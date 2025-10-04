# ArgoCD Demo Application

A comprehensive demonstration repository showcasing ArgoCD functionality, including GitOps workflows, monitoring with Prometheus and Grafana, RBAC implementation, and progressive delivery strategies using Argo Rollouts.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Part 1: ArgoCD Setup and Basic Operations](#part-1-argocd-setup-and-basic-operations)
- [Part 2: Monitoring with Prometheus and Grafana](#part-2-monitoring-with-prometheus-and-grafana)
- [Part 3: RBAC and Security](#part-3-rbac-and-security)
- [Part 4: Progressive Delivery with Argo Rollouts](#part-4-progressive-delivery-with-argo-rollouts)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before starting, ensure you have the following installed and configured:

### Required Tools

- **kubectl** - Verify installation with:

  ```bash
  kubectl version --client
  ```

- **kubeconfig** file - Default location: `~/.kube/config`

  ```powershell
  # Windows PowerShell verification
  if (Test-Path "$env:USERPROFILE\.kube\config") {
      Write-Host "kubeconfig file found at: $env:USERPROFILE\.kube\config"
  } else {
      Write-Host "kubeconfig file NOT found at default location"
  }
  ```

- **Minikube** - Required for local Kubernetes cluster
- **Docker Engine** - Must be running before starting Minikube

### Important Notes

- **CoreDNS** is automatically provided when a Kubernetes cluster is running. For MicroK8s users, enable with: `microk8s enable dns && microk8s stop && microk8s start`
- This tutorial uses Minikube as the Kubernetes environment

---

## Part 1: ArgoCD Setup and Basic Operations

### 1.1 Bootstrap ArgoCD

Start your local Kubernetes cluster and install ArgoCD:

```bash
# Start Minikube with Docker driver
minikube start --driver=docker

# Verify Minikube is running
minikube version

# Create ArgoCD namespace and install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argocd/stable/manifests/install.yaml

# Install ArgoCD CLI (Windows with Chocolatey - requires admin access)
choco install argocd-cli
```

### 1.2 Access ArgoCD UI

```bash
# Port forward to access ArgoCD API server
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Access the UI at `https://localhost:8080`

- **Username**: `admin`
- **Password**: Retrieve with `argocd admin initial-password -n argocd`
- **Note**: You may see a certificate warning (`ERR_CERT_AUTHORITY_INVALID`). Click "Advanced" and proceed.

### 1.3 Login via CLI

```bash
# Login to ArgoCD (accept insecure connection when prompted)
argocd login localhost:8080
```

### 1.4 Register Kubernetes Cluster

```bash
# List available contexts
kubectl config get-contexts -o name

# Register Minikube cluster with ArgoCD
argocd cluster add minikube

# Verify cluster registration (ensure API server is port-forwarded)
argocd cluster list
```

### 1.5 Create ArgoCD Application

```bash
# Set current namespace to argocd
kubectl config set-context --current --namespace=argocd

# Create 'guestbook' application from this repository
argocd app create guestbook \
  --repo https://github.com/samtaitai/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

**GUI Alternative**: When creating an application via the ArgoCD UI:

- Repository URL: `https://github.com/samtaitai/argocd-example-apps`
- Path: `guestbook`
- Cluster URL: `https://kubernetes.default.svc` (deploys to your Minikube cluster)
- Destination Namespace: `default`

### 1.6 Sync Application

```bash
# View application details
argocd app get guestbook

# Sync application (performs kubectl apply of manifests)
argocd app sync guestbook

# Verify sync status
argocd app list
```

### 1.7 Simulate and Correct Drift

Demonstrate how ArgoCD detects and corrects configuration drift:

```bash
# Check current deployments
kubectl get deployment -n default

# Simulate drift by scaling replicas
kubectl scale deployment guestbook-ui --replicas=3 -n default

# Detect drift (should show "OutOfSync")
argocd app get guestbook

# Correct drift (restores to Git state)
argocd app sync guestbook

# Verify correction
kubectl get deployment -n default
kubectl describe deployment guestbook-ui -n default
```

### 1.8 Demonstrate Rollback

Show how ArgoCD enables easy rollbacks through Git:

1. **Modify replicas in Git**: Edit `guestbook-ui-deployment.yaml` (change replicas from 1 to 3)
2. **Sync changes**: `argocd app sync guestbook`
3. **Verify running replicas**: `kubectl get deployment guestbook-ui -n default`
4. **Revert in Git**:
   ```bash
   git revert HEAD
   # Save and exit vim editor with :wq
   git push
   git pull  # Update your local working directory
   ```
5. **Sync rollback**: `argocd app sync guestbook`
6. **Verify rollback**: Replicas should return to 1

### 1.9 Health Monitoring

Demonstrate ArgoCD's health monitoring capabilities:

```bash
# Introduce a bad deployment by editing guestbook-ui-deployment.yaml
```

```yaml
spec:
  containers:
    - image: gcr.io/google-samples/gb-frontend:nonexistent-tag # Bad image tag
```

```bash
# Push changes
git push

# Sync application
argocd app sync guestbook

# Monitor app status (should show degraded health)
argocd app get guestbook

# Watch pods fail in real-time
kubectl get pods -n default -w

# Fix by reverting
git revert HEAD
git push
git pull

# Sync and verify
argocd app sync guestbook
argocd app get guestbook
```

---

## Part 2: Monitoring with Prometheus and Grafana

### 2.1 Enable ArgoCD Metrics

```bash
# Verify Application Controller is running
kubectl get pods -n argocd | findstr application-controller

# Create metrics service
kubectl apply -f argocd-metrics-services.yaml

# Verify service creation
kubectl get services -n argocd

# Enable metrics endpoint
kubectl port-forward svc/argocd-metrics -n argocd 8082:8082

# Verify metrics are accessible
curl http://localhost:8082/metrics
```

### 2.2 Install Prometheus

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Apply Prometheus configuration
kubectl apply -f prometheus-argocd-config.yaml

# Access Prometheus UI
kubectl port-forward svc/prometheus-service -n monitoring 9090:9090
```

Access Prometheus at `http://localhost:9090`

### 2.3 Verify Prometheus Metrics Collection

Key metrics to verify:

1. **argocd_app_sync_total** - Total number of application syncs
2. **argocd_app_info** - Application information and health status (health is a label)
3. **argocd_cluster_api_resources** - Number of monitored Kubernetes API resources

```bash
# Trigger sync to generate metrics
argocd app sync guestbook

# Query metrics via API
curl "http://localhost:9090/api/v1/query?query=argocd_app_info"
```

### 2.4 Install and Configure Grafana

```bash
# Apply Grafana configuration
kubectl apply -f grafana-config.yaml

# Verify Grafana service
kubectl get svc -n monitoring

# Access Grafana UI
kubectl port-forward svc/grafana-service -n monitoring 3000:3000
```

Access Grafana at `http://localhost:3000`

**Configure Data Source**:

1. Login to Grafana (default: admin/admin)
2. Add Prometheus data source
3. URL: `http://prometheus-service:9090`

### 2.5 Create Grafana Dashboard

Create a dashboard with the following queries:

- `argocd_app_info`
- `argocd_app_sync_total`
- `argocd_cluster_api_resources`

### 2.6 Configure Prometheus Alerts

```bash
# Create alert rules
kubectl apply -f prometheus-alert-rules.yaml

# Update Prometheus configuration
kubectl apply -f prometheus-argocd-config.yaml

# Restart Prometheus to load new rules
kubectl delete pod -l app=prometheus -n monitoring

# Verify pod recreation
kubectl get pods -n monitoring | findstr prometheus

# Port forward if needed
kubectl port-forward svc/prometheus-service -n monitoring 9090:9090
```

Verify alert rules are loaded by navigating to the "Alerts" section in Prometheus UI.

### 2.7 Install Alertmanager

```bash
# Create Alertmanager
kubectl apply -f alertmanager-config.yaml

# Update Prometheus configuration (uncomment Alertmanager section)
kubectl apply -f prometheus-argocd-config.yaml

# Restart Prometheus
kubectl delete pod -l app=prometheus -n monitoring

# Access Alertmanager UI
kubectl port-forward svc/alertmanager-service -n monitoring 9093:9093
```

Access Alertmanager at `http://localhost:9093`

### 2.8 Trigger Alerts

**Trigger OutOfSync Alert**:

```bash
# Scale deployment manually to create drift
kubectl scale deployment guestbook-ui --replicas=3 -n default

# Verify change
kubectl get deployment guestbook-ui -n default

# Alert will be pending for 30s, then fire
```

**Trigger Unhealthy Alert**:

```bash
# Delete service to simulate degraded health
kubectl delete service guestbook-ui -n default

# Verify service is gone
kubectl get services -n default | findstr guestbook
```

**View Firing Alerts**:

- Prometheus: `http://localhost:9090` → Status → Rules → Alerts tab
- Alertmanager: `http://localhost:9093` → Alerts page

**Restore Services**:

```bash
# Restore deleted service
kubectl apply -f guestbook/guestbook-ui-svc.yaml

# Reset replica count
argocd app sync guestbook
```

---

## Part 3: RBAC and Security

### Prerequisites

```bash
# Ensure Minikube is running
minikube start --driver=docker

# Port-forward ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 3.1 Create AppProject

Create a custom project to restrict access to specific Git repositories and namespaces:

```bash
# Create restricted project
kubectl apply -f restricted-project.yaml

# Verify project creation
kubectl get appproject -n argocd restricted-demo-project
```

**Key Concepts**:

- **Kubernetes namespace**: Where the workload runs
- **ArgoCD project**: Defines permissions for managing workloads

### 3.2 Create Local Users

```bash
# Create users in ArgoCD
kubectl apply -f argocd-cm-users.yaml

# Set passwords for users
argocd account update-password --account readonly-user
argocd account update-password --account deploy-user

# Apply RBAC configuration
kubectl apply -f argocd-rbac-cm.yaml

# Verify RBAC config
kubectl get configmap argocd-rbac-cm -n argocd -o yaml

# Restart ArgoCD server to apply changes
kubectl rollout restart deployment argocd-server -n argocd
kubectl rollout status deployment argocd-server -n argocd
```

### 3.3 Create Test Application

```bash
# Create application for testing RBAC
kubectl apply -f test-application.yaml

# Verify application
kubectl get application -n argocd restricted-guestbook
```

### 3.4 Test RBAC Permissions

**Test Read-Only User**:

```bash
# Login as readonly-user
argocd login localhost:8080 --username readonly-user --password readonly123 --insecure

# Allowed operations
argocd app list
argocd app get restricted-guestbook
argocd proj get restricted-demo-project

# Denied operations (will fail)
argocd app sync restricted-guestbook
argocd app delete restricted-guestbook
```

**Test Deploy User**:

```bash
# Logout first
argocd logout

# Login as deploy-user
argocd login localhost:8080 --username deploy-user --password deploy123 --insecure

# Allowed operations
argocd app sync restricted-guestbook

# Denied operations (will fail)
argocd app delete restricted-guestbook
argocd proj role create restricted-demo-project deploy-role
```

### 3.5 Monitor Access Logs

**View Server Logs for Permission Denials**:

```powershell
# PowerShell
kubectl logs -f deployment/argocd-server -n argocd | Select-String -Pattern "permission denied|unauthorized|403"
```

**View Application Event Logs**:

```bash
# Shows sync and health events, including access attempts
kubectl describe application restricted-guestbook -n argocd
```

### 3.6 RBAC Security Benefits

AppProject and RBAC secure the GitOps pipeline by:

- Restricting which Git repositories can be used
- Limiting deployment targets to specific namespaces
- Controlling user permissions (read-only vs deploy)
- Providing audit trails of user actions
- Enforcing least-privilege access principles

---

## Part 4: Progressive Delivery with Argo Rollouts

### 4.1 Install Argo Rollouts (Optional - if not installed)

```bash
# Create Argo Rollouts namespace
kubectl create namespace argo-rollouts

# Install Argo Rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install kubectl plugin
kubectl krew install argo-rollouts
```

### 4.2 Canary Deployment Strategy

**Setup**:

```bash
# Create namespace
kubectl create namespace canary-demo

# Apply canary rollout manifest
kubectl apply -f canary-rollout.yaml
```

**Trigger Update and Monitor**:

```bash
# Update application image (edit canary-rollout.yaml)
kubectl apply -f canary-rollout.yaml

# Watch rollout progress
kubectl argo rollouts get rollout <rollout-name> -n canary-demo --watch

# Pause rollout to inspect traffic distribution
kubectl argo rollouts pause <rollout-name> -n canary-demo

# Resume rollout
kubectl argo rollouts promote <rollout-name> -n canary-demo

# Verify final state
kubectl get pods -n canary-demo
```

Access application at `http://localhost:8080`

**Canary Strategy Benefits**:

- Gradual traffic shifting reduces blast radius
- Ability to pause and validate at each step
- Quick rollback if issues detected
- Real production traffic testing with minimal risk

### 4.3 BlueGreen Deployment Strategy

**Setup**:

```bash
# Create namespace
kubectl create namespace bluegreen-demo

# Apply bluegreen rollout manifest
kubectl apply -f bluegreen-rollout.yaml

# Access services
# Preview: http://localhost:8082
# Production: http://localhost:8081
```

**Trigger Update and Promote**:

```bash
# Update application image (edit bluegreen-rollout.yaml)
kubectl apply -f bluegreen-rollout.yaml

# Watch rollout progress
kubectl argo rollouts get rollout <rollout-name> -n bluegreen-demo --watch

# Promote preview to production
kubectl argo rollouts promote <rollout-name> -n bluegreen-demo

# Verify promotion
kubectl get services -n bluegreen-demo
```

**BlueGreen Strategy Benefits**:

- Complete environment isolation
- Instant rollback capability
- Full testing before production switch
- Zero-downtime deployments
- Simple cutover process

### 4.4 Compare Deployment Strategies

| Strategy      | Use Case                             | Risk Level | Rollback Speed | Complexity |
| ------------- | ------------------------------------ | ---------- | -------------- | ---------- |
| **Canary**    | Gradual validation with live traffic | Low        | Medium         | Medium     |
| **BlueGreen** | Full validation before cutover       | Very Low   | Instant        | Low        |

---

## Troubleshooting

### Session Expiration

If you encounter authentication errors:

```bash
# Error: "token is expired"
argocd login localhost:8080
```

### Connection Refused Errors

If ArgoCD cluster registration fails:

```bash
# Ensure Minikube is running
minikube status

# Restart port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Login Issues

If user login fails, check server logs:

```bash
kubectl logs -f deployment/argocd-server -n argocd
```

Restore login capability if needed:

```bash
kubectl patch configmap argocd-cm -n argocd --type=merge \
  -p='{"data":{"accounts.readonly-user":"login","accounts.deploy-user":"login"}}'

kubectl rollout restart deployment argocd-server -n argocd
```

### Port Forwarding

Ensure the following ports are forwarded when needed:

- ArgoCD Server: `8080` → `443`
- Prometheus: `9090` → `9090`
- Grafana: `3000` → `3000`
- Alertmanager: `9093` → `9093`
- ArgoCD Metrics: `8082` → `8082`

---

## Repository Structure

```
.
├── argocd-metrics-services.yaml
├── prometheus-argocd-config.yaml
├── prometheus-alert-rules.yaml
├── alertmanager-config.yaml
├── grafana-config.yaml
├── restricted-project.yaml
├── argocd-cm-users.yaml
├── argocd-rbac-cm.yaml
├── test-application.yaml
├── canary-rollout.yaml
├── bluegreen-rollout.yaml
└── guestbook/
    ├── guestbook-ui-deployment.yaml
    └── guestbook-ui-svc.yaml
```

---

## Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Argo Rollouts Documentation](https://argoproj.github.io/argo-rollouts/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)

---

## License

This demo repository is for educational purposes.
