# ArgoCD RBAC ImplementatThis project will:
- Allow deployments only from `https://github.com/samtaitai/argocd-example-apps` repository
- Restrict deployments to `default` namespace only
- Allow only basic Kubernetes resources (no RBAC, CRDs,Guide

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

```bash
# Set password for readonly-user
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "accounts.readonly-user.password": "'$(argocd account bcrypt --password readonly123)'"
  }}'

# Set password for deploy-user
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "accounts.deploy-user.password": "'$(argocd account bcrypt --password deploy123)'"
  }}'
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

# Try to modify project (should fail)
argocd proj role create restricted-demo-project test-role
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

# View application status (should work)
argocd app get restricted-guestbook

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

# Try to create new application outside project (should fail)
argocd app create test-app --repo https://github.com/other/repo.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default
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

### 7.3 Monitor RBAC Policy Violations

```powershell
# Check for RBAC violations in server logs (Windows PowerShell)
kubectl logs deployment/argocd-server -n argocd --tail=100 | Select-String -Pattern "rbac" -CaseSensitive:$false

# Monitor real-time access denials
kubectl logs -f deployment/argocd-server -n argocd | Select-String -Pattern "denied|forbidden|unauthorized"
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

### 8.3 Application Event Logs

Check application events for access attempts:

```powershell
# View recent application events (Windows PowerShell)
kubectl get events -n argocd --field-selector involvedObject.name=restricted-guestbook --sort-by='.lastTimestamp'

# Check application status for permission-related messages
argocd app get restricted-guestbook --show-params
```

## Step 9: Testing Edge Cases and Validation

Verify the security boundaries are properly enforced.

### 9.1 Test Cross-Project Access

```bash
# Try to access applications in different projects (should fail)
argocd app create test-default --repo https://github.com/samtaitai/argocd-example-apps.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default --project default

# Try to sync applications in default project (should fail for restricted users)
argocd app sync some-default-project-app
```

### 9.2 Test Repository Restrictions

```bash
# Try to create app from unauthorized repository (should fail)
argocd app create unauthorized-app --repo https://github.com/unauthorized/repo.git --path app --dest-server https://kubernetes.default.svc --dest-namespace demo-namespace --project restricted-demo-project
```

### 9.3 Test Namespace Restrictions

```bash
# Try to deploy to unauthorized namespace (should fail)
argocd app create restricted-app-wrong-ns --repo https://github.com/samtaitai/argocd-example-apps.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace monitoring --project restricted-demo-project
```

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
kubectl patch configmap argocd-cm -n argocd --type='json' -p='[{"op": "remove", "path": "/data/accounts.readonly-user"}, {"op": "remove", "path": "/data/accounts.deploy-user"}]'

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

## Expected Results Summary

| User Type | Can View Apps | Can Sync Apps | Can Delete Apps | Can Modify Project |
|-----------|---------------|---------------|-----------------|-------------------|
| readonly-user | ✅ | ❌ | ❌ | ❌ |
| deploy-user | ✅ | ✅ | ❌ | ❌ |
| admin | ✅ | ✅ | ✅ | ✅ |

## Troubleshooting

### Windows PowerShell Specific Commands

For Windows users, here are the PowerShell equivalents of common monitoring commands:

```powershell
# Monitor ArgoCD server logs for permission denials
kubectl logs -f deployment/argocd-server -n argocd | Select-String -Pattern "permission|denied|forbidden" -CaseSensitive:$false

# Get last 50 log lines and filter for RBAC issues
kubectl logs deployment/argocd-server -n argocd --tail=50 | Select-String -Pattern "rbac|permission" -CaseSensitive:$false

# Check if ArgoCD server is running
kubectl get pods -n argocd | Select-String "argocd-server"

# View application status
kubectl get applications -n argocd
```

### Common Issues

1. **RBAC not taking effect**: Restart argocd-server deployment
2. **User login fails**: Check password was set correctly in argocd-secret
3. **Permissions too restrictive**: Verify RBAC policies syntax
4. **UI not showing restrictions**: Clear browser cache and re-login

### Validation Commands

```bash
# Test RBAC policy syntax
argocd admin settings rbac validate --policy-file argocd-rbac-cm.yaml

# Test specific permissions
argocd admin settings rbac can readonly-user get applications "restricted-demo-project/restricted-guestbook"
argocd admin settings rbac can deploy-user sync applications "restricted-demo-project/restricted-guestbook"
argocd admin settings rbac can deploy-user delete applications "restricted-demo-project/restricted-guestbook"
```

This guide provides a complete implementation of RBAC in ArgoCD with practical testing scenarios and expected outcomes.
