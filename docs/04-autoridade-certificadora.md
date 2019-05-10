# Provisionando a CA e Gerando Certificados TLS

Nesse lab você irá provisionar uma [Infraestrutura PKI (Infraestrutura de Chave Privada)](https://en.wikipedia.org/wiki/Public_key_infrastructure) utilizando o toolkit da CloudFare, [cfssl](https://github.com/cloudflare/cfssl) e então utilizá-la para subir uma Autoridade Certificadora e gerar certificados TLS para os seguintes componentes: etcd, kube-apiserver, kubelet e kube-proxy.

## Autoridade Certificadora

Nessa seção você irá provisionar uma Autoridade Certificadora que pode ser utilizada para gerar certificados TLS adicionais.

Crie o arquivo de configuração da CA:

```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

Crie a requisição de assinatura do certificado da CA:

```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "BR",
      "L": "Sao Paulo",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Sao Paulo"
    }
  ]
}
EOF
```

Gere o certificado da CA e a chave privada:

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Resultados:

```
ca-key.pem
ca.pem
```

## Certificados de Cliente e Servidor

Nessa seção você irá gerar os certificados de cliente e servidor para cada um dos componentes do Kubernetes e um certificado de cliente para o usuário `admin` do Kubernetes.


### O Certificado de Cliente do Admin

Crie a requisição de assinatura do certificado de cliente do `admin`:

```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "BR",
      "L": "Sao Paulo",
      "O": "system:masters",
      "OU": "Kubernetes do Jeito Dificil",
      "ST": "Sao Paulo"
    }
  ]
}
EOF
```

Gere o certificado de cliente e a chave privada do `admin`:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

Resultados:

```
admin-key.pem
admin.pem
```

### Os Certificados de Cliente do Kubelet

Kubernetes utiliza um [modo de autorização de "propósito especial"] (https://kubernetes.io/docs/admin/authorization/node/) chamado _Node Authorizer_ (Autorizador de Nó), que autoriza especificamente requisições de API feitas pelos [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet). Para serem autorizados pelo Autorizador de Nó, os Kubelets devem utilizar uma credencial que os identifiquem como parte do grupo `system:nodes`, com um usuário parte de `system:node:<nomeDoNó>`. Nessa seção você irá criar um certificado para cada _worker_ do Kubernetes que atenda aos requisitos do Autorizador de Nós.

Gere um certificado e uma chave privada para cada nó _worker_ & _controller_ do Kubernetes:

```
for nodetype in worker controller ; do
  for count in 0 1 2; do

    instance=$nodetype-$count
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "BR",
      "L": "Sao Paulo",
      "O": "system:nodes",
      "OU": "Kubernetes do Jeito Dificil",
      "ST": "Sao Paulo"
    }
  ]
}
EOF

EXTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')

INTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].networkIP)')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${IP_EXTERNO},${IP_INTERNO} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}

  done
done
```

Resultados:

```
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```
### Certificado do Cliente Controller Manager

Gere um certificado e uma chave privada para o `kube-controller-manager` :

```
{

cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "BR",
      "L": "Sao Paulo",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes do Jeito Dificil",
      "ST": "Sao Paulo"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

Results:

```
kube-controller-manager-key.pem
kube-controller-manager.pem
```

### O Certificado de Cliente do kube-proxy

Crie a requisição de assinatura do certificado de cliente `kube-proxy`:

```
{


cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "BR",
      "L": "Sao Paulo",
      "O": "system:node-proxier",
      "OU": "Kubernetes do Jeito Dificil",
      "ST": "Sao Paulo"
    }
  ]
}
EOF

}
```

Gere o certificado de cliente e a chave privada do `kube-proxy`:

```
{

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

Resultados:

```
kube-proxy-key.pem
kube-proxy.pem
```

### Certificado de do kube scheduller

Gere o certificado e a chave privada do `kube-scheduler`:

```
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "BR",
      "L": "Sao Paulo",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes do Jeito Dificil",
      "ST": "Sao Paulo"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```

Results:

```
kube-scheduler-key.pem
kube-scheduler.pem
```


### Os Certificados dos Servidores de API Kubernetes

O endereço estático de IP do `kubernetes-the-hard-way` será incluído na lista de nomes de sujeitos alternativos para o certificado do Servidor de API do Kubernetes. Isso irá garantir que o certificado possa ser validado por clientes remotos.

Recupere o endereço de IP estático do `kubernetes-the-hard-way`:

```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

Crie a requisição de assinatura do certificado do Servidor de API do Kubernetes:

```
{

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "BR",
      "L": "Sao Paulo",
      "O": "Kubernetes",
      "OU": "Kubernetes do Jeito Dificil",
      "ST": "Sao Paulo"
    }
  ]
}
EOF

}
```

Gere o certificado e a chave privada do Servidor de API do Kubernetes:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

Resultados:

```
kubernetes-key.pem
kubernetes.pem
```
## The Service Account Key Pair


O Kubernetes Controller Manager utiliza um par de chaves para gerar e assinar tokens de conta de serviço, conforme descrito na documentação de [gerenciando service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/).

Gerando o par de chaves para `service-account`:

```
{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "BR",
      "L": "Sao Paulo",
      "O": "Kubernetes",
      "OU": "Kubernetes do Jeito Dificil",
      "ST": "Sao Paulo"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

Results:

```
service-account-key.pem
service-account.pem
```

## Distribua os Certificados de Cliente e Servidor

Copie os certificados e chaves privadas apropriadas para cada instância de _worker_:

```
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done
```

Copie os certificados e chaves privadas apropriadas para cada instância de controladora:

```
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
   ${instance}-key.pem ${instance}.pem \
   service-account-key.pem service-account.pem ${instance}:~/
done
```

> Os certificados de cliente do `kube-proxy`, `kube-controller-manager`, `kube-scheduler` e do `kubelet` serão utilizados para gerar arquivos de configuração para autenticação de clientes no próximo lab.

Próximo: [Gerando Arquivos de Configuração do Kubernetes para Autenticação](05-arquivos-configuracao-kubernetes.md)
