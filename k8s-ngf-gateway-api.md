# Kubernetes Single-Node Setup (NGINX Gateway Fabric)

A guide to installing a single-node Kubernetes cluster on a Lima VM, with k9s, Helm, MetalLB, and NGINX Gateway Fabric as a Gateway API controller for local testing.

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

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Set up kubeconfig for your user and allow scheduling on the control plane (required for single-node):

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Install the Flannel CNI network plugin:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Verify the node is ready:

```bash
kubectl get nodes
```

---

## 5. Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## 6. Gateway API with MetalLB and NGINX Gateway Fabric (VM-local testing)

On a bare-metal or VM-based cluster there is no cloud provider to assign IPs to `LoadBalancer` services, so ingress controllers will sit in `<pending>` indefinitely. MetalLB solves this. Since this is just for local testing inside the VM, we use a dummy network interface to give MetalLB a private IP range that is only visible inside the VM — no special Lima networking or WiFi bridging required.

### A note on Gateway API

Gateway API is the successor to the `Ingress` resource. Rather than a single `Ingress` object with annotations, it separates concerns across three resources:

- **`GatewayClass`** — names the controller implementation (set once, cluster-scoped)
- **`Gateway`** — defines a listener (port, protocol, TLS) and which routes can bind to it
- **`HTTPRoute`** — defines routing rules (hostnames, paths, backends)

This separation makes it easier to delegate route management to application teams while keeping infrastructure concerns with the platform team. NGINX Gateway Fabric is F5/NGINX's official Gateway API implementation — a natural evolution from the classic NGINX Ingress Controller toward the Gateway API model.

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

### Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml

# Wait for MetalLB pods to be ready
kubectl wait --namespace metallb-system --for=condition=ready pod --selector=app=metallb --timeout=90s
```

Create the IP pool and advertisement config:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: dummy-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.200.100-192.168.200.200
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: dummy-advert
  namespace: metallb-system
spec:
  ipAddressPools:
    - dummy-pool
  interfaces:
    - dummy0
EOF
```

### Install the Gateway API CRDs

NGF does not bundle its own Gateway API CRDs, so they must be installed before deploying the controller. We install the standard channel, which includes the stable `HTTPRoute` resource:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/latest/download/standard-install.yaml
```

Verify the CRDs are present:

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

You should see `gatewayclasses`, `gateways`, and `httproutes` among the results.

### Install NGINX Gateway Fabric

```bash
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --namespace nginx-gateway \
  --create-namespace
```

Verify the NGF control plane pod is running:

```bash
kubectl get pods -n nginx-gateway
```

Confirm the `GatewayClass` named `nginx` was created and is accepted:

```bash
kubectl get gatewayclass
```

You should see `nginx` with `ACCEPTED` showing `True`.

### Create the Gateway

Unlike other controllers, NGF uses a split control plane / data plane architecture. The data plane (the NGINX deployment and its LoadBalancer service) is not provisioned until a `Gateway` object is created. Apply the Gateway first and wait for the data plane service to appear before deploying the test application:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: web
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
EOF
```

Watch for the data plane service to appear in the `default` namespace:

```bash
kubectl get svc -w
```

Once you see a `LoadBalancer` service named `nginx-gateway-nginx` with an IP from the `192.168.200.x` range, press `ctrl-c` and proceed.

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
    - name: nginx-gateway
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
kubectl get gateway nginx-gateway
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

> **Note:** On corporate-managed Macs, security software may prevent sshuttle from installing its `pf` (packet filter) rules. sshuttle will report "Connected" but traffic will not route — curl times out and the browser cannot connect. If this happens, use SSH port forwarding instead.

### Alternative: SSH port forwarding

SSH port forwarding achieves the same result without `pf` rules. This command forwards a local port directly to the gateway IP inside the VM:

```bash
ssh -i ~/.lima/_config/user -o StrictHostKeyChecking=no \
  -L 8080:192.168.200.100:80 \
  127.0.0.1 -p $(limactl list --format '{{.SSHLocalPort}}' fullkube)
```

With the tunnel open, access the cluster via `localhost:8080`:

```bash
curl localhost:8080 -H "Host: myapp.local"
```

For browser testing, add a `/etc/hosts` entry pointing `myapp.local` to `127.0.0.1`:

```bash
echo "127.0.0.1 myapp.local" | sudo tee -a /etc/hosts
```

Then `http://myapp.local:8080` works in a browser. The tunnel runs in the foreground — `ctrl-c` to stop it.

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
