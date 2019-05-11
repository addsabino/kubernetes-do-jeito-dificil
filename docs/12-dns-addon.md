# Implantando o Add-on de DNS do Cluster

Nesse lab você irá implantar o [Add-on de DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) que provê descoberta de serviços baseadas em DNS, fornecido pelo [CoreDNS](https://coredns.io/), para aplicações executando dentro do cluster do Kubernetes.

## O Add-on de DNS do Cluster

Implante o add-on de cluster `coredns`:

```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml
```

> saída

```
service "kube-dns" created
serviceaccount "kube-dns" created
configmap "kube-dns" created
deployment.extensions "kube-dns" created
```

Liste os pods criados pela implantação `kube-dns`:

```
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> saída

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-699f8ddd77-j7mhr   1/1     Running   0          24s
coredns-699f8ddd77-xtc7b   1/1     Running   0          24s
```

## Verificação

Crie uma implantação `busybox`

```
kubectl run busybox --image=busybox --command -- sleep 3600
```

Liste o pod criado pela implantação `busybox`:

```
kubectl get pods -l run=busybox
```

> saída

```
NAME                       READY     STATUS    RESTARTS   AGE
busybox-2125412808-mt2vb   1/1       Running   0          15s
```

Recupere o nome completo do pod `busybox`:

```
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Execute uma busca de DNS pelo serviço `kubernetes`  dentro do pod `busybox`:

```
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> saída

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

Próximo: [Smoke Test (Teste de Fumaça)](13-smoke-test.md)
