# Fixes and Modifications Applied to Kubeflow Scripts

This document summarizes all fixes and modifications applied to the original scripts from [andyatmiami/kubeflow-scripts](https://github.com/andyatmiami/kubeflow-scripts).

---

## 1. Podman Support for `build-and-deploy.sh`

**Script:** `notebooks-v1/build-and-deploy.sh`  
**Issue:** The original script only supported Docker, which is incompatible with Fedora 42 (kernel 6.14.2)  
**Solution:** Added Podman support via `--podman` flag

### Changes Made:

#### a) Global Variables (lines 36-38)
```bash
# Global variable for container runtime (set by parse_arguments)
use_podman=""
kind_cmd=""
```

#### b) Command Line Argument Parsing (lines 451-454)
```bash
--podman)
    use_podman="podman"
    shift
    ;;
```

#### c) Help Text (lines 475-476)
```bash
echo "  --podman             Use podman instead of docker for building and kind operations"
echo "                       (default: docker)"
```

#### d) Prerequisites Check (lines 810-820)
```bash
# Check for appropriate container runtime
if [ -n "$use_podman" ]; then
    if ! command -v podman >/dev/null 2>&1; then
        print_error "podman is not installed or not in PATH"
        return 1
    fi
else
    if ! command -v docker >/dev/null 2>&1; then
        print_error "docker is not installed or not in PATH"
        return 1
    fi
fi
```

#### e) Image Loading Function (line 791)
```bash
if ! eval "$kind_cmd load docker-image \"$full_image\""; then
```
Changed from hardcoded `kind load` to use `$kind_cmd` variable.

#### f) Main Function Initialization (lines 1092-1097)
```bash
# Set kind command based on use_podman flag
if [ -n "$use_podman" ]; then
    kind_cmd="KIND_EXPERIMENTAL_PROVIDER=podman kind"
else
    kind_cmd="kind"
fi
```

### Why This Fix Was Necessary:

**Problem:**
- Fedora 42 (Rawhide) ships with kernel 6.14.2, which is bleeding-edge
- Docker (even v28.5.1) has issues with systemd inside containers on this kernel
- Kind clusters using Docker fail with: `kubelet is unhealthy due to: The kubelet is not running`
- Error: `The HTTP call equal to 'curl -sSL http://127.0.0.1:10248/healthz' returned error: Get "http://127.0.0.1:10248/healthz": context deadline exceeded`

**Why Podman Works:**
- Podman is designed specifically for Red Hat-based systems (Fedora/RHEL)
- Better integration with newer kernels and cgroup v2
- Native support for rootless containers
- Better handling of systemd inside containers

### Usage:
```bash
./build-and-deploy.sh --repo ~/Projects/notebooks-security/ --podman
```

---

## 2. JWT/JWKS Authentication Fix for `setup-kind.sh`

**Script:** `setup/setup-kind.sh`  
**Issue:** Missing `jwksUri` in Istio RequestAuthentication causes "Jwks doesn't have key to match kid or alg from Jwt" error  
**Solution:** Patch RequestAuthentication to include JWKS endpoint after Kubeflow setup

### Changes Made:

#### Addition to `_setup_kubeflow()` function (lines 395-398)
```bash
# Fix RequestAuthentication to include jwksUri for JWT validation
# This is a workaround for missing jwksUri in upstream Kubeflow manifests
kubectl patch requestauthentication dex-jwt -n istio-system --type=json \
    -p='[{"op": "add", "path": "/spec/jwtRules/0/jwksUri", "value": "http://dex.auth.svc.cluster.local:5556/dex/keys"}]'
```

### Why This Fix Was Necessary:

**Problem:**
- The upstream Kubeflow manifests (v1.10.2) deploy a `RequestAuthentication` resource without the `jwksUri` field
- Without `jwksUri`, Istio's ingress gateway cannot fetch Dex's public keys (JWKS) to validate JWT tokens
- When users try to access the Kubeflow dashboard, they get: `jwt_authn_access_denied{Jwks_doesn't_have_key_to_match_kid_or_alg_from_Jwt}`
- This results in HTTP 401 Unauthorized errors even with valid credentials

**Root Cause:**
- Istio's JWT authentication requires either:
  - `jwks`: Inline JSON Web Key Set (static)
  - `jwksUri`: URL to fetch JWKS dynamically (recommended for rotating keys)
- Dex rotates its signing keys periodically for security
- The upstream manifest only specifies the `issuer` but not where to get the keys

**What the Fix Does:**
1. Waits for Kubeflow installation to complete
2. Patches the `dex-jwt` RequestAuthentication resource in `istio-system` namespace
3. Adds `jwksUri: http://dex.auth.svc.cluster.local:5556/dex/keys`
4. Istio can now fetch Dex's current public keys to validate JWT tokens

**Expected RequestAuthentication Configuration After Fix:**
```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: dex-jwt
  namespace: istio-system
spec:
  jwtRules:
  - forwardOriginalToken: true
    fromHeaders:
    - name: Authorization
      prefix: 'Bearer '
    issuer: http://dex.auth.svc.cluster.local:5556/dex
    jwksUri: http://dex.auth.svc.cluster.local:5556/dex/keys  # <- This line is added
    outputClaimToHeaders:
    - claim: email
      header: kubeflow-userid
    - claim: groups
      header: kubeflow-groups
  selector:
    matchLabels:
      app: istio-ingressgateway
```

### Verification:

After the fix is applied, you can verify it worked:

```bash
# Check the RequestAuthentication has jwksUri
kubectl get requestauthentication dex-jwt -n istio-system -o jsonpath='{.spec.jwtRules[0].jwksUri}'
# Expected output: http://dex.auth.svc.cluster.local:5556/dex/keys

# Test that Dex JWKS endpoint is accessible
kubectl run curl-test --image=curlimages/curl:latest --rm -i --restart=Never \
  --command -- curl -s http://dex.auth.svc.cluster.local:5556/dex/keys
# Expected: JSON response with keys array

# Check istio-ingressgateway logs for JWT errors (should be none)
kubectl logs -n istio-system deployment/istio-ingressgateway | grep -i "jwt\|jwks"
```

### User Action Required:

After redeploying the cluster, users should:
1. Clear browser cookies for `kubeflow.example.com`
2. Or use an incognito/private browsing window
3. Login again with credentials: `user@example.com` / `12341234`

The old JWT tokens in the browser are signed with old keys that no longer exist after Dex restarts.

---

## 3. Additional System Configuration Required (Fedora 42)

**Note:** This is not a script modification, but a prerequisite system configuration for Fedora 42.

### Increase inotify Limits

**Issue:** Profiles deployment fails with "too many open files" error  
**Solution:** Increase inotify limits

```bash
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances=512" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

**Why This is Needed:**
- Kubernetes controllers watch many resources simultaneously
- Fedora's default inotify limits are too low for Kubeflow
- Without this, deployment fails during profiles setup

---

## Summary of Script Status

### `setup-kind.sh` (setup/setup-kind.sh)
- ✅ **Original Podman support:** Already present in original script (no changes needed)
- ✅ **JWT/JWKS fix:** **ADDED** - Patches RequestAuthentication after Kubeflow setup

### `build-and-deploy.sh` (notebooks-v1/build-and-deploy.sh)
- ✅ **Podman support:** **ADDED** - Full support via `--podman` flag
- ✅ **Image loading:** Modified to use dynamic `$kind_cmd` variable

---

## Testing the Fixes

### Test Podman Support:
```bash
# Test setup-kind.sh with Podman
cd ~/Projects/kubeflow-scripts/setup
./setup-kind.sh --notebooks-v1 --podman

# Test build-and-deploy.sh with Podman
cd ~/Projects/kubeflow-scripts/notebooks-v1
./build-and-deploy.sh --repo ~/Projects/notebooks-security/ --podman
```

### Test JWT/JWKS Fix:
```bash
# After deployment, check RequestAuthentication
kubectl get requestauthentication dex-jwt -n istio-system -o yaml | grep jwksUri

# Access dashboard and verify no JWT errors
kubectl -n istio-system port-forward svc/istio-ingressgateway 8443:443
# Open browser: https://kubeflow.example.com:8443/
# Login with: user@example.com / 12341234
```

---

## Future Considerations

### Upstream Fix Needed:
The JWT/JWKS fix is a **workaround** for an issue in the upstream Kubeflow manifests. Ideally:
- The `RequestAuthentication` in the upstream Kubeflow manifests should include `jwksUri`
- This would eliminate the need for the post-deployment patch
- Consider submitting a PR to [kubeflow/manifests](https://github.com/kubeflow/manifests)

### Alternative Solutions Considered:
1. **Inline JWKS:** Could specify `jwks` instead of `jwksUri` with static keys
   - ❌ Bad: Keys would be stale after Dex restart
   - ❌ Bad: Doesn't support key rotation

2. **Modify Dex Configuration:** Could configure Dex to use static keys
   - ❌ Bad: Reduces security (no key rotation)
   - ❌ Bad: Requires modifying more components

3. **Current Solution (jwksUri patch):** Best approach
   - ✅ Supports dynamic key rotation
   - ✅ Minimal modification (single kubectl patch)
   - ✅ Works immediately after deployment

---

## References

- Original scripts: [andyatmiami/kubeflow-scripts](https://github.com/andyatmiami/kubeflow-scripts)
- Kubeflow manifests: [kubeflow/manifests v1.10.2](https://github.com/kubeflow/manifests/tree/v1.10.2)
- Istio RequestAuthentication: [Istio Security Docs](https://istio.io/latest/docs/reference/config/security/request_authentication/)
- Dex JWKS endpoint: [Dex Documentation](https://dexidp.io/docs/)
- Kind with Podman: [Kind Experimental Podman Provider](https://kind.sigs.k8s.io/docs/user/rootless/)

