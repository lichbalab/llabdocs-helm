
# LLab Docs Application

## Overview
LLab Docs Application is an application designed to handle document signing and signature validation. 

## Prerequisites
- Java 11 or higher
- Maven 3.6.0 or higher

## Project Structure
LLab Docs Application is a Spring Boot application designed to handle document signing and signature validation. It leverages the `cmc-spring-sdk` for integration with CMC services.

## Prerequisites
- Java 11 or higher
- Maven 3.6.0 or higher

## Setup

### 1. Clone the Repository
git clone https://github.com/lichbalab/llab-docs.git
cd llab-docs

### 2. Build the Project

```bash
mvn clean package
```

### 3. Run the Application

```bash
mvn spring-boot:run -Dspring-boot.run.arguments="--server.port=8081 --cmc.sdk.base-url=http://localhost:8080"
```

### 4. Build Docker Image
If you prefer Docker, you can build an image using:
```bash
docker build -t lichbalab:llabdocs-2024.1 .
docker build -t registry.digitalocean.com/lichbalab-registry/lichbalab:llabdocs-2024.1 .
docker push registry.digitalocean.com/lichbalab-registry/lichbalab:llabdocs-2024.1
docker buildx create --use                                                                     
docker buildx build --platform linux/amd64,linux/arm64 -t registry.digitalocean.com/lichbalab-registry/lichbalab:llabdocs-2024.1 --push .

```
This command builds a Docker image tagged `lichbalab:llabdocs-2024.1`.

### 5. Run Docker Image
Run the Docker image with:
```bash
docker-compose up -d
```
This starts the Docker container in detached mode.

### 6. Deploy the Application in K8s
Run the Docker image with:
```bash
kubectl apply -f kubernetes-logs-pvc.yaml
kubectl apply -f kubernetes-deployment.yaml 
```

### 7. Scale Application
Run the Docker image with:
```bash
kubectl scale deployment llabdocs-deployment --replicas=5 
```

### 8. Get Service details 
```bash
kubectl get svc llabdocs-service
```

### 9. Start K8S GUI
```bash
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443

kubectl apply -f dashboard-adminuser.yaml

kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath="{.data.token}" | base64 -d

```

### 10. View logs

To view logs directly from the running pod:
```bash
kubectl logs -f deployment/llabdocs-deployment  
```

To check the contents of the logs directory:
```bash
kubectl exec -it deployment/llabdocs-deployment -- ls -la /logs   
```

To check the contents of the logs directory:
```bash
kubectl exec -it deployment/llabdocs-deployment -- cat /logs/application.log   
kubectl exec -it deployment/llabdocs-deployment -- tail -f /logs/application.log     
```

Copy logs from the pod to your local machine:
```bash
kubectl cp default/$(kubectl get pod -l app=llabdocs -o jsonpath='{.items[0].metadata.name}'):/logs/application.log ./application.log
```

Get Services:
```bash
 kubectl get services 
```

### 10. Deploy build to Cloud 

1 Build package:
```bash
mvn clean install -DskipTests=true docker buildx build --platform linux/amd64 -t registry.digitalocean.com/lichbalab-registry/lichbalab:llabdocs-2024.1 --push .
```

2 Build image and push it to registry:
```bash
docker buildx create --use
docker buildx build --platform linux/amd64 -t registry.digitalocean.com/lichbalab-registry/lichbalab:llabdocs-2024.1 --push .
```

3 Apply deployment:
```bash
kubectl apply -f kubernetes-deployment.yaml
```
4 Get logs
```bash
kubectl logs <pod-code>
```


