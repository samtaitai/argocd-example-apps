# Enable Argo CD Metrics Endpoint - Step by Step Guide

*Note: The Argo CD Application Controller already has metrics enabled by default on port 8082, so no configuration changes are needed to enable the metrics endpoint.*

## Prerequisites: Verify Argo CD is Running

Before proceeding, ensure Argo CD is properly installed and running:

```cmd
REM Check if Argo CD namespace exists
kubectl get namespace argocd

REM Check if Argo CD pods are running
kubectl get pods -n argocd

REM Specifically check Application Controller
kubectl get pods -n argocd | findstr application-controller
```

The Application Controller pod should be in "Running" status. If not, you need to install or fix your Argo CD installation first.

## Step 1: Create the Metrics Service

Create a service to expose the metrics endpoint:

### For Windows Command Prompt:
```cmd
kubectl apply -f argocd-metrics-services.yaml
```
*Note: This file contains the Application Controller metrics service (port 8082) with your required metrics*

## Step 2: Verify Metrics Endpoints

Test that the Application Controller metrics endpoint is working:

### For Windows Command Prompt:
```cmd
REM Port forward to test metrics (Application Controller - port 8082)
kubectl port-forward svc/argocd-metrics -n argocd 8082:8082

REM In another CMD window, test the metrics endpoint
curl http://localhost:8082/metrics
```

**Important**: Some metrics like `argocd_app_sync_total` and `argocd_app_health_status` only appear after:
- Applications are deployed and managed by Argo CD
- Sync operations have occurred
- Health checks have been performed

If you see `argocd_app_info` but not the other metrics, trigger a sync operation in Argo CD UI or run:
```cmd
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Then access Argo CD at `https://localhost:8080` and perform a sync operation on your application.

## Step 3: Install and Configure Prometheus

Install Prometheus with custom scrape configuration to collect your required Argo CD metrics:

```cmd
REM Create prometheus namespace
kubectl create namespace monitoring

REM Apply the custom Prometheus configuration
kubectl apply -f prometheus-argocd-config.yaml
```

*Note: Use the `prometheus-argocd-config.yaml` file which contains the complete Prometheus setup with Argo CD scrape configuration*

## Step 4: Verify Prometheus is Collecting Argo CD Metrics

### Access Prometheus UI:
```cmd
REM Port forward to Prometheus
kubectl port-forward svc/prometheus-service -n monitoring 9090:9090
```

Then open your browser to `http://localhost:9090`

### Verify the Required Metrics:

1. **Check argocd_app_info** (includes health status in labels):
   - In Prometheus UI, go to Graph tab
   - Query: `argocd_app_info`
   - This shows general information about applications including health_status and sync_status in labels

2. **Check argocd_app_sync_total**:
   - Query: `argocd_app_sync_total`
   - This shows the total number of sync operations by phase (Succeeded, Failed, etc.)

3. **Check argocd_cluster_api_resources**:
   - Query: `argocd_cluster_api_resources`
   - This shows the number of monitored Kubernetes API resources

### Command Line Verification:
```cmd
REM Test if Prometheus is scraping metrics
curl "http://localhost:9090/api/v1/query?query=argocd_app_info"
curl "http://localhost:9090/api/v1/query?query=argocd_app_sync_total"
curl "http://localhost:9090/api/v1/query?query=argocd_cluster_api_resources"
```

### Troubleshooting Commands:
```cmd
REM Check if Prometheus pods are running
kubectl get pods -n monitoring

REM Check Prometheus logs
kubectl logs -f deployment/prometheus -n monitoring

REM Verify Prometheus targets are up
REM Go to http://localhost:9090/targets in browser
```

## Step 5: Install Grafana and Connect to Prometheus

Install Grafana to visualize your Argo CD metrics:

### Install Grafana:
```cmd
REM Apply the Grafana configuration (uses monitoring namespace like Prometheus)
kubectl apply -f grafana-config.yaml
```

*Note: This installs Grafana in the same `monitoring` namespace as Prometheus for easier connectivity*

### Access Grafana:
```cmd
REM Port forward to Grafana
kubectl port-forward svc/grafana-service -n monitoring 3000:3000
```

Then open your browser to `http://localhost:3000`
- Username: `admin`
- Password: `admin123`

### Configure Prometheus Data Source in Grafana:

1. **Add Data Source**:
   - Go to Configuration > Data Sources > Add data source
   - Select "Prometheus"
   - URL: `http://prometheus-service:9090` (since both are in monitoring namespace)
   - Click "Save & Test"

2. **Create Dashboard**:
   - Go to "+" > Dashboard > Add new panel
   - Use these queries for your Argo CD metrics:
     - **Application Info**: `argocd_app_info`
     - **Sync Operations**: `argocd_app_sync_total`
     - **API Resources**: `argocd_cluster_api_resources`

### Verify Grafana Setup:
```cmd
REM Check Grafana pod is running
kubectl get pods -n monitoring | findstr grafana

REM Port forward to access Grafana UI
kubectl port-forward svc/grafana-service -n monitoring 3000:3000
```

## Step 6: Add Basic Alert Rules in Prometheus

Configure Prometheus to alert on Argo CD application health and sync status issues:

### Create Alert Rules:

```cmd
REM Apply the alert rules configuration
kubectl apply -f prometheus-alert-rules.yaml
```

### Update Prometheus Configuration:

Since we need to modify the existing Prometheus configuration to load alert rules, reapply the updated config:

```cmd
REM Reapply the updated Prometheus configuration (now includes rule loading)
kubectl apply -f prometheus-argocd-config.yaml
```

*Note: This updates the ConfigMap, but the running Prometheus pod won't automatically pick up the changes*

### Restart Prometheus to Load Alert Rules:

```cmd
REM Delete the existing Prometheus pod to force it to reload the new configuration
kubectl delete pod -l app=prometheus -n monitoring

REM Check that Prometheus pod is running again with new config
kubectl get pods -n monitoring | findstr prometheus
```

*Note: Kubernetes will automatically recreate the pod with the updated ConfigMap*

### Verify Alert Rules are Loaded:

```cmd
REM Port forward to Prometheus (if not already running)
kubectl port-forward svc/prometheus-service -n monitoring 9090:9090
```

Then open your browser to `http://localhost:9090` and:
1. Go to **Status > Rules** to see your alert rules
2. Go to **Alerts** to see active/pending alerts
3. Go to **Status > Targets** to verify Argo CD metrics are being scraped

## Step 7: Install Alertmanager and Configure Default Receiver

Install Alertmanager to handle alert notifications from Prometheus:

### Create Alertmanager Configuration:

```cmd
REM Apply the Alertmanager configuration with dummy receiver
kubectl apply -f alertmanager-config.yaml
```

*Note: This uses the `alertmanager-config.yaml` file which contains the complete Alertmanager setup with ConfigMap, Deployment, and Service*

### Update Prometheus to Send Alerts to Alertmanager:

Update the Prometheus configuration to include Alertmanager integration:

```cmd
REM Apply the updated Prometheus configuration (now with Alertmanager integration)
kubectl apply -f prometheus-argocd-config.yaml
```

*Note: The `prometheus-argocd-config.yaml` file now includes Alertmanager integration in the `alerting` section*

### Restart Prometheus to Connect to Alertmanager:

```cmd
REM Restart Prometheus to load the updated configuration
kubectl delete pod -l app=prometheus -n monitoring

REM Check both Prometheus and Alertmanager are running
kubectl get pods -n monitoring | findstr "prometheus\|alertmanager"
```

### Access Alertmanager UI:

```cmd
REM Port forward to Alertmanager
kubectl port-forward svc/alertmanager-service -n monitoring 9093:9093
```

Then open your browser to `http://localhost:9093`

### Verify Alertmanager Setup:

1. **Check Alertmanager Status**:
   - Go to `http://localhost:9093` to see Alertmanager UI
   - Check **Status** page to see configuration
   - Check **Alerts** page to see any active alerts

2. **Verify Prometheus-Alertmanager Connection**:
   - Go to Prometheus at `http://localhost:9090`
   - Navigate to **Status > Runtime & Build Information**
   - Check **Status > Targets** to see if Alertmanager is listed

### Default Configuration Details:

- **Dummy Receiver**: Uses a webhook to `localhost:5001` (non-existent endpoint)
- **Grouping**: Groups alerts by alertname to avoid spam
- **Timing**: 10s group wait, 1h repeat interval
- **No Real Notifications**: Alerts are received but not sent anywhere (safe for testing)

**Note**: This dummy configuration receives and processes alerts but doesn't send real notifications. To enable real notifications, you would replace the webhook receiver with email, Slack, or other notification methods.

## Step 8: Trigger Alerts by Creating Sync Drift and Health Issues

Test the complete alerting pipeline by creating realistic scenarios that trigger alerts:

### Scenario 1: Create Sync Drift by Manual Deployment Changes

Manually edit a live Kubernetes deployment to cause sync drift between Git and cluster state:

```cmd
REM First, ensure you have a guestbook application deployed via Argo CD
REM If not deployed, you can deploy it first:
REM kubectl apply -f guestbook/

REM Edit the live deployment to change replica count from 1 to 3
kubectl scale deployment guestbook-ui --replicas=3 -n default

REM Verify the change was applied
kubectl get deployment guestbook-ui -n default
```

This creates a sync drift because:
- **Git state**: `replicas: 1` (in your Git repository)
- **Cluster state**: `replicas: 3` (after manual change)
- **Argo CD detects**: `sync_status != "Synced"`

### Scenario 2: Simulate Degraded Health by Deleting Resources

Delete a required resource to simulate application health degradation:

```cmd
REM Delete the guestbook service to break the application
kubectl delete service guestbook-ui -n default

REM Check that the service is gone
kubectl get services -n default | findstr guestbook
```

This creates health issues because:
- **Missing Service**: Application cannot be accessed
- **Argo CD detects**: `health_status != "Healthy"`

### Wait for Alert Conditions:

```cmd
REM Wait ~1 minute for ArgoCDAppOutOfSync alert (sync drift)
REM Wait ~1 minute for ArgoCDAppUnhealthy alert (missing service)
REM Use this time to check Prometheus and Alertmanager
```

## Step 9: Verify Alerts in Prometheus UI and Alertmanager

Monitor the alert progression through both Prometheus and Alertmanager:

### Check Alerts in Prometheus UI:

```cmd
REM Access Prometheus (if not already port-forwarded)
kubectl port-forward svc/prometheus-service -n monitoring 9090:9090
```

**Prometheus Alert Verification**:
1. Go to `http://localhost:9090`
2. Navigate to **Status > Rules** to see loaded alert rules
3. Go to **Alerts** tab to see firing alerts:
   - **ArgoCDAppOutOfSync**: Should show "FIRING" (red) due to sync drift
   - **ArgoCDAppUnhealthy**: Should show "FIRING" (red) due to missing service
4. Click on each alert to see details like:
   - Alert labels (app name, namespace, etc.)
   - Alert annotations (description, summary)
   - Alert state (Pending → Firing)

### Check Alerts in Alertmanager UI:

```cmd
REM Access Alertmanager (if not already port-forwarded)
kubectl port-forward svc/alertmanager-service -n monitoring 9093:9093
```

**Alertmanager Alert Verification**:
1. Go to `http://localhost:9093`
2. Check **Alerts** page to see received alerts from Prometheus:
   - Should show the same alerts that are firing in Prometheus
   - Shows grouping (alerts grouped by alertname)
   - Shows silencing options
3. Go to **Status** page to verify:
   - Configuration is loaded correctly
   - Prometheus is listed as an alert source

### Verify Alert Flow:

1. **Prometheus**: 
   - Evaluates alert rules every 15s
   - Detects conditions and fires alerts
   - Sends alerts to Alertmanager

2. **Alertmanager**:
   - Receives alerts from Prometheus
   - Groups alerts by alertname
   - Attempts to send to webhook (fails safely to localhost:5001)
   - Shows alerts in UI

### Restore Normal State (Clean Up):

After verifying alerts, restore the applications to normal state:

```cmd
REM Restore the deleted service
kubectl apply -f guestbook/guestbook-ui-svc.yaml

REM Restore sync by triggering Argo CD sync to revert the manual changes
kubectl port-forward svc/argocd-server -n argocd 8080:443
REM Then access https://localhost:8080 and click "Sync" on the guestbook app
REM This will revert the replicas back to 1 (the Git state)

REM Alternative: Use Argo CD CLI if available
REM argocd app sync guestbook
```

### Expected Alert Resolution:

After restoring normal state:
1. **Prometheus Alerts**: Should transition from "FIRING" → "RESOLVED" 
2. **Alertmanager**: Should show resolved alerts (if configured with `send_resolved: true`)
3. **Timeline**: Allow ~1 minute for alert conditions to clear

This completes the full alerting pipeline test: **Application Issues → Metrics → Prometheus Rules → Alerts → Alertmanager → (Notifications)**


