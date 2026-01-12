# Java + MySQL + Kubernetes Demo

This project is a simple Spring Boot REST API connected to a MySQL database, containerized with Docker and deployable to Kubernetes using a `Deployment` for the app and a `StatefulSet` for MySQL.

## Project structure

- `pom.xml` – Maven build configuration (Spring Boot, Web, Data JPA, MySQL, Test)
- `src/main/java/com/example/demo`  
  - `DemoApplication.java` – main Spring Boot application  
  - `User.java` – JPA entity  
  - `UserRepository.java` – Spring Data JPA repository  
  - `UserController.java` – REST controller (`/api/users`)
- `src/main/resources/application.properties` – DB + JPA configuration (using environment variables)
- `src/test/java/com/example/demo/DemoApplicationTests.java` – basic Spring test
- `Dockerfile` – multi-stage Docker build for the app
- `k8s/` – Kubernetes manifests:  
  - `mysql-secret.yaml` – MySQL credentials (Secret)  
  - `mysql-configmap.yaml` – DB name (ConfigMap)  
  - `mysql-pvc.yaml` – PersistentVolumeClaim for MySQL data  
  - `mysql-statefulset.yaml` – MySQL `StatefulSet` + Service  
  - `app-deployment.yaml` – app `Deployment`  
  - `app-service.yaml` – app `Service` (NodePort)

## Prerequisites

- Java 17+
- Maven 3.8+
- Docker
- Kubernetes cluster (minikube, Docker Desktop, microk8s, etc.)
- `kubectl` configured to talk to your cluster

On Windows PowerShell, run the commands in this README from the project root:  
`C:\Users\admin\java-mysql-k8s-demo`

---

## 1. Build and test locally

From the project root:

```powershell
mvn clean test
```

Run the app (requires a MySQL instance reachable at `localhost:3306` or you adjust `application.properties`):

```powershell
mvn spring-boot:run
```

Test the API (example with PowerShell + `Invoke-WebRequest`):

```powershell
# Get users
Invoke-WebRequest -Uri "http://localhost:8080/api/users" -Method GET

# Create user
Invoke-WebRequest -Uri "http://localhost:8080/api/users" -Method POST \
  -ContentType "application/json" \
  -Body '{"name":"Alice"}'
```

---

## 2. Build Docker image

From the project root:

```powershell
# Replace YOUR_DOCKERHUB_USERNAME with your Docker Hub username
docker build -t YOUR_DOCKERHUB_USERNAME/java-mysql-demo:1.0.0 .
```

(Optional) Test the app with a local MySQL container:

```powershell
# Start MySQL container
docker run -d --name mysql-demo `
  -e MYSQL_ROOT_PASSWORD=rootpass `
  -e MYSQL_DATABASE=demo `
  -e MYSQL_USER=demo `
  -e MYSQL_PASSWORD=demopass `
  -p 3306:3306 `
  mysql:8.0

# Start app container
docker run -d --name java-demo `
  -e DB_HOST=host.docker.internal `
  -e DB_PORT=3306 `
  -e DB_NAME=demo `
  -e DB_USER=demo `
  -e DB_PASSWORD=demopass `
  -p 8080:8080 `
  YOUR_DOCKERHUB_USERNAME/java-mysql-demo:1.0.0
```

Test again at `http://localhost:8080/api/users`.

---

## 3. Push Docker image to registry

```powershell
docker login

docker push YOUR_DOCKERHUB_USERNAME/java-mysql-demo:1.0.0
```

Update `k8s/app-deployment.yaml` to use this image name (replace `YOUR_DOCKERHUB_USERNAME`).

---

## 4. Deploy MySQL to Kubernetes (StatefulSet)

Apply Secret, ConfigMap, PVC, and StatefulSet manifests:

```powershell
kubectl apply -f k8s/mysql-secret.yaml
kubectl apply -f k8s/mysql-configmap.yaml
kubectl apply -f k8s/mysql-pvc.yaml
kubectl apply -f k8s/mysql-statefulset.yaml
```

Check that the MySQL pod is running:

```powershell
kubectl get pods
```

You should see something like:

- `mysql-0` in `Running` status

---

## 5. Deploy the Java app to Kubernetes

Make sure `k8s/app-deployment.yaml` references your pushed image:

```yaml
image: YOUR_DOCKERHUB_USERNAME/java-mysql-demo:1.0.0
```

Apply the Deployment and Service:

```powershell
kubectl apply -f k8s/app-deployment.yaml
kubectl apply -f k8s/app-service.yaml
```

Check status:

```powershell
kubectl get pods
kubectl get svc
```

You should see:

- A pod with label `app=demo-app` in `Running` state
- Service `demo-app` of type `NodePort` exposing port `30080`

> Note: The readiness/liveness probes in `app-deployment.yaml` point to `/actuator/health`.  
> You can either add `spring-boot-starter-actuator` to `pom.xml` or change the probe paths to `/api/users`.

---

## 6. Access the app in Kubernetes

### If using minikube

Get the service URL:

```powershell
minikube service demo-app --url
```

Open the printed URL in a browser, appending `/api/users`.

### If using another cluster

Use the node IP and NodePort (default `30080`):

```powershell
# Replace <node-ip> with the IP of any Kubernetes node
Invoke-WebRequest -Uri "http://<node-ip>:30080/api/users" -Method GET

Invoke-WebRequest -Uri "http://<node-ip>:30080/api/users" -Method POST \
  -ContentType "application/json" \
  -Body '{"name":"Bob"}'
```

If you can create and retrieve users, the Java app and MySQL `StatefulSet` are successfully connected.

---

## 7. Clean up

To remove all created resources from the cluster:

```powershell
kubectl delete -f k8s/app-service.yaml
kubectl delete -f k8s/app-deployment.yaml
kubectl delete -f k8s/mysql-statefulset.yaml
kubectl delete -f k8s/mysql-pvc.yaml
kubectl delete -f k8s/mysql-configmap.yaml
kubectl delete -f k8s/mysql-secret.yaml
```
