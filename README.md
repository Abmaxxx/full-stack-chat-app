# Real-Time Chat App — Dockerized & Deployed on Kubernetes

A full-stack real-time chat application built with React, Node.js, and MongoDB — containerized with Docker and deployed on Kubernetes (Minikube) with Ingress-based routing.

![Architecture](https://readme-typing-svg.demolab.com?font=Fira+Code&size=18&pause=1000&color=2E9EF7&center=true&vCenter=true&width=700&lines=React+%7C+Node.js+%7C+MongoDB;Docker+%7C+Kubernetes+%7C+Nginx;Multi-tier+Cloud-Native+Deployment)

<p align="center">
  <img src="./images/architecture.png" alt="Project Architecture" width="100%">
</p>

---

## 📌 Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React (Vite), Tailwind CSS, served via Nginx |
| Backend | Node.js, Express, Socket.io |
| Database | MongoDB |
| Containerization | Docker (multi-stage builds) |
| Orchestration | Kubernetes (Minikube) |
| Routing | Nginx Ingress Controller |
| Storage | Kubernetes PersistentVolumeClaim |

---

## 🏗️ Architecture

```
                         ┌─────────────┐
                         │   Browser   │
                         └──────┬──────┘
                                │
                         ┌──────▼──────┐
                         │   Ingress   │
                         │ routes by   │
                         │    path     │
                         └──────┬──────┘
                    ┌───────────┴───────────┐
                 path: /                 path: /api
                    │                         │
          ┌─────────▼─────────┐    ┌──────────▼─────────┐
          │  frontend-service  │    │   backend-service   │
          │  ClusterIP, :80    │    │  ClusterIP, :5001   │
          └─────────┬─────────┘    └──────────┬─────────┘
                    │                         │
          ┌─────────▼─────────┐    ┌──────────▼─────────┐
          │   frontend pod     │    │    backend pod      │
          │ React + Nginx      │    │  Node.js + Express  │
          └────────────────────┘    └──────────┬─────────┘
                                               │
                                    ┌──────────▼─────────┐
                                    │    mongo-service    │
                                    │  ClusterIP, :27017  │
                                    └──────────┬─────────┘
                                               │
                                    ┌──────────▼─────────┐
                                    │   MongoDB pod       │
                                    │   + PVC Storage     │
                                    └────────────────────┘
```

---

## 📁 Repository Structure

```
.
├── frontend/
│   ├── Dockerfile          # Multi-stage build (Node → Nginx)
│   ├── nginx.conf          # Nginx config with /api proxy
│   ├── src/
│   └── ...
├── backend/
│   ├── Dockerfile          # Node.js production image
│   ├── src/
│   │   ├── index.js        # Express app entry point
│   │   ├── routes/
│   │   └── lib/
│   └── ...
└── k8s/
    ├── namespace.yml
    ├── mongo-pvc.yml
    ├── mongo-deployment.yml
    ├── mongo-service.yml
    ├── backend-deployment.yml
    ├── backend-service.yml
    ├── frontend-deployment.yml
    ├── frontend-service.yml
    └── ingress.yml
```

---

## 🐳 Docker Setup

### Frontend — Multi-Stage Build

```dockerfile
# Stage 1: Build React app
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Backend

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ENV NODE_ENV=production
EXPOSE 5001
CMD ["node", "src/index.js"]
```

---

## ☸️ Kubernetes Deployment

### Prerequisites

- Minikube installed and running
- kubectl configured
- Docker images pushed to Docker Hub

### Deploy Step by Step

**1. Create namespace**
```bash
kubectl apply -f k8s/namespace.yml
```

**2. Deploy MongoDB**
```bash
kubectl apply -f k8s/mongo-pvc.yml
kubectl apply -f k8s/mongo-deployment.yml
kubectl apply -f k8s/mongo-service.yml
```

**3. Deploy Backend**
```bash
kubectl apply -f k8s/backend-deployment.yml
kubectl apply -f k8s/backend-service.yml
```

**4. Deploy Frontend**
```bash
kubectl apply -f k8s/frontend-deployment.yml
kubectl apply -f k8s/frontend-service.yml
```

**5. Enable Ingress and apply**
```bash
minikube addons enable ingress
kubectl apply -f k8s/ingress.yml
```

**6. Verify everything is running**
```bash
kubectl get pods -n namespace-for-app
kubectl get services -n namespace-for-app
kubectl get ingress -n namespace-for-app
```

**7. Access the app**
```bash
# Map the Ingress IP to a hostname
echo "<ingress-ip> myapp.com" | sudo tee -a /etc/hosts

# Or use port-forward for quick access
kubectl port-forward -n namespace-for-app service/frontend-service 8080:80 --address 0.0.0.0
```

Then open: `http://<your-ip>:8080`

---

## ⚙️ Environment Variables

| Variable | Description |
|---|---|
| `PORT` | Port the Express server listens on (5001) |
| `NODE_ENV` | Environment — `production` or `development` |
| `MONGODB_URI` | MongoDB connection string |
| `JWT_SECRET` | Secret key for signing JWTs |

> In Kubernetes, these are defined in `backend-deployment.yml`. For production, move secrets to a Kubernetes `Secret` resource.

---

## 🔑 Key Design Decisions

**Multi-stage frontend build** — Node.js used only to build the React app. Final image contains only Nginx and the compiled static files — no source code, no `node_modules`.

**Service-name-based internal routing** — Nginx proxies `/api/` and `/socket.io/` to `backend-service`, the Kubernetes internal DNS name. This avoids hardcoded IPs and works natively with Kubernetes service discovery.

**PersistentVolumeClaim for MongoDB** — Ensures chat data survives pod restarts and rescheduling. Data is stored on a persistent volume separate from the pod lifecycle.

**Path-based Ingress routing** — A single external entry point routes `/api/*` to the backend and everything else to the frontend. No need to expose multiple services externally.

**Namespace isolation** — All resources deployed under a dedicated namespace (`namespace-for-app`) for clean separation and easier management.

---

## 🐛 Real Debugging Encountered

**Backend crash-looping on startup** — Backend was starting before `mongo-service` existed. Fixed by applying MongoDB manifests first, then restarting the backend deployment.

**MongoDB PVC version mismatch** — PVC had leftover metadata from a newer MongoDB version. Fixed by force-deleting and recreating the PVC.

**Nginx "host not found in upstream"** — `nginx.conf` was pointing to `backend` (Docker Compose service name) instead of `backend-service` (Kubernetes service name). Fixed by updating `nginx.conf` and rebuilding the frontend image.

**ImagePullBackOff on frontend** — Docker Hub image name had a typo (`chat-app_frontend` vs `chatapp-fronted`). Fixed by correcting the image name in `frontend-deployment.yml`.

