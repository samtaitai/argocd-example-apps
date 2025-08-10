# ArgoCD RBAC Implementation Guide

This guide demonstrates how to implement Role-Based Access Control (RBAC) in ArgoCD using:
- AppProjects with restricted access
- Local ArgoCD accounts
- Role-based permissions
- Access testing and verification

## Prerequisites

- ArgoCD installed and running
- kubectl access to the cluster
- ArgoCD CLI installed (optional but recommended)

## Step 1: Create a Restricted AppProject

Create an AppProject that restricts access to specific Git repositories and Kubernetes namespaces.

### 1.1 Create the AppProject

```bash
# Apply the restricted project configuration
kubectl apply -f restricted-project.yaml
```

This project will:
- Allow deployments only from `https://github.com/samtaitai/argocd-example-apps` repository
- Restrict deployments to `demo-namespace` only
- Allow only basic Kubernetes resources (no RBAC, Secrets, etc.)

### 1.2 Verify Project Creation

```bash
# Check if the project was created
kubectl get appproject -n argocd restricted-demo-project

# View project details
kubectl describe appproject -n argocd restricted-demo-project
```

## Step 2: Configure Local Users in argocd-cm

Create local ArgoCD users that can be assigned to specific roles.

### 2.1 Update argocd-cm ConfigMap

```bash
# Apply the user configuration
kubectl apply -f argocd-cm-users.yaml
```

This creates two local users:
- `readonly-user`: For read-only access
- `deploy-user`: For deployment operations

### 2.2 Set Initial Passwords

**Windows CMD (Two-step process):**
```cmd
REM Step 1: Generate hashes (copy the output from each command)
argocd account bcrypt --password readonly123
argocd account bcrypt --password deploy123

REM Step 2: Use the hash outputs in these commands (replace HASH_HERE with actual values)
kubectl -n argocd patch secret argocd-secret -p "{\"stringData\": {\"accounts.readonly-user.password\": \"$2a$10$MXSeDSfiW4dBQAldEzMaE.79qTWfmLEyZRSEckgygYrV6VhZySqVO\"}}"
kubectl -n argocd patch secret argocd-secret -p "{\"stringData\": {\"accounts.deploy-user.password\": \"$2a$10$Ngu.7KNUAoNL9FzFF9xI9.g1CmRDyQO.PAloVbvrJTM3w2PKjjCSm\"}}"
```
## Step 3: Define RBAC Roles in argocd-rbac-cm

Create role-based permissions that control what each user can do.

### 3.1 Apply RBAC Configuration

```bash
# Apply the RBAC configuration
kubectl apply -f argocd-rbac-cm.yaml
```

This configuration defines:
- **read-only-role**: Can only view applications and projects
- **deploy-only-role**: Can sync applications but cannot modify projects or delete applications

### 3.2 Verify RBAC Configuration

```bash
# Check the RBAC ConfigMap
kubectl get configmap argocd-rbac-cm -n argocd -o yaml

# Restart ArgoCD server to pick up changes
kubectl rollout restart deployment argocd-server -n argocd
```

## Step 4: Create a Test Application

Create an application in the restricted project for testing permissions.

### 4.1 Create Test Application

```bash
# Apply the test application
kubectl apply -f test-application.yaml
```

### 4.2 Verify Application Creation

```bash
# Check application status
kubectl get application -n argocd restricted-guestbook

# Access ArgoCD UI to see the application
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## Step 5: Test Read-Only User Access

Test the read-only user's permissions and limitations.

### 5.1 Login as Read-Only User

```bash
# Login via CLI
argocd login localhost:8080 --username readonly-user --password readonly123 --insecure

# Or via UI at https://localhost:8080
# Username: readonly-user
# Password: readonly123
```

### 5.2 Test Read Operations (Should Succeed)

```bash
# List applications (should work)
argocd app list

# Get application details (should work)
argocd app get restricted-guestbook

# View project details (should work)
argocd proj get restricted-demo-project
```

### 5.3 Test Write Operations (Should Fail)

```bash
# Try to sync application (should fail)
argocd app sync restricted-guestbook
# Expected: "permission denied" error

# Try to delete application (should fail)
argocd app delete restricted-guestbook
# Expected: "permission denied" error
```

## Step 6: Test Deploy-Only User Access

Test the deploy-only user's permissions.

### 6.1 Login as Deploy-Only User

```bash
# Logout previous user and login as deploy user
argocd logout

argocd login localhost:8080 --username deploy-user --password deploy123 --insecure
```

### 6.2 Test Deployment Operations (Should Succeed)

```bash
# Sync application (should work)
argocd app sync restricted-guestbook

# Wait for sync to complete
argocd app wait restricted-guestbook
```

### 6.3 Test Restricted Operations (Should Fail)

```bash
# Try to delete application (should fail)
argocd app delete restricted-guestbook
# Expected: "permission denied" error

# Try to modify project settings (should fail)
argocd proj role create restricted-demo-project deploy-role
# Expected: "permission denied" error
```

## Step 7: Monitor Access Attempts and Denials

Track access attempts and view audit logs.

### 7.1 Check ArgoCD Server Logs

```powershell
# View ArgoCD server logs for access attempts (Windows PowerShell)
kubectl logs -f deployment/argocd-server -n argocd | Select-String -Pattern "permission denied|unauthorized|403"

# View audit logs (if enabled)
kubectl logs -f deployment/argocd-server -n argocd | Select-String -Pattern "audit"
```

### 7.2 Check Application Event Logs

```bash
# View application events
kubectl describe application restricted-guestbook -n argocd

# Check for sync events and access attempts
argocd app get restricted-guestbook --show-operation
```

## Step 8: Verify Access Denied Messages

Document the expected error messages and UI behaviors.

### 8.1 CLI Error Messages

When testing with restricted users, you should see messages like:

```bash
# Read-only user trying to sync:
FATA[0000] rpc error: code = PermissionDenied desc = permission denied: applications, sync, restricted-demo-project/restricted-guestbook, sub: readonly-user, iat: 1691234567

# Deploy-only user trying to delete:
FATA[0000] rpc error: code = PermissionDenied desc = permission denied: applications, delete, restricted-demo-project/restricted-guestbook, sub: deploy-user, iat: 1691234567
```

### 8.2 UI Error Messages

In the ArgoCD UI, restricted actions will show:
- Disabled/grayed out buttons
- "Insufficient permissions" tooltips
- HTTP 403 error messages in browser console
- Notification banners with permission denied messages

## Step 10: Cleanup and Reset

Clean up the test environment when done.

### 10.1 Remove Test Resources

```bash
# Delete test application
kubectl delete application restricted-guestbook -n argocd

# Delete test project
kubectl delete appproject restricted-demo-project -n argocd

# Remove RBAC configuration (restore original)
kubectl delete configmap argocd-rbac-cm -n argocd

# Remove user configuration (restore original)
kubectl patch configmap argocd-cm -n argocd --type=merge -p="{\"data\":{\"accounts.readonly-user\":null,\"accounts.deploy-user\":null}}"

# Restart ArgoCD server
kubectl rollout restart deployment argocd-server -n argocd
```

### 10.2 Verify Cleanup

```bash
# Verify resources are removed
kubectl get appproject -n argocd
kubectl get application -n argocd
argocd account list
```
