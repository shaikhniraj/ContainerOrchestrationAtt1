# Microservices Kubernetes Deployment on Minikube

Four containerized Node.js microservices deployed to a local Kubernetes cluster (Minikube). The Gateway Service is the single entry point; it reaches the other three services through Kubernetes DNS using their Service names.

Source repository: <https://github.com/shaikhniraj/ContainerOrchestrationAtt1>

## Contents

1. [Architecture](#architecture)
2. [Submission structure](#submission-structure)
3. [Prerequisites](#prerequisites)
4. [Step 1 – Minikube setup](#step-1--minikube-setup)
5. [Step 2 – Build and load the images](#step-2--build-and-load-the-images)
6. [Step 3 – Deploy with kubectl apply](#step-3--deploy-with-kubectl-apply)
7. [Step 4 – Test the services](#step-4--test-the-services)
8. [Step 5 – Logs](#step-5--logs)
9. [Manifest summary](#manifest-summary)
10. [Troubleshooting](#troubleshooting)
11. [Cleanup](#cleanup)

## Architecture

```
                    host machine
                         |
        kubectl port-forward 3003  /  NodePort 30003
                         |
                +-----------------+
                | gateway-service |  :3003  (NodePort)
                +-----------------+
                 /       |        \
   /api/users   /        | /api/products   \  /api/orders
               v         v                  v
      +--------------+ +-----------------+ +---------------+
      | user-service | | product-service | | order-service |
      |    :3000     | |      :3001      | |     :3002     |
      |  ClusterIP   | |    ClusterIP    | |   ClusterIP   |
      +--------------+ +-----------------+ +---------------+
```

| Service | Port | Service type | Endpoints | In-cluster DNS name |
|---|---|---|---|---|
| User Service | 3000 | ClusterIP | `GET /users`, `GET /health` | `http://user-service:3000` |
| Product Service | 3001 | ClusterIP | `GET /products`, `GET /health` | `http://product-service:3001` |
| Order Service | 3002 | ClusterIP | `GET /orders`, `POST /orders`, `GET /health` | `http://order-service:3002` |
| Gateway Service | 3003 | NodePort (30003) | `GET /api/users`, `GET /api/products`, `GET /api/orders`, `POST /api/orders`, `GET /health` | `http://gateway-service:3003` |

The three backend services are internal only (ClusterIP). Only the gateway is exposed outside the cluster.

## Submission structure

```
submission/
├── deployments/
│   ├── user-service.yaml
│   ├── product-service.yaml
│   ├── order-service.yaml
│   └── gateway-service.yaml
├── services/
│   ├── user-service.yaml
│   ├── product-service.yaml
│   ├── order-service.yaml
│   └── gateway-service.yaml
├── screenshots/
│   ├── pods.png
│   ├── logs.png
│   ├── service-test.png
│   └── (additional setup, build and test evidence)
└── README.md
```

The bonus Ingress task was not attempted. The application is reached with `kubectl port-forward` (or the gateway NodePort).

## Prerequisites

- Docker Desktop
- Minikube (v1.39.0 used here)
- `kubectl`
- PowerShell (the commands below use Windows path syntax)

The application source code and Dockerfiles are in the repository under `Microservices-Task-main/Microservices/`. All commands are run from the repository root.

## Step 1 – Minikube setup

Start a single-node cluster with the Docker driver and confirm the node is `Ready`:

```powershell
minikube start --driver=docker --cpus=2 --memory=2048
kubectl get nodes
```

![Minikube start and node Ready](./screenshots/Kubernets-Install.png)

## Step 2 – Build and load the images

Build one image per service. The tags must match the `image:` field in the Deployment manifests.

```powershell
docker build -t user-service:v1    .\Microservices-Task-main\Microservices\user-service
docker build -t product-service:v1 .\Microservices-Task-main\Microservices\product-service
docker build -t order-service:v1   .\Microservices-Task-main\Microservices\order-service
docker build -t gateway-service:v1 .\Microservices-Task-main\Microservices\gateway-service
```

![Build of user-service image](./screenshots/docker-build-1.png)

![Build of product-service and order-service images](./screenshots/docker-build-2.png)

![Build of gateway-service image](./screenshots/docker-build-3.png)

The images exist only in the local Docker daemon, so copy them into the Minikube node. The Deployments use `imagePullPolicy: IfNotPresent`, so Kubernetes uses these local images and does not try to pull from a registry.

```powershell
minikube image load user-service:v1
minikube image load product-service:v1
minikube image load order-service:v1
minikube image load gateway-service:v1
```

![Images loaded into Minikube](./screenshots/image-load.png)

## Step 3 – Deploy with kubectl apply

```powershell
kubectl apply -f .\submission\deployments\
kubectl apply -f .\submission\services\
```

![kubectl apply output: 4 deployments and 4 services created](./screenshots/kubectl-deploy.png)

Check that every pod is `1/1 Running` and that the Services exist:

```powershell
kubectl get pods -o wide
kubectl get services
```

![Running pods and services](./screenshots/pods.png)

All four pods are `Running` and `READY 1/1`, which means each one passed its readiness probe. The three backend Services are `ClusterIP`; `gateway-service` is `NodePort` (`3003:30003`).

## Step 4 – Test the services

### Option A – Port-forward the gateway

In one terminal, forward the gateway Service to the host and leave it running:

```powershell
kubectl port-forward service/gateway-service 3003:3003
```

In a second terminal, call each route through the gateway:

```powershell
Invoke-RestMethod http://localhost:3003/api/users
Invoke-RestMethod http://localhost:3003/api/products
Invoke-RestMethod http://localhost:3003/api/orders
```

`curl.exe http://localhost:3003/api/users` works as well.

![Gateway returning users and products](./screenshots/service-test.png)

The users come from `user-service` and the products from `product-service`; the gateway fetched them over the cluster network. `/api/orders` returns an empty list because no order has been created yet.

Create an order through the gateway and read it back:

```powershell
Invoke-RestMethod -Method Post `
  -Uri http://localhost:3003/api/orders `
  -ContentType "application/json" `
  -Body '{"userId":1,"productId":1}'

Invoke-RestMethod http://localhost:3003/api/orders
```

![Order created and then listed through the gateway](./screenshots/test-order-services.png)

The `POST` travels host → gateway-service → order-service, and the follow-up `GET` returns the stored order. This confirms inter-service communication in both read and write paths.

### Option B – Call services directly by Service name

The backend Services are not reachable from the host, but any pod can call them by name. The images are Alpine-based, so `wget` is available inside the containers:

```powershell
kubectl exec deploy/gateway-service -- wget -qO- http://user-service:3000/users
kubectl exec deploy/gateway-service -- wget -qO- http://product-service:3001/products
kubectl exec deploy/gateway-service -- wget -qO- http://order-service:3002/orders
kubectl exec deploy/order-service   -- wget -qO- http://product-service:3001/health
```

Expected output of the first command:

```json
[{"id":1,"name":"John Doe"},{"id":2,"name":"Jane Smith"}]
```

### Option C – NodePort

The gateway is also published on NodePort `30003`. With the Docker driver on Windows the node IP is not directly routable, so let Minikube open a tunnel:

```powershell
minikube service gateway-service --url
```

## Step 5 – Logs

```powershell
kubectl logs deployment/gateway-service
kubectl logs deployment/user-service
kubectl logs deployment/product-service
kubectl logs deployment/order-service
```

![Gateway service log](./screenshots/logs.png)

The provided applications log only their startup line (`Gateway service running on port 3003`) and do not log individual requests. The log shows the container started and is listening; the proof of service-to-service communication is the successful gateway responses in Step 4.

## Manifest summary

Every Deployment contains:

| Requirement | Implementation |
|---|---|
| Container image | `<service-name>:v1`, `imagePullPolicy: IfNotPresent` |
| Resource requests | `100m` CPU, `128Mi` memory |
| Resource limits | `250m` CPU, `256Mi` memory |
| Environment variables | `NODE_ENV=production`, `PORT`; order and gateway also get `USER_SERVICE_URL`, `PRODUCT_SERVICE_URL` (gateway: `ORDER_SERVICE_URL`) |
| Readiness probe | `GET /health` on the service port, starts after 5s, every 10s |
| Liveness probe | `GET /health` on the service port, starts after 15s, every 20s |
| Labels and selectors | `app: <service-name>` on the Deployment, the pod template and the Service selector |

Service discovery works because each Service name (`user-service`, `product-service`, `order-service`) matches the hostname the gateway code calls, and each Service port matches the container port.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ErrImagePull` / `ImagePullBackOff` | Image is not inside Minikube, or the tag differs from the manifest | `minikube image ls` to check, then `minikube image load <name>:v1` |
| Pod `Running` but `0/1` ready | Readiness probe failing | `kubectl describe pod <pod>` and read the Events section; check the `/health` path and port |
| `CrashLoopBackOff` | Application exits on start | `kubectl logs <pod> --previous` |
| Gateway returns `{"error":"Error fetching users"}` | Backend pod not ready, or Service name/port mismatch | `kubectl get endpoints` — each Service must list a pod IP; an empty list means the selector does not match the pod labels |
| `port-forward` fails with "address already in use" | Port 3003 is taken on the host | Use another local port: `kubectl port-forward service/gateway-service 8080:3003` |
| `localhost:3003` refuses the connection | Port-forward terminal was closed | Restart `kubectl port-forward` and keep it open |
| `kubectl` cannot connect to the cluster | Minikube is stopped | `minikube status`, then `minikube start` |
| Code change not picked up | Old image still cached under the same tag | Rebuild, `minikube image load` again, then `kubectl rollout restart deployment/<name>` |

Useful general commands:

```powershell
kubectl get all
kubectl describe deployment <name>
kubectl get events --sort-by=.lastTimestamp
```

## Cleanup

```powershell
kubectl delete -f .\submission\services\
kubectl delete -f .\submission\deployments\
minikube stop
```
