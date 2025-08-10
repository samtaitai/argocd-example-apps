# Argo Rollouts Deployment Strategy Guide

## Prerequisites

- Minikube running and configured
- kubectl installed and configured
- Docker installed
- Git access to this repository

## Step 1: Install Argo Rollouts in Minikube

Install Argo Rollouts controller, CLI plugin, and dashboard in your Kubernetes cluster.

### 1.1 Create Argo Rollouts Namespace

```bash
# Create namespace for Argo Rollouts
kubectl create namespace argo-rollouts
```

### 1.2 Install Argo Rollouts Controller

```bash
# Install the latest Argo Rollouts manifests
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Verify installation - wait for pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argo-rollouts -n argo-rollouts --timeout=300s

# Check controller status
kubectl get pods -n argo-rollouts
```

### 1.3 Install Argo Rollouts CLI Plugin

The `kubectl argo rollouts` plugin provides enhanced commands for managing rollouts.

```powershell
# Download and install the kubectl argo rollouts plugin for Windows
# Get the latest release URL
$latestRelease = Invoke-RestMethod -Uri "https://api.github.com/repos/argoproj/argo-rollouts/releases/latest"
$windowsAsset = $latestRelease.assets | Where-Object { $_.name -like "*windows-amd64*" }
$downloadUrl = $windowsAsset.browser_download_url

# Download the plugin
Invoke-WebRequest -Uri $downloadUrl -OutFile "kubectl-argo-rollouts.exe"

# Check where kubectl is installed
$kubectlPath = (Get-Command kubectl).Source
Write-Host "kubectl is located at: $kubectlPath"

# Option 1: Place in same directory as kubectl (recommended)
$kubectlDir = Split-Path $kubectlPath -Parent
Move-Item "kubectl-argo-rollouts.exe" "$kubectlDir\kubectl-argo-rollouts.exe"
Write-Host "Installed kubectl-argo-rollouts.exe to: $kubectlDir"

# Option 2: Alternative - create a local plugins directory if above fails
# New-Item -ItemType Directory -Path "$env:USERPROFILE\.kubectl\plugins" -Force
# Move-Item "kubectl-argo-rollouts.exe" "$env:USERPROFILE\.kubectl\plugins\kubectl-argo-rollouts.exe"
# $env:PATH += ";$env:USERPROFILE\.kubectl\plugins"

# Verify installation
kubectl argo rollouts version

# Expected output should show both client and server versions
```

## Step 2: Deploy Guestbook Application Using Canary Strategy

Use the existing guestbook application to demonstrate canary deployment strategy with traffic shifting.

### 2.1 Create Canary Application Manifests

First, let's create the namespace:

```bash
# Create application namespace
kubectl create namespace canary-demo
```

The canary rollout configuration is provided in `canary-rollout.yaml` which converts your existing guestbook application to use:
- Rollout resource with canary strategy (20% → 40% → 60% → 80% → 100% traffic progression)
- 10-second pauses between each step (fast for demos) - **The rollout will pause automatically at each step**
- Your familiar guestbook UI (`gcr.io/google-samples/gb-frontend:v5`)

**Important**: The canary deployment will pause automatically at each step for 10 seconds. You'll see "Rollout is paused (CanaryPauseStep)" in the events - this is normal and expected behavior.

### 2.2 Deploy the Canary Application

```bash
# Apply the canary rollout
kubectl apply -f canary-rollout.yaml

# Verify deployment
kubectl get rollout -n canary-demo
kubectl get pods -n canary-demo
```

### 2.3 Access the Guestbook Application

```bash
# Port forward to access the guestbook
kubectl port-forward svc/guestbook-ui-service -n canary-demo 8080:80

# If you see the wrong deployment, clean up (powershell)
Get-Process | Where-Object {$_.ProcessName -eq "kubectl" -and $_.CommandLine -like "*port-forward*"} | Stop-Process -Force

# In another terminal, test the guestbook application
Invoke-RestMethod http://localhost:8080
# Or open in browser: http://localhost:8080
```

## Step 3: Demonstrate Canary Traffic Shifting

Trigger an update and observe the canary deployment process with traffic shifting.

### 3.1 Monitor Using Argo Rollouts CLI

```bash
# Watch rollout progress in real-time using Argo Rollouts CLI
kubectl argo rollouts get rollout guestbook-canary -n canary-demo --watch

# Check detailed rollout status
kubectl argo rollouts status guestbook-canary -n canary-demo

# List all rollouts
kubectl argo rollouts list rollouts -n canary-demo

# Monitor replicasets and pods (traditional kubectl)
kubectl get replicasets,pods -n canary-demo -l app=guestbook-ui --watch
```

### 3.2 Trigger a Canary Update

```bash
# Update to a different version to trigger a rollout
# Create a patch file for reliable JSON handling in PowerShell
@'
{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "guestbook-ui",
            "image": "nginx:1.15.4"
          }
        ]
      }
    }
  }
}
'@ | Out-File -FilePath patch-canary-hello.json -Encoding utf8

# Apply the patch
kubectl patch rollout guestbook-canary -n canary-demo --type merge --patch-file patch-canary-hello.json

# Clean up the patch file
Remove-Item patch-canary-hello.json

# IMPORTANT: Fix service port configuration for nginx compatibility
# When switching from guestbook to nginx, fix the service to use direct port numbers
@'
{
  "spec": {
    "ports": [
      {
        "port": 80,
        "targetPort": 80,
        "protocol": "TCP",
        "name": "http"
      }
    ]
  }
}
'@ | Out-File -FilePath patch-service-port.json -Encoding utf8

kubectl patch service guestbook-ui-service -n canary-demo --type merge --patch-file patch-service-port.json
Remove-Item patch-service-port.json

# Watch the rollout progress using Argo Rollouts CLI
kubectl argo rollouts get rollout guestbook-canary -n canary-demo --watch

# Pause the rollout manually using Argo Rollouts CLI (too fast)
kubectl argo rollouts pause guestbook-canary -n canary-demo

# Resume the rollout when ready using Argo Rollouts CLI
kubectl argo rollouts promote guestbook-canary -n canary-demo

# Check replica distribution
kubectl get replicasets -n canary-demo -l app=guestbook-ui

# Alternative: Check status periodically
# output: healthy at the end 
kubectl argo rollouts status guestbook-canary -n canary-demo

# Get structured status output using traditional kubectl (powershell)
kubectl get rollout guestbook-canary -n canary-demo -o jsonpath='{.status.currentStepIndex}/{.spec.strategy.canary.steps[*]} - {.status.phase} - {.status.message}'

# Now test the nginx welcome page
kubectl port-forward svc/guestbook-ui-service -n canary-demo 8080:80
# You should now see nginx welcome page at http://localhost:8080
```

## Step 4: Deploy Second Guestbook Application Using Blue-Green Strategy

Use the guestbook application again to demonstrate the blue-green strategy.

### 4.1 Create Blue-Green Application Manifests

```bash
# Create namespace for blue-green demo
kubectl create namespace bluegreen-demo
```

The blue-green rollout configuration is provided in `bluegreen-rollout.yaml` which includes:
- A Rollout resource with blue-green strategy using the same guestbook application
- Two services: `guestbook-ui-active` (production) and `guestbook-ui-preview` (testing)
- Manual promotion (autoPromotionEnabled: false)

### 4.2 Deploy the Blue-Green Application

```bash
# Apply the blue-green rollout
kubectl apply -f bluegreen-rollout.yaml

# Verify deployment
kubectl get rollout -n bluegreen-demo
kubectl get services -n bluegreen-demo
kubectl get pods -n bluegreen-demo
```

### 4.3 Access Both Services

```bash
# Port forward active service (production traffic)
kubectl port-forward svc/guestbook-ui-active -n bluegreen-demo 8081:80

# Port forward preview service (for testing new version)
kubectl port-forward svc/guestbook-ui-preview -n bluegreen-demo 8082:80

# Test active service
Invoke-RestMethod http://localhost:8081
# Or open in browser: http://localhost:8081

# Test preview service
Invoke-RestMethod http://localhost:8082
# Or open in browser: http://localhost:8082
```

## Step 5: Demonstrate Blue-Green Preview and Promotion

Show how a preview version is deployed and then promoted to production.

### 5.1 Monitor Initial Blue-Green State

```bash
# Check current state
kubectl get rollout guestbook-bluegreen -n bluegreen-demo -o wide
```

### 5.2 Trigger Blue-Green Update
```bash
@'
{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "guestbook-ui",
            "image": "nginx:1.15.4"
          }
        ]
      }
    }
  }
}
'@ | Out-File -FilePath patch-bluegreen-hello.json -Encoding utf8

# Apply the patch
kubectl patch rollout guestbook-bluegreen -n bluegreen-demo --type merge --patch-file patch-bluegreen-hello.json

# Clean up the patch file
Remove-Item patch-bluegreen-hello.json

# Watch the rollout progress using Argo Rollouts CLI
kubectl argo rollouts get rollout guestbook-bluegreen -n bluegreen-demo --watch

# Check detailed status
# output: paused
kubectl argo rollouts status guestbook-bluegreen -n bluegreen-demo
```

### 5.5 Promote to Production Using Argo Rollouts CLI

```bash
# Promote the preview version to active using Argo Rollouts CLI (RECOMMENDED)
kubectl argo rollouts promote guestbook-bluegreen -n bluegreen-demo
```

### 5.6 Monitor Using Argo Rollouts CLI

Instead of using a dashboard, monitor the blue-green deployment with:

```bash
# Use Argo Rollouts CLI for enhanced monitoring
kubectl argo rollouts get rollout guestbook-bluegreen -n bluegreen-demo

# Check rollout status
# output healthy
kubectl argo rollouts status guestbook-bluegreen -n bluegreen-demo

# Monitor pod status and labels
kubectl get pods -n bluegreen-demo -o wide --show-labels

# Port forward active service (production traffic)
kubectl port-forward svc/guestbook-ui-active -n bluegreen-demo 8081:80
# and visit localhost:8081 should be nginx
```

## Step 6: Trigger Updates and Compare Strategies

Demonstrate how each strategy handles updates differently.

### 6.1 Canary Strategy Update Behavior

```bash
# Trigger another canary update using Argo Rollouts CLI
kubectl argo rollouts set image guestbook-canary guestbook-ui=nginx:1.16.1 -n canary-demo

# Observe gradual traffic shifting using Argo Rollouts CLI
kubectl argo rollouts get rollout guestbook-canary -n canary-demo --watch

# Monitor replicaset changes (traditional kubectl)
kubectl get replicasets -n canary-demo -l app=guestbook-ui --watch

# Key characteristics:
# - Gradual traffic increase (20% -> 40% -> 60% -> 80% -> 100%)
# - Both versions running simultaneously
# - Automatic progression with pauses
# - Can abort at any step using: kubectl argo rollouts abort guestbook-canary -n canary-demo
```

### 6.2 Blue-Green Strategy Update Behavior

```bash
# Trigger another blue-green update using Argo Rollouts CLI
kubectl argo rollouts set image guestbook-bluegreen guestbook-ui=nginx:1.16.1 -n bluegreen-demo

# Observe all-or-nothing switch using Argo Rollouts CLI
kubectl argo rollouts get rollout guestbook-bluegreen -n bluegreen-demo --watch

# Monitor service endpoints (traditional kubectl)
kubectl get endpoints -n bluegreen-demo --watch

# Promote the preview version to active using Argo Rollouts CLI (RECOMMENDED)
kubectl argo rollouts promote guestbook-bluegreen -n bluegreen-demo

# Key characteristics:
# - Preview deployment first (0% production traffic)
# - Manual promotion required (autoPromotionEnabled: false)
# - Instant traffic switch (0% -> 100%) using: kubectl argo rollouts promote guestbook-bluegreen -n bluegreen-demo
# - Old version available for rollback
```
## Step 7: Demonstrate Rollback Scenarios

Show how to handle failed deployments and rollbacks.

### 7.3 Manual Rollback

```bash
# Check rollout history - Argo Rollouts doesn't have history command
# Instead, use these commands to see rollout information:
kubectl argo rollouts get rollout guestbook-canary -n canary-demo
kubectl argo rollouts get rollout guestbook-bluegreen -n bluegreen-demo

# Rollback to previous version using Argo Rollouts CLI
kubectl argo rollouts undo guestbook-canary -n canary-demo
kubectl argo rollouts undo guestbook-bluegreen -n bluegreen-demo

# Rollback to specific revision using Argo Rollouts CLI
# If you're at revision 6 and want to go back to revision 1:
kubectl argo rollouts undo guestbook-canary -n canary-demo --to-revision=1
kubectl argo rollouts undo guestbook-bluegreen -n bluegreen-demo --to-revision=1

# You can also rollback to any specific revision (2, 3, 4, 5, etc.)
# kubectl argo rollouts undo guestbook-canary -n canary-demo --to-revision=3

# Note: Traditional kubectl rollout commands don't work with Argo Rollouts
# kubectl rollout history/undo only work with standard Deployments, not Rollout resources

# Verify rollback using Argo Rollouts CLI
kubectl argo rollouts get rollout guestbook-canary -n canary-demo
kubectl argo rollouts get rollout guestbook-bluegreen -n bluegreen-demo
```

## Step 9: Best Practices and Troubleshooting

### 9.1 Best Practices

**For Canary Deployments:**
- Start with small traffic percentages (5-10%)
- Include sufficient pause durations for observation
- Set up proper monitoring and analysis
- Define clear success/failure criteria
- Use `maxSurge` and `maxUnavailable` appropriately

**For Blue-Green Deployments:**
- Disable auto-promotion for critical applications
- Test preview thoroughly before promotion
- Set appropriate `scaleDownDelaySeconds`
- Monitor resource usage (double pods during transition)
- Have rollback procedures ready

### 9.2 Common Issues and Solutions

**Issue: Rollout stuck in progressing state**
```bash
# Check pod status
kubectl get pods -n <namespace> -l app=<app-name>

# Check events using Argo Rollouts CLI
kubectl argo rollouts get rollout <rollout-name> -n <namespace>

# Check replicaset status
kubectl get replicasets -n <namespace> -l app=<app-name>

# Possible solutions using Argo Rollouts CLI:
kubectl argo rollouts abort <rollout-name> -n <namespace>
kubectl argo rollouts restart <rollout-name> -n <namespace>

# Alternative: Traditional kubectl approach
kubectl patch rollout <rollout-name> -n <namespace> --type merge -p='{"status":{"abort":true}}'
kubectl rollout restart rollout/<rollout-name> -n <namespace>
```

**Issue: Services not updating selector**
```bash
# Check service endpoints
kubectl get endpoints -n <namespace>

# Verify service selectors match rollout labels
kubectl describe service <service-name> -n <namespace>

# Check rollout status and pod labels
kubectl get pods -n <namespace> --show-labels
```

**Issue: Port-forward error "Pod does not have a named port"**
```bash
# Error: Pod 'pod-name' does not have a named port 'http'
# This happens when switching between different container images

# Solution: Fix service to use direct port numbers instead of named ports
@'
{
  "spec": {
    "ports": [
      {
        "port": 80,
        "targetPort": 80,
        "protocol": "TCP",
        "name": "http"
      }
    ]
  }
}
'@ | Out-File -FilePath patch-service-port.json -Encoding utf8

kubectl patch service <service-name> -n <namespace> --type merge --patch-file patch-service-port.json
Remove-Item patch-service-port.json

# Alternative: Check pod port configuration
kubectl describe pod <pod-name> -n <namespace> | Select-String -Pattern "Port"
```

**Issue: Analysis failing**
```bash
# Check analysis run logs using Argo Rollouts CLI
kubectl argo rollouts get rollout <rollout-name> -n <namespace>

# Traditional kubectl for detailed analysis inspection
kubectl get analysisrun -n <namespace>
kubectl describe analysisrun <analysis-run-name> -n <namespace>

# Check if analysis template exists
kubectl get analysistemplate -n <namespace>
```

### 9.3 Monitoring and Observability

```bash
# View rollout metrics (if Prometheus is configured)
kubectl port-forward -n argo-rollouts svc/argo-rollouts-metrics 8090:8090

# Access metrics at: http://localhost:8090/metrics

# Key metrics to monitor:
# - rollout_phase
# - rollout_phase_duration
# - analysis_run_phase
# - analysis_run_metric_phase
```

## Step 10: Cleanup

Clean up the demo environment.

### 10.1 Remove Demo Applications

```bash
# Delete canary demo
kubectl delete namespace canary-demo

# Delete blue-green demo
kubectl delete namespace bluegreen-demo

# Verify cleanup
kubectl get namespaces | Select-String "demo"
```

### 10.2 Remove Argo Rollouts (Optional)

```bash
# Remove Argo Rollouts installation
kubectl delete -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Delete namespace
kubectl delete namespace argo-rollouts

# Remove kubectl argo rollouts plugin (if installed)
# Windows: Remove from PATH or delete file from your kubectl plugins directory
# Example: Remove-Item "$env:ProgramFiles\kubectl\kubectl-argo-rollouts.exe"
```

### 10.3 Verify Complete Cleanup

```bash
# Check for remaining resources using Argo Rollouts CLI
kubectl argo rollouts list rollouts --all-namespaces

# Traditional kubectl for additional resources
kubectl get rollouts --all-namespaces
kubectl get analysistemplates --all-namespaces
kubectl get analysisruns --all-namespaces
```

## Summary

This guide demonstrated:

1. **Installation** of Argo Rollouts controller and CLI plugin in Minikube
2. **Canary deployment** with gradual traffic shifting (20% → 40% → 60% → 80% → 100%) using `kubectl argo rollouts`
3. **Blue-Green deployment** with preview and manual promotion using `kubectl argo rollouts promote`
4. **CLI commands** for rollout management using both Argo Rollouts CLI and traditional kubectl
5. **Update triggers** and strategy comparisons
6. **Rollback scenarios** and failure handling using `kubectl argo rollouts abort` and `kubectl argo rollouts undo`
7. **Advanced features** like analysis templates
8. **Troubleshooting** common issues with both CLI approaches
9. **Best practices** for production use

### Key Argo Rollouts CLI Commands Summary

| Operation | Argo Rollouts CLI | Traditional kubectl |
|-----------|-------------------|-------------------|
| **Monitor Rollout** | `kubectl argo rollouts get rollout <name>` | `kubectl describe rollout <name>` |
| **Check Status** | `kubectl argo rollouts status <name>` | `kubectl get rollout <name>` |
| **Pause Rollout** | `kubectl argo rollouts pause <name>` | `kubectl patch rollout <name> --type merge -p='{"spec":{"paused":true}}'` |
| **Promote/Resume** | `kubectl argo rollouts promote <name>` | `kubectl patch rollout <name> --type merge -p='{"spec":{"paused":false}}'` |
| **Abort Rollout** | `kubectl argo rollouts abort <name>` | `kubectl patch rollout <name> --type merge -p='{"status":{"abort":true}}'` |
| **Update Image** | `kubectl argo rollouts set image <name> <container>=<image>` | `kubectl patch rollout <name> -p='{"spec":{"template":{"spec":{"containers":[{"name":"<container>","image":"<image>"}]}}}}'` |
| **Rollback** | `kubectl argo rollouts undo <name>` | Not available - use Argo Rollouts CLI |
| **List Rollouts** | `kubectl argo rollouts list rollouts` | `kubectl get rollouts` |
| **View History** | Use `kubectl argo rollouts get rollout <name>` | Not available - use Argo Rollouts CLI |

### Key Differences Summary

| Feature | Canary | Blue-Green |
|---------|--------|------------|
| **Traffic Distribution** | Gradual (20%→40%→60%→80%→100%) | All-or-nothing (0%→100%) |
| **Resource Usage** | Variable during rollout | Double during preview |
| **Risk Level** | Lower (gradual exposure) | Medium (instant switch) |
| **Rollback Speed** | Immediate abort possible | Instant switch back |
| **Complexity** | Higher (traffic management) | Lower (simple switch) |
| **Testing** | Live traffic testing | Full preview testing |
| **Best For** | High-traffic applications | Critical applications |

Choose the strategy based on your application requirements, risk tolerance, and infrastructure capabilities.
