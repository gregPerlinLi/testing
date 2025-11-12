# Strix Helm Chart

Official Helm Chart for deploying [Strix](https://github.com/usestrix/strix) - Open-source AI Security Testing Agent on Kubernetes.

## Overview

Strix are autonomous AI agents that act like real hackers - they run your code dynamically, find vulnerabilities, and validate them through actual proof-of-concepts. This Helm chart provides a production-ready deployment of Strix on Kubernetes with Docker-in-Docker support for isolated sandbox environments.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PV provisioner support in the underlying infrastructure (if persistence is enabled)
- LLM API key (OpenAI, Anthropic, or compatible provider)

## Installation

### Add Helm Repository (if published)

```bash
helm repo add strix https://charts.usestrix.com
helm repo update
```

### Install from local chart

```bash
# Clone the repository
git clone https://github.com/usestrix/strix.git
cd strix/helm

# Install the chart
helm install my-strix . \
  --namespace strix \
  --create-namespace \
  --set strix.llm.apiKey="your-api-key-here" \
  --set strix.llm.provider="openai/gpt-4"
```

### Install with custom values

```bash
helm install my-strix . \
  --namespace strix \
  --create-namespace \
  --values custom-values.yaml
```

## Configuration

### Basic Configuration

| Parameter          | Description        | Default            |
|--------------------|--------------------|--------------------|
| `replicaCount`     | Number of replicas | `1`                |
| `image.registry`   | Image registry     | `docker.io`        |
| `image.repository` | Image repository   | `usestrix/strix`   |
| `image.tag`        | Image tag          | `Chart appVersion` |
| `image.pullPolicy` | Image pull policy  | `IfNotPresent`     |

### Strix Configuration

| Parameter                 | Description                   | Default        |
|---------------------------|-------------------------------|----------------|
| `strix.llm.provider`      | LLM provider and model        | `openai/gpt-4` |
| `strix.llm.apiKey`        | LLM API key (required)        | `""`           |
| `strix.llm.apiBase`       | Custom API base URL           | `""`           |
| `strix.perplexity.apiKey` | Perplexity API key for search | `""`           |
| `strix.nonInteractive`    | Run in headless mode          | `true`         |
| `strix.target`            | Default target for scanning   | `""`           |
| `strix.instructions`      | Custom instructions           | `""`           |

### Docker-in-Docker Configuration

| Parameter                      | Description             | Default   |
|--------------------------------|-------------------------|-----------|
| `dind.enabled`                 | Enable Docker-in-Docker | `true`    |
| `dind.image.repository`        | DinD image repository   | `docker`  |
| `dind.image.tag`               | DinD image tag          | `24-dind` |
| `dind.resources.limits.cpu`    | DinD CPU limit          | `2000m`   |
| `dind.resources.limits.memory` | DinD memory limit       | `4Gi`     |

### Persistence Configuration

| Parameter                   | Description        | Default         |
|-----------------------------|--------------------|-----------------|
| `persistence.enabled`       | Enable persistence | `true`          |
| `persistence.size`          | PVC size           | `10Gi`          |
| `persistence.accessMode`    | Access mode        | `ReadWriteOnce` |
| `persistence.storageClass`  | Storage class      | `""`            |
| `persistence.existingClaim` | Use existing PVC   | `""`            |

### Ingress

Expose Strix via Kubernetes Ingress to route external traffic to the Service.

- ingress.enabled: Enable/disable Ingress
- ingress.className: IngressClass name (e.g., nginx, traefik)
- ingress.annotations: Controller/Cert-Manager annotations
- ingress.hosts: Host/path rules (pathType usually Prefix)
- ingress.tls: TLS secrets for HTTPS

Example (NGINX + cert-manager):

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: strix.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts: [strix.example.com]
      secretName: strix-tls
```

Apply:

```bash
helm upgrade --install my-strix . -n strix --create-namespace \
  --set strix.llm.apiKey="<your-api-key>" \
  --set ingress.enabled=true \
  --set ingress.className=nginx \
  --set ingress.hosts[0].host=strix.example.com
```

### Resources

| Parameter                   | Description    | Default |
|-----------------------------|----------------|---------|
| `resources.limits.cpu`      | CPU limit      | `2000m` |
| `resources.limits.memory`   | Memory limit   | `4Gi`   |
| `resources.requests.cpu`    | CPU request    | `500m`  |
| `resources.requests.memory` | Memory request | `1Gi`   |

## Usage Examples

### 1. Basic Security Scan

```bash
# Get pod name
POD_NAME=$(kubectl get pods -n strix -l "app.kubernetes.io/name=strix" -o jsonpath="{.items[0].metadata.name}")

# Run security scan on a target
kubectl exec -it $POD_NAME -n strix -- strix --target https://example.com
```

### 2. Scan GitHub Repository

```bash
kubectl exec -it $POD_NAME -n strix -- \
  strix --target https://github.com/org/repo
```

### 3. Headless Scan with Custom Instructions

```bash
kubectl exec -it $POD_NAME -n strix -- \
  strix -n --target https://api.example.com \
  --instructions "Focus on authentication and authorization vulnerabilities"
```

### 4. Copy Results Locally

```bash
# Copy scan results from pod
kubectl cp $POD_NAME:/workspace/agent_runs ./local-results -n strix
```

## Advanced Configuration

### Custom Values Example

Create a `custom-values.yaml` file:

```yaml
replicaCount: 1

strix:
  llm:
    provider: "anthropic/claude-sonnet-3-5"
    apiKey: "sk-ant-xxxxx"
  perplexity:
    apiKey: "pplx-xxxxx"
  target: "https://myapp.example.com"

persistence:
  enabled: true
  size: 20Gi
  storageClass: "fast-ssd"

resources:
  limits:
    cpu: 4000m
    memory: 8Gi
  requests:
    cpu: 1000m
    memory: 2Gi

dind:
  resources:
    limits:
      cpu: 3000m
      memory: 6Gi
```

Then install:

```bash
helm install my-strix . -f custom-values.yaml -n strix --create-namespace
```

### Using with Local LLM (Ollama)

```yaml
strix:
  llm:
    provider: "ollama/llama3"
    apiKey: "not-required"
    apiBase: "http://ollama-service:11434"
```

## Chart Signing (Provenance)

This chart can be optionally signed in CI and publish provenance files (.prov).

Setup:
- Generate a GPG key and export the private key (ASCII armored) and passphrase.
- Add repository secrets: HELM_GPG_PRIVATE_KEY and HELM_GPG_PASSPHRASE.

Verify locally:
```bash
# After adding the repo and updating index
helm verify strix-<version>.tgz --keyring ~/.gnupg/pubring.gpg
```

Note: The workflow auto-detects the key and runs `helm package --sign`.

## Upgrading

```bash
# Upgrade with new values
helm upgrade my-strix . -n strix --set strix.llm.provider="openai/gpt-4o"

# Upgrade with values file
helm upgrade my-strix . -n strix -f custom-values.yaml
```

## Uninstalling

```bash
helm uninstall my-strix -n strix
```

To also delete the PVC:

```bash
kubectl delete pvc -n strix -l "app.kubernetes.io/name=strix"
```

## Security Considerations

1. **Privileged Containers**: Docker-in-Docker requires privileged mode. Ensure your cluster security policies allow this.
2. **API Keys**: Store sensitive API keys using Kubernetes Secrets or external secret management solutions.
3. **Network Policies**: Consider implementing network policies to restrict egress traffic.
4. **RBAC**: Use appropriate service accounts with minimal required permissions.
5. **Scan Targets**: Only scan systems you own or have explicit permission to test.

## Troubleshooting

### Pod not starting

```bash
# Check pod status
kubectl describe pod $POD_NAME -n strix

# Check logs
kubectl logs $POD_NAME -n strix
kubectl logs $POD_NAME -n strix -c dind
```

### Docker daemon not accessible

```bash
# Verify DinD container is running
kubectl exec -it $POD_NAME -n strix -c dind -- docker info

# Check if main container can access Docker
kubectl exec -it $POD_NAME -n strix -- docker ps
```

### Persistence issues

```bash
# Check PVC status
kubectl get pvc -n strix

# Check PV binding
kubectl describe pvc -n strix
```

## Support

- 📖 Documentation: https://usestrix.com
- 💬 Discord: https://discord.gg/YjKFvEZSdZ
- 🐛 Issues: https://github.com/usestrix/strix/issues
- ⭐ GitHub: https://github.com/usestrix/strix

## License

Apache 2.0 - See [LICENSE](https://github.com/usestrix/strix/blob/main/LICENSE) for details.