# WSO2 Agentic Engineer Module for OpenChoreo

Installs the WSO2 Agentic Engineer platform on an existing OpenChoreo cluster.

## Architecture

```
Control Plane cluster
├── openchoreo-control-plane   ← OC API server + gateway-default
└── wso2-ae                    ← Agentic Engineer services
      asdlc-api (:9090)  ──────── calls OC API ──→ openchoreo-control-plane
      agents-service (:3400)
      asdlc-console (:3000)
      asdlc-postgres

Data Plane cluster
└── openchoreo-data-plane
      gateway-operator
      api-configuration trait → routes component traffic through WSO2 API Platform

Workflow Plane cluster
└── openchoreo-workflow-plane
      dockerfile-builder ClusterWorkflow
      app-factory-coding-agent ClusterWorkflow
```

## What This Module Registers in OC

| Resource | Kind | Extension Point |
|---|---|---|
| `service` | ClusterComponentType | CI / Workflow plane |
| `web-application` | ClusterComponentType | CI / Workflow plane |
| `api-configuration` | ClusterTrait | API Gateway (data plane) |
| `dockerfile-builder` | ClusterWorkflow | CI (workflow plane) |
| `app-factory-coding-agent` | ClusterWorkflow | CI (workflow plane) |
| `asdlc-api-client-binding` | ClusterAuthzRoleBinding | OC admin access |
| `administrators-group-binding` | ClusterAuthzRoleBinding | OC admin access |

## Prerequisites

- OpenChoreo v1.0.0 installed
- `kubectl`, `helm` v3.x, `envsubst` available
- Docker images built and pushed to a registry (see Building Images below)

## Step 1 — Install Gateway Operator on the data plane

Skip if already installed (e.g. from the `ai-wso2-agent-manager` module).

```bash
export DATA_PLANE_NS=openchoreo-data-plane

# Check first
helm list -n ${DATA_PLANE_NS} | grep gateway-operator

# Install if not present
helm install gateway-operator \
  oci://ghcr.io/wso2/api-platform/helm-charts/gateway-operator \
  --version 0.4.0 \
  --namespace ${DATA_PLANE_NS} \
  --create-namespace \
  --timeout 600s \
  --set logging.level=info \
  --set gateway.helm.chartVersion="0.9.0"
```

## Step 3 — Install the platform services

Edit `values/agentic-engineer.yaml` with your domain and Anthropic key, then:

```bash
helm install wso2-agentic-engineer \
  oci://ghcr.io/senithkay/wso2-agentic-engineer \
  --version 0.1.0 \
  --namespace wso2-ae \
  --create-namespace \
  -f values/agentic-engineer.yaml
```

No manual secret creation needed — the chart auto-generates `GITHUB_WEBHOOK_SECRET`,
`OAUTH_STATE_SIGNING_KEY`, the RSA task signing key, and the PostgreSQL password on first install.

## Step 4 — Register into OpenChoreo

```bash
# Wait for OC controller webhook
kubectl wait -n openchoreo-control-plane \
  --for=condition=available --timeout=300s \
  deployment/controller-manager

kubectl apply -f resources/rbac.yaml
kubectl apply -f resources/docker-build-workflow.yaml
kubectl apply -f resources/app-factory-coding-agent.yaml

helm install wso2-ae-platform-resources \
  oci://ghcr.io/senithkay/wso2-ae-platform-resources \
  --version 0.1.0
```

## Step 5 — Expose via the OC gateway

```bash
export AE_CONSOLE_HOSTNAME=asdlc.example.com
export AE_API_HOSTNAME=asdlc-api.example.com

envsubst < resources/httproute-console.yaml | kubectl apply -f -
envsubst < resources/httproute-api.yaml | kubectl apply -f -
```

Create DNS records pointing both hostnames to the OC gateway LoadBalancer IP:
```bash
kubectl get svc -n openchoreo-control-plane -l app=gateway-default
```

## Verification

```bash
# Platform services running
kubectl get pods -n wso2-ae

# OC resources registered
kubectl get clustercomponenttype service web-application
kubectl get clustertrait api-configuration
kubectl get clusterworkflow dockerfile-builder app-factory-coding-agent
kubectl get clusterauthzrolebinding asdlc-api-client-binding

# HTTPRoutes accepted
kubectl get httproute -n wso2-ae
```

## Building Images

If you're building from source rather than using pre-published images:

```bash
cd /path/to/wso2-agentic-engineer-2

docker build -t ghcr.io/<your-org>/asdlc-api:latest ./asdlc-service
docker build -t ghcr.io/<your-org>/agents-service:latest ./agents
docker build -t ghcr.io/<your-org>/asdlc-console:latest -f console/Dockerfile .

docker push ghcr.io/<your-org>/asdlc-api:latest
docker push ghcr.io/<your-org>/agents-service:latest
docker push ghcr.io/<your-org>/asdlc-console:latest
```

Then override image repositories in values:
```yaml
asdlcApi:
  image:
    repository: ghcr.io/<your-org>/asdlc-api
agentsService:
  image:
    repository: ghcr.io/<your-org>/agents-service
console:
  image:
    repository: ghcr.io/<your-org>/asdlc-console
```

## Uninstallation

```bash
kubectl delete -f resources/httproute-console.yaml
kubectl delete -f resources/httproute-api.yaml
helm uninstall wso2-ae-platform-resources
helm uninstall wso2-agentic-engineer -n wso2-ae
kubectl delete -f resources/app-factory-coding-agent.yaml
kubectl delete -f resources/docker-build-workflow.yaml
kubectl delete -f resources/rbac.yaml
kubectl delete namespace wso2-ae
```
