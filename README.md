# zwavejs2mqtt-kustomize

Kubernetes manifests for deploying [zwave-js-ui](https://github.com/zwave-js/zwave-js-ui) (formerly zwavejs2mqtt) using [Kustomize](https://kustomize.io/), with first-class support for [FluxCD](https://fluxcd.io/) GitOps workflows.

---

## Table of Contents

- [Overview](#overview)
- [What Problem Does This Solve?](#what-problem-does-this-solve)
- [Integration with Home Assistant](#integration-with-home-assistant)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Configuration](#configuration)
- [Installation — FluxCD (GitOps)](#installation--fluxcd-gitops)
- [Installation — kubectl (direct)](#installation--kubectl-direct)
- [Upgrading](#upgrading)
- [Maintenance](#maintenance)
- [Troubleshooting](#troubleshooting)

---

## Overview

This repository provides production-ready Kubernetes manifests to run **zwave-js-ui** on your cluster. It is built around Kustomize so it works out-of-the-box with both plain `kubectl` and FluxCD.

Key features:

- **Single-replica StatefulSet** — keeps Z-Wave state consistent across restarts.
- **Z-Wave USB device passthrough** — the container runs in privileged mode so it can access `/dev/tty*` USB serial devices on the host.
- **ClusterIP Service** — exposes the HTTP UI (port `8091`) and WebSocket (port `3000`) internally.
- **Automatic image updates via [Renovate](https://docs.renovatebot.com/)** — the `renovate.json` configuration keeps the container image up-to-date and the repository's CI pipeline automatically creates a matching Git tag for each new image version.
- **FluxCD-ready** — the `flux/` directory contains ready-to-use `GitRepository` and `Kustomization` examples.

---

## What Problem Does This Solve?

Running zwave-js-ui on bare metal or in a VM is straightforward. Running it on Kubernetes requires careful handling of:

- **USB device access** — the Z-Wave controller (e.g. Aeotec Z-Stick, Zooz USB) must be accessible inside the container.
- **Persistent configuration** — zwave-js-ui stores its configuration and Z-Wave mesh data on disk; this must survive pod restarts.
- **Secrets management** — the session secret should not be hard-coded.

This repository solves all three problems and provides a GitOps-friendly structure.

---

## Integration with Home Assistant

zwave-js-ui acts as the bridge between your Z-Wave USB controller and Home Assistant:

```
Z-Wave USB Controller
        │
        ▼
  zwave-js-ui (this project)
        │  WebSocket (port 3000)
        ▼
  Home Assistant (Z-Wave JS integration)
```

### Home Assistant Configuration

1. In Home Assistant, go to **Settings → Devices & Services → Add Integration**.
2. Search for and add the **Z-Wave JS** integration.
3. When prompted for the WebSocket URL, enter:
   ```
   ws://<zwave-js-ui-cluster-ip>:3000
   ```
   Replace `<zwave-js-ui-cluster-ip>` with the ClusterIP of the `zwave-js-ui` Service, or if Home Assistant also runs in the same cluster, you can use the DNS name:
   ```
   ws://zwave-js-ui.zwave.svc.cluster.local:3000
   ```
4. Complete the integration wizard.

> **Note:** Home Assistant must be able to reach port `3000` on the zwave-js-ui Service. If Home Assistant is outside the cluster, you may need to expose that port via a `NodePort` or `LoadBalancer` service, or a reverse proxy.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Kubernetes cluster (1.25+) | Any distribution (k3s, k8s, microk8s, etc.) |
| A Z-Wave USB controller attached to a cluster node | e.g. Aeotec Z-Stick Gen5, Zooz ZST10 |
| `kubectl` configured for your cluster | `kubectl version` should show both client and server |
| Kustomize 5.x | Bundled with `kubectl` ≥ 1.21 or install separately |
| FluxCD (optional) | Only required for the GitOps installation method |
| Node storage on the host | Default path: `/srv/container-data/zwavejs2mqtt` |

> **USB device note:** The manifests use `privileged: true` so the container can open USB serial devices (e.g. `/dev/ttyACM0`). Ensure `allowPrivilegeEscalation: true` is acceptable in your cluster's Pod Security policy/admission configuration.

---

## Repository Structure

```
.
├── kustomization.yaml       # Root Kustomize entry point
├── statefulset.yaml         # StatefulSet for zwave-js-ui
├── service.yaml             # ClusterIP Service (ports 8091 and 3000)
├── secret.yaml              # Default Secret — REPLACE before deploying
├── renovate.json            # Renovate bot image-update config
├── flux/
│   ├── gitrepository.yaml   # FluxCD GitRepository source
│   └── kustomization.yaml   # FluxCD Kustomization
└── examples/
    └── secret.yaml          # Annotated Secret template
```

---

## Configuration

### 1. Secret — Session Secret

The `secret.yaml` at the root contains a **default placeholder** secret. You **must** replace this before deploying to production.

Generate a strong secret and base64-encode it:

```bash
# Generate a random 32-byte hex secret
SECRET=$(openssl rand -hex 32)

# Base64-encode it for use in the Secret manifest
echo -n "$SECRET" | base64
```

Then replace the `session_secret` value in `secret.yaml`, or create the secret directly with kubectl:

```bash
kubectl create secret generic zwave-js-secret \
  --from-literal=session_secret="$(openssl rand -hex 32)" \
  --namespace=zwave
```

> **GitOps users:** Do **not** commit a real secret in plain text. Use [SOPS](https://fluxcd.io/flux/guides/mozilla-sops/) or [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) to encrypt it before committing.

### 2. Host Path (Storage)

By default, zwave-js-ui's configuration is stored at `/srv/container-data/zwavejs2mqtt` on the **Kubernetes node** that runs the pod. Change the `path` in `statefulset.yaml` to match your preferred location:

```yaml
# statefulset.yaml
volumes:
  - name: config
    hostPath:
      path: /srv/container-data/zwavejs2mqtt  # <-- change this
```

Ensure the path exists and is writable on the target node before deploying.

### 3. Timezone

The manifests mount `/usr/share/zoneinfo` from the host and set the timezone to `America/New_York`. Change the `subPath` in `statefulset.yaml` to match your timezone:

```yaml
# statefulset.yaml
volumeMounts:
  - name: zoneinfo
    mountPath: /etc/localtime
    subPath: America/New_York  # <-- change to your timezone (e.g. Europe/London)
    readOnly: true
```

### 4. Z-Wave USB Device

If your Z-Wave USB controller is not at `/dev/ttyACM0`, you will need to add a `hostPath` device mount. Add the following to the `volumeMounts` and `volumes` sections in `statefulset.yaml`:

```yaml
# Under spec.template.spec.containers[0].volumeMounts:
- name: zwave-device
  mountPath: /dev/ttyACM0

# Under spec.template.spec.volumes:
- name: zwave-device
  hostPath:
    path: /dev/ttyACM0   # <-- change to your device path
```

### 5. Namespace

All manifests are namespace-agnostic by default. It is recommended to deploy into a dedicated namespace (e.g. `zwave`):

```bash
kubectl create namespace zwave
```

When using Kustomize or FluxCD you can set the target namespace via the `targetNamespace` field in the `Kustomization` resource (see the FluxCD section below).

---

## Installation — FluxCD (GitOps)

This method is recommended if you manage your cluster with FluxCD.

### Step 1: Create the namespace and secret

```bash
kubectl create namespace zwave

kubectl create secret generic zwave-js-secret \
  --from-literal=session_secret="$(openssl rand -hex 32)" \
  --namespace=zwave
```

> If you want to manage the secret declaratively in Git, use SOPS or Sealed Secrets to encrypt `examples/secret.yaml` before committing.

### Step 2: Apply the FluxCD source

Copy `flux/gitrepository.yaml` to your FluxCD repository (typically under `clusters/<cluster-name>/` or `flux-system/`) and update the `tag` to the latest release:

```yaml
# flux/gitrepository.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: zwavejs2mqtt-kustomize
  namespace: flux-system
spec:
  interval: 1h
  url: https://github.com/turbo5000c/zwavejs2mqtt-kustomize
  ref:
    tag: "REPLACE_WITH_LATEST_TAG"   # <-- check https://github.com/turbo5000c/zwavejs2mqtt-kustomize/tags
```

Apply it:

```bash
kubectl apply -f flux/gitrepository.yaml
```

### Step 3: Apply the FluxCD Kustomization

Review and apply `flux/kustomization.yaml`:

```bash
kubectl apply -f flux/kustomization.yaml
```

### Step 4: Verify the deployment

```bash
# Watch Flux reconcile the Kustomization
flux get kustomizations zwavejs2mqtt

# Check the pod is running
kubectl get pods -n zwave
```

Expected output:

```
NAME              READY   STATUS    RESTARTS   AGE
zwave-js-ui-0     1/1     Running   0          2m
```

### Keeping Up-to-Date with FluxCD

Renovate will automatically open pull requests when a new zwave-js-ui image is published. Once merged, the CI workflow creates a matching Git tag. FluxCD's `GitRepository` will detect the new tag and reconcile the updated manifests.

To pin to a new release, update the `tag` field in `flux/gitrepository.yaml` and commit the change to your FluxCD repository.

---

## Installation — kubectl (direct)

This method is for users who want to deploy quickly without GitOps tooling.

### Step 1: Clone the repository

```bash
git clone https://github.com/turbo5000c/zwavejs2mqtt-kustomize.git
cd zwavejs2mqtt-kustomize
```

### Step 2: Create the namespace

```bash
kubectl create namespace zwave
```

### Step 3: Create the secret

```bash
kubectl create secret generic zwave-js-secret \
  --from-literal=session_secret="$(openssl rand -hex 32)" \
  --namespace=zwave
```

> **Do not apply the default `secret.yaml`** directly — it contains a placeholder value that is not secure.

### Step 4: Review and customise the manifests

- Edit `statefulset.yaml` to set the correct host path and timezone.
- Review `service.yaml` and change the service type if you need external access.

### Step 5: Apply the manifests

```bash
kubectl apply -k . --namespace=zwave
```

> The `-k` flag tells kubectl to use Kustomize mode.

### Step 6: Verify

```bash
kubectl get pods -n zwave
kubectl get svc -n zwave
```

### Step 7 (optional): Access the UI

Port-forward to access the web UI locally:

```bash
kubectl port-forward svc/zwave-js-ui 8091:8091 -n zwave
```

Then open [http://localhost:8091](http://localhost:8091) in your browser.

---

## Upgrading

### With FluxCD

1. Renovate will automatically open a PR updating the image tag in `statefulset.yaml`.
2. Merge the PR.
3. The CI workflow creates a new Git tag matching the image version.
4. FluxCD detects the new tag and reconciles the updated manifests automatically.

To manually trigger a reconciliation:

```bash
flux reconcile kustomization zwavejs2mqtt --with-source
```

### With kubectl

1. Pull the latest changes:
   ```bash
   git pull origin main
   ```
2. Re-apply the manifests:
   ```bash
   kubectl apply -k . --namespace=zwave
   ```
3. Verify the pod restarts on the new image:
   ```bash
   kubectl rollout status statefulset/zwave-js-ui -n zwave
   ```

---

## Maintenance

### Viewing Logs

```bash
kubectl logs -f statefulset/zwave-js-ui -n zwave
```

### Restarting the Pod

```bash
kubectl rollout restart statefulset/zwave-js-ui -n zwave
```

### Backing Up Configuration

The Z-Wave mesh configuration lives in the host path you configured (default: `/srv/container-data/zwavejs2mqtt`). Back up this directory regularly.

### Checking Resource Usage

```bash
kubectl top pod -n zwave
```

---

## Troubleshooting

### Pod is stuck in `Pending`

- Check if the node has the required USB device.
- Verify the host path directory exists on the node.
- Run `kubectl describe pod zwave-js-ui-0 -n zwave` and look at the Events section.

### Pod crashes with `CrashLoopBackOff`

- Check the logs: `kubectl logs zwave-js-ui-0 -n zwave`
- Verify the Z-Wave USB device path is correct.
- Ensure the `SESSION_SECRET` environment variable is populated (check the Secret exists in the correct namespace).

### Home Assistant cannot connect

- Confirm the pod is running: `kubectl get pods -n zwave`
- Confirm the service is created: `kubectl get svc -n zwave`
- Test connectivity from the Home Assistant pod:
  ```bash
  kubectl exec -it <ha-pod> -n <ha-namespace> -- curl http://zwave-js-ui.zwave.svc.cluster.local:8091
  ```
- If Home Assistant is outside the cluster, ensure port `3000` is reachable (consider `NodePort` or a reverse proxy).

### FluxCD Kustomization is not reconciling

```bash
flux get kustomizations zwavejs2mqtt
flux logs --kind=Kustomization --name=zwavejs2mqtt --namespace=flux-system
```

### Permission denied on USB device

- Ensure the pod is running with `privileged: true` (already set in `statefulset.yaml`).
- Check that your cluster's admission policy allows privileged pods.

---

## License

See [LICENSE](LICENSE) if present, or refer to the upstream [zwave-js-ui license](https://github.com/zwave-js/zwave-js-ui/blob/master/LICENSE).
