# llabdocs-helm

1  Install ArgoCD:
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
2 Install Certificate Manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.2/cert-manager.yaml
```

3 Install ingress nginx
```bash
helm upgrade --install ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--namespace ingress-nginx --create-namespace
```
4 Create namespace for application llabdocs
```bash
kubectl create namespace backend
```

1  Verify Ingress Controller is Running:
```bash
kubectl get svc -n ingress-nginx
```

2  Create secret in namespace:
```bash
kubectl create secret generic do-registry \
--from-file=.dockerconfigjson=docker-config.json \
--type=kubernetes.io/dockerconfigjson \
--namespace backend
```

3  Get LB details:
```bash
doctl compute load-balancer list
doctl compute load-balancer get <lb-id>
doctl compute certificate list
```
4  Edit ingress controller ConfigMap:
```bash
kubectl describe configmap ingress-nginx-controller -n ingress-nginx
kubectl patch configmap ingress-nginx-controller -n ingress-nginx --patch '{"data":{"allow-snippet-annotations":"true"}}'
```
5  K8s Dashboard port forwarding:
```bash
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```
6 ArgoCD port forwarding
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
7 Get password, username admin
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
8 Check helm template:
```bash
helm template llabdocs . --debug
```