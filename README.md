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
- `kubectl` and `helm` v3.x available
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

## Step 2 — Install everything in one command

Edit `values/agentic-engineer.yaml` with your domain and credentials, then:

```bash
helm install wso2-agentic-engineer \
  oci://ghcr.io/senithkay/wso2-agentic-engineer-bundle \
  --version 0.1.0 \
  --namespace wso2-ae \
  --create-namespace \
  -f values/agentic-engineer.yaml
```

This single command:
1. Creates the `wso2-ae` namespace
2. Deploys all platform services (asdlc-api, agents-service, console, postgres, openbao)
3. Auto-generates all secrets (webhook secret, OAuth state key, postgres password, RSA task signing key)
4. Registers ClusterComponentType, ClusterTrait, ClusterWorkflow, ClusterAuthzRoleBinding into OC
5. Exposes the console and API through the OC gateway via HTTPRoutes

No manual secret creation or `kubectl apply` steps needed.

## Values reference

See `values/agentic-engineer.yaml` for the minimal override file. Key fields:

| Value | Description |
|---|---|
| `expose.consoleHostname` | DNS hostname for the console (e.g. `asdlc.example.com`) |
| `expose.apiHostname` | DNS hostname for the API (e.g. `asdlc-api.example.com`) |
| `wso2-agentic-engineer.console.publicURL` | Full public URL for the console |
| `wso2-agentic-engineer.console.thunderPublicURL` | Full public URL for Thunder IdP |
| `wso2-agentic-engineer.anthropic.apiKey` | Anthropic API key for AI flows |
| `wso2-agentic-engineer.openbao.token` | OpenBao token (default: `root`, override for prod) |
| `wso2-agentic-engineer.github.appSlug` | GitHub App slug (optional, for App-mode GitHub connect) |

## DNS setup

After install, get the OC gateway LoadBalancer IP:

```bash
kubectl get svc -n openchoreo-control-plane -l app=gateway-default
```

Create DNS A records pointing `expose.consoleHostname` and `expose.apiHostname` to that IP.

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

If building from source rather than using pre-published images:

```bash
cd /path/to/wso2-agentic-engineer-2

docker build -t ghcr.io/<your-org>/asdlc-api:latest ./asdlc-service
docker build -t ghcr.io/<your-org>/agents-service:latest ./agents
docker build -t ghcr.io/<your-org>/asdlc-console:latest -f console/Dockerfile .

docker push ghcr.io/<your-org>/asdlc-api:latest
docker push ghcr.io/<your-org>/agents-service:latest
docker push ghcr.io/<your-org>/asdlc-console:latest
```

Then override image repositories in `values/agentic-engineer.yaml`:

```yaml
wso2-agentic-engineer:
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

## Upgrading

```bash
helm upgrade wso2-agentic-engineer \
  oci://ghcr.io/senithkay/wso2-agentic-engineer-bundle \
  --version <new-version> \
  --namespace wso2-ae \
  -f values/agentic-engineer.yaml
```

Auto-generated secrets (webhook secret, OAuth state key, postgres password) are preserved across upgrades.

## Uninstalling

```bash
helm uninstall wso2-agentic-engineer -n wso2-ae

# Clean up OC resources (not removed by helm uninstall)
kubectl delete clusterworkflow dockerfile-builder app-factory-coding-agent
kubectl delete clustercomponenttype service web-application
kubectl delete clustertrait api-configuration
kubectl delete clusterauthzrolebinding asdlc-api-client-binding administrators-group-binding

kubectl delete namespace wso2-ae
```
