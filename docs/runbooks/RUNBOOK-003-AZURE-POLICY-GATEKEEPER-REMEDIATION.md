# RUNBOOK-003 — Azure Policy / Gatekeeper Read-Only Root Filesystem Remediation

## 1. Purpose

This runbook documents the investigation and remediation of a Kubernetes security-policy violation detected by Azure Policy through Gatekeeper.

The objective was to ensure the `demo-app` workload runs with a read-only container root filesystem while retaining a writable `/tmp` location for application runtime needs.

This runbook also records an important platform lesson: the AKS cluster already had an Azure-managed Gatekeeper deployment because Azure Policy for Kubernetes was enabled. A second upstream Gatekeeper installation was therefore unnecessary and was removed after ownership was verified.

---

## 2. Environment

| Item | Value |
|---|---|
| Kubernetes platform | Azure Kubernetes Service (AKS) |
| Policy engine integration | Azure Policy for Kubernetes + Gatekeeper |
| Application | `demo-app` |
| Application namespace | `demo-app` |
| GitOps engine | Argo CD |
| GitOps repository | `Davintus/azure-devsecops-platform-gitops` |
| Policy | `K8sAzureV3ReadOnlyRootFilesystem` |
| Enforcement mode | `dryrun` |
| Remediation commit | `0b018fa14e06428a2bd8fde43d93946191b10d1d` |

The policy constraint is installed and managed by the Azure Policy addon and targets Kubernetes Pods. The captured policy status shows `enforcementAction: dryrun`, so this exercise demonstrates detection and remediation rather than admission-time blocking.

---

## 3. Why Gatekeeper Was Already Present

Before installing any additional policy engine, the cluster was inspected.

Azure Policy was already enabled on the AKS cluster and had provisioned Gatekeeper components under `gatekeeper-system`, including:

- `gatekeeper-controller`
- `gatekeeper-audit`
- Gatekeeper webhook configuration
- Azure Policy-managed ConstraintTemplates and Constraints

The existing Gatekeeper deployment was identified by Azure-specific ownership metadata and the Azure-managed image.

### Important lesson

Do **not** install upstream Gatekeeper independently when the AKS cluster is already using the Azure Policy-managed Gatekeeper integration unless there is a specific, validated requirement to do so.

Installing another Gatekeeper deployment can create duplicate controllers sharing the same CRDs and webhook infrastructure.

---

## 4. Duplicate Gatekeeper Installation Incident

An upstream Gatekeeper manifest was initially applied:

```powershell
kubectl apply `
  -f "https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.20.1/deploy/gatekeeper.yaml"
```

Inspection showed that this created an additional:

```text
gatekeeper-controller-manager
```

while the Azure-managed deployment already existed as:

```text
gatekeeper-controller
gatekeeper-audit
```

The Azure-managed components used the Azure Policy-managed Gatekeeper image and carried Azure management labels.

The shared webhook configuration and service were older, Azure-managed resources and were therefore retained.

### Corrective action

Only the accidentally installed upstream controller deployment was removed:

```powershell
kubectl delete deployment gatekeeper-controller-manager `
  --namespace gatekeeper-system
```

The following Azure-managed resources were deliberately retained:

- `gatekeeper-controller`
- `gatekeeper-audit`
- Gatekeeper CRDs
- Gatekeeper webhook configurations
- Gatekeeper webhook service
- Azure Policy components

### Result

After cleanup, the Gatekeeper namespace contained the expected Azure-managed audit and controller workloads, with both running successfully.

---

## 5. Policy Under Investigation

The affected policy was:

```text
K8sAzureV3ReadOnlyRootFilesystem
```

The corresponding Azure Policy definition reference was:

```text
ReadOnlyRootFileSystemInKubernetesCluster
```

The Constraint was identified as:

```text
constraint-installed-by: azure-policy-addon
managed-by: azure-policy-addon
```

The policy was operating in:

```text
enforcementAction: dryrun
```

This means the policy records violations without rejecting the workload during admission.

---

## 6. Initial Violation

The initial audit identified two `demo-app` Pods violating the read-only root filesystem requirement.

The captured Gatekeeper status reported:

```text
totalViolations: 2
```

The affected Pods were:

```text
demo-app-847466b75b-c7chb
demo-app-847466b75b-wqfpw
```

The violation message was:

```text
Readonly root filesystem is required for container.
```

This was a genuine workload configuration issue rather than a false positive.

---

## 7. Runtime Validation Before Remediation

Before changing the manifest, the container filesystem behaviour was tested.

### Test `/tmp`

```powershell
kubectl exec `
  --namespace demo-app `
  deploy/demo-app `
  -- `
  sh -c "touch /tmp/security-test && echo 'TMP_WRITE_OK' || echo 'TMP_WRITE_FAILED'"
```

The test returned:

```text
TMP_WRITE_OK
```

### Test application filesystem

```powershell
kubectl exec `
  --namespace demo-app `
  deploy/demo-app `
  -- `
  sh -c "touch /app/security-test && echo 'APP_WRITE_OK' || echo 'APP_WRITE_FAILED'"
```

Before remediation, this returned:

```text
APP_WRITE_OK
```

This confirmed that the application container's root filesystem was writable and therefore did not satisfy the intended security control.

---

## 8. GitOps Remediation

The remediation was performed in the GitOps repository rather than by manually editing the live Deployment.

The Deployment security context was changed to:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
```

Because the application may require temporary runtime writes, an `emptyDir` volume was mounted at `/tmp`:

```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp
```

with:

```yaml
volumes:
  - name: tmp
    emptyDir: {}
```

This creates a writable temporary filesystem without making the application container's root filesystem writable.

---

## 9. Manifest Validation

The corrected manifest was rendered and validated before deployment.

The intended rendered Deployment contains:

```text
readOnlyRootFilesystem: true
runAsNonRoot: true
allowPrivilegeEscalation: false
capabilities.drop: ALL
volumeMounts: /tmp
volumes: emptyDir
```

The corrected GitOps revision was:

```text
0b018fa14e06428a2bd8fde43d93946191b10d1d
```

A previous GitOps revision contained an incorrect `volumeMounts` placement. That manifest was rejected during server-side validation and subsequently corrected. This is useful evidence that manifest validation should happen before allowing a GitOps change to become the desired state.

---

## 10. Argo CD Deployment

Argo CD automatically reconciled the GitOps repository.

Validation showed:

```text
demo-app-dev   Synced   Healthy
```

The application was running from GitOps revision:

```text
0b018fa14e06428a2bd8fde43d93946191b10d1d
```

Argo CD was configured with:

```text
Automated: true
Prune: true
Self Heal: true
```

Therefore the remediation followed the intended GitOps flow:

```text
Git change
   ↓
GitHub
   ↓
Argo CD detects desired-state change
   ↓
AKS Deployment updated
   ↓
New Pods created
   ↓
Azure Policy / Gatekeeper audits workload
```

---

## 11. Post-Remediation Kubernetes Configuration

The live Deployment reported:

```json
{
  "allowPrivilegeEscalation": false,
  "capabilities": {
    "drop": [
      "ALL"
    ]
  },
  "readOnlyRootFilesystem": true,
  "runAsNonRoot": true
}
```

The `/tmp` volume was:

```json
[
  {
    "emptyDir": {},
    "name": "tmp"
  }
]
```

The container mount was:

```json
[
  {
    "mountPath": "/tmp",
    "name": "tmp"
  }
]
```

The replacement Pods were both healthy and running.

---

## 12. Runtime Security Validation After Remediation

### `/tmp` remains writable

```powershell
kubectl exec `
  --namespace demo-app `
  deploy/demo-app `
  -- `
  sh -c "touch /tmp/security-test && echo 'TMP_WRITE_OK' || echo 'TMP_WRITE_FAILED'"
```

Result:

```text
TMP_WRITE_OK
```

### Application filesystem is read-only

```powershell
kubectl exec `
  --namespace demo-app `
  deploy/demo-app `
  -- `
  sh -c "touch /app/security-test && echo 'APP_WRITE_OK' || echo 'APP_WRITE_FAILED'"
```

Result:

```text
APP_WRITE_FAILED
touch: cannot touch '/app/security-test': Read-only file system
```

This demonstrates the intended security boundary:

```text
Container root filesystem
        │
        └── READ ONLY

/tmp
 │
 └── writable emptyDir volume
```

The temporary test file was then removed:

```powershell
kubectl exec `
  --namespace demo-app `
  deploy/demo-app `
  -- `
  sh -c "rm -f /tmp/security-test"
```

---

## 13. Final Policy Validation

The Azure Policy-managed Gatekeeper Constraint was checked again after the GitOps deployment.

Final audit result:

```text
totalViolations: 0
```

The Constraint continued to operate in:

```text
enforcementAction: dryrun
```

Therefore the result should be interpreted as:

> The `demo-app` workload no longer violates the specific `ReadOnlyRootFileSystemInKubernetesCluster` policy during audit.

It should **not** be interpreted as:

> Every Azure Policy constraint in the cluster has zero violations.

Other Azure Policy constraints had separate findings, including resource-limit findings affecting workloads such as Argo CD.

---

## 14. Note on `vap.k8s.io`

The captured policy status also contained:

```text
enforcementPoint: vap.k8s.io
message: K8sNativeValidation engine is missing
state: error
```

This status was observed alongside active Gatekeeper audit/controller processing.

It should not be described as evidence that the Gatekeeper audit itself failed. The same Constraint reported active audit/controller processing and ultimately:

```text
totalViolations: 0
```

No change was made to the cluster based solely on this separate enforcement-point status.

---

## 15. Security Significance

The remediation improves container isolation by preventing the application process from modifying the container image filesystem at runtime.

The configuration also preserves a controlled writable location for temporary application data.

The resulting security controls are:

| Control | Configuration |
|---|---|
| Root filesystem | Read-only |
| Temporary storage | `/tmp` via `emptyDir` |
| Privilege escalation | Disabled |
| Linux capabilities | All dropped |
| User execution | Non-root |
| Policy detection | Azure Policy + Gatekeeper |
| Deployment control | GitOps / Argo CD |
| Policy mode | `dryrun` |

This provides defence in depth across the container image, Kubernetes workload specification, GitOps reconciliation, and policy audit layers.

---

## 16. Lessons Learned

### 16.1 Check platform-managed components first

Before installing Kubernetes security tooling manually, determine whether AKS already provides the capability through an Azure-managed integration.

### 16.2 Avoid duplicate controllers

Two Gatekeeper controllers sharing the same CRDs and webhook resources can introduce unnecessary complexity and unpredictable ownership.

### 16.3 Validate manifests before Git push

The first remediation attempt contained an indentation/structure error. Server-side validation identified the issue before Argo CD deployed it.

Recommended validation:

```powershell
kubectl kustomize .\apps\demo-app\overlays\dev
```

followed by:

```powershell
kubectl apply `
  --dry-run=server `
  -k .\apps\demo-app\overlays\dev
```

### 16.4 Runtime tests are valuable

Policy compliance should be supported by runtime evidence where practical.

In this case:

```text
/tmp      → writable
/app      → read-only
```

provided direct evidence that the intended control was actually effective.

### 16.5 Remediate through GitOps

The desired state was changed in Git and reconciled by Argo CD rather than manually modifying the live Deployment.

This keeps the security configuration reproducible and auditable.

---

## 17. Verification Checklist

Use the following checklist after future changes:

```text
[ ] Confirm Azure Policy/Gatekeeper ownership
[ ] Confirm no duplicate Gatekeeper installation exists
[ ] Confirm ConstraintTemplate is present
[ ] Confirm Constraint is present
[ ] Check current violation count
[ ] Validate Kustomize output
[ ] Run server-side dry run
[ ] Commit and push GitOps change
[ ] Confirm Argo CD is Synced
[ ] Confirm Argo CD is Healthy
[ ] Confirm live securityContext
[ ] Confirm required writable volumes
[ ] Perform runtime filesystem test
[ ] Re-check policy violation count
[ ] Record evidence in the repository
```

---

## 18. Useful PowerShell Commands

### Check Gatekeeper workloads

```powershell
kubectl get pods `
  --namespace gatekeeper-system
```

### Check Azure Policy components

```powershell
kubectl get pods `
  --namespace kube-system `
  -l app=azure-policy
```

### Check the Constraint

```powershell
kubectl get k8sazurev3readonlyrootfilesystem
```

### Inspect the Constraint

```powershell
kubectl get k8sazurev3readonlyrootfilesystem `
  azurepolicy-k8sazurev3readonlyrootfilesyst-402f3adf28eaa2cbe45f `
  -o yaml
```

### Check Argo CD

```powershell
kubectl get application demo-app-dev `
  --namespace argocd
```

### Check Deployment security context

```powershell
kubectl get deployment demo-app `
  --namespace demo-app `
  -o jsonpath="{.spec.template.spec.containers[0].securityContext}"
```

### Check volumes

```powershell
kubectl get deployment demo-app `
  --namespace demo-app `
  -o jsonpath="{.spec.template.spec.volumes}"
```

### Check volume mounts

```powershell
kubectl get deployment demo-app `
  --namespace demo-app `
  -o jsonpath="{.spec.template.spec.containers[0].volumeMounts}"
```

---

## 19. Evidence Summary

The final validation demonstrated:

```text
GitOps revision:
0b018fa14e06428a2bd8fde43d93946191b10d1d

Argo CD:
Synced / Healthy

Root filesystem:
Read-only

/tmp:
Writable

/app:
Read-only

Azure Policy ReadOnlyRootFilesystem:
0 violations
```

This completes the remediation evidence for the `demo-app` ReadOnlyRootFilesystem policy.

---

## 20. Portfolio Value

This incident is useful portfolio evidence because it demonstrates more than simply installing Gatekeeper.

It shows:

1. Platform discovery before deploying security tooling.
2. Identification of Azure-managed Kubernetes policy components.
3. Safe removal of an accidental duplicate controller.
4. Investigation of a real policy violation.
5. Secure Kubernetes workload design.
6. GitOps-based remediation.
7. Argo CD reconciliation.
8. Runtime security validation.
9. Policy post-remediation validation.
10. Documentation of an operational lesson learned.

The important architectural principle is:

```text
Azure Policy
     │
     ▼
Gatekeeper
     │
     ▼
Policy audit
     │
     ▼
GitOps remediation
     │
     ▼
Argo CD
     │
     ▼
AKS workload
     │
     ▼
Runtime validation
```
