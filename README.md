# Kubeflow Scripts

Scripts for deploying Kubeflow on Kind with Podman support.

## 🚀 Features

- **Setup Script:** Deploy complete Kubeflow platform on Kind cluster
- **Build & Deploy Script:** Build and deploy custom Kubeflow components from local repository
- **Podman Support:** Works on systems with bleeding-edge kernels (Fedora 42+)
- **Notebooks v1 & v2:** Support for both Kubeflow Notebooks versions

## 📋 Prerequisites

- `podman` or `docker`
- `kubectl`
- `kind`
- `kustomize`
- `jq`, `yq`
- `git`, `make`

## 🔧 Quick Start

### 1. Deploy Kubeflow with Notebooks v1

```bash
cd setup
./setup-kind.sh --notebooks-v1 --podman
```

### 2. Deploy Custom Components

```bash
cd notebooks-v1
./build-and-deploy.sh --repo /path/to/your/notebooks-repo --podman
```

### 3. Access Kubeflow

```bash
kubectl -n istio-system port-forward svc/istio-ingressgateway 8443:443
```

Open: https://kubeflow.example.com:8443/

**Login:**
- Username: `user@example.com`
- Password: `12341234`

## 📖 Documentation

See [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md) for detailed deployment instructions, troubleshooting, and system requirements.

## 🔄 Changes from Original

This repository is based on [andyatmiami/kubeflow-scripts](https://github.com/andyatmiami/kubeflow-scripts) with the following enhancements:

- ✅ Added Podman support to `build-and-deploy.sh`
- ✅ Comprehensive deployment documentation
- ✅ Tested on Fedora 42 (kernel 6.14.2)

## 🐛 Troubleshooting

### Fedora 42 / Bleeding-edge Kernels

If you encounter errors with Docker on newer kernels:

1. **Use Podman:** Add `--podman` flag to all commands
2. **Increase inotify limits:**
   ```bash
   echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
   echo "fs.inotify.max_user_instances=512" | sudo tee -a /etc/sysctl.conf
   sudo sysctl -p
   ```
3. **Add hosts entry:**
   ```bash
   echo "127.0.0.1 kubeflow.example.com" | sudo tee -a /etc/hosts
   ```

## 📄 License

Based on Kubeflow manifests and notebooks repositories, which are Apache 2.0 licensed.

## 🤝 Contributing

Contributions welcome! Please ensure:
- Scripts are tested on both Docker and Podman
- Documentation is updated
- Backward compatibility is maintained

## 📚 Resources

- [Kubeflow Documentation](https://www.kubeflow.org/docs/)
- [Kind Documentation](https://kind.sigs.k8s.io/)
- [Podman Documentation](https://podman.io/)
- [Original Repository](https://github.com/andyatmiami/kubeflow-scripts)

