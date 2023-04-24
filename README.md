# Apisix Deployment

This repository based on [Apisix Helm Chart](https://github.com/apache/apisix-helm-chart) and [Casdoor K8s](https://raw.githubusercontent.com/casdoor/casdoor/master/k8s.yaml)

## Pre-Deployment
Make sure you have docker, kubectl, kind and helm installed and running.

*Note*: [Kind](https://kind.sigs.k8s.io/) is a tool for running local Kubernetes clusters using Docker containers "nodes". kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.
For MacOS, simply run `brew install kind`

## Deployment
### 1. Create a local K8s

```bash
cd deployment

kind create cluster
```

2. Install Kubernetes dashboard (optional)
- To visualize the deployment during the installation process, you should install Kubernetes dashboard

```bash
cd deployment/dashboard


helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm install dashboard kubernetes-dashboard/kubernetes-dashboard -n kubernetes-dashboard --create-namespace
```

- Access dashboard

```bash
kubectl proxy
```

- Kubectl will make Dashboard available at http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:dashboard-kubernetes-dashboard:https/proxy/#/login.

Create token to log in to dashboard:

```bash
kubectl apply -f kube-dashboard.yaml
kubectl -n kubernetes-dashboard create token admin-user

# copy token to login to dashboard
```

### 2. Install Apisix
The script below installs APISIX:


```bash
cd deployment/apisix
helm package .

helm upgrade --install apisix apisix-1.3.1.tgz \
  --create-namespace \
  --namespace apisix \
  --set gateway.type=NodePort \
  --set dashboard.enabled=true
  
kubectl get service --namespace apisix
```

Since we uses `kind` to build a local K8s cluster, the `apisix-gateway` NodePort is not accessible, an additional step is needed before validation, i.e. forwarding port 80 from the cluster to port 9000 on the local machine.

```bash
kubectl port-forward service/apisix-dashboard 9000:80 --namespace apisix
```

Access http://localhost:9000 and login to APISIX Dashboard (admin/admin).

Create Routing with `authz-casdoor` plugin via Apisix Admin API endpoint
```bash
curl "http://127.0.0.1:9180/apisix/admin/routes/1" -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -X PUT -d '
{
  "methods": ["GET"],
  "uri": "/web1/*",
  "plugins": {
    "authz-casdoor": {
        "endpoint_addr":"http://localhost:8000",
        "callback_url":"http://localhost:9080/web1/callback",
        "client_id":"7ceb9b7fda4a9061ec1c",
        "client_secret":"3416238e1edf915eac08b8fe345b2b95cdba7e04"
    }
  },
  "upstream": {
    "type": "roundrobin",
    "nodes": {
      "httpbin.org:80": 1
    }
  }
}'
```

### 3. Install Casdoor
In the root of directory, run the commands below:
```bash
cd deployment/portal-casdoor

helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install mysql oci://registry-1.docker.io/bitnamicharts/mysql \
    --create-namespace \
    --namespace apisix \
    --set auth.rootPassword="password" \
    --set auth.database=casdoor

kubectl apply -f k8s.yaml --namespace=apisix

kubectl get service --namespace apisix
```

And soon you can see the result via command `kubectl get pods`
Access url `http://localhost:8000` to see the dashboard, login with username/password `admin/123`

Now you can try access `http://localhost:9080/web1/anything`, the browser will redirect you to the login, for set up application to authnz and authorize please go to next chapter

### 4. Setup
Follow this docs for more information [Apisix vs Casdoor](https://confluence.zalopay.vn/display/ZTM/%5BPCT-CE%5D+Apisix+integrate+with+Casdoor)
