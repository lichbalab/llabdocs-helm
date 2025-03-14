# llabdocs-helm

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