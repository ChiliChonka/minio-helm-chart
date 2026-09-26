# MinIO Umbrella Chart – Standalone on NFS

## Produktion (TASK-355)

- **Release / Namespace:** `minio` / `minio-server`
- **Produktionswerte:** ausschließlich `helm-values/minio-helm-chart/minio-values.yaml` im Repo
  `helm-values` (die `values.yaml` hier sind die Chart-Defaults).
- **Rollout:** aus einem sauberen, committeten Stand beider Repos:

  ```bash
  python3 ../helm-values/scripts/compare-release.py minio minio-server minio \
    -f ../helm-values/minio-helm-chart/minio-values.yaml     # erst vergleichen
  helm upgrade minio minio -n minio-server -f ../helm-values/minio-helm-chart/minio-values.yaml
  ```

- **Tags:** `prod/r<Revision>` markiert den Commit, aus dem Helm-Revision `<Revision>` ausgerollt
  wurde (die Tag-Nachricht nennt den `helm-values`-Commit). **Kein `chart-*`-Tag für Rollouts** —
  der löst `helm-release.yaml` aus und veröffentlicht das Chart nach GHCR.

| Stand | Chart | Nachweis |
|---|---|---|
| **Produktion** | siehe `helm list -n minio-server` und den jüngsten Tag `prod/r*` | `compare-release.py` danach: keine Abweichungen |

## Prerequisites

- Kubernetes cluster with Gateway API (HTTPRoute, e.g. Envoy Gateway)
- NFS share on your NAS (NFSv4)
- TLS terminated at the Gateway (managed separately)
- Namespace created

## Quick Start

### 1. Create namespace + credentials

```bash
kubectl create namespace minio

kubectl create secret generic minio-root-credentials \
  --from-literal=rootUser=<admin> \
  --from-literal=rootPassword=$(openssl rand -base64 32) \
  -n minio
```

### 2. Prepare NFS export on the NAS

Make sure the following is set up on your NAS:

- Export path exists (e.g. `/minio`)
- NFSv4 enabled
- Permissions set:

```bash
# On the NAS
chown -R 1000:1000 /minio
chmod -R 775 /minio
```

### 3. Adjust values.yaml

At a minimum, adjust these values in `values.yaml`:

```yaml
nfs:
  server: "192.168.1.100"     # IP of your NAS
  path: "/minio"      # NFS export path
```

As well as `httpRoute` (gateway, hostnames) and `MINIO_BROWSER_REDIRECT_URL` in the `minio:` block.

### 4. Pull dependencies + install

```bash
cd minio
helm dependency update
helm install minio . -n minio -f values.yaml
```

### 5. Upgrade

```bash
helm upgrade minio . -n minio -f values.yaml
```

## Architecture

```text
┌─────────────────────────────────────────────────────┐
│  Umbrella Chart                                     │
│                                                     │
│  ┌──────────────┐  ┌──────────────────────────────┐ │
│  │ nfs-pv.yaml  │  │MinIO Subchart (charts.min.io)| │
│  │ nfs-pvc.yaml │  │                              │ │
│  │              │  │  - Standalone Deployment     │ │
│  │  NFS ──────────►│  - S3 API Service            │ │
│  │  PV/PVC      │  │  - Console Service           │ │
│  │ httproute    │  │                              │ │
│  └──────────────┘  └──────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
                         │
           ┌─────────────┼─────────────┐
           ▼                           ▼
   api.minio.example.com    console.minio.example.com
     (S3 compatible)             (Web UI)
```

## Known Issues

- **MinIO Chart v5.4.0 Bug:** There is a known bug where `replicas` is
  hardcoded to 16 in standalone mode (GitHub Issue #21480). If this occurs,
  either downgrade to v5.3.0 or manually scale the StatefulSet to 1 replica.
- **NFS + MinIO:** Not officially recommended, but works reliably in
  standalone mode, especially with NFSv4.

## Uninstall

```bash
helm uninstall minio -n minio
# PV has Retain policy, delete manually if desired:
kubectl delete pv <release-name>-minio-nfs-pv
```
