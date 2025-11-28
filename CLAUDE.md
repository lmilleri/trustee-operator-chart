# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the **legacy monolithic Helm chart** for deploying the Trustee Operator on OpenShift/Kubernetes. It combines operator installation and TrusteeConfig CR deployment in a single chart.

**Note:** For new deployments, prefer the modular charts in `../charts/` (trustee-operator, trustee-config, and trustee umbrella chart). See [../CLAUDE.md](../CLAUDE.md) for details on the new architecture.

This chart is maintained for backward compatibility and as a reference implementation.

## Chart Architecture

This chart deploys all resources in a single Helm release using Helm hooks to control deployment order:

### Core Resources (deployed immediately)
1. **Namespace** (`templates/namespace.yaml`) - Creates `trustee-operator-system` namespace
2. **CatalogSource** (`templates/catalog-source.yaml`) - Deployed to `openshift-marketplace` namespace
3. **OperatorGroup** (`templates/operator-group.yaml`) - Scopes operator to target namespace
4. **Subscription** (`templates/subscription.yaml`) - Subscribes to operator channel with automatic approval

### Post-Install Hooks (executed after core resources)

The chart uses Helm hooks with weighted execution to solve the CRD timing problem:

1. **Hook Weight 0**: RBAC Resources (`templates/crd-wait-rbac.yaml`)
   - ServiceAccount: `trustee-crd-waiter`
   - ClusterRole with permissions to check CRD availability
   - ClusterRoleBinding

2. **Hook Weight 1**: CRD Wait Job (`templates/crd-wait-job.yaml`)
   - Kubernetes Job that polls for `trusteeconfigs.confidentialcontainers.org` CRD
   - Waits up to 5 minutes (60 attempts × 5 seconds)
   - Automatically detects `oc` or `kubectl` CLI tool
   - Fails if CRD doesn't become available

3. **Hook Weight 2**: TrusteeConfig CR (`templates/trusteeconfig.yaml`)
   - Custom Resource of kind `TrusteeConfig`
   - API: `confidentialcontainers.org/v1alpha1`
   - Only created after CRD wait job succeeds

## Key Architectural Patterns

### Helm Hook Ordering

The chart relies heavily on Helm hooks to ensure correct resource creation order. Hooks are defined via annotations:

```yaml
annotations:
  "helm.sh/hook": post-install,post-upgrade
  "helm.sh/hook-weight": "1"
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

**Critical**: The hook weights must maintain this order:
- RBAC (weight 0) → CRD Wait Job (weight 1) → TrusteeConfig CR (weight 2)

### CRD Timing Problem

The operator installs its CRDs asynchronously after the Subscription is created. Without the CRD wait job, the TrusteeConfig CR would fail to create because the CRD doesn't exist yet. The wait job solves this by polling until the CRD is available.

## Common Helm Commands

### Validate and Test

```bash
# Lint the chart
helm lint .

# Test template rendering
helm template test-release .

# Dry run to see what would be installed
helm install test-release . --dry-run --debug

# Package the chart
helm package .
```

### Install and Upgrade

```bash
# Install with default values
helm install my-trustee-operator .

# Install with custom values
helm install my-trustee-operator . \
  --set trusteeConfig.spec.profileType=Restrictive \
  --set trusteeConfig.spec.kbsServiceType=NodePort

# Install from a values file
helm install my-trustee-operator . -f custom-values.yaml

# Upgrade existing release
helm upgrade my-trustee-operator .

# View release status
helm status my-trustee-operator

# List all releases
helm list
```

### Uninstall

```bash
# Uninstall release
helm uninstall my-trustee-operator
```

## Monitoring Deployment

```bash
# Check operator installation status
oc get csv -n trustee-operator-system

# View operator logs
oc logs -n trustee-operator-system deployment/trustee-operator-controller-manager

# Check TrusteeConfig status
oc get trusteeconfig -n trustee-operator-system
oc describe trusteeconfig trusteeconfig -n trustee-operator-system

# Verify KBS deployment (created by operator from TrusteeConfig)
oc get pods -n trustee-operator-system
oc get svc -n trustee-operator-system

# Check hook job status during install
oc get jobs -n trustee-operator-system
oc logs -n trustee-operator-system job/trusteeconfig-crd-wait
```

## Key Configuration Values

See `values.yaml` for all configurable options. The chart has been simplified to minimize configuration complexity:

### Namespace
- `namespace.name` - Target namespace (default: `trustee-operator-system`)

### TrusteeConfig
- `trusteeConfig.name` - Name of the TrusteeConfig CR (default: `trusteeconfig`)
- `trusteeConfig.spec.profileType` - Attestation policy: `Permissive` (dev/test) or `Restrictive` (production)
- `trusteeConfig.spec.kbsServiceType` - Service exposure: `ClusterIP`, `NodePort`, or `LoadBalancer`

## Common Customizations

### Change Namespace

To deploy to a different namespace:

```bash
helm install my-trustee-operator . \
  --set namespace.name=my-custom-namespace
```

### Use Different TrusteeConfig Name

```bash
helm install my-trustee-operator . \
  --set trusteeConfig.name=my-trusteeconfig
```

## Template Helper Functions

Defined in `templates/_helpers.tpl`:

- `trustee-operator.name` - Chart name (truncated to 63 chars)
- `trustee-operator.fullname` - Full application name
- `trustee-operator.chart` - Chart name and version
- `trustee-operator.labels` - Common labels for all resources
- `trustee-operator.selectorLabels` - Selector labels

## Important Concepts

### TrusteeConfig API
- **Group**: `confidentialcontainers.org`
- **Version**: `v1alpha1`
- **Kind**: `TrusteeConfig`
- **CRD Name**: `trusteeconfigs.confidentialcontainers.org`

### Profile Types
- **Permissive**: Relaxed attestation policies for development/testing
- **Restrictive**: Strict attestation policies for production

### KBS (Key Broker Service)
The operator creates KBS resources from the TrusteeConfig. The `kbsServiceType` determines how the service is exposed.

### Child Resources
When the operator processes a TrusteeConfig, it creates:
- KbsConfig custom resources
- Deployments for KBS pods
- Services for KBS endpoints
- ConfigMaps and Secrets for configuration
