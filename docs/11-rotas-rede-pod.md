# Provisionando Rotas de Rede do Pod

Pods agendados para um nó recebem um endereço de IP da faixa de CIDR do Nó do Pod em questão. A essa altura, Pods não podem se comunicar com outros Pods em execução em nós diferentes devido a [rotas](https://cloud.google.com/compute/docs/vpc/routes) de rede faltantes.

Nesse lab você irá criar uma rota para cada nó _worker_ que mapeia a faixa CIDR do nó do Pod para o endereço de IP interno do nó.

> Existem [outras maneiras](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) de implementar o modelo de rede do Kubernetes.

## A Tabela de Roteamento

Nessa seção você irá coletar a informação necessária para criar rotas na rede VPC `kubernetes-the-hard-way`.

Imprima o endereço de IP interno e a faixa de CIDR do Pod para cada instância _worker_:

```
function routes(){
zone=(a b c)
for enum in 0 1 2; do
  for type in worker controller ; do
  instance=$type-$enum
  gcloud compute instances describe ${instance} \
    --format 'value[separator=" "](networkInterfaces[0].networkIP,metadata.items[0].value)'\
    --zone southamerica-east1-${zone[$enum]}
done
done
}
routes
```

> saída

```
10.240.0.20 10.200.0.0/24
10.240.0.10 10.200.201.0/24
10.240.0.21 10.200.1.0/24
10.240.0.11 10.200.202.0/24
10.240.0.22 10.200.2.0/24
10.240.0.12 10.200.203.0/24
```

## Rotas

Crie rotas de rede para cada instância _worker_:

```
routes |while read host route  ; do
  nome=$( echo $route|sed -e 's/\./-/g;s/\//-/')
  gcloud compute routes create kubernetes-route-$nome \
    --network kubernetes-the-hard-way \
    --next-hop-address $host\
    --destination-range $route
done
```

Liste as rotas na rede VPC `kubernetes-the-hard-way`:

```
gcloud compute routes list --filter "network: kubernetes-the-hard-way"
```

> saída

```
NAME                            NETWORK                  DEST_RANGE     NEXT_HOP                  PRIORITY
default-route-440209973dd9874f  kubernetes-the-hard-way  10.240.0.0/24  kubernetes-the-hard-way   1000
default-route-baa92d48e6cb2d8f  kubernetes-the-hard-way  0.0.0.0/0      default-internet-gateway  1000
kubernetes-route-10-200-0-0-24  kubernetes-the-hard-way  10.200.0.0/24  10.240.0.20               1000
kubernetes-route-10-200-1-0-24  kubernetes-the-hard-way  10.200.1.0/24  10.240.0.21               1000
kubernetes-route-10-200-2-0-24  kubernetes-the-hard-way  10.200.2.0/24  10.240.0.22               1000
```

Próximo: [Implantando o Add-on de DNS do Cluster](12-dns-addon.md)
