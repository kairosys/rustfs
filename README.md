# rustfs

Kubernetes deployment module for **RustFS** (S3-compatible object storage) in the `furseal` namespace of a local dev cluster. This repository ships runtime data and Kubernetes manifests only — there are no Rust/Go sources, build scripts, or CI config here; it consumes the upstream container image rather than compiling from source within this repo.

## Layout

- `k8s/rustfs-deployment.yaml` — Service (`ClusterIP`, port 9000→targetPort 9000) + Deployment (replicas: 1, command `["rustfs", "/data"]`) + nginx Ingress in namespace `furseal`. Image is `rustfs/rustfs:latest`; change the image/tag when you need a different version.
- `k8s/rustfs-secret.yaml` — Secret consumed via bulk env injection; keys referenced are `RUSTFS_ACCESS_KEY`/`RUSTFS_SECRET_KEY`. No ConfigMaps or env files exist here.
- `data/` — generated at runtime (RustFS metadata under `.rustfs.sys/`, plus Langfuse event buckets). Consumed in place by the container through a hostPath mount; **do not edit while RustFS is running**. Back up from `<host>/mnt/workspaces/rustfs/data`, not this directory.

## Storage

Storage is a hostPath volume declared with `type: DirectoryOrCreate` (from the manifest's `volumes[].hostPath`):

- Host path : `/mnt/workspaces/rustfs/data` — source of truth on the node
- Mount      : `/data` inside the container (the argument passed to `rustfs`)

Notes:

- No PersistentVolumeClaim or StorageClass object exists in this repo. The host directory is auto-created empty by Kubernetes (`DirectoryOrCreate`) if absent, so a new cluster node will have no prior data until seeded.
- Runtime leftovers appear under `data/.rustfs.sys/` after the workload stops (config pool / IAM format JSON written there). These are not config files to edit while RustFS is running.

## Configuration & secrets

Environment injected by this repo (values verbatim from the manifests):

| Env key (exact)        | Value as defined                                              | Binding location                              |
|------------------------|--------------------------------------------------------------|-----------------------------------------------|
| `RUSTFS_ADDRESS`       | `0.0.0.0:9000`                                                | Inline under `Containers[].env[]` in the Deployment |
| `RUSTFS_SERVER_DOMAINS`| `s3.localhost,rustfs,rustfs.furseal.svc.cluster.local`        | Same; keep synced with Ingress `rules.host`    |

`RUSTFS_SECRET_KEY` lives in that same object. Both keys are injected at once via bulk env injection referencing one Secret named exactly, not as individual per-key references. No credential value is reproduced in this README to avoid committing secrets; prefer a sealed Secret (e.g., kubeseal) before production since the repository's secret file stores plaintext stringData and must be rotated off that form.

## Networking & ports

| Layer                                | Port(s)                       | Purpose                          | Source-defined routing                                  | Notes |
|--------------------------------------|------------------------------|----------------------------------|---------------------------------------------------------|-------|
| Container (`spec.containers[0].ports`) | `containerPort: 9000`, name `s3`        | S3 API presented to clients      | DNS-resolvable within the cluster                         | Single containerized workload in this Deployment; container arg is bucket root `/data`. No readiness or liveness probes are defined here. |
| Service (`kind: Service`, `ClusterIP`) | port `9000`, targetPort `9000` (name `s3`), selector app=`rustfs`                                          | Cluster-internal endpoint for the Deployment | DNS hostname `<svc>.<namespace>.cluster.local`; also one of the servers listed | Selector is label based (`app=rustfs`); no NodePort, LoadBalancer IP, console web port, or TLS secret are declared by any manifest here. The Ingress only routes to this Service **port** (`9000`); nothing additional is exported for client traffic. |

## Kubernetes architecture

One file holds three objects: a Service, the Deployment (one replica), and an nginx Ingress in namespace `furseal`. Ingress details from the source:

- annotation: `nginx.ingress.kubernetes.io/proxy-body-size: "500m"`
- `ingressClassName: "nginx"`
- host rules: `s3.localhost` and `*.s3.localhost`, each routing `/` to Service `rustfs` port `9000`
- the backend references that single Service by name

The Deployment also sets the two env keys above inline; no ConfigMap is consumed here. This is not sharded — one replica and one hostPath mount, so the node hosting the workload is authoritative for both data and config at runtime.

## Deployment guide (sequential)

No `Namespace` object ships in these manifests, and no StorageClass/PVC needs creating (hostPath + DirectoryOrCreate handles it). Apply into an existing cluster namespace:

```bash
# 0. prerequisites — Namespace must pre-exist on your cluster; host path will be auto-created if absent
kubectl create namespace furseal              # idempotent: ignore the message if it already exists
mkdir -p /mnt/workspaces/rustfs/data          # only needed beforehand when you must load data first

# 1. The referenced Secret must exist before the Deployment/SVC reference it, to avoid webhook "referent not found":
kubectl -n furseal apply -f k8s/rustfs-secret.yaml      # defines name = rustfs-secret (this repo's secret object)

# 2. Service + Deployment + Ingress with env injected from this repository only:
kubectl -n furseal apply -f k8s/rustfs-deployment.yaml   # Service port 9000->targetPort 9000, selector app=rustfs; Ingress routes / to s3.localhost and *.s3.localhost
```

## Verification & diagnostics (not sourced from these manifests)

These YAML objects declare no readiness or liveness probe. The commands below verify the workload as-defined and confirm live env matches the inline keys/values specified under `k8s/rustfs-deployment.yaml`; they may be omitted if deploying strictly by manifest:

```bash
kubectl -n furseal get pods,svc,ingress   # Pods become Running; svc keeps 1 endpoint per pod IP matching selector app=rustfs; Ingress address includes s3.localhost / *.s3.localhost (from the rules defined above)

# Optional — RustFS exposes a health path in its HTTP layer and is not present as an object in this repo:
kubectl -n furseal port-forward svc/rustfs 9000 && curl http://localhost:9000/minio/health/live   # expect HTTP 200, but it is NOT declared here

# Confirm the runtime env matches only what's declared by inline keys (no file edits required):
kubectl -n furseal exec deploy/rustfs -- env | grep -E '^(RUSTFS_ADDRESS|RUSTFS_SERVER_DOMAINS)=...'   # should print: RUSTFS_ADDRESS=0.0.0.0:9000  and  RUSTFS_SERVER_DOMAINS=s3.localhost,rustfs,rustfs.furseal.svc.cluster.local
```

## Restore checklist (what this repo tracks vs what you bring)

- namespace `furseal` must already exist on the cluster before apply;
- host path `/mnt/workspaces/rustfs/data` is mounted into the live container at as its bucket root (`/data`);
- runtime state relevant during restore lives under `.rustfs.sys/` within that data directory and is managed by RustFS itself — do not hand-edit it while the service is stopped or running without a sealed-secret-backed key first;
- there are no ConfigMaps consumed here.