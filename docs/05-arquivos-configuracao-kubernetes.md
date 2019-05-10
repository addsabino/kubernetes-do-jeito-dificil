# Gerando Arquivos de Configuração do Kubernetes para Autenticação

Nesse lab você irá criar [arquivos de configuração do Kubernetes](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), também conhecidos como kubeconfigs, que possibilitam clientes Kubernetes localizarem e autenticarem com os Servidores de API do Kubernetes.

## Configurações de Autenticação do Cliente

Nessa seção você irá criar arquivos kubeconfig para os clientes `kubelet`, `kube-proxy`, `controller manager` e `scheduler`.


### Endereço de IP público do Kubernetes

Cada kubeconfig requer um Servidor de API do Kubernetes para se conectar. Para suportar alta disponibilidade o endereço de IP vinculado ao balanceador de carga externo fazendo frente ao Servidor de API do Kubernetes será utilizado.

Recupere o endereço de IP estático do `kubernetes-the-hard-way`:

```
ENDERECO_PUBLICO_KUBERNETES=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

### O Arquivo kubelet de Configuração do Kubernetes

Ao criar os arquivos kubeconfig, o certificado de cliente que corresponde com o nome do nó do Kubelet deve ser utilizado. Isso irá garantir que os Kubelets serão devidamente autorizados pelo [Autorizador de Nós](https://kubernetes.io/docs/admin/authorization/node/) do Kubernetes.

Crie um arquivo kubeconfig para cada nó :

```
for nodetype in controller worker; do
  for enum in 0 1 2; do
    instance=$nodetype-$enum
    kubectl config set-cluster kubernetes-the-hard-way \
      --certificate-authority=certs/ca.pem \
      --embed-certs=true \
      --server=https://${ENDERECO_PUBLICO_KUBERNETES}:6443 \
      --kubeconfig=config/${instance}.kubeconfig

    kubectl config set-credentials system:node:${instance} \
      --client-certificate=certs/${instance}.pem \
      --client-key=certs/${instance}-key.pem \
      --embed-certs=true \
      --kubeconfig=config/${instance}.kubeconfig

    kubectl config set-context default \
      --cluster=kubernetes-the-hard-way \
      --user=system:node:${instance} \
      --kubeconfig=config/${instance}.kubeconfig

    kubectl config use-context default --kubeconfig=config/${instance}.kubeconfig
  done
done
```

Resultados:

```
config/
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
controller-0.kubeconfig
controller-1.kubeconfig
controller-2.kubeconfig
```

### O Arquivo de Configuração kube-proxy do Kubernetes

Crie um arquivo kubeconfig para o serviço `kube-proxy`:

```
{
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=certs/ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=config/kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
  --client-certificate=certs/kube-proxy.pem \
  --client-key=certs/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=config/kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=kube-proxy \
  --kubeconfig=config/kube-proxy.kubeconfig


kubectl config use-context default --kubeconfig=config/kube-proxy.kubeconfig

}
```

Resultados:

```
kube-proxy.kubeconfig
```

### The kube-controller-manager Kubernetes Configuration File

Gere um arquivo kubeconfig para o serviço `kube-controller-manager` :

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=certs/ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=config/kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=certs/kube-controller-manager.pem \
    --client-key=certs/kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=config/kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=config/kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=config/kube-controller-manager.kubeconfig
}
```

Resultados:

```
kube-controller-manager.kubeconfig
```


### The kube-scheduler Kubernetes Configuration File

Gere um arquivo kubeconfig par a o serviço `kube-scheduler`:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=certs/ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=config/kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=certs/kube-scheduler.pem \
    --client-key=certs/kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=config/kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=config/kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=config/kube-scheduler.kubeconfig
}

```

Results:

```
kube-scheduler.kubeconfig
```

### Configuracão para o admin

Gere um arquivo kubeconfig para o usuário `admin`:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=certs/ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=config/admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=certs/admin.pem \
    --client-key=certs/admin-key.pem \
    --embed-certs=true \
    --kubeconfig=config/admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=config/admin.kubeconfig

  kubectl config use-context default --kubeconfig=config/admin.kubeconfig
}
```

Resultados:

```
admin.kubeconfig
```


##

### Distribua os Arquivos de Configuração do Kubernetes

Copie os arquivos kubeconfig `kubelet` e `kube-proxy` apropriados para cada instância todas as instâncias:

```
for nodetype in controller worker; do
  for enum in 0 1 2; do
    instance=$nodetype-$enum
    gcloud compute scp config/${instance}.kubeconfig config/kube-proxy.kubeconfig ${instance}:~/
  done
done
```

Copie os  arquivos kubeconfig de `kube-controller-manager` e `kube-scheduler` para cada instância controller:

```
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp config/admin.kubeconfig config/kube-controller-manager.kubeconfig \
  config/kube-scheduler.kubeconfig ${instance}:~/
done
```

Próximo: [Gerando a Configuração e a Chave de Encriptação de Dados](06-chaves-encriptacao-dados.md)
