# 🐳 Docker & Kubernetes

## Docker Core Concepts

### Image vs Container
- **Image** = blueprint (read-only layers)
- **Container** = running instance of an image

### Dockerfile Best Practices
```dockerfile
# ✅ Good Dockerfile (multi-stage, minimal)
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .

# Non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "server.js"]
```

**Key rules:**
- Use specific tags, never `latest` in production
- Order layers: rarely-changing → frequently-changing
- Use `.dockerignore` to exclude `node_modules`, `.git`, etc.
- One process per container
- Multi-stage builds to keep image size small

---

## 📋 Docker Cheatsheet

```bash
# Build & Run
docker build -t myapp:1.0 .
docker run -d -p 8080:3000 --name myapp myapp:1.0
docker run --rm -it myapp:1.0 /bin/sh    # interactive, auto-remove

# Inspect & Debug
docker logs -f myapp                      # follow logs
docker exec -it myapp /bin/sh            # shell into container
docker inspect myapp                      # full config JSON
docker stats                              # live resource usage
docker top myapp                          # processes inside container

# Image Management
docker images
docker pull nginx:alpine
docker tag myapp:1.0 myregistry/myapp:1.0
docker push myregistry/myapp:1.0
docker rmi $(docker images -f "dangling=true" -q)   # remove dangling

# Cleanup
docker system prune -af --volumes        # nuclear cleanup
docker volume ls / docker volume prune
```

### Docker Compose
```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}

volumes:
  pgdata:
```

```bash
docker compose up -d          # start detached
docker compose logs -f app    # follow app logs
docker compose down -v        # stop + remove volumes
```

---

## ☸️ Kubernetes Cheatsheet

### Core Objects
```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 15
```

```yaml
# Service
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP   # ClusterIP | NodePort | LoadBalancer
```

### kubectl Commands
```bash
# Context & Namespace
kubectl config get-contexts
kubectl config use-context my-cluster
kubectl config set-context --current --namespace=dev

# Deploy & Manage
kubectl apply -f deployment.yaml
kubectl get all -n dev
kubectl describe pod myapp-xxx -n dev
kubectl logs -f myapp-xxx -c myapp       # -c for specific container
kubectl exec -it myapp-xxx -- /bin/sh

# Scaling & Rollouts
kubectl scale deployment myapp --replicas=5
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp    # rollback

# Debugging
kubectl get events --sort-by=.lastTimestamp
kubectl top pods -n dev
kubectl port-forward svc/myapp-svc 8080:80
```

---

## 🛠️ Hands-On Project: Dockerized App + K8s Deployment

**What you'll build:** A Python Flask API → Docker image → K8s Deployment + Service + HPA

📁 See [`projects/`](./projects/) folder for full YAML manifests and Dockerfile.

**Architecture:**
```
Internet → Ingress → Service (ClusterIP) → Deployment (3 replicas) → Pods
                                                    ↑
                                               HPA (2-10 replicas based on CPU)
```

---

## 🎯 Key Interview Questions

1. **Difference between CMD and ENTRYPOINT?**  
   `ENTRYPOINT` defines the executable; `CMD` provides default arguments. `CMD` can be overridden at runtime.

2. **What is a pod vs a deployment?**  
   A Pod is the smallest K8s unit (one or more containers). A Deployment manages Pod replicas, rolling updates, and self-healing.

3. **How does a Kubernetes rolling update work?**  
   New pods are created before old ones are terminated, ensuring zero downtime. Controlled by `maxSurge` and `maxUnavailable`.

4. **What are resource requests vs limits?**  
   Requests = guaranteed minimum (used for scheduling). Limits = hard cap (container is killed if exceeded).

5. **How do you expose a service externally in K8s?**  
   Use `type: LoadBalancer` (cloud), `NodePort`, or an Ingress controller.
