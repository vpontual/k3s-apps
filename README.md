# k3s Homelab Applications

This repository contains the configuration to deploy and manage applications on a K3s cluster using Helmfile.

## Prerequisites

Before you begin, ensure you have the following tools installed and configured:

*   `kubectl`: Configured to connect to your K3s cluster.
*   `helm`: The Kubernetes package manager.
*   `helmfile`: The declarative spec for deploying Helm charts.
*.  

## 1. Secret Configuration

TO DO 
 - CURRENTLY SECRETS ARE HARDCODED IN THE CONFIGURATION FILES
 - THE INSTRUCTIONS FOR THE SECRETS BELOW DO NOT WORK YET

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
helmfile apply
```

### Deploy a Single Service

To deploy a specific service, use the `-l` (selector) flag with the name of the release. Here are the commands for each service:

*   **Vaultwarden**
    ```bash
    helmfile apply -l name=vaultwarden
    ```

*   **Open WebUI**
    ```bash
    helmfile apply -l name=open-webui
    ```

*   **Glance**
    ```bash
    helmfile apply -l name=glance
    ```

## 3. Customization

Each application has a `base.yaml` file with the default configuration. To customize an application, you should not edit the `base.yaml` file directly. Instead, you should create an `override.yaml` file.

This approach allows you to receive updates to the `base.yaml` files without creating merge conflicts with your local changes.

### Customization Workflow

1.  **Copy the Example:** For the application you want to customize, copy the `override.yaml.example` to a new file named `override.yaml`. For example, for Vaultwarden:

    ```bash
    cp values/vaultwarden/override.yaml.example values/vaultwarden/override.yaml
    ```

2.  **Add Your Overrides:** Add your custom values to the `override.yaml` file. Any value you set in this file will override the default value in `base.yaml`.

3.  **Find Configurable Values:** To see all possible configuration options for a chart, you can use the `helm show values` command. For example, for Vaultwarden:

    ```bash
    helm show values vaultwarden/vaultwarden
    ```

    Use the output of this command to find the values you want to override in your `override.yaml`.