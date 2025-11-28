# Trustee Operator Helm Chart

This Helm chart deploys the Trustee Operator on OpenShift/Kubernetes. The Trustee Operator manages the Key Broker Service (KBS) for attestation and secret delivery in confidential computing environments as part of the Confidential Containers project.

## Prerequisites

- Kubernetes 1.19+ or OpenShift 4.x
- Helm 3.0+
- Operator Lifecycle Manager (OLM) installed on the cluster

## Installing the Chart

To install the chart with the release name `my-trustee-operator`:

```bash
helm install my-trustee-operator ./trustee-operator-chart
```

Or from a packaged chart:

```bash
helm package trustee-operator-chart
helm install my-trustee-operator trustee-operator-1.0.0.tgz
```

## Uninstalling the Chart

To uninstall/delete the `my-trustee-operator` deployment:

```bash
helm uninstall my-trustee-operator
```

## Configuration

The following table lists the configurable parameters of the Trustee Operator chart and their default values.

### Namespace Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `namespace.name` | Name of the namespace to deploy to | `trustee-operator-system` |

### TrusteeConfig Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `trusteeConfig.name` | Name of the TrusteeConfig CR | `trusteeconfig` |
| `trusteeConfig.spec.profileType` | Attestation profile type (`Permissive` or `Restrictive`) | `Permissive` |
| `trusteeConfig.spec.kbsServiceType` | KBS service type (`ClusterIP`, `NodePort`, or `LoadBalancer`) | `ClusterIP` |

## Customizing Values

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example:

```bash
helm install my-trustee-operator ./trustee-operator-chart \
  --set trusteeConfig.spec.profileType=Restrictive \
  --set trusteeConfig.spec.kbsServiceType=NodePort
```

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart:

```bash
helm install my-trustee-operator ./trustee-operator-chart -f my-values.yaml
```

Example `my-values.yaml`:

```yaml
namespace:
  name: my-trustee-namespace

trusteeConfig:
  name: my-trusteeconfig
  spec:
    profileType: Restrictive
    kbsServiceType: LoadBalancer
```

## Deployment Order

The chart uses Helm hooks to ensure resources are created in the correct order:

### Main Resources (installed immediately)
1. Namespace
2. CatalogSource (in openshift-marketplace)
3. OperatorGroup
4. Subscription

### Post-Install Hooks (executed after main resources)
The chart uses Helm hooks with weighted execution order to ensure the TrusteeConfig CR is only created after the operator has installed its CRDs:

1. **Hook Weight 0**: RBAC resources (ServiceAccount, ClusterRole, ClusterRoleBinding) for CRD verification
2. **Hook Weight 1**: CRD Wait Job - A Kubernetes Job that polls for the TrusteeConfig CRD to become available (waits up to 5 minutes)
3. **Hook Weight 2**: TrusteeConfig CR - Created only after the CRD wait job succeeds

This hook-based approach solves the timing issue where the TrusteeConfig custom resource would fail to create because the operator hadn't yet installed its CRD.

## Monitoring the Deployment

After installation, monitor the deployment:

```bash
# Check operator installation status
oc get csv -n <namespace>

# View operator logs
oc logs -n <namespace> deployment/trustee-operator-controller-manager

# Check TrusteeConfig status
oc get trusteeconfig -n <namespace>

# Verify KBS deployment
oc get pods -n <namespace>
oc get svc -n <namespace>
```

## Upgrading

To upgrade the chart:

```bash
helm upgrade my-trustee-operator ./trustee-operator-chart
```

## Architecture

For more information about the architecture and components, see the [CLAUDE.md](../CLAUDE.md) file in the repository root.

## Support

For issues and support, visit the [Trustee Operator GitHub repository](https://github.com/confidential-containers/trustee-operator).
