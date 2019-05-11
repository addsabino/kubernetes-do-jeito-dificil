# Configurando o kubectl para Acesso Remoto

Nesse lab vorê irá gerar um arquivo kubeconfig para o utilitário de linha de comando `kubectl` baseado nas credenciais `admin` de usuário.

> Execute os comandos nesse lab a partir do mesmo diretório utilizado para gerar os certificados admin dos clientes.

## O Arquivo de Configuração Admin do Kubernetes

Cada kubeconfig requer um Servidor de API do Kubernetes para se conectar. Para suportar alta disponibilidade, o endereço de IP atribuído ao balanceador de carga externo fazendo frente aos Servidores de API do Kubernetes será utilizado.

> Execute os comandos no diretório usado para gerar os certificados

```
{
  KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
    --region $(gcloud config get-value compute/region) \
    --format 'value(address)')

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```

## Verificação

Valide a saúde do cluster remoto do Kubernetes:

```
kubectl get componentstatuses
```

> saída

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

Liste os nós no cluster remoto do Kubernetes:

```
kubectl get nodes
```

> saída

```
NAME       STATUS    ROLES     AGE       VERSION
worker-0   Ready     <none>    2m        v1.8.0
worker-1   Ready     <none>    2m        v1.8.0
worker-2   Ready     <none>    2m        v1.8.0
```

Próximo: [Provisionando Rotas de Rede do Pod](11-rotas-rede-pod.md)
