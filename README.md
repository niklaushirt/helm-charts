
# short version
`cd helm-charts`
`helm dependency update ./lunchbadger && helm upgrade -f path/to/values.yaml --debug lb  ./lunchbadger`

#Automation TODO
- Create lunchbadger SA instead of default
- remove traefik
- migrate kubeless to official k8s chart
# lunchbadger helm-charts


## Installation Steps
0. Configure helm
`helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/`
`helm repo add jfelten https://jfelten.github.io/helm-charts/charts`
`cd helm-charts/lunchbadger`
`helm dependency update`  - this is to load all updated dependencies

0A. Run in test mode:

`helm dependency update ./lunchbadger && helm install -f ./sk/sk.values.yaml --debug  --name lb  ./lunchbadger --dry-run`

0B Gitea after install 
# create admin user:
# ssh into pod : gitea admin create-user --name=test --password=test --email=test@xx.com --admin

# generate token
# curl -X POST "http://localhost:3000/api/v1/users/test/tokens" -H "accept: application/x-www-form-urlencoded" -H "authorization: Basic dGVzdDp0ZXN0" -F name=ttxxx

# response sha1 is the access key
# {"id":7,"name":"ttxxx","sha1":"2283d9f73439c7b34a644197875e1bf84923a960"}
# update git-api deployment manually

1. Install Traefik (not required. should work with nginx)

```
helm install ./charts/traefik --name=traefik
```


2. Grab Traefik IP.

```
export TRAEFIK_IP=$(kubectl get svc traefik-ingress-service -n kube-system -o jsonpath="{.spec.clusterIP}")
```
3. Redacted: Redis secret is now mounted to actualizer

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
--set gateway.ingressAddress=$TRAEFIK_IP \
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
