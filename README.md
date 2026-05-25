# gitops

Kubernetes application and platform configuration for the AKS cluster provisioned by [single-aks](https://github.com/Ntnick-22/single-aks).

## Repository Structure

```
.github/workflows/
  helm.yaml          CI/CD for platform Helm releases (Terraform)
  k8s.yaml           CI/CD for raw Kubernetes manifests

platform/
  helm/              Terraform layer — platform tooling (nginx-ingress, cert-manager)
  apps/              Kubernetes manifests — applications and cluster-wide resources
    demo-app/        Sample nginx app (deployment, service, ingress)
    cluster-issuer.yaml   Let's Encrypt ClusterIssuer via Cloudflare DNS01
    cloudflare-secret.yaml  Cloudflare API token for cert-manager
```

## Platform Layer (`platform/helm`)

Manages cluster-wide infrastructure tools via Terraform + Helm provider 3.x.

| Release | Namespace | Purpose |
|---|---|---|
| `ingress-nginx` | `ingress-nginx` | Nginx ingress controller with Azure LoadBalancer |
| `cert-manager` | `cert-manager` | Automatic TLS certificate management via Let's Encrypt |

State is stored remotely in Azure Blob Storage (`aksk8state`, key: `05_helm/terraform.tfstate`).

The Helm provider authenticates to the cluster using `~/.kube/config` — the CI workflow sets this up via `az aks get-credentials` + `kubelogin` before running Terraform.

## Application Layer (`platform/apps`)

Raw Kubernetes manifests applied via `kubectl`. Currently deploys:

- **demo-app** — nginx demo container, exposed via ingress at `myaks.nt-nick.link`
- **ClusterIssuer** — Let's Encrypt production issuer using Cloudflare DNS01 challenge
- **Cloudflare secret** — API token used by cert-manager to create DNS TXT records

## TLS Flow

```
cert-manager sees Ingress annotation: cert-manager.io/cluster-issuer: letsencrypt-prod
  → requests certificate from Let's Encrypt
  → Let's Encrypt issues DNS01 challenge
  → cert-manager uses Cloudflare API token to create TXT record
  → Let's Encrypt verifies → issues certificate
  → stored in Kubernetes secret: demo-app-tls
  → nginx-ingress uses it for TLS termination
```

## Traffic Flow

```
Browser → DNS (myaks.nt-nick.link → 50.85.154.165)
  → Azure Load Balancer (auto-provisioned by AKS for LoadBalancer service type)
  → nginx-ingress controller
      - TLS termination using demo-app-tls secret
      - routes by host header → demo-app service (ClusterIP :80)
  → demo-app pod
```

## CI/CD Workflows

### helm.yaml
- Triggers on push to `main` when files under `platform/helm/**` change
- Runs `terraform init → plan → apply`
- Sets up kubeconfig via `az aks get-credentials` + `kubelogin` before Terraform

### k8s.yaml
- Triggers on push to `main` when files under `platform/apps/**` change
- Runs `kubectl apply` for cluster-wide resources and app manifests

### Authentication
Both workflows use GitHub OIDC federation to authenticate to Azure — no stored credentials. Required secrets:

| Secret | Purpose |
|---|---|
| `AZURE_CLIENT_ID` | Federated identity client ID |
| `AZURE_TENANT_ID` | Azure AD tenant |
| `AZURE_SUBSCRIPTION_ID` | Target subscription |
| `AZURE_RESOURCE_GROUP` | AKS resource group |
| `AZURE_AKS_NAME` | AKS cluster name |
