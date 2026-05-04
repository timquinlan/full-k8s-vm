# Kubernetes Single-Node Lab Setup
A collection of guides for standing up a single-node Kubernetes cluster on a Lima VM,
varying only in their ingress implementation. All guides share the same base stack
(kubeadm, Flannel, MetalLB, k9s, Helm) and the same dummy-interface trick for
self-contained LoadBalancer IP assignment — no cloud provider required.

> **Note:** These guides use Lima to provision the VM, as it's a lightweight and
> widely available option for macOS users. The Lima-specific step is isolated to
> Step 0 in each guide — everything from Step 1 onward runs on standard Ubuntu
> regardless of how it was provisioned. However, there are some Lima-specific network
> settings that you may have to adjust.

## Guides

| Guide | Ingress Implementation | API Style |
|---|---|---|
| [F5 NGINX Ingress](./F5_NGINX_ingress.md) | F5 NGINX Ingress Controller | `networking.k8s.io/v1 Ingress` |
| [Traefik Ingress](./traefik-ingress.md) | Traefik | `networking.k8s.io/v1 Ingress` |
| [Traefik Gateway API](./traefik-gateway-api.md) | Traefik | Gateway API `HTTPRoute` |
| [NGINX Gateway Fabric](./k8s-ngf-gateway-api.md) | NGINX Gateway Fabric | Gateway API `HTTPRoute` |
| [Envoy Gateway](./k8s-envoy-gateway.md) | Envoy Gateway | Gateway API `HTTPRoute` |

## Which guide should I use?

**NGINX or Traefik Ingress** — if you want a familiar, widely-documented setup using
the classic `Ingress` resource. Both work identically from an application perspective;
the difference is the controller and its annotation conventions.

**Traefik Gateway API** — if you want to work with the next-generation Gateway API,
which separates infrastructure concerns (`Gateway`) from routing concerns (`HTTPRoute`).
This is the direction the Kubernetes ecosystem is heading and is worth learning even
for lab use.

**NGINX Gateway Fabric** — if you want a Gateway API implementation backed by NGINX,
or are already familiar with the F5/NGINX ecosystem and want to see how the same vendor
approaches the newer API model.

**Envoy Gateway** — if you want a Gateway API implementation backed by Envoy Proxy,
the same data plane used by Istio and many other cloud-native projects. A good choice
for exploring the broader CNCF ecosystem.

## Shared components

All guides provision the same base environment:
- **Lima VM** — lightweight macOS VM host (step 0 only; skip if you have your own Ubuntu VM)
- **kubeadm** — cluster bootstrap
- **Flannel** — CNI network plugin
- **k9s** — terminal UI for cluster management
- **Helm** — package manager
- **MetalLB** — bare-metal LoadBalancer IP assignment via a dummy network interface
- **sshuttle** — lightweight SSH tunnel for accessing the cluster from your Mac
