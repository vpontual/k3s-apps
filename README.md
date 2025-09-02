# k3s Homelab Applications

This repository contains the configuration to deploy and manage applications on a K3s cluster using Helmfile.

## Prerequisites

Before you begin, ensure you have the following tools installed and configured:

*   `kubectl`: Configured to connect to your K3s cluster.
*   `helm`: The Kubernetes package manager.
*   `helmfile`: The declarative spec for deploying Helm charts.

## 1. Secret Configuration

Some applications require Kubernetes secrets to be created **before** you deploy them. Please create the following secrets.

### Vaultwarden Admin Token

This secret stores the admin token for Vaultwarden's admin interface.

**IMPORTANT:** Replace `YOUR_SUPER_SECRET_TOKEN` with your own strong, random token.

```bash
kubectl create secret generic vaultwarden-secret \
  --from-literal=ADMIN_TOKEN='YOUR_SUPER_SECRET_TOKEN' \
  -n vaultwarden
```

### Open WebUI API Key

This secret stores the API key for Open WebUI.

**IMPORTANT:** Replace `YOUR_API_KEY` with a secure key of your choice.

```bash
kubectl create secret generic open-webui-secrets \
  --from-literal=openai-api-key='YOUR_API_KEY' \
  -n open-webui
```

## 2. Deployment

You can deploy all services at once or one by one.

### Deploy All Services

To deploy all applications defined in `helmfile.yaml`, run:

```bash
helmfile sync
```

### Deploy a Single Service

To deploy a specific service, use the `-l` (selector) flag with the name of the release. Here are the commands for each service:

*   **Vaultwarden**
    ```bash
    helmfile sync -l name=vaultwarden
    ```

*   **Open WebUI**
    ```bash
    helmfile sync -l name=open-webui
    ```

*   **Glance**
    ```bash
    helmfile sync -l name=glance
    ```

## 3. Configuration Options

The `values/` directory contains the specific configuration for each application. To see all possible configuration options for a chart, you can use the `helm show values` command.

This is useful for customizing the `base.yaml` files for each application.

*   **Vaultwarden**
    ```bash
    helm show values vaultwarden/vaultwarden
    ```

*   **Open WebUI**
    ```bash
    helm show values open-webui/open-webui
    ```

*   **Glance**
    ```bash
    helm show values glance/glance
    ```