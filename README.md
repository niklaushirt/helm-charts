# LunchBadger Helm Chart Deployment

## Local Deployment Prerequisites
- Minikube
  - 2 GB, 4 CPU VM configuration recommended, i.e. `minikube start --cpus 4 --memory 4096`
  - enable minikube addons
    - registry-cred
    - dashboard (optional)
    - metrics-server (optional)
- Helm
- Kubectl
- AWS credentials and access token (for now this is needed because Docker images are stored in LB AWS ECR)
- awscli

## AWS Deployment Additonal Prerequisites
- Kubernetes 1.10+

## LunchBadger Main Installation Steps
The LunchBadger Helm charts deploy LB microservices to the `default` namespace and creates a `customer` namespace where all user application pods are deployed.

### Configure Helm and Update Dependencies
```
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
helm repo add jfelten https://jfelten.github.io/helm-charts/charts
git clone git@github.com:LunchBadger/helm-charts.git
helm dependency update lunchbadger # this is to load all updated dependencies
```

### Install Tiller
```
kubectl create serviceaccount tiller --namespace kube-system
kubectl apply -f tiller-rbac-config.yaml
helm init --service-account tiller
```

### Helm Values.yaml
There are three directories with sample Helm values.yaml files:
- akt
- sk
- ks

Use the `akt/akt.values.yaml` file to deploy locally to minikube.

### Initial Install Dry Run
```
cd helm-charts
helm dependency update ./lunchbadger && helm install -f <path/to/values.yaml> --debug --name lb --namespace default --dry-run ./lunchbadger > dry-run.txt
```
example using akt Helm values file:
```
helm dependency update ./lunchbadger && helm install -f akt/akt.values.yaml --debug --name lb --namespace default --dry-run ./lunchbadger > dry-run.txt
```
inspect the dry-run.txt file for debug level output to Helm


### Initial Install
```
cd helm-charts
helm dependency update ./lunchbadger && helm install -f <path/to/values.yaml> --debug --name lb --namespace default ./lunchbadger
```

### Upgrade
```
cd helm-charts
helm dependency update ./lunchbadger && helm upgrade -f <path/to/values.yaml> --debug lb  ./lunchbadger
```

### Helpful Helm Commands for Failed Deployments
If you want to get a clean release deployed, it may take several attempts.  Here are some useful commands.
```
helm ls --all lb    # get status on lb release
helm del --purge lb # delete a previous release entirely
```
if you delete a release PersistentVolumeClaims will remain (to safeguard data on purpose), to delete them before re-running use the following:
```
kubectl delete pvc --all --namespace default
```
## Post Installation Steps
There are a number of manual post installation steps that have not been automated and or integrated into the Helm charts

### Install Traefik
Express Gateway acts as the point of Ingress for LunchBadger. Express Gatewa routes directly to Traefik as an ingress controller that dyanmically wires in Kubernetes services.

```
helm install stable/traefik --name traefik --namespace kube-system
```

### Gitea Post Installation Manual Steps
The following steps have not been automated but should be in the future.

#### create Gitea admin user:
```
export POD_NAME=$(kubectl get po -n default -l"app=lb-gitea" -o jsonpath="{.items[0].metadata.name}") && kubectl exec $POD_NAME -c gitea -it -- gitea admin create-user --name=test --password=test --email=test@test.com --admin
```
note: double check the label and namespace, the above example assumes you followed the installation instructions above and used `lb` as a release name and deployed to the `default` namespace

#### generate token
after creating admin user, while ssh'ed into the gitea container:
```
export POD_NAME=$(kubectl get po -n default -l"app=lb-gitea" -o jsonpath="{.items[0].metadata.name}") && kubectl exec $POD_NAME -c gitea -it -- curl -X POST "http://localhost:3000/api/v1/users/test/tokens" -H "accept: application/x-www-form-urlencoded" -H "authorization: Basic dGVzdDp0ZXN0" -F name=ttxxx
```

the returned response form the command above contains a sha1 which is the gitea access key (example below)
```
{"id":7,"name":"ttxxx","sha1":"2283d9f73439c7b34a644197875e1bf84923a960"}
```

#### update GITEA_TOKEN
patch the GITEA_TOKEN value in git-api deployment
```
export DEPLOY_NAME=$(kubectl get deploy -n default -l"app=git-api" -o jsonpath="{.items[0].metadata.name}") && kubectl patch deploy $DEPLOY_NAME --type json -p '[ { "op": "replace", "path": "/spec/template/spec/containers/0/env/1/value", "value": "8cae1e9df6da692abc3b563373efc750268be016"} ]'
```
after updating the GITEA_TOKEN, the `lb-git-api` pod should be re-deployed

## Additional Optional Components

### Install Prometheus

```
helm install ./charts/prometheus \
--name=monitoring \
--set rbac.create=true \
--set server.persistentVolume.storageClass=standard \
--set alertmanager.persistentVolume.storageClass=standard \
--namespace=kube-system
```

### Install Grafana

```
helm install ./charts/grafana \
--name=monitoring-viz \
--set server.persistentVolume.storageClass=standard \
--namespace=kube-system
```

### Install Kubernetes Dashboard
Note: not needed if using minikube addon

```
helm install ./charts/kubernetes-dashboard \
--name=kubernetes-dashboard \
--set rbac.create=true \
--namespace=kube-system
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

## Additional Notes
The current Helm chart is unfinished work, adjustments were being made to target
EKS, and Kubernetes 1.10+

### AWS Load Balancers
SSH for git clone, https and http for client and k8s APIs must be exposed via AWS ELB otherwise it is not accessible

In this helm chart check for exposure via ALB attibutes
https://github.com/LunchBadger/helm-charts/blob/cb53fa320fd3234d7d3c44e6ff717764ed68bf81/lunchbadger/gateway/templates/service.yaml#L10

To expose Gitea reposistories, the gitea-ssh-service must be expoesd for SSH forwarding.

### Snapshots
Once volume are created ensure (and automate in future) backup rules to snapshot volumes.

Ensure these volumes are added:
- redis-data-lb-redis-master-0
- gitea-postgres
- lb-gitea

Manual Rule creation guide
https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/TakeScheduledSnapshot.html

## Automation TODO
- Create LunchBadger serviceaccount instead of using default
- remove traefik
- migrate Kubeless to official k8s chart

## Highly Recommended Kubernetes Tooling:
- [kubectx and kubens](https://github.com/ahmetb/kubectx) - context and namespace switching
- [k9s](https://k9ss.io/) - cli cluster management
- [stern](https://github.com/wercker/stern) - pod/container log viewing and tailing
