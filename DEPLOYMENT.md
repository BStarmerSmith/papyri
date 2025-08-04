# K3s Deployment Guide

This guide documents the deployment of the HobbyHelper application stack on a K3s cluster using GitOps with Flux.

## System Architecture

Our deployment uses the following components:

- **K3s**: Lightweight Kubernetes distribution
- **Flux**: GitOps tool for managing Kubernetes resources
- **Traefik**: Ingress controller (built into K3s)
- **MongoDB**: Database for the application
- **HobbyHelper Backend**: Go-based API backend
- **HobbyHelper Frontend**: React-based web frontend
- **Prometheus & Grafana**: Monitoring stack

## Prerequisites

- Linux server with minimum specs:
  - 8 cores / 16 threads (AMD R7-7840HS)
  - 16 GB RAM
  - 512 GB storage
- Tailscale installed and configured
- Domain configured to point to your server

## Initial Setup

### 1. Install K3s

```bash
# Install K3s with default configuration (includes Traefik)
curl -sfL https://get.k3s.io | sh -

# Get kubeconfig
mkdir -p ~/.kube
sudo cat /etc/rancher/k3s/k3s.yaml > ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config
```

### 2. Bootstrap Flux

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux with your Git repository
flux bootstrap github \
  --owner=BStarmerSmith \
  --repository=papyri \
  --branch=main \
  --path=clusters/talos \
  --personal
```

## Repository Structure

The GitOps repository is organized as follows:

```
papyri/
├── .github/
│   └── workflows/
│       ├── backend-ci.yaml
│       └── frontend-ci.yaml
├── clusters/
│   └── talos/
│       ├── apps/
│       │   ├── hobbyhelper/
│       │   │   ├── backend.yaml
│       │   │   ├── frontend.yaml
│       │   │   ├── ingress.yaml
│       │   │   ├── mongodb.yaml
│       │   │   └── kustomization.yaml
│       │   └── kustomization.yaml
│       ├── flux-system/
│       │   ├── apps.yaml
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   ├── infrastructure.yaml
│       │   ├── kustomization.yaml
│       │   └── monitoring.yaml
│       ├── infrastructure/
│       │   ├── namespace.yaml
│       │   └── kustomization.yaml
│       └── monitoring/
│           ├── prometheus-stack.yaml
│           └── kustomization.yaml
└── DEPLOYMENT.md
```

## Deployments

### 1. Infrastructure

The infrastructure components include:
- Namespace definitions for `hobbyhelper` and `monitoring`

### 2. Applications

The HobbyHelper application stack consists of:

- **MongoDB StatefulSet**:
  - Persistent storage with local-path StorageClass
  - Credentials stored in a Kubernetes Secret

- **Backend Deployment**:
  - Go API running on port 3000
  - Connects to MongoDB
  - Health checks configured

- **Frontend Deployment**:
  - React application served through Nginx
  - Configured to proxy API requests to the backend

- **Ingress Configuration**:
  - Traefik IngressRoute resources for HTTP and HTTPS
  - Automatic HTTPS redirection
  - Routing for frontend and API paths

### 3. Monitoring

The monitoring stack includes:
- Prometheus for metrics collection
- Grafana for visualization
- Node Exporter for host metrics
- AlertManager for alerting

## CI/CD Pipeline

The CI/CD pipeline is implemented with GitHub Actions:

1. **Backend CI/CD**:
   - Triggered on changes to the backend code
   - Runs tests, builds Docker image
   - Pushes image to GitHub Container Registry
   - Updates the image tag in the GitOps repository

2. **Frontend CI/CD**:
   - Triggered on changes to the frontend code
   - Lints code, builds Docker image
   - Pushes image to GitHub Container Registry
   - Updates the image tag in the GitOps repository

## Access and Management

### Accessing Applications

- HobbyHelper Frontend: https://hobby.example.com
- HobbyHelper API: https://hobby.example.com/api
- Grafana Dashboard: https://grafana.hobby.example.com

### Kubectl Commands

```bash
# View all pods
kubectl get pods -A

# Check logs for a specific pod
kubectl logs -n hobbyhelper deployment/hobbyhelper-backend

# Describe a service
kubectl describe -n hobbyhelper svc/mongodb

# Port-forward to access a service locally
kubectl port-forward -n hobbyhelper svc/mongodb 27017:27017
```

### Flux Commands

```bash
# Check Flux components status
flux check

# Get all Kustomizations
flux get kustomizations

# Trigger reconciliation of a specific Kustomization
flux reconcile kustomization apps

# Get image policies
flux get images all -A
```

## Troubleshooting

### Common Issues

1. **Pods not starting**:
   - Check events: `kubectl get events -n hobbyhelper`
   - Check pod status: `kubectl describe pod -n hobbyhelper <pod-name>`

2. **Database connection issues**:
   - Verify MongoDB is running: `kubectl get pods -n hobbyhelper -l app=mongodb`
   - Check backend logs: `kubectl logs -n hobbyhelper deployment/hobbyhelper-backend`

3. **Ingress not working**:
   - Check Traefik logs: `kubectl logs -n kube-system -l app.kubernetes.io/name=traefik`
   - Verify IngressRoute configuration: `kubectl get ingressroute -n hobbyhelper`

4. **CI/CD pipeline failures**:
   - Check GitHub Actions logs in the repository
   - Ensure FLUX_PAT secret is configured with appropriate permissions

## Maintenance

### Backup Strategy

1. **MongoDB Data**:
   ```bash
   # Create a backup job
   kubectl create job --from=cronjob/mongodb-backup backup-manual -n hobbyhelper
   ```

2. **Kubernetes Resources**:
   ```bash
   # Export all resources in the hobbyhelper namespace
   kubectl get all -n hobbyhelper -o yaml > hobbyhelper-backup.yaml
   ```

### Scaling

```bash
# Scale backend deployment
kubectl scale deployment -n hobbyhelper hobbyhelper-backend --replicas=4

# Scale frontend deployment
kubectl scale deployment -n hobbyhelper hobbyhelper-frontend --replicas=4
```

### Updating Applications

1. Push changes to the respective application repository
2. CI/CD pipeline will automatically build and deploy the new version

## Security Considerations

1. **Secrets Management**:
   - Kubernetes Secrets are used for storing credentials
   - Consider using a more secure solution like HashiCorp Vault for production

2. **Network Security**:
   - Tailscale provides secure access to the cluster
   - All external access is via HTTPS

3. **Resource Isolation**:
   - Separate namespaces for applications and monitoring
   - Resource limits defined for all containers

## Future Enhancements

1. **Certificate Management**:
   - Implement cert-manager for automated certificate management

2. **Backup Automation**:
   - Set up CronJobs for regular MongoDB backups

3. **High Availability**:
   - Add additional K3s worker nodes for HA setup
