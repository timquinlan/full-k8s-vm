# Kubernetes Single-Node Setup (Cilium)

A guide to installing a single-node Kubernetes cluster on a Lima VM, with k9s, Helm, and a fully Cilium-native stack for local testing. Unlike the other guides in this collection, Cilium replaces three separate components — Flannel (CNI), MetalLB (LoadBalancer IP assignment), and a standalone Gateway API controller — with a single deployment.

## Prerequisites

* macOS with [Lima](https://lima-vm.io) installed (for VM creation — the rest of the guide works on any Ubuntu VM regardless of how it was created)
* At least 2 CPUs and 2GB RAM
* `sudo` access on the VM

---

## 0. Create the VM (macOS / Lima)

> **Note:** This step is macOS-specific. If you already have an Ubuntu VM from another source, skip to step 1.

Install Lima if you haven't already:

```bash
brew install lima
```

Create and start the VM:

```bash
limactl create --name fullkube --cpus 4 --memory 8 --tty=false
limactl start fullkube
```

Open a shell on the VM:

```bash
limactl shell fullkube
```

All subsequent steps are run inside this shell unless otherwise noted.

---

## 1. System Preparation

Disable swap, load required kernel modules, and configure networking:

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# Load bridge module now and on reboot
sudo modprobe br_netfilter
echo br_netfilter | sudo tee /etc/modules-load.d/br_netfilter.conf

# Enable required sysctl settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

---

## 2. Install containerd

```bash
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

---

## 3. Install kubeadm, kubelet, and kubectl

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 4. Bootstrap the Cluster

Cilium replaces kube-proxy entirely using eBPF. Tell kubeadm to skip the kube-proxy addon so it is never installed:

```bash
sudo kubeadm init --skip-phases=addon/kube-proxy
```

Set up kubeconfig for your user and allow scheduling on the control plane (required for single-node):

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Verify the node status:

```bash
kubectl get nodes
```

> **Note:** The node will show `NotReady` at this point — this is expected. Cilium acts as the CNI and will be installed in step 6. The node will transition to `Ready` once Cilium is running.

---

## 5. Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## 6. Cilium: CNI, LB-IPAM, L2 Announcements, and Gateway API (VM-local testing)

On a bare-metal or VM-based cluster there is no cloud provider to assign IPs to `LoadBalancer` services, and without a CNI the node cannot run workloads. In this guide Cilium handles all of it. Since this is just for local testing inside the VM, we use a dummy network interface to give Cilium a private IP range that is only visible inside the VM — no special Lima networking or WiFi bridging required.

### A note on the Cilium-native stack

In the other guides in this collection, Flannel handles pod networking, MetalLB handles LoadBalancer IP assignment and L2 announcement, and a separate controller handles Gateway API routing. Here, Cilium replaces all three:

- **CNI** — Cilium manages pod networking using eBPF, and also replaces kube-proxy
- **LB-IPAM** — a `CiliumLoadBalancerIPPool` resource defines the IP range; Cilium assigns addresses from it automatically
- **L2 Announcements** — a `CiliumL2AnnouncementPolicy` resource tells Cilium to respond to ARP requests on the dummy interface, making LoadBalancer IPs reachable inside the VM
- **Gateway API** — Cilium's built-in Gateway API controller handles `Gateway` and `HTTPRoute` resources natively

### Create a dummy network interface

```bash
# Bring up a dummy interface immediately
sudo ip link add dummy0 type dummy
sudo ip link set dummy0 up
sudo ip addr add 192.168.200.1/24 dev dummy0
```

To make it persist across reboots:

```bash
cat <<EOF | sudo tee /etc/systemd/network/10-dummy0.netdev
[NetDev]
Name=dummy0
Kind=dummy
EOF

cat <<EOF | sudo tee /etc/systemd/network/20-dummy0.network
[Match]
Name=dummy0

[Network]
Address=192.168.200.1/24
EOF

sudo systemctl restart systemd-networkd
```

### Install the Cilium CLI

The Cilium CLI provides the `cilium` command for verifying the install, checking status, and troubleshooting. Install it on the VM:

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')
curl -L -o /tmp/cilium-linux-${ARCH}.tar.gz \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${ARCH}.tar.gz
sudo tar xzvfC /tmp/cilium-linux-${ARCH}.tar.gz /usr/local/bin
rm /tmp/cilium-linux-${ARCH}.tar.gz
cilium version --client
```

### Install the Gateway API CRDs

Cilium does not bundle the Gateway API CRDs. Install them before Cilium so the operator detects them on first boot and automatically creates the `cilium` GatewayClass. Cilium 1.14+ requires the experimental channel because it depends on `TLSRoute` (`gateway.networking.k8s.io/v1alpha2`), which is not included in the standard channel:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/experimental-install.yaml
```

Verify the CRDs are present:

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

You should see `gatewayclasses`, `gateways`, `httproutes`, and `tlsroutes` among the results.

### Install Cilium

Add the Cilium Helm repo:

```bash
helm repo add cilium https://helm.cilium.io/ && helm repo update
```

Cilium needs to know the API server address to act as a full kube-proxy replacement. Capture it from the node, then install:

```bash
API_SERVER_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=${API_SERVER_IP} \
  --set k8sServicePort=6443 \
  --set l2announcements.enabled=true \
  --set gatewayAPI.enabled=true \
  --set operator.replicas=1
```

Wait for Cilium to be fully ready:

```bash
cilium status --wait
```

Verify the node is now `Ready`:

```bash
kubectl get nodes
```

### Configure LB-IPAM

Create an IP pool pointing at the dummy interface range. Cilium will draw from this range when assigning addresses to `LoadBalancer` services:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: dummy-pool
spec:
  blocks:
    - start: "192.168.200.100"
      stop: "192.168.200.200"
EOF
```

### Configure L2 Announcements

Tell Cilium to respond to ARP requests for LoadBalancer IPs on the dummy interface:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: dummy-l2-policy
spec:
  interfaces:
    - dummy0
  loadBalancerIPs: true
EOF
```

### Confirm the GatewayClass

Confirm that Cilium created a `GatewayClass` named `cilium` and that it is accepted:

```bash
kubectl get gatewayclass
```

You should see `cilium` with `ACCEPTED` showing `True`.

### Create the Gateway

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cilium-gateway
spec:
  gatewayClassName: cilium
  listeners:
    - name: web
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
EOF
```

Watch for Cilium to provision a `LoadBalancer` service for the gateway:

```bash
kubectl get svc -w
```

Once you see a `LoadBalancer` service named `cilium-gateway` with an IP from the `192.168.200.x` range, press `ctrl-c` and proceed.

### Deploy a test application

Deploy a vanilla nginx instance and wire it up through an `HTTPRoute` to verify the full stack is working:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-nginx
  template:
    metadata:
      labels:
        app: test-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: test-nginx
spec:
  selector:
    app: test-nginx
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: test-nginx
spec:
  parentRefs:
    - name: cilium-gateway
  hostnames:
    - "myapp.local"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: test-nginx
          port: 80
EOF
```

Verify the `Gateway` has been programmed and the `HTTPRoute` has been accepted:

```bash
kubectl get gateway cilium-gateway
kubectl get httproute test-nginx
```

Both should show `PROGRAMMED` / `Accepted` in their status columns.

Verify the full stack is working:

```bash
curl 192.168.200.100 -H "Host: myapp.local"
```

A successful response will show the default nginx welcome page HTML.

---

## 7. Accessing the Cluster from Your Laptop (sshuttle)

For demos or browser testing from your Mac, sshuttle acts like a lightweight VPN over SSH — it makes the `192.168.200.x` subnet routable from your Mac without any changes to the VM.

Install sshuttle on your Mac:

```bash
brew install sshuttle
```

Then run:

> NOTE: "fullkube" is the name of the VM we created at the start; if you named it something different, adjust this command.

```bash
sshuttle -r 127.0.0.1:$(limactl list --format '{{.SSHLocalPort}}' fullkube) 192.168.200.0/24 \
  --ssh-cmd 'ssh -i ~/.lima/_config/user -o StrictHostKeyChecking=no'
```

Verify you can reach the gateway from your Mac:

```bash
curl 192.168.200.100 -H "Host: myapp.local"
```

Your Mac can now reach `192.168.200.100` directly. For clean demo URLs, add a `/etc/hosts` entry on your Mac:

```bash
echo "192.168.200.100 myapp.local" | sudo tee -a /etc/hosts
```

Then `http://myapp.local` works in a browser with no port numbers. sshuttle runs in the foreground — just `ctrl-c` to stop it when done.

---

## 8. Install k9s (optional)

k9s is a terminal UI for managing Kubernetes clusters. It's optional — everything can be done with `kubectl` — but it makes navigating resources, tailing logs, and inspecting pods significantly easier.

```bash
cd /tmp
ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')
curl -LO https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_${ARCH}.tar.gz
tar -xzf k9s_Linux_${ARCH}.tar.gz
chmod +x k9s && sudo mv k9s /usr/local/bin/
k9s version
```

**Useful k9s keybindings:**

| Key | Action |
| --- | --- |
| `:pod` | Jump to pod view |
| `:namespace` | Switch namespaces |
| `?` | Show help/keybindings |
| `ctrl-c` | Exit |
