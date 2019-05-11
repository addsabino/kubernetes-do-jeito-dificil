# Subindo o Cluster etcd

Componentes do Kubernetes são _stateless_ (não mantém estado), mas armazenam o estado do cluster no [etcd](https://github.com/coreos/etcd). Nesse lab você irá subir um cluster de etcd com três nós e configurá-lo para alta disponibilidade e acesso remoto seguro.

## Pré-requisitos

Os comandos nesse lab devem ser executados em cada instância de controladora:  `controller-0`, `controller-1`, e `controller-2`. Conecte em cada controladora utilizando o comando `gcloud`. Exemplo:

```
gcloud compute ssh controller-0
```

## Subindo um Membro de Cluster etcd

### Faça o Download e Instale os Binários do etcd

Faça o Download dos binários oficiais do etcd a partir do projeto [coreos/etcd](https://github.com/coreos/etcd) no GitHub:

```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
```

Extraia e instale o servidor `etcd` e o utilitário de linha de comando `etcdctl`:

```
{
  tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
  sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
}
```

### Configure o Servidor etcd

Crie o diretório do etcd dados e configuração e copie as chaves privadas para diretório de configuração

```
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
}
```

O IP interno da instância será utilizado para servir requisições de cliente e comunicar com cada um dos pares do cluster etcd. Recupere o endereço de IP interno para a instância computacional em uso:

```
IP_INTERNO=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```

Cada membro etcd deve ter um nome único dentro de um cluster etcd. Configure o nome do etcd para bater com o _hostname_ a instância computacional corrente:

```
NOME_ETCD=$(hostname -s)
```

Crie o arquivo _unit_ `etcd.service` do systemd:

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${NOME_ETCD} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${IP_INTERNO}:2380 \\
  --listen-peer-urls https://${IP_INTERNO}:2380 \\
  --listen-client-urls https://${IP_INTERNO}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls https://${IP_INTERNO}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Inicie o Servidor etcd

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}

```

> Lembre de executar os comandos acima em cada nó de controladora: `controller-0`, `controller-1`, e `controller-2`.

## Verificação

Liste os membros do cluster etcd:

```
ETCDCTL_API=3 etcdctl member list
```

> saída

```
3a57933972cb5131, started, controller-2, https://10.240.0.12:2380, https://10.240.0.12:2379
f98dc20bce6225a0, started, controller-0, https://10.240.0.10:2380, https://10.240.0.10:2379
ffed16798470cab5, started, controller-1, https://10.240.0.11:2380, https://10.240.0.11:2379
```

Próximo: [Subindo o Plano de Controle do Kubernetes](08-subindo-controladoras-kubernetes.md)
