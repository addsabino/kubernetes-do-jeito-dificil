# Limpeza

Nesse lab você irá deletar os recursos computacionais criados durante esse tutorial.

## Instâncias Computacionais

Delete a controladora e a instância computacional dos _workers_:

```
zone=(a b c)
for enum in 0 1 2; do
  instance=worker-${enum}
  gcloud -q compute instances delete  $instance \
    --zone southamerica-east1-${zone[$enum]}
  instance=controller-${enum}
  gcloud -q compute instances delete  $instance \
    --zone southamerica-east1-${zone[$enum]}
done
```

## Rede

Remova os recursos de rede do balanceador de carga externo:

```
{
  gcloud -q compute forwarding-rules delete kubernetes-forwarding-rule \
    --region $(gcloud config get-value compute/region)

  gcloud -q compute target-pools delete kubernetes-target-pool

  gcloud -q compute http-health-checks delete kubernetes

  gcloud -q compute addresses delete kubernetes-the-hard-way
}
```

Remova as regras de firewall do `kubernetes-the-hard-way`:

```
gcloud -q compute firewall-rules delete \
  kubernetes-the-hard-way-allow-nginx-service \
  kubernetes-the-hard-way-allow-internal \
  kubernetes-the-hard-way-allow-external \
  kubernetes-the-hard-way-allow-health-check
```

Remova a rede VPC `kubernetes-the-hard-way`:

```
{
  gcloud -q compute routes delete \
    kubernetes-route-10-200-0-0-24 \
    kubernetes-route-10-200-1-0-24 \
    kubernetes-route-10-200-2-0-24

  gcloud -q compute networks subnets delete kubernetes

  gcloud -q compute networks delete kubernetes-the-hard-way
}
```
