# Kubeflow Deployment Guide - Fedora 42 (Kernel 6.14.2)

## Environment Context

**System:** Fedora 42 (Rawhide)  
**Kernel:** 6.14.2-300.fc42.x86_64  
**Issue:** Docker incompatibility with bleeding-edge kernel  
**Solution:** Use Podman instead of Docker

> **📝 Note:** For detailed information about all fixes and modifications applied to the scripts, see [FIXES.md](./FIXES.md)

---

## Deployment Process

### Prerequisites

1. **Install required tools:**
   ```bash
   sudo dnf install -y podman kubectl kind kustomize jq yq git make
   ```

2. **Increase inotify limits (IMPORTANT for Fedora):**
   ```bash
   echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
   echo "fs.inotify.max_user_instances=512" | sudo tee -a /etc/sysctl.conf
   sudo sysctl -p
   ```

3. **Add Kubeflow domain to /etc/hosts:**
   ```bash
   echo "127.0.0.1 kubeflow.example.com" | sudo tee -a /etc/hosts
   ```

---

### Step 1: Deploy Full Kubeflow with Notebooks v1

```bash
cd ~/Projects/kubeflow-scripts/setup
./setup-kind.sh --notebooks-v1 --podman
```

**What this does:**
- ✅ Creates kind cluster with Podman (Kubernetes v1.32.0)
- ✅ Installs cert-manager
- ✅ Installs Istio service mesh
- ✅ Installs Dex (authentication)
- ✅ Installs OAuth2-Proxy
- ✅ Installs Kubeflow Profiles
- ✅ Installs Central Dashboard
- ✅ Installs Notebooks v1 components:
  - Network policies
  - PodDefaults admission webhook
  - Notebook Controller (upstream)
  - Jupyter Web App (upstream)
  - PVCViewer Controller
  - Volumes Web App (upstream)
  - Tensorboard Controller (upstream)
  - Tensorboards Web App (upstream)
- ✅ Creates sample profile (`kubeflow-user-example-com`)
- ✅ Sets up HTTPS with self-signed certificates
- ✅ Configures local access

**Expected output:**
```
Next steps:

- Run the following command to enable local access to Kubeflow Dashboard:

  kubectl -n istio-system port-forward svc/istio-ingressgateway 8443:443

- Open the following URL in your browser:

  https://kubeflow.example.com:8443/

- Login with the following credentials:

  Username: user@example.com
  Password: 12341234
```

**Wait time:** ~10-15 minutes (depending on internet speed and system resources)

---

### Step 2: Deploy Custom Components from Your Repository

After the full Kubeflow installation is complete, redeploy specific components with your custom code:

```bash
cd ~/Projects/kubeflow-scripts/notebooks-v1
./build-and-deploy.sh --repo ~/Projects/notebooks-security/ --podman
```

**What this does:**
- ✅ Builds Docker images with your custom code using Podman
- ✅ Loads images into kind cluster using Podman
- ✅ Redeploys controllers:
  - Notebook Controller (your custom version)
  - PVCViewer Controller (your custom version)
  - Tensorboard Controller (your custom version)
- ✅ Redeploys CRUD web apps:
  - Jupyter Web App (your custom version)
  - Volumes Web App (your custom version)
  - Tensorboards Web App (your custom version)

**Expected output:**
```
[INFO] All components (controllers and crud-web-apps) have been successfully built and deployed!
```

**Wait time:** ~10-15 minutes (includes building frontend with npm dependencies)

---

### Step 3: Access Kubeflow Dashboard

1. **Start port-forwarding:**
   ```bash
   kubectl -n istio-system port-forward svc/istio-ingressgateway 8443:443
   ```

2. **Open browser:**
   ```
   https://kubeflow.example.com:8443/
   ```

3. **Login:**
   - **Username:** `user@example.com`
   - **Password:** `12341234`

4. **Navigate to web apps:**
   - **Jupyter (Notebook Servers):** Click "Notebook Servers" in the left menu
   - **Volumes (PVCs):** Click "Volumes" in the left menu
   - **Tensorboards:** Click "Tensorboards" in the left menu

---

## Verification Commands

### Check if all pods are running:
```bash
kubectl get pods -n kubeflow
```

Expected: All pods should be `Running` with `READY` showing correct container counts (e.g., `2/2`, `3/3`).

### Check deployed components:
```bash
kubectl get deployments -n kubeflow
```

Expected deployments:
- `centraldashboard`
- `jupyter-web-app-deployment`
- `notebook-controller-deployment`
- `profiles-deployment`
- `pvcviewer-controller-manager`
- `tensorboard-controller-deployment`
- `tensorboards-web-app-deployment`
- `volumes-web-app-deployment`

### Check custom image versions:
```bash
kubectl get deployment -n kubeflow jupyter-web-app-deployment -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Expected: Should show your custom image tag (e.g., `ghcr.io/kubeflow/notebooks/jupyter-web-app:a264293-dirty`)

### Check authentication is working:
```bash
kubectl get pods -n auth
kubectl get pods -n oauth2-proxy
```

Expected: `dex` and `oauth2-proxy` pods should be `Running`.

---

## Troubleshooting

### Issue: Port-forward disconnects
**Solution:** Restart port-forward
```bash
pkill -f port-forward
kubectl -n istio-system port-forward svc/istio-ingressgateway 8443:443
```

### Issue: Pods stuck in `ContainerCreating`
**Solution:** Check events
```bash
kubectl describe pod <pod-name> -n kubeflow
```

### Issue: "No user detected" error after login
**Solution:** Check authorization policies
```bash
kubectl get authorizationpolicies -n istio-system
kubectl get authorizationpolicies -n kubeflow
```

Ensure these exist:
- `istio-ingressgateway-oauth2-proxy` (in istio-system)
- `istio-ingressgateway-require-jwt` (in istio-system)

### Issue: Image not loading into kind cluster
**Solution:** Manually load image
```bash
KIND_EXPERIMENTAL_PROVIDER=podman kind load docker-image <image-name:tag>
```

---

## Cleanup

### Delete the cluster:
```bash
KIND_EXPERIMENTAL_PROVIDER=podman kind delete cluster --name kind
```

### Remove port-forward:
```bash
pkill -f port-forward
```

---

## Summary of Key Differences from Other Machines

**On standard systems (Ubuntu, Debian, older Fedora):**
- Docker works out of the box
- No need for `--podman` flag
- No need to increase inotify limits (usually)

**On Fedora 42 with kernel 6.14.2:**
- ✅ **MUST use Podman** (`--podman` flag)
- ✅ **MUST increase inotify limits** before deployment
- ✅ All other functionality works identically

The scripts are portable - the same deployment process works on any system, just use the appropriate container runtime flag.

---

## References

- **[FIXES.md](./FIXES.md)** - Complete documentation of all fixes applied to scripts
- Original scripts: [andyatmiami/kubeflow-scripts](https://github.com/andyatmiami/kubeflow-scripts)
- Kubeflow manifests: [kubeflow/manifests v1.10.2](https://github.com/kubeflow/manifests/tree/v1.10.2)
- Kind documentation: [kind.sigs.k8s.io](https://kind.sigs.k8s.io/)
- Podman with Kind: [Kind Experimental Podman Provider](https://kind.sigs.k8s.io/docs/user/rootless/)

