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

6. Install Grafana

```
helm install ./charts/grafana \
--name=monitoring-viz \
--set server.persistentVolume.storageClass=standard \
--namespace=kube-system
```

7. Install Kubernetes Dashboard

```
helm install ./charts/kubernetes-dashboard \
--name=kubernetes-dashboard \
--set rbac.create=true \
--namespace=kube-system
```

8. Install LunchBadger

```
helm install . \
--name=lunchbadger \
--set storageClass.enabled=true \
--set storageClass.name=standard \
--set "storageClass.zones=us-west-2b\, us-west-2c" \
--set gateway.traefikAddress=$TRAEFIK_IP \
--set actualizer.redisPassword=$REDIS_PASSWORD \
--set actualizer.customerDomain=dev.lunchbadger.io \
--set kube-watcher.ingressHost=kube-watcher.dev-api.lunchbadger.com \
--set configstore.ingressHost=dev-api.lunchbadger.com \
--set configstore.persistence.storageClass=standard
```


## Grafana

1. Get password

Note: Username is `admin`.

```
kubectl get secret --namespace kube-system monitoring-viz-grafana -o jsonpath="{.data.grafana-admin-password}" | base64 --decode ; echo
```

2. Get pod name

```
export POD_NAME=$(kubectl get pods --namespace kube-system -l "app=monitoring-viz-grafana,component=grafana" -o jsonpath="{.items[0].metadata.name}")
```

3. Forward port

```
kubectl --namespace=kube-system port-forward $POD_NAME 3000 
```

Go to http://localhost:3000 to login.

## Kubernetes Dashboard

1. Run proxy

```
kubectl proxy
```

2. Access URL

http://localhost:8001/api/v1/namespaces/kube-system/services/kubernetes-dashboard:/proxy/
