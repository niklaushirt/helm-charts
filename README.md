# lunchbadger helm-charts

## Installation Steps

1. Install Traefik

```
helm install ./charts/traefik --name=traefik
```


2. Grab Traefik IP.

```
export TRAEFIK_IP=$(kubectl get svc traefik-ingress-service -n kube-system -o jsonpath="{.spec.clusterIP}")
```

3. Install Redis

```
helm install ./charts/redis \
--name=eg-identity \
--set persistence.storageClass=standard
```

4. Grab Redis password

```
export REDIS_PASSWORD=$(kubectl get secret --namespace default eg-identity-redis -o jsonpath="{.data.redis-password}" | base64 --decode)
```

5. Install Prometheus

```
helm install ./charts/prometheus \
--name=monitoring \
--set rbac.create=true \
--set server.persistentVolume.storageClass=standard \
--set alertmanager.persistentVolume.storageClass=standard \
--namespace=kube-system
```

5. Install LunchBadger

```
helm install . \
--name=lunchbadger \
--set storageClass.enabled=true
--set storageClass.name=standard
--set gateway.traefikAddress=$TRAEFIK_IP \
--set actualizer.redisPassword=$REDIS_PASSWORD \
--set configstore.ingressHost=dev-api.lunchbadger.com \
--set configstore.persistence.storageClass=standard
```
