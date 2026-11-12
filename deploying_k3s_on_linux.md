# Deploying K3s on a Linux VM

This guide installs a single-node K3s cluster on an Ubuntu or Debian Linux VM and verifies that the cluster, networking and DNS are working.

The K3s server acts as both the Kubernetes control plane and a worker node by default. A default single-server installation uses SQLite rather than etcd.

## 1. Prepare the Linux VM

Recommended minimum resources for a learning VM:

- 2 vCPUs
- 2 GB RAM
- 20 GB disk
- A static or stable private IP address
- Outbound HTTPS access

Check the operating system, hostname, addresses and available resources:

```bash
cat /etc/os-release
hostnamectl
ip addr
free -h
df -h
```

Update the operating system:

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

K3s does not normally require swap to be disabled, but you can inspect it:

```bash
swapon --show
```

If you want to disable swap for lab consistency:

```bash
sudo swapoff -a
```

If the uncomplicated firewall is enabled, allow the Kubernetes API, Flannel VXLAN and kubelet ports:

```bash
sudo ufw allow 6443/tcp
sudo ufw allow 8472/udp
sudo ufw allow 10250/tcp
sudo ufw status
```

These ports are required between nodes in a multi-node cluster. Restrict their source ranges to the lab network where possible.

| Port | Purpose |
|---|---|
| `6443/tcp` | Kubernetes API server |
| `8472/udp` | Flannel VXLAN networking in the default configuration |
| `10250/tcp` | Kubelet API |

## 2. Install K3s

Install the latest stable K3s release using the official installer:

```bash
curl -sfL https://get.k3s.io | sh -
```

The installer:

- Installs the `k3s` binary.
- Configures and starts the `k3s` systemd service.
- Installs the bundled `kubectl` command.
- Writes the administrator kubeconfig to `/etc/rancher/k3s/k3s.yaml`.
- Uses an embedded SQLite datastore for a default single-server installation.

For a disposable learning VM, the following makes the kubeconfig readable by non-root users:

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server --write-kubeconfig-mode=644" \
  sh -
```

Do not use a world-readable administrator kubeconfig on a shared or production server. Prefer the normal restrictive permissions and copy it securely to the individual administrator account as shown later in this guide.

To install a specific K3s version rather than the latest stable release:

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_VERSION='<k3s-version>' \
  sh -
```

## 3. Verify the K3s systemd service

Inspect the service:

```bash
sudo systemctl status k3s --no-pager
```

Confirm that it is enabled and active:

```bash
sudo systemctl is-enabled k3s
sudo systemctl is-active k3s
```

The expected results are:

```text
enabled
active
```

Enable and start the service manually if necessary:

```bash
sudo systemctl enable --now k3s
```

Inspect recent logs:

```bash
sudo journalctl -u k3s -n 100 --no-pager
```

Follow the service logs while troubleshooting:

```bash
sudo journalctl -u k3s -f
```

## 4. Verify the Kubernetes API and node

Use the K3s-provided `kubectl` command:

```bash
sudo k3s kubectl get nodes
sudo k3s kubectl get nodes -o wide
```

The node should report `Ready` and have the `control-plane,master` role.

Check the Kubernetes versions and cluster endpoint:

```bash
sudo k3s kubectl version
sudo k3s kubectl cluster-info
```

Check all system Pods:

```bash
sudo k3s kubectl get pods -A -o wide
```

Wait for all current Pods to become ready:

```bash
sudo k3s kubectl wait \
  --for=condition=Ready pod \
  --all \
  --all-namespaces \
  --timeout=120s
```

Inspect the cluster workloads and services:

```bash
sudo k3s kubectl get deployments,daemonsets,statefulsets,services -A
```

Test the API server readiness endpoint:

```bash
sudo k3s kubectl get --raw='/readyz?verbose'
```

## 5. Configure kubectl for the normal user

Inspect the K3s kubeconfig:

```bash
sudo ls -l /etc/rancher/k3s/k3s.yaml
```

Copy it to the current user's kubeconfig directory:

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/rancher/k3s/k3s.yaml "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
chmod 600 "$HOME/.kube/config"
```

Set the kubeconfig for the current shell:

```bash
export KUBECONFIG="$HOME/.kube/config"
```

Optionally persist it for Bash sessions:

```bash
echo 'export KUBECONFIG="$HOME/.kube/config"' >> "$HOME/.bashrc"
source "$HOME/.bashrc"
```

Verify access without `sudo`:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

The default K3s context is normally named `default`.

If this kubeconfig is copied to another machine, replace `127.0.0.1` in its `server` URL with the K3s server's reachable address. Protect the file because it contains cluster-administrator credentials.

## 6. Verify CoreDNS and networking

Inspect CoreDNS:

```bash
kubectl -n kube-system get deployment coredns
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system get service kube-dns
```

Inspect the K3s system components:

```bash
kubectl -n kube-system get pods -o wide
kubectl get daemonsets -A
```

Typical default K3s components include:

- CoreDNS
- Local Path Provisioner
- Metrics Server
- Traefik
- ServiceLB

Create a temporary Pod:

```bash
kubectl run network-test \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 300
```

Wait for it to become ready:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/network-test \
  --timeout=120s
```

Test Kubernetes DNS from inside the Pod:

```bash
kubectl exec network-test -- nslookup kubernetes.default.svc.cluster.local
```

Remove the temporary Pod:

```bash
kubectl delete pod network-test
```

## 7. Deploy and test a sample application

Create an NGINX Deployment:

```bash
kubectl create deployment hello \
  --image=nginx:stable
```

Expose it through a cluster-internal Service:

```bash
kubectl expose deployment hello \
  --port=80 \
  --target-port=80
```

Wait for the Deployment and inspect its resources:

```bash
kubectl rollout status deployment/hello --timeout=120s
kubectl get deployment hello
kubectl get pods -l app=hello -o wide
kubectl get service hello
kubectl get endpointslice -l kubernetes.io/service-name=hello
```

Create a temporary client Pod:

```bash
kubectl run curl-test \
  --image=curlimages/curl:8.10.1 \
  --restart=Never \
  --command -- sleep 300
```

Wait for the client and call the Service:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/curl-test \
  --timeout=120s

kubectl exec curl-test -- \
  curl --fail --silent --show-error http://hello
```

Remove the sample resources:

```bash
kubectl delete deployment hello
kubectl delete service hello
kubectl delete pod curl-test
```

## 8. Run health checks

Run these checks to confirm that K3s is healthy:

```bash
sudo systemctl is-active k3s
kubectl get --raw='/readyz?verbose'
kubectl get nodes
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
```

Confirm that the API server and kubelet ports are listening:

```bash
sudo ss -lntup | grep -E ':(6443|10250)\b'
```

Record node details:

```bash
kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,KUBELET:.status.nodeInfo.kubeletVersion,OS:.status.nodeInfo.osImage,RUNTIME:.status.nodeInfo.containerRuntimeVersion'
```

The installation is working when:

1. `systemctl is-active k3s` returns `active`.
2. The readiness endpoint reports successful checks.
3. The node reports `Ready`.
4. The system Pods are running or completed successfully.
5. The DNS test resolves the Kubernetes Service.
6. The test application responds through its Service.

## 9. Optionally add a worker node

On the K3s server, display the node-registration token:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Treat this token as a secret. Do not write it to logs, documentation or source control.

Find the server's reachable private IP address:

```bash
hostname -I
```

On the worker VM, install the K3s agent:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL='https://<server-private-ip>:6443' \
  K3S_TOKEN='<node-token>' \
  sh -
```

Verify the agent service on the worker:

```bash
sudo systemctl status k3s-agent --no-pager
sudo systemctl is-active k3s-agent
sudo journalctl -u k3s-agent -n 100 --no-pager
```

Verify the new node from the server:

```bash
kubectl get nodes -o wide
```

For a multi-node cluster, ensure that the worker can reach the server on `6443/tcp` and that the required Flannel traffic is permitted between nodes.

## 10. Uninstall K3s

The uninstall commands remove K3s and its local cluster data. Use them only on a disposable VM or a server that is intentionally being decommissioned.

Uninstall a K3s server:

```bash
sudo /usr/local/bin/k3s-uninstall.sh
```

Uninstall a K3s agent:

```bash
sudo /usr/local/bin/k3s-agent-uninstall.sh
```
