# Customizing CoreDNS for Local Domain Resolution in k3s

This guide explains how to customize the CoreDNS configuration in a k3s Kubernetes cluster to resolve local domains (e.g., `*.example.com`) to specific IP addresses. This is useful for routing in-cluster traffic to local servers that are not managed by Kubernetes, especially when those servers host multiple services under different domains.

This method uses the `import` functionality of the default k3s CoreDNS configuration, which is robust and will persist across cluster upgrades. The CoreDNS configuration in k3s is set up to load any additional `*.server` files provided in the `coredns-custom` ConfigMap.

## Problem

You need pods within the k3s cluster to resolve any hostname under your local domains (e.g., `service.casarp.us` or `service.casavp.com`) to specific IP addresses on your local network.

## Solution

The solution is to create custom CoreDNS server block files for each local domain. These blocks use the `template` plugin to act as a wildcard, answering any query for that domain or its subdomains with the specified IP address. These configuration files are then loaded into CoreDNS via a special `ConfigMap` named `coredns-custom`.

---

## Step-by-Step Guide

### 1. Create the Custom Server Configuration Files

First, create a file for each domain that defines the new server block. The k3s CoreDNS setup will import any file ending with the `.server` extension. Make sure the files are placed in the servers folder. For clarity, it's best to name each file after the domain you are configuring (e.g., `casarp.server`, `casavp.server`).

These files contain the logic to match your domain and answer with the desired IP.

**Example for `casarp.us` resolving to `10.0.154.1`:**

```coredns
casarp.us:53 {
    errors
    cache 30
    rewrite name regex (.*)\.casarp\.us casarp.us
    forward . 10.0.154.1
}
```

**Example for `casavp.com` resolving to `10.0.153.1`:**

```coredns
casavp.com:53 {
    errors
    cache 30
    rewrite name regex (.*)\.casavp\.com casavp.com
    forward . 10.0.153.1
}
```

**Explanation:**
*   `casarp.us:53`: Defines a new server block that will handle all queries for the `casarp.us` zone.
*   `rewrite name regex (.*)\.casarp\.us casarp.us` tells CoreDNS to rewrite any subdomain like foo.casarp.us, bar.casarp.us, etc., into casarp.us before resolving. It ensures that all *.casarp.us resolves to the IP defined by the forward directive.

### 2. Create or Update the Custom ConfigMap

The k3s CoreDNS deployment is pre-configured to look for and load configurations from a `ConfigMap` named `coredns-custom` in the `kube-system` namespace.

To ensure only the necessary `.server` files are loaded, you should create the `ConfigMap` using only files located in the ./server folder.

If you are updating your custom configmap, first delete the existing one with

```bash
kubectl delete configmap coredns-custom --namespace=kube-system 
```

then execute

```bash
kubectl create configmap coredns-custom --namespace=kube-system \
  --from-file=./servers/
```

This command creates a `ConfigMap` containing only `casarp.server` and `casavp.server`, which is the cleanest way to provide the custom configuration to CoreDNS.

### 3. Verify the Changes

Creating or updating the `coredns-custom` `ConfigMap` will trigger the CoreDNS pods to reload the configuration. You can watch the pods to ensure they restart/reload correctly.

```bash
# Watch the CoreDNS pods. You should see them restart or reload.
kubectl get pods -n kube-system -l k8s-app=kube-dns --watch
```
Wait for the pod to be in a `Running` and `READY 1/1` state. If it gets stuck in `CrashLoopBackOff`, there is a syntax error in one of your configuration files.

### 4. Test the DNS Resolution

Finally, test that the resolution is working from inside the cluster.

**A. Start a debug pod:**
The `nicolaka/netshoot` image is excellent as it contains `dig` and other useful networking tools.
```bash
kubectl run -it dns-debug --image=nicolaka/netshoot --restart=Never -- /bin/bash
```

**B. Run tests from the debug pod's shell:**
Use the `dig` command to test your domains.
```bash
# Test a subdomain for the first domain
dig homeassistant.casarp.us

# Test the bare domain
dig casarp.us

# Test a subdomain for the second domain
dig something.casavp.com

# Test the bare domain
dig casavp.com
```

In all cases, you should see a `status: NOERROR` and an `ANSWER SECTION` showing a direct `A` record pointing to the IP you configured for that domain.

**C. Clean up the debug pod:**
Once you are finished testing, exit the pod's shell (`exit`) and delete the pod.
```bash
kubectl delete pod dns-debug
```