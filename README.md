# MongoDB + Mongo Express on Kubernetes

A small Kubernetes project that runs a **MongoDB** database and **Mongo Express** (a web-based admin UI for MongoDB) in a cluster. It is a good demo of the core Kubernetes building blocks: Deployments, Services, ConfigMaps, and Secrets working together.

---

## 1. What this project does

| Component | What it is | Who can reach it |
|---|---|---|
| **MongoDB** | The database | Only other pods inside the cluster |
| **Mongo Express** | A web UI to browse and manage the database | You, from your browser (via NodePort) |

You open Mongo Express in the browser, and it talks to MongoDB inside the cluster on your behalf.

---

## 2. Architecture

```
                         Kubernetes cluster
 ┌────────────────────────────────────────────────────────────────┐
 │                                                                │
 │   ┌──────────────┐        ┌───────────────────┐                │
 │   │  Secret      │        │  ConfigMap        │                │
 │   │ mongo-secret │        │  mongo-config     │                │
 │   │ user + pass  │        │ mongo-url =       │                │
 │   └──────┬───────┘        │  "mongo-service"  │                │
 │          │                └─────────┬─────────┘                │
 │          │ (used by both)           │                          │
 │          ▼                          ▼                          │
 │   ┌──────────────┐          ┌───────────────────┐              │
 │   │ mongo pod    │◄─────────│ mongo-express pod │              │
 │   │ port 27017   │ internal │ port 8081         │              │
 │   └──────▲───────┘ traffic  └─────────▲─────────┘              │
 │          │                            │                        │
 │   ┌──────┴────────┐          ┌────────┴──────────┐             │
 │   │ mongo-service │          │ webapp-service    │             │
 │   │ ClusterIP     │          │ NodePort 30107    │             │
 │   │ (internal)    │          │ (external)        │             │
 │   └───────────────┘          └────────▲──────────┘             │
 └───────────────────────────────────────┼────────────────────────┘
                                         │
                                  Your browser
                              http://<node-ip>:30107
```

---

## 3. Files and what each one does

### `secret.yaml` – credentials
Creates a Kubernetes **Secret** named `mongo-secret` holding the MongoDB root username and password.

- Values under `data:` must be **base64-encoded**. Base64 is *encoding, not encryption*.
- Used by **both** deployments:
  - MongoDB uses it to create the root user on first start.
  - Mongo Express uses it to log in to MongoDB.

> **Do not commit real credentials to Git.** Keep `secret.yaml` in `.gitignore`, or create the secret with the command in the setup section below.

To encode your own values:
```bash
echo -n 'myuser' | base64
echo -n 'mypassword' | base64
```
Always use `-n`, otherwise a newline gets encoded into the value and authentication fails.

### `mongo-config.yaml` – non-secret configuration
Creates a **ConfigMap** named `mongo-config` with one key:

```
mongo-url: mongo-service
```

This is the hostname Mongo Express should connect to. It is **not sensitive**, so it goes in a ConfigMap rather than a Secret. Keeping it here means that if the DB service is renamed, you change one place instead of editing the web app deployment.

### `mongo-app.yaml` – the database (two objects in one file)

**1. Deployment `mongo-deployment`**
- Runs 1 replica of the `mongo:8.0` image.
- Exposes container port `27017` (MongoDB's default).
- Sets `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD` from `mongo-secret`. The official Mongo image reads these on first startup and creates the admin user.

**2. Service `mongo-service`** (default type: ClusterIP)
- Gives the MongoDB pod a **stable internal DNS name**: `mongo-service`.
- Pods get random IPs that change when they restart; the Service name does not.
- It selects pods with label `app: mongo` and forwards port `27017` to the container's `27017`.
- Being ClusterIP, it is **only reachable inside the cluster**. The database is not exposed to the internet.

### `web-app.yaml` – the web UI (two objects in one file)

**1. Deployment `webapp-deployment`**
- Runs 1 replica of `mongo-express`.
- Listens on container port `8081`.
- Environment variables:
  - `ME_CONFIG_MONGODB_ADMINUSERNAME` – from `mongo-secret` (`mongo-user`)
  - `ME_CONFIG_MONGODB_ADMINPASSWORD` – from `mongo-secret` (`mongo-password`)
  - `ME_CONFIG_MONGODB_SERVER` – from ConfigMap `mongo-config` (`mongo-url`), which resolves to `mongo-service`

**2. Service `webapp-service`** (type: NodePort)
- Opens port `30107` on every cluster node and forwards it to port `8081` on the Mongo Express pod.
- This is how you reach the UI from outside the cluster.
- NodePort values must be in the range **30000–32767**.

---

## 4. How the pieces connect (step-by-step workflow)

1. **Secret and ConfigMap are created first.** They are just stored data in the cluster; nothing runs yet.
2. **MongoDB deployment starts.** Kubernetes pulls `mongo:8.0`, starts a pod, and injects the username and password from the Secret as environment variables. MongoDB creates its root user.
3. **`mongo-service` is created.** It finds the MongoDB pod using the label `app: mongo` and creates the DNS name `mongo-service` that points to it.
4. **Mongo Express deployment starts.** It receives the same credentials from the Secret, plus the server hostname (`mongo-service`) from the ConfigMap.
5. **Mongo Express connects** to `mongo-service:27017` using cluster DNS. Kubernetes routes that to the MongoDB pod.
6. **`webapp-service` exposes the UI.** Traffic arriving on `<node-ip>:30107` is forwarded to the Mongo Express pod on port `8081`.
7. **You open the browser,** use the UI, and every action you take is sent by Mongo Express to MongoDB.

### How the label matching works
Services and Deployments are linked by **labels**, not by name:

```
Deployment pod template label:   app: mongo
Service selector:                app: mongo     ← matches, so traffic is routed
```

If the labels do not match, the Service has no endpoints and connections fail.

---

## 5. Prerequisites

- A running Kubernetes cluster (Minikube, kind, Docker Desktop, or a cloud cluster)
- `kubectl` configured to talk to it

---

## 6. Setup and deployment

### Step 1 – Create the secret
Either apply the file:
```bash
kubectl apply -f secret.yaml
```
or create it directly without any file (recommended for real use):
```bash
kubectl create secret generic mongo-secret \
  --from-literal=mongo-user=YOUR_USER \
  --from-literal=mongo-password=YOUR_PASSWORD
```

### Step 2 – Create the ConfigMap
```bash
kubectl apply -f mongo-config.yaml
```

### Step 3 – Deploy MongoDB
```bash
kubectl apply -f mongo-app.yaml
```

### Step 4 – Deploy Mongo Express
```bash
kubectl apply -f web-app.yaml
```

**Order matters:** the Secret and ConfigMap must exist before the Deployments that reference them. If they don't, pods get stuck in `CreateContainerConfigError`.

### Step 5 – Verify everything is running
```bash
kubectl get all
kubectl get configmap
kubectl get secret
```
You should see both pods in `Running` state and both services listed.

---

## 7. Accessing the web UI

**Minikube:**
```bash
minikube service webapp-service
```
or manually:
```bash
minikube ip          # then open http://<that-ip>:30107
```

**Docker Desktop / kind / other clusters:** open `http://localhost:30107` or `http://<node-ip>:30107`. On kind you may need extra port mapping, or use port-forward:
```bash
kubectl port-forward service/webapp-service 8081:8081
# then open http://localhost:8081
```

Some Mongo Express versions also show a basic-auth login prompt (the default is often `admin` / `pass`). Check the image documentation for your version.

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Pod status `CreateContainerConfigError` | Secret or ConfigMap missing | Apply `secret.yaml` and `mongo-config.yaml` first |
| Mongo Express pod crash-looping | Cannot connect to MongoDB or wrong credentials | `kubectl logs <webapp-pod>`; check the Secret values and that the mongo pod is `Running` |
| Authentication failed | Secret was base64-encoded with a trailing newline | Re-encode using `echo -n` |
| Can't change credentials after first run | Mongo only creates the root user on first initialization | Delete the pod's data (see persistence note) and redeploy |
| UI not reachable on 30107 | Wrong node IP, or cluster does not expose NodePorts to localhost | Use `minikube service webapp-service` or `kubectl port-forward` |
| Service has no endpoints | Label mismatch | `kubectl get endpoints mongo-service`; compare labels and selectors |

Useful commands:
```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get endpoints
kubectl exec -it <mongo-pod> -- mongosh -u <user> -p <password>
```

---

## 9. Important limitations (know these)

- **No persistent storage.** The MongoDB deployment has no PersistentVolume. **If the MongoDB pod restarts or is deleted, all data is lost.** For anything beyond a demo, add a PersistentVolumeClaim and mount it at `/data/db`.
- **Single replica, no replication.** There is no high availability. A production setup would use a StatefulSet or a replica set.
- **`mongo-express:latest` is unpinned.** A future image update can change behavior (newer versions changed some environment variable names). Pin a specific version tag.
- **Base64 is not encryption.** Treat `secret.yaml` as sensitive. Use Sealed Secrets, SOPS, or External Secrets Operator for real deployments.
- **No resource limits or health probes.** Add `resources` and `livenessProbe` / `readinessProbe` for production use.
- **Mongo Express should not be publicly exposed.** It is an admin tool with full database access. Restrict it with network policies, authentication, or use port-forward only.

---

## 10. Cleanup

```bash
kubectl delete -f web-app.yaml
kubectl delete -f mongo-app.yaml
kubectl delete -f mongo-config.yaml
kubectl delete -f secret.yaml
```

---

## 11. Key concepts used (quick reference)

| Concept | Purpose in this project |
|---|---|
| **Deployment** | Keeps the desired number of pods running and restarts them if they fail |
| **Pod** | The running container(s); here, one MongoDB and one Mongo Express |
| **Service (ClusterIP)** | Stable internal name and IP for MongoDB |
| **Service (NodePort)** | Exposes Mongo Express outside the cluster |
| **ConfigMap** | Stores non-sensitive config (the DB hostname) |
| **Secret** | Stores sensitive config (DB username and password) |
| **Labels and selectors** | How Services find the right pods |
| **Cluster DNS** | Lets pods reach each other by service name (`mongo-service`) |
