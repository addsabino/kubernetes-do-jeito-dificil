# Subindo os Nós Worker do Kubernetes

Nesse lab você irá subir três nós _worker_ do Kubernetes. Os seguintes componentes serão instalados em cada nó:[runc](https://github.com/opencontainers/runc), [plugins de rede para contêiner](https://github.com/containernetworking/cni), [cri-containerd](https://github.com/kubernetes-incubator/cri-containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet) e [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Pré-requisitos

Os comandos nesse lab devem ser executados em cada instância de controladora:  `worker-0`, `worker-1`, e `worker-2`. Conecte em cada controladora utilizando o comando `gcloud`. Exemplo:

```
gcloud compute ssh worker-0
```

### Executando comandos com tmux
O [tmux](https://github.com/tmux/tmux/wiki) pode ser utilizado para executar comandos em várias instâncias computacionais ao mesmo tempo. Veja o [Executando comandos em paralelo com o tmux](01-pre-requisitos.md#executando-comandos-em-paralelo-com-o-tmux) Na sessão de prerequisitos


## Provisionando um Nó _Worker_ do Kubernetes

Instale as dependências de SO:

```
{
  sudo apt install ip_vs
  sudo apt-get update
  sudo apt-get -y install socat conntrack ipset

  curl -fsSL https://get.docker.com/ | sudo sh

# Necessário para o roteamento de pods funcionar
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
ip_vs
EOF



}
```

> O binário socat habilita o suporte ao comando `kubectl port-forward`.

### Faça o Download e Instale os Binários do _Worker_

```
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.12.0/crictl-v1.12.0-linux-amd64.tar.gz \
  https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz \
  https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubelet
```

Crie os diretórios de instalação:

```
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Instale os binários do _worker_:

```
{
  chmod +x kubectl kube-proxy kubelet
  sudo mv kubectl kube-proxy kubelet /usr/local/bin/
  sudo tar -xvf crictl-v1.12.0-linux-amd64.tar.gz -C /usr/local/bin/
  sudo tar -xvf cni-plugins-amd64-v0.6.0.tgz -C /opt/cni/bin/
}
```

### Configure a rede CNI

Recupere a faixa CIDR do Pod para a instância computacional corrente:

```
POD_CIDR=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/pod-cidr)
```

Crie o arquivo de configuração de rede `bridge`:

```
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

Crie o arquivo de configuração de rede `loopback`:

```
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.1",
    "type": "loopback"
}
EOF
```

# Configure o containerd

### Configure containerd

Crie o arquivo de configuração do `containerd`:

```
sudo mkdir -p /etc/containerd/
```

```
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
    [plugins.cri.containerd.untrusted_workload_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
    [plugins.cri.containerd.gvisor]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
EOF
```

Crie o arquivo de systemd unit `containerd.service`:

```
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### Configure o Kubelet

```
{
  sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
  sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.pem /var/lib/kubernetes/
}
```

Create the `kubelet-config.yaml` configuration file:

```
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

> a configuração do `resolvConf` é usada para evitar loops quando o CoreDNS é usado para descoberta de serviços e o `systemd-resolved` usado.

Crie o arquivo _unit_ `kubelet.service` do systemd:

```
cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

```

### Configure o Proxy do Kubernetes

```
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Crie o arquivo yaml de configuração `kube-proxy-config.yaml`:

```
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

Crie o arquivo _unit_ `kube-proxy.service` do systemd:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Inicie os Serviços do _Worker_


```
{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd kubelet kube-proxy
  sudo systemctl start containerd kubelet kube-proxy
}
```


> Lembre-se de executar os comandos acima em cada um dos nós de _worker_: `worker-0`, `worker-1` e `worker-2`.

## Verificação

Conecte em um dos nós de controladora:

```
gcloud compute ssh controller-0 \
  --command "kubectl get nodes --kubeconfig admin.kubeconfig"
```

Liste os nós de Kubernetes registrados:

```
kubectl get nodes
```

> saída

```
NAME       STATUS    ROLES     AGE       VERSION
worker-0   Ready     <none>    1m        v1.8.0
worker-1   Ready     <none>    1m        v1.8.0
worker-2   Ready     <none>    1m        v1.8.0
```

Próximo: [Configurando o kubectl para Acesso Remoto](10-configurando-kubectl.md)
