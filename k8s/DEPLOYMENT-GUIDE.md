# Healthcare Appointment System — Deployment Guide

A complete guide to running the Healthcare microservices system using **Docker Compose** and **Kubernetes (Minikube)**.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Option 1: Running with Docker Compose](#option-1-running-with-docker-compose)
4. [Option 2: Running on Kubernetes (Minikube)](#option-2-running-on-kubernetes-minikube)
5. [API Testing](#api-testing)
6. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
Client (Browser / Postman)
         │
         ▼ port 8080
   ┌─────────────┐
   │ API Gateway  │
   └──────┬───────┘
          │
    ┌─────┴──────┐
    ▼            ▼
┌─────────┐  ┌──────────────┐       ┌────────────────────┐
│ Patient │  │ Appointment  │──────▶│ Kafka (Events)     │
│ Service │  │ Service      │       └────────┬───────────┘
│ (8081)  │  │ (8082)       │                │
└────┬────┘  └──────┬───────┘                ▼
     │              │                ┌───────────────────┐
     ▼              ▼                │ Notification      │
┌─────────┐  ┌──────────────┐       │ Service (8083)    │
│ MySQL   │  │ MySQL        │       └───────────────────┘
│ Patient │  │ Appointment  │
│ (3306)  │  │ (3307)       │
└─────────┘  └──────────────┘
```

| Service | Port | Database | Dependencies |
|---------|------|----------|-------------|
| API Gateway | 8080 | — | Patient Service, Appointment Service |
| Patient Service | 8081 | MySQL (patientdb) on 3306 | MySQL |
| Appointment Service | 8082 | MySQL (appointmentdb) on 3307 | MySQL, Kafka, Patient Service |
| Notification Service | 8083 | — | Kafka |

---

## Prerequisites

- **Java 17** — `java --version`
- **Maven 3.9+** — `mvn --version`
- **Docker Desktop** — `docker --version`
- **Minikube** (for Kubernetes) — `minikube version`
- **kubectl** (for Kubernetes) — `kubectl version --client`

---

## Option 1: Running with Docker Compose

This is the simplest way to run the entire system. One command starts all infrastructure and services.

### Step 1: Build All Services

Open a terminal in each service directory and build the JAR files:

```bash
# Patient Service
cd healthcare-patient-service/patient-service
mvn clean package -DskipTests

# Appointment Service
cd healthcare-appointment-service/appointment-service
mvn clean package -DskipTests

# Notification Service
cd healthcare-notification-service/notification-service
mvn clean package -DskipTests

# API Gateway
cd healthcare-api-gateway/api-gateway
mvn clean package -DskipTests
```

### Step 2: Start Everything

From the root `Assignment` directory:

```bash
docker compose up -d --build
```

This starts **8 containers**:

| Container | Image | Port |
|-----------|-------|------|
| mysql-patient | mysql:8.0 | 3306 |
| mysql-appointment | mysql:8.0 | 3307 |
| zookeeper | confluentinc/cp-zookeeper:7.5.0 | 2181 |
| kafka | confluentinc/cp-kafka:7.5.0 | 9092 |
| patient-service | Built from Dockerfile | 8081 |
| appointment-service | Built from Dockerfile | 8082 |
| notification-service | Built from Dockerfile | 8083 |
| api-gateway | Built from Dockerfile | 8080 |

### Step 3: Verify

```bash
# Check all containers are running
docker ps

# Test the gateway
curl http://localhost:8080/actuator/health

# Check MySQL data
docker exec mysql-patient mysql -uroot -proot patientdb -e "SELECT * FROM patients;"
docker exec mysql-appointment mysql -uroot -proot appointmentdb -e "SELECT * FROM appointments;"
```

### Step 4: Stop Everything

```bash
docker compose down
```

### How Docker Networking Works

Inside Docker Compose, services communicate using **container names** instead of `localhost`:

| Local Dev | Docker Compose |
|-----------|---------------|
| `localhost:3306` | `mysql-patient:3306` |
| `localhost:3307` | `mysql-appointment:3306` (internal port) |
| `localhost:9092` | `kafka:9092` |
| `localhost:8081` | `patient-service:8081` |
| `localhost:8082` | `appointment-service:8082` |

The `docker-compose.yml` uses Spring Boot environment variable overrides so the `application.yaml` files remain unchanged for local development.

---

## Option 2: Running on Kubernetes (Minikube)

### Step 1: Install Minikube

**Windows (via winget):**
```powershell
winget install Kubernetes.minikube
```

**Verify installation:**
```powershell
# Add to PATH if needed
$env:Path += ";C:\Program Files\Kubernetes\Minikube"

minikube version
# Output: minikube version: v1.38.1
```

**Install kubectl:**
```powershell
winget install Kubernetes.kubectl
```

```powershell
kubectl version --client
# Output: Client Version: v1.34.1
```

### Step 2: Start Minikube Cluster

```powershell
# Ensure System32 is in PATH (needed for icacls.exe on Windows)
$env:Path += ";C:\Windows\System32"

# Start minikube with Docker driver
minikube start --driver=docker
```

Expected output:
```
😄  minikube v1.38.1 on Microsoft Windows 11
✨  Using the docker driver based on user configuration
👍  Starting "minikube" primary control-plane node
🏄  Done! kubectl is now configured to use "minikube" cluster
```

**Verify the cluster:**
```powershell
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   68s   v1.35.1
```

### Step 3: Build Docker Images Inside Minikube

Minikube has its own Docker daemon. We build images directly inside it so Kubernetes can use them with `imagePullPolicy: Never`.

```powershell
# Build all 4 service images
minikube image build -t patient-service:1.0 ./healthcare-patient-service/patient-service
minikube image build -t appointment-service:1.0 ./healthcare-appointment-service/appointment-service
minikube image build -t notification-service:1.0 ./healthcare-notification-service/notification-service
minikube image build -t api-gateway:1.0 ./healthcare-api-gateway/api-gateway
```

**Verify images are loaded:**
```powershell
minikube image ls | Select-String "service|gateway"
```

### Step 4: Deploy to Kubernetes

Apply all manifests from the `k8s/` directory:

```powershell
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-mysql.yaml
kubectl apply -f k8s/02-kafka.yaml
kubectl apply -f k8s/03-patient-service.yaml
kubectl apply -f k8s/04-appointment-service.yaml
kubectl apply -f k8s/05-notification-service.yaml
kubectl apply -f k8s/06-api-gateway.yaml
```

Or apply all at once:
```powershell
kubectl apply -f k8s/
```

### Step 5: Verify Pods Are Running

```powershell
kubectl get pods -n healthcare
```

Expected output (after ~2 minutes):
```
NAME                                   READY   STATUS    AGE
api-gateway-745667f9fc-d9hqm           1/1     Running   5m
appointment-service-6c8b5b4dc9-hdj66   1/1     Running   5m
kafka-7896d4bdcb-qd9sj                 1/1     Running   10m
mysql-appointment-644ff658f8-fmmsx     1/1     Running   25m
mysql-patient-5bff5ff9c8-9vrmt         1/1     Running   25m
notification-service-86997cf9b-6dmjm   1/1     Running   25m
patient-service-5785d7fd56-m4n9c       1/1     Running   25m
```

**Check services:**
```powershell
kubectl get svc -n healthcare
```

### Step 6: Access the Application

**Option A — Port Forward (recommended):**
```powershell
kubectl port-forward svc/api-gateway 9090:8080 -n healthcare
```
Then access at `http://localhost:9090`

**Option B — Minikube Service Tunnel:**
```powershell
minikube service api-gateway -n healthcare
```
This opens a tunnel and gives you a URL like `http://127.0.0.1:53635`

> ⚠️ On Windows with Docker driver, keep the terminal open — the tunnel stops if closed.

### Step 7: Open Minikube Dashboard (Web UI)

```powershell
minikube dashboard
```

This opens the Kubernetes Dashboard in your browser. Switch the **namespace dropdown** (top-left) to **`healthcare`** to see all pods, deployments, and services.

### Kubernetes Manifest Structure

```
k8s/
├── 00-namespace.yaml           # Creates 'healthcare' namespace
├── 01-mysql.yaml               # MySQL for Patient + Appointment (with health checks)
├── 02-kafka.yaml               # Kafka in KRaft mode (no Zookeeper needed)
├── 03-patient-service.yaml     # Patient Service + init container waits for MySQL
├── 04-appointment-service.yaml # Appointment Service + init container waits for MySQL, Kafka, Patient
├── 05-notification-service.yaml# Notification Service + init container waits for Kafka
└── 06-api-gateway.yaml         # API Gateway (NodePort on 30080) + init container waits for services
```

**Key design decisions:**
- **Init containers** ensure services start in the right order (wait for dependencies)
- **`imagePullPolicy: Never`** tells K8s to use locally built images
- **Environment variables** override Spring Boot configs for K8s networking
- **API Gateway** uses `NodePort` type to be accessible outside the cluster

### Step 8: Clean Up

```powershell
# Delete all resources
kubectl delete namespace healthcare

# Stop minikube
minikube stop

# Delete the cluster entirely
minikube delete
```

---

## API Testing

All requests go through the **API Gateway** (port 8080 for Docker, or the forwarded port for K8s).

### Patient Service

```bash
# Register a patient
curl -X POST http://localhost:8080/api/patients \
  -H "Content-Type: application/json" \
  -d '{"name":"Vinay Alapaty","email":"vinay@example.com","phone":"9876543210"}'

# Get all patients
curl http://localhost:8080/api/patients

# Get patient by ID
curl http://localhost:8080/api/patients/1

# Check if patient exists
curl http://localhost:8080/api/patients/1/exists

# Update patient
curl -X PUT http://localhost:8080/api/patients/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Vinay Updated","email":"vinay.new@example.com","phone":"1234567890"}'

# Delete patient
curl -X DELETE http://localhost:8080/api/patients/2
```

### Appointment Service

```bash
# Book appointment (triggers Saga pattern + Kafka event)
curl -X POST http://localhost:8080/api/appointments \
  -H "Content-Type: application/json" \
  -d '{"patientId":1,"doctorId":101,"appointmentDate":"2026-05-15","appointmentTime":"10:30:00"}'

# Get appointment by ID
curl http://localhost:8080/api/appointments/1

# Get all appointments for a patient
curl http://localhost:8080/api/appointments/patient/1

# Cancel appointment (triggers CANCELLED Kafka event)
curl -X PUT http://localhost:8080/api/appointments/1/cancel
```

### Gateway Actuator

```bash
curl http://localhost:8080/actuator/health
curl http://localhost:8080/actuator/gateway/routes
curl http://localhost:8080/actuator/metrics
```

---

## Checking Database Data

### Docker Compose

When running via Docker Compose, use `docker exec` to query MySQL directly:

```bash
# Patient Database (port 3306)
docker exec mysql-patient mysql -uroot -proot patientdb -e "SELECT * FROM patients;"

# Appointment Database (port 3307)
docker exec mysql-appointment mysql -uroot -proot appointmentdb -e "SELECT * FROM appointments;"

# Show all tables
docker exec mysql-patient mysql -uroot -proot patientdb -e "SHOW TABLES;"
docker exec mysql-appointment mysql -uroot -proot appointmentdb -e "SHOW TABLES;"

# Describe table schema
docker exec mysql-patient mysql -uroot -proot patientdb -e "DESCRIBE patients;"
docker exec mysql-appointment mysql -uroot -proot appointmentdb -e "DESCRIBE appointments;"

# Count records
docker exec mysql-patient mysql -uroot -proot patientdb -e "SELECT COUNT(*) FROM patients;"
docker exec mysql-appointment mysql -uroot -proot appointmentdb -e "SELECT COUNT(*) FROM appointments;"

# Interactive MySQL shell
docker exec -it mysql-patient mysql -uroot -proot patientdb
docker exec -it mysql-appointment mysql -uroot -proot appointmentdb
```

### Kubernetes (Minikube)

When running on Kubernetes, use `kubectl exec` to access the MySQL pods:

```bash
# Find the MySQL pod names
kubectl get pods -n healthcare -l app=mysql-patient
kubectl get pods -n healthcare -l app=mysql-appointment

# Patient Database
kubectl exec -it -n healthcare deploy/mysql-patient -- mysql -uroot -proot patientdb -e "SELECT * FROM patients;"

# Appointment Database
kubectl exec -it -n healthcare deploy/mysql-appointment -- mysql -uroot -proot appointmentdb -e "SELECT * FROM appointments;"

# Describe table schema
kubectl exec -it -n healthcare deploy/mysql-patient -- mysql -uroot -proot patientdb -e "DESCRIBE patients;"
kubectl exec -it -n healthcare deploy/mysql-appointment -- mysql -uroot -proot appointmentdb -e "DESCRIBE appointments;"

# Interactive MySQL shell
kubectl exec -it -n healthcare deploy/mysql-patient -- mysql -uroot -proot patientdb
kubectl exec -it -n healthcare deploy/mysql-appointment -- mysql -uroot -proot appointmentdb
```

### Checking Kafka Events / Notification Logs

```bash
# Docker Compose
docker logs notification-service | findstr "NOTIFICATION"

# Kubernetes
kubectl logs -l app=notification-service -n healthcare | Select-String "NOTIFICATION|Event|ACTION"
```

---

## Troubleshooting

### Docker Compose

| Issue | Solution |
|-------|----------|
| Port already in use | `docker compose down` then retry, or check `netstat -ano | findstr :8080` |
| MySQL not ready | Wait for health check: `docker ps` should show `(healthy)` |
| Service can't connect to DB | Ensure container names match in env vars |

### Minikube

| Issue | Solution |
|-------|----------|
| `icacls.exe` not found | Add `C:\Windows\System32` to PATH |
| Image pull errors | Use `minikube image build` instead of `docker build` |
| Pod stuck in `Init:0/1` | Check dependency pods are running: `kubectl get pods -n healthcare` |
| Pod in `CrashLoopBackOff` | Check logs: `kubectl logs <pod-name> -n healthcare` |
| Can't access service | Use `kubectl port-forward` or `minikube service` |
| Dashboard shows no pods | Switch namespace to `healthcare` in the dropdown |
