# WSO2 Agentic Engineer Module for OpenChoreo

This module installs the WSO2 Agentic Engineer platform capabilities into an existing OpenChoreo installation.

## What This Module Registers

| Resource | Kind | Extension Point |
|---|---|---|
| `service` | ClusterComponentType | Teaches OC how to deploy backend service components |
| `web-application` | ClusterComponentType | Teaches OC how to deploy SPA frontend components |
| `api-configuration` | ClusterTrait | Adds WSO2 API Platform management to any component endpoint |
| `dockerfile-builder` | ClusterWorkflow | Docker image build pipeline on the Workflow plane |
| `app-factory-coding-agent` | ClusterWorkflow | AI coding agent runner on the Workflow plane |
| `asdlc-api-client-binding` | ClusterAuthzRoleBinding | Grants asdlc-api service account admin role in OC |
| `administrators-group-binding` | ClusterAuthzRoleBinding | Grants Thunder Administrators group admin role in OC |

## Prerequisites

- OpenChoreo v1.0.0 installed
- WSO2 API Platform Gateway Operator installed in the data plane (see Step 1)
- `kubectl` configured for the target cluster
- `helm` v3.x

## Installation

### Step 1 — Install Gateway Operator (skip if already installed from another module)

```bash
export DATA_PLANE_NS=openchoreo-data-plane

helm install gateway-operator \
  oci://ghcr.io/wso2/api-platform/helm-charts/gateway-operator \
  --version 0.4.0 \
  --namespace ${DATA_PLANE_NS} \
  --create-namespace \
  --timeout 600s \
  --set logging.level=info \
  --set gateway.helm.chartVersion="0.9.0"

kubectl wait --for=condition=Available deployment \
  -l app.kubernetes.io/name=gateway-operator \
  -n ${DATA_PLANE_NS} --timeout=300s
```

### Step 2 — Apply raw workflow and RBAC manifests

These contain Argo Workflow template syntax and are applied directly.

```bash
# Wait for OC controller webhook to be ready first
kubectl wait -n openchoreo-control-plane \
  --for=condition=available --timeout=300s \
  deployment/controller-manager

kubectl apply -f resources/rbac.yaml
kubectl apply -f resources/docker-build-workflow.yaml
kubectl apply -f resources/app-factory-coding-agent.yaml
```

### Step 3 — Install the platform resources Helm chart

This registers ComponentTypes, the ClusterTrait, and authorization into OpenChoreo.

```bash
helm install wso2-ae-platform-resources ./helm \
  --set authz.apiClientId=asdlc-api-client \
  --set authz.administratorsGroup=Administrators
```

To use a custom values file:

```bash
helm install wso2-ae-platform-resources ./helm -f my-values.yaml
```

## Verification

```bash
# ClusterComponentTypes
kubectl get clustercomponenttype service web-application

# ClusterTrait
kubectl get clustertrait api-configuration

# ClusterWorkflows
kubectl get clusterworkflow dockerfile-builder app-factory-coding-agent

# ClusterAuthzRoleBindings
kubectl get clusterauthzrolebinding asdlc-api-client-binding administrators-group-binding
```

## Uninstallation

```bash
helm uninstall wso2-ae-platform-resources

kubectl delete -f resources/app-factory-coding-agent.yaml
kubectl delete -f resources/docker-build-workflow.yaml
kubectl delete -f resources/rbac.yaml
```
