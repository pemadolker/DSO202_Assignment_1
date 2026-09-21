# DSO202 Assignment 1: Three-Tier Task Tracker on Kubernetes

For this assignment I deployed the provided Task Tracker (an nginx frontend, a Node.js backend and a PostgreSQL database) to a local kind cluster. Everything lives in the namespace `dso202-assignment-01`. No application code was written, so most of the work here is the Kubernetes configuration and showing that it works.

## 1. Repository layout and how to deploy

```text
assignment-1/
├── kind-config.yaml        # cluster config: maps host port 30080 to the node
├── namespace.yaml
├── configmap.yaml
├── secret.yaml
├── quota.yaml              # ResourceQuota and LimitRange
├── rbac.yaml               # optional bonus (Task 8)
├── database/  (pvc.yaml, deployment.yaml, service.yaml)
├── backend/   (deployment.yaml, service.yaml)
├── frontend/  (deployment.yaml, service.yaml)
├── evidence/               # screenshots used in this README
└── README.md
```

```bash
kind create cluster --name dso202-assignment --config kind-config.yaml
kubectl apply -f namespace.yaml -f configmap.yaml -f secret.yaml -f quota.yaml
kubectl apply -f database/
kubectl apply -f backend/
kubectl apply -f frontend/
```

The app is then available at `http://localhost:30080`.

The cluster from Practical 1 was already running. Before starting I checked that the node was `Ready`, that kind's default StorageClass (`standard`) existed for the database volume, and that host port 30080 was mapped into the node. The port mapping has to be set when a kind cluster is created, because the "node" is really a Docker container, so I added it in `kind-config.yaml`.

![Cluster ready](<evidence/image.png>)

## 2. The images I used (and why)

The brief says to use `sarojsanyasi/dso202-{frontend,backend,db}:1.0`. When I inspected them with `docker buildx imagetools inspect`, all three were published for `linux/arm64` only. My machine is `x86_64` and the kind node reports `amd64`, so Kubernetes could not run them here.

To carry on, I built the three images myself from the source code that came with the assignment (`Assignment-1-app-code/`), using the provided Dockerfiles without changing them, and pushed them to my own Docker Hub as `pdolker/dso202-{frontend,backend,db}:1.0`. The only difference in my manifests compared with using the official images is the `image:` line in each Deployment, and I marked those lines with `# TEMP`. If the official images are republished for amd64, switching back means changing those three lines and running `kubectl apply` again.

<!-- Optional, only if it is true: add a sentence such as "I told the module tutor about this by email on <date>." -->

## 3. Task 1: Architecture note

When I run `kubectl apply`, the request goes to the kube-apiserver, which validates it and saves the desired state in etcd. The Deployment controller (part of the kube-controller-manager) sees that a Deployment wants one Pod and asks for it through a ReplicaSet. The kube-scheduler chooses a node for the new Pod, which here is the single kind node. On that node the kubelet takes over: it tells containerd to pull the image and start the container, mounts the volume, and passes in the ConfigMap and Secret values as environment variables. kube-proxy sets up the rules that send Service traffic to the right Pods, and CoreDNS lets Pods use Service names like `db-svc` instead of IP addresses.

I used one Deployment and one Service per tier, plus shared configuration objects:

| Tier | Objects | Why |
|---|---|---|
| Database | Deployment (1 replica), PVC, headless Service `db-svc` | The Deployment recreates the Pod if it dies. The PVC keeps the data outside the Pod. A headless Service gives the database a DNS name inside the namespace without a virtual IP, and nothing outside can reach it. |
| Backend | Deployment, ClusterIP Service `backend-svc` | The backend keeps no data, so it can be replaced any time. ClusterIP means it is only reachable from inside the cluster. |
| Frontend | Deployment, NodePort Service `frontend-svc` | It is the only thing that needs to be reached from outside, so it gets the only NodePort (30080). |
| Shared | Namespace, ConfigMap `app-config`, Secret `db-credentials`, ResourceQuota, LimitRange | Separation from other workloads, configuration kept out of the images, and limits on resource use. |

## 4. Task 2: ConfigMap and Secret

The non-sensitive values are in the ConfigMap `app-config`: `DB_HOST`, `DB_PORT`, `DB_NAME`, `APP_PORT`, `CORS_ORIGIN`, `POSTGRES_DB` and `BACKEND_URL`. The credentials are in the Secret `db-credentials`: `DB_USER`, `DB_PASSWORD`, `POSTGRES_USER` and `POSTGRES_PASSWORD`.

The backend reads `DB_*` and the official PostgreSQL image reads `POSTGRES_*`. They are different names for the same information, so I supply both sets with identical values (`DB_NAME` and `POSTGRES_DB` both hold the database name, and so on). If only one set existed, one of the two tiers would not be configured. `DB_HOST` is set to `db-svc`, the name of the database Service.

In the Deployments I list each variable one by one (`configMapKeyRef` / `secretKeyRef`) instead of loading a whole ConfigMap with `envFrom`. That way each tier only receives the keys it needs, and it is easy to see in the YAML that the database uses `POSTGRES_*` and the backend uses `DB_*`.

**Secret caveat.** Kubernetes Secrets are only base64-encoded, and they are not encrypted at rest by default. Anyone who can read the Secret can decode it in one command, which the screenshot below shows for the username. I stored the values in `secret.yaml` as base64 (`data:`) so that no password appears as plain text in the file I commit, but that is encoding, not protection. Making Secrets properly secure is outside this assignment, so I have only documented it.

![ConfigMap, Secret and quota created, Secret decoded](<evidence/image copy.png>)

## 5. Tasks 3 to 5: The three tiers

| Resource | Name | Notes | `tier` label |
|---|---|---|---|
| PVC | `db-pvc` | 1Gi, ReadWriteOnce, default StorageClass `standard` | database |
| Deployment | `db` | 1 replica, mounts the PVC at `/var/lib/postgresql/data`, uses `POSTGRES_*` | database |
| Service | `db-svc` | headless (`clusterIP: None`), port 5432 | database |
| Deployment | `backend` | 1 replica, uses `DB_*` | backend |
| Service | `backend-svc` | ClusterIP, port 8080 | backend |
| Deployment | `frontend` | 1 replica, uses `BACKEND_URL` | frontend |
| Service | `frontend-svc` | NodePort 30080 to port 8080 | frontend |

Every Pod, Deployment and Service has a `tier` label (and `app: task-tracker`). The Deployment and Service selectors match on both labels.

**Database.** The PVC is mounted at PostgreSQL's data directory, so the data is stored on the volume and not in the container's own filesystem. I set the Deployment's strategy to `Recreate`. With the default rolling update Kubernetes would start a new database Pod before stopping the old one, and two PostgreSQL servers cannot share one data directory. `Recreate` stops the old Pod first. kind's default StorageClass only creates the volume once a Pod that uses the claim is scheduled, so the PVC can show `Pending` for a few seconds at first, which is normal.

![Database tier](<evidence/image copy 2.png>)

**Backend.** `DB_HOST` points at `db-svc`. When the backend starts before the database is ready it logs that it is retrying and keeps trying, which is what the start-up log shows, so I did not need any probes or ordering tricks. The Service is ClusterIP only.

![Backend tier](<evidence/image copy 3.png>)

**Frontend.** The image writes `BACKEND_URL` into a file called `config.js` when the container starts, using the value from the ConfigMap. The screenshot shows the substituted value, so changing the URL only needs a Pod restart, not an image rebuild.

![Frontend tier](<evidence/image copy 4.png>)

All the resources together, with the labels:

![All resources](<evidence/image copy 5.png>)

**Something I found about the browser and the backend.** The frontend is a static page, so its JavaScript runs in the user's browser and calls `BACKEND_URL` from there. Since `backend-svc` is a ClusterIP name that only exists inside the cluster, a browser on my machine cannot resolve it, so the page loads but cannot reach the backend. The brief accepts `curl` through a port-forwarded backend for the CRUD evidence, so my main evidence uses that. For the browser screenshots in 7a I made it work temporarily by adding `127.0.0.1 backend-svc` to `/etc/hosts` and running `kubectl port-forward svc/backend-svc 8080:8080`. That changed nothing in the manifests, and I removed the hosts entry afterwards. `BACKEND_URL` in the ConfigMap stays as the cluster-internal address the brief asks for.

## 6. Task 6: ResourceQuota and LimitRange

I set requests and limits on every container. A namespace with a CPU/memory quota needs each Pod to declare them (or get them from a LimitRange), so it is better to set them explicitly. The sizes are small because the app is small:

| | Requests (CPU / memory) | Limits (CPU / memory) |
|---|---|---|
| db | 100m / 128Mi | 500m / 256Mi |
| backend | 50m / 64Mi | 250m / 256Mi |
| frontend | 25m / 32Mi | 100m / 64Mi |
| **Total** | **175m / 224Mi** | **850m / 576Mi** |
| With one extra backend Pod during a rolling update | 225m / 288Mi | 1100m / 832Mi |

I sized the quota from those totals and did not pick round numbers at random:

- **requests 500m CPU / 512Mi:** covers the rolling-update peak (225m / 288Mi) with room for a temporary debug Pod.
- **limits 2 CPU / 1Gi:** covers the peak of 1100m / 832Mi with some spare.
- **pods 8:** three running, one extra during a rolling update, one for testing, plus a couple spare.
- **persistentvolumeclaims 2:** I need one, and one is spare.
- **services.nodeports 1 and services.loadbalancers 0:** only the frontend is allowed to be exposed, so the backend or database cannot accidentally be given a NodePort or LoadBalancer. This backs up the brief's rule.

The LimitRange gives any container that declares nothing a default of 50m / 64Mi requested and 250m / 256Mi limit. The minimum is 10m / 16Mi and the maximum is 500m / 512Mi. The maximum matches the biggest container, the database, with some headroom, so no single container can take a big share of the quota. All three Deployments fit inside these bounds, and the "Used" column below matches my totals.

![Requirements check and quota usage](<evidence/image copy 6.png>)

The same screenshot shows that the image tags are `1.0` and not `latest`, that only the frontend is a NodePort, and that the search for a plaintext password in the manifests finds nothing.

## 7. Task 7: Verification

### 7a. Full CRUD cycle

I used `curl` through a port-forwarded backend: list the tasks, create one (POST), read it, update its status to `in_progress` (PUT), delete it (`HTTP 204`) and then read it again to get a `404`.

![CRUD cycle](<evidence/image copy 7.png>)

I also did the same things in the web page using the temporary hosts entry explained in section 5:

![Frontend connected to the backend](<evidence/image copy 15.png>)

Playing around Frontend - adding, updating and deleting tasks 

![Frontend after adding, updating and deleting tasks](<evidence/image copy 16.png>)

![alt text](<evidence/image copy 17.png>)


### 7b. Service DNS from inside the frontend Pod

From inside the frontend Pod I ran `curl http://backend-svc:8080/api/status` and got `{"status":"ok","db":"connected"}`. There was no port-forward involved, so the Pod is resolving the Service name through CoreDNS. I also ran `nslookup`: `backend-svc` resolves to its cluster IP, while the headless `db-svc` resolves straight to the database Pod's own IP, which is the difference a headless Service makes.

![DNS](<evidence/image copy 8.png>)

### 7c. Self-healing and data persistence

First I created a task and listed the Deployment, ReplicaSet and Pod for the backend:

![Before deleting the Pod](<evidence/image copy 9.png>)

Then I deleted the backend Pod with `kubectl delete pod -l tier=backend` and watched with `kubectl get pods --watch`. The old Pod goes to `Terminating` and a new Pod with a different name is created and reaches `Running` without me doing anything:

![Backend Pod deleted and recreated](<evidence/image copy 10.png>)

The task I created before the deletion is still there:

![Task still retrievable](<evidence/image copy 11.png>)

The ReplicaSet is what recreated the Pod: it is told to keep one backend Pod running, sees zero, and starts another. The task survived because it is stored by PostgreSQL on the PersistentVolume, not inside the backend Pod, so the lifecycle of the Pod and the lifecycle of the data are separate.

As an extra test I deleted the database Pod as well. The PVC stayed `Bound`, a new database Pod started using the same volume, and the same task was still returned:

![Database Pod deleted](<evidence/image copy 12.png>)

### 7d. Declarative vs imperative

I created the ConfigMap declaratively with `kubectl apply -f configmap.yaml`, and then created a similar one imperatively with `kubectl create configmap app-config-imp --from-literal=...`. Then I ran the imperative command a second time:

![Declarative vs imperative](<evidence/image copy 13.png>)

What I saw: applying the unchanged file again says `unchanged`, while repeating the `create` command fails with `AlreadyExists`. The declarative object has the label `app=task-tracker`, but the imperative one has no labels, because I only gave it the three keys as flags.

| | Declarative (`apply -f`) | Imperative (`create ...`) |
|---|---|---|
| Running it again | Safe, says `unchanged` or `configured` | Fails with `AlreadyExists` |
| Record of what I wanted | A YAML file I can keep in git and review | Only in the cluster and my shell history |
| Completeness | Labels, namespace and all keys in one place | Only what I remembered to type as flags |
| Speed | Needs a file first | Quickest for a quick test |

For this assignment, declarative is clearly better because everything has to be reproducible and submitted as files. Imperative commands are handy for quick experiments and debugging.

## 8. Task 8 (bonus): Namespace RBAC

`rbac.yaml` creates a ServiceAccount `readonly-sa`, a Role `namespace-reader` and a RoleBinding. The Role allows get/list/watch on Pods, Pod logs, Services, ConfigMaps, PVCs, Deployments and ReplicaSets, and I left Secrets out on purpose. Because it is a Role and not a ClusterRole, it only applies inside `dso202-assignment-01`. I checked it with `kubectl auth can-i --as=system:serviceaccount:dso202-assignment-01:readonly-sa`: reading is allowed in the namespace, while deleting Pods, reading Secrets and reading Pods in `kube-system` are all refused.

![RBAC checks](<evidence/image copy 14.png>)

## 9. Problems I ran into and what I changed

- **The published images did not run on my machine.** Explained in section 2. I rebuilt them from the provided source and changed only the `image:` lines.
- **`CreateContainerConfigError` on the database Pod.** I had deployed the database before creating `app-config` and `db-credentials`, so the Pod could not find the values it was told to read. Creating them and restarting the Deployment fixed it, and now I apply the ConfigMap and Secret first.
- **`kubectl port-forward` failed with "address already in use".** An old port-forward was still holding port 8080. I found it with `ss -ltnp` and stopped it, and I now close each tunnel straight after using it.
- **The backend logged `EAI_AGAIN` and "retrying" when it started.** The database name could not be resolved yet because the database Pod was still starting. It is the retry behaviour the brief asks for: the backend kept trying and connected on its own.
- **The web page could not reach the backend.** Explained in section 5.
- **One backend Pod restarted once, a few seconds after starting.** The container exited with code 133 (SIGTRAP), and the log from the previous container showed an internal error from Node's V8 engine. I checked that it was not a memory limit (that would show `OOMKilled`, exit code 137) and not a configuration problem. Kubernetes restarted the container by itself and the data was not affected. I did not find the root cause, and I mention it because it is another example of Kubernetes restarting a failed container.

## 10. Limitations

- Each tier has one replica, so there is no high availability. A Pod is recreated when it dies, but there is a short gap.
- Secrets are only base64-encoded (section 4), and `CORS_ORIGIN` is `*` for classroom simplicity; a real deployment would restrict it to known origins.
- I did not add readiness probes, since the brief keeps this assignment to Unit I. The backend's own retry loop covers the start-up order between backend and database.
- My images are amd64-only (section 2).

