# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap three Kubernetes worker nodes. The following components will be installed on each node: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [cri-containerd](https://github.com/kubernetes-incubator/cri-containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Prerequisites

The commands in this lab must be run on each worker instance: `worker-0`, `worker-1`, and `worker-2`. Login to each worker instance using the `vagrant ssh` command. Example:

```
vagrant ssh worker-0
```

## Download Worker Binaries

```
wget -q --show-progress --https-only --timestamping \
  https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz \
  https://github.com/kubernetes-incubator/cri-containerd/releases/download/v1.0.0-alpha.0/cri-containerd-1.0.0-alpha.0.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubelet
```

## Provisioning a Kubernetes Worker Node

Install the OS dependencies:

```
sudo apt update && sudo apt -y install socat
```

> The socat binary enables support for the `kubectl port-forward` command.


## Install Worker Binaries
Create the installation directories:

```
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Install the worker binaries:

```
sudo tar -xvf /vagrant/cni-plugins-amd64-v0.6.0.tgz -C /opt/cni/bin/
```

```
sudo tar -xvf /vagrant/cri-containerd-1.0.0-alpha.0.tar.gz -C /
```

```
(cd /vagrant && sudo cp kubectl kube-proxy kubelet /usr/local/bin/)
```

```
(cd /usr/local/bin && sudo chmod +x kubectl kube-proxy kubelet)
```

### Get Worker Internal IP

```
INTERNAL_IP=$(ip -4 --oneline addr | grep -v secondary | grep -oP '(192\.168\.100\.[0-9]{1,3})(?=/)')
```

### Configure the Kubelet

```
(cd /vagrant && sudo cp ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/)
```

```
(cd /vagrant && sudo cp ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig)
```

```
(cd /vagrant && sudo cp ca.pem /var/lib/kubernetes/)
```

Create the `kubelet.service` systemd unit file:

```
cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=cri-containerd.service
Requires=cri-containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --allow-privileged=true \\
  --anonymous-auth=false \\
  --authorization-mode=Webhook \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --cluster-dns=10.32.0.10 \\
  --cluster-domain=cluster.local \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/cri-containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --require-kubeconfig \\
  --runtime-request-timeout=25m \\
  --tls-cert-file=/var/lib/kubelet/${HOSTNAME}.pem \\
  --tls-private-key-file=/var/lib/kubelet/${HOSTNAME}-key.pem \\
  --v=2 \\
  --node-ip=${INTERNAL_IP}
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Proxy

```
(cd /vagrant && sudo cp kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig)
```

Create the `kube-proxy.service` systemd unit file:

```
cat > kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --cluster-cidr=10.244.0.0/16 \\
  --kubeconfig=/var/lib/kube-proxy/kubeconfig \\
  --proxy-mode=iptables \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the cri-containerd
Create the `cri-containerd.service` systemd unit file:

```
cat > cri-containerd.service <<EOF
[Unit]
Description=Kubernetes containerd CRI shim
Requires=network-online.target
After=containerd.service

[Service]
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/cri-containerd --logtostderr --stream-addr ${INTERNAL_IP}
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Worker Services

```
sudo mv kubelet.service kube-proxy.service cri-containerd.service /etc/systemd/system/
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable containerd cri-containerd kubelet kube-proxy
```

```
sudo systemctl start containerd cri-containerd kubelet kube-proxy
```

> Remember to run the above commands on each worker node: `worker-0`, `worker-1`, and `worker-2`.

## Verification

Login to one of the controller nodes:

```
vagrant ssh controller-0
```

List the registered Kubernetes nodes:

```
kubectl get nodes
```

> output

```
NAME       STATUS     ROLES     AGE       VERSION
worker-0   NotReady   <none>    1m        v1.8.0
worker-1   NotReady   <none>    1m        v1.8.0
worker-2   NotReady   <none>    1m        v1.8.0
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
