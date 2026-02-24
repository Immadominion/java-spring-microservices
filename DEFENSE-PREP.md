# Project Defense Preparation — Group 10: Monitoring (Prometheus/Grafana)

## Patient Management Microservices System

---

## TABLE OF CONTENTS

1. [Project Overview](#1-project-overview)
2. [Architecture Deep Dive](#2-architecture-deep-dive)
3. [How to Run the Project](#3-how-to-run-the-project)
4. [DevOps Relevance](#4-devops-relevance)
5. [Group 10 Topic: Monitoring with Prometheus & Grafana](#5-monitoring-with-prometheus--grafana)
6. [Defense Q&A Preparation](#6-defense-qa-preparation)
7. [Live Demo Script](#7-live-demo-script)
8. [Cost Analysis](#8-cost-analysis)
9. [AWS Deployment](#9-aws-deployment)
10. [CI/CD Pipeline (GitHub Actions → AWS)](#10-cicd-pipeline-github-actions--aws)

---

## 1. PROJECT OVERVIEW

This is a **Patient Management System** built with Java 21, Spring Boot 3.4.x, and a microservices architecture. It allows healthcare staff to manage patient records — create, read, update, and delete patients — with authentication, billing integration, and event-driven analytics.

### What the System Does

- **Register/login** users with JWT-based authentication
- **CRUD operations** on patient records
- **Automatic billing account creation** when a patient is registered (via gRPC)
- **Event streaming** for analytics when patients are created (via Kafka)
- **API Gateway** as a single entry point with JWT validation

### Technology Stack

| Technology           | Purpose                                           |
|---------------------|---------------------------------------------------|
| Java 21             | Programming language                              |
| Spring Boot 3.4.x  | Application framework                             |
| Spring Cloud Gateway| API Gateway / routing / filtering                 |
| PostgreSQL          | Persistent data storage (auth + patient DBs)      |
| Apache Kafka        | Asynchronous event streaming                      |
| gRPC + Protobuf     | Synchronous inter-service communication           |
| Docker              | Containerization                                  |
| AWS CDK + LocalStack| Infrastructure as Code (original design)          |
| Prometheus          | Metrics collection & monitoring                   |
| Grafana             | Metrics visualization & dashboards                |
| JWT (JJWT)          | Authentication tokens                             |
| SpringDoc OpenAPI   | API documentation                                 |

---

## 2. ARCHITECTURE DEEP DIVE

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                        CLIENT / BROWSER                         │
 └──────────────────────────────┬──────────────────────────────────┘
                                │ HTTP (port 4004)
                                ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │                        API GATEWAY                               │
 │  • Routes /auth/** → Auth Service (no JWT check)                │
 │  • Routes /api/patients/** → Patient Service (JWT required)     │
 │  • JwtValidation filter calls auth-service /validate            │
 │  Port: 4004                                                     │
 └──────────┬────────────────────────────────┬─────────────────────┘
            │                                │
    ┌───────▼───────┐              ┌─────────▼──────────┐
    │  AUTH SERVICE  │              │  PATIENT SERVICE   │
    │  Port: 4005   │              │  Port: 4000        │
    │               │              │                    │
    │  • POST /login│              │  • GET /patients   │
    │  • GET /valid. │              │  • POST /patients  │
    │               │              │  • PUT /patients/id│
    │  PostgreSQL   │              │  • DEL /patients/id│
    │  JWT + BCrypt │              │                    │
    └───────────────┘              │  PostgreSQL + JPA  │
                                   └──┬────────────┬───┘
                            gRPC call │            │ Kafka publish
                                      │            │ (topic: "patient")
                          ┌───────────▼──┐   ┌────▼──────────────┐
                          │   BILLING    │   │ ANALYTICS SERVICE │
                          │   SERVICE    │   │  Port: 4002       │
                          │  Port: 4001  │   │                   │
                          │  gRPC: 9001  │   │  Kafka consumer   │
                          │              │   │  Logs patient     │
                          │  Creates     │   │  events           │
                          │  billing     │   └───────────────────┘
                          │  accounts    │
                          └──────────────┘

 ┌──────────────────── MONITORING STACK ────────────────────────────┐
 │  PROMETHEUS (port 9090) ──scrapes──> /actuator/prometheus        │
 │       │                              on all 5 services           │
 │       ▼                                                          │
 │  GRAFANA (port 3000) ──queries──> Prometheus                    │
 │       • Pre-built dashboard with 11 panels                      │
 │       • Health, HTTP rates, latency, JVM, CPU, threads, etc.    │
 └──────────────────────────────────────────────────────────────────┘
```

### Communication Patterns

| Pattern       | Where Used                                | Why                                |
|--------------|-------------------------------------------|------------------------------------|
| **REST/HTTP** | Client → Gateway → Patient/Auth Service   | Standard synchronous CRUD          |
| **gRPC**     | Patient Service → Billing Service         | Low-latency binary inter-service   |
| **Kafka**    | Patient Service → Analytics Service       | Async event-driven, decoupled      |

### Flow When Creating a Patient

1. Client sends `POST /api/patients` to API Gateway (port 4004) with JWT in header
2. Gateway extracts `Bearer` token, calls Auth Service `GET /validate` to verify it
3. If valid, Gateway forwards request to Patient Service (port 4000)
4. Patient Service saves patient to PostgreSQL
5. Patient Service calls Billing Service via **gRPC** to create billing account
6. Patient Service publishes `PatientEvent` protobuf to **Kafka** topic `patient`
7. Analytics Service (Kafka consumer) receives and logs the event
8. Patient Service returns `201 Created` with patient data

---

## 3. HOW TO RUN THE PROJECT

### Prerequisites

You need **Docker Desktop** installed on your Mac. That's it.

```bash
# Install Docker Desktop (if not already installed)
brew install --cask docker
```

After installing, **open Docker Desktop** from Applications and wait for it to start.

### Start Everything

```bash
cd /Users/mac/Documents/school/java-spring-microservices

# Build and start all services (first time takes ~5-10 minutes)
docker compose up --build -d

# Watch the logs
docker compose logs -f
```

### Verify It's Working

```bash
# Check all containers are running
docker compose ps

# Test login (get JWT token)
curl -s -X POST http://localhost:4004/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@test.com","password":"password123"}' | jq .

# Use the token to get patients (replace <TOKEN> with actual token)
curl -s http://localhost:4004/api/patients \
  -H "Authorization: Bearer <TOKEN>" | jq .

# Create a new patient
curl -s -X POST http://localhost:4004/api/patients \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"name":"Test Patient","email":"test@example.com","address":"123 Main St","dateOfBirth":"1990-01-15","registeredDate":"2025-01-01"}' | jq .
```

### Access Points

| Service              | URL                          |
|---------------------|------------------------------|
| API Gateway          | <http://localhost:4004>        |
| Auth Service         | <http://localhost:4005>        |
| Patient Service      | <http://localhost:4000>        |
| Billing Service      | <http://localhost:4001>        |
| Analytics Service    | <http://localhost:4002>        |
| **Prometheus**       | **<http://localhost:9090>**    |
| **Grafana**          | **<http://localhost:3000>**    |
| Patient API Docs     | <http://localhost:4000/swagger-ui.html> |
| Auth API Docs        | <http://localhost:4005/swagger-ui.html> |

### Grafana Login

- Username: `admin`
- Password: `admin`
- Dashboard is auto-provisioned: look in "Patient Management System" folder

### Stop Everything

```bash
docker compose down          # Stop containers
docker compose down -v       # Stop + delete all data volumes
```

---

## 4. DEVOPS RELEVANCE

This project is a **textbook DevOps case study**. Here's how it maps:

### Containerization (Docker)

- Every microservice has a **multi-stage Dockerfile** (builder + runner)
- Build stage: Maven 3.9.9 + JDK 21 compiles the code
- Runtime stage: Slim OpenJDK 21 image runs the JAR
- **Why it matters**: Consistent environments, no "works on my machine" problem

### Container Orchestration (Docker Compose / AWS ECS)

- `docker-compose.yml` defines the entire stack declaratively
- Service dependencies, health checks, networking — all in one file
- Original project uses **AWS CDK** to deploy to **ECS Fargate** (serverless containers)
- **Why it matters**: Infrastructure as Code (IaC), reproducible deployments

### Microservices Architecture

- 5 independently deployable services
- Each service has its own database (database-per-service pattern)
- Services communicate via REST, gRPC, and Kafka
- **Why it matters**: Independent scaling, fault isolation, team autonomy

### CI/CD Pipeline Readiness

- Multi-stage Docker builds are CI/CD friendly
- Integration tests exist (REST Assured)
- Each service can be built, tested, and deployed independently
- **Why it matters**: Faster release cycles, automated quality gates

### Infrastructure as Code (IaC)

- AWS CDK stack defines VPC, RDS, MSK (Kafka), ECS, ALB
- LocalStack for local development testing
- **Why it matters**: Version-controlled infrastructure, repeatable environments

### Observability (YOUR GROUP TOPIC)

- **Monitoring**: Prometheus + Grafana (metrics)
- **Logging**: Each service outputs structured logs
- Spring Boot Actuator exposes health checks and metrics
- **Why it matters**: You can't manage what you can't measure

---

## 5. MONITORING WITH PROMETHEUS & GRAFANA

### What is Prometheus?

Prometheus is an **open-source monitoring and alerting system**. It:

- **Pulls (scrapes)** metrics from your services at regular intervals
- Stores metrics in a **time-series database**
- Provides **PromQL** query language for analysis
- Can trigger **alerts** based on metric thresholds

### What is Grafana?

Grafana is an **open-source visualization platform**. It:

- Connects to data sources like Prometheus
- Creates **dashboards** with graphs, gauges, and alerts
- Provides **real-time monitoring** of system health

### How It Works in This Project

```
┌──────────────┐   scrape every 10s   ┌──────────────────┐
│   Service    │ ◄──────────────────── │   PROMETHEUS     │
│ /actuator/   │   (pull model)       │   (port 9090)    │
│  prometheus  │                      │                  │
└──────────────┘                      │  Time-series DB  │
                                      └────────┬─────────┘
                                               │ PromQL queries
                                               ▼
                                      ┌──────────────────┐
                                      │    GRAFANA        │
                                      │   (port 3000)     │
                                      │                  │
                                      │  📊 Dashboards    │
                                      │  🚨 Alerts        │
                                      └──────────────────┘
```

### What We Added to Enable Monitoring

1. **Spring Boot Actuator** dependency — exposes `/actuator/health`, `/actuator/metrics`
2. **Micrometer Prometheus Registry** — formats metrics in Prometheus format at `/actuator/prometheus`
3. **application.properties config** — exposes health, info, prometheus, metrics endpoints
4. **prometheus.yml** — tells Prometheus where to scrape each service
5. **Grafana provisioning** — auto-configures Prometheus as data source + loads dashboard

### Metrics Available (What Prometheus Scrapes)

| Category              | Example Metrics                                     | What It Tells You                    |
|----------------------|-----------------------------------------------------|--------------------------------------|
| **HTTP Requests**     | `http_server_requests_seconds_count`                | Total requests per endpoint          |
|                      | `http_server_requests_seconds_sum`                  | Total time spent serving requests    |
|                      | `http_server_requests_seconds_bucket`               | Latency distribution (histograms)    |
| **JVM Memory**        | `jvm_memory_used_bytes`                             | How much heap/non-heap is used       |
|                      | `jvm_memory_max_bytes`                              | Maximum memory available             |
| **JVM GC**            | `jvm_gc_pause_seconds_sum`                          | Time spent in garbage collection     |
| **CPU**               | `process_cpu_usage`                                 | CPU utilization per service          |
|                      | `system_cpu_usage`                                  | Overall system CPU                   |
| **Threads**           | `jvm_threads_live_threads`                          | Active thread count                  |
| **Connection Pool**   | `hikaricp_connections_active`                       | Active DB connections                |
|                      | `hikaricp_connections_idle`                         | Idle DB connections                  |
| **Kafka**             | `kafka_consumer_records_lag_max`                    | Consumer lag (analytics service)     |
| **Service Health**    | `up`                                                | 1 = healthy, 0 = down               |

### Pre-Built Dashboard Panels

Our Grafana dashboard includes **11 panels**:

1. **Service Health Status** — Color-coded UP/DOWN for all services
2. **HTTP Request Rate** — Requests per second across services
3. **HTTP Response Time (p95)** — 95th percentile latency
4. **JVM Heap Memory Used** — Memory consumption over time
5. **JVM Memory Max vs Used** — Memory headroom per service
6. **CPU Usage per Service** — Process-level CPU utilization
7. **JVM Live Threads** — Thread count monitoring
8. **HTTP Error Rate (4xx + 5xx)** — Error tracking
9. **Kafka Consumer Lag** — Are messages being processed fast enough?
10. **GC Pause Duration** — Garbage collection impact
11. **HikariCP Connection Pool** — Database connection health

### Key PromQL Queries to Know

```promql
# Is a service up?
up{job="patient-service"}

# Request rate per second (last 5 minutes)
rate(http_server_requests_seconds_count{job="patient-service"}[5m])

# 95th percentile response time
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{job="patient-service"}[5m]))

# Error rate (4xx and 5xx responses)
rate(http_server_requests_seconds_count{status=~"4..|5.."}[5m])

# JVM heap memory used
jvm_memory_used_bytes{area="heap", job="patient-service"}

# CPU usage
process_cpu_usage{job="patient-service"}

# Kafka consumer lag
kafka_consumer_records_lag_max{job="analytics-service"}
```

---

## 6. DEFENSE Q&A PREPARATION

### Q: What is monitoring and why is it important in microservices?

**A:** Monitoring is the practice of collecting, analyzing, and acting on metrics from your systems. In microservices, it's critical because:

- You have **many moving parts** (5 services, 2 databases, Kafka, etc.)
- A failure in one service can **cascade** to others
- Without monitoring, you're **blind** to performance issues until users complain
- It enables **proactive problem detection** vs reactive firefighting

### Q: Why Prometheus specifically? Why not just look at logs?

**A:** Logs tell you **what happened** (events). Metrics tell you **how things are performing** over time (trends). Prometheus excels because:

- **Pull-based model**: Prometheus scrapes services, so services don't need to know about monitoring
- **Time-series data**: You can see trends, detect anomalies, set alerts
- **PromQL**: Powerful query language for aggregation and analysis
- **Low overhead**: Minimal impact on service performance
- Logs + metrics together = full observability

### Q: What's the difference between Prometheus and Grafana?

**A:** Prometheus **collects and stores** metrics. Grafana **visualizes** them. Prometheus is the engine, Grafana is the dashboard. Prometheus can also alert, but Grafana provides richer visualization options.

### Q: How does the pull model work?

**A:** Prometheus periodically (every 10s in our config) makes HTTP GET requests to each service's `/actuator/prometheus` endpoint. Each service exposes metrics in Prometheus text format. This is different from a push model (like Datadog agent) where services send metrics to a central collector.

### Q: What metrics matter most for this project?

**A:**

1. **Service health (up/down)** — Are all 5 services running?
2. **HTTP latency (p95/p99)** — Are patients experiencing slow responses?
3. **Error rates (4xx/5xx)** — Are requests failing?
4. **JVM memory** — Are services running out of memory?
5. **Kafka consumer lag** — Is analytics service keeping up with events?
6. **DB connection pool** — Are we running out of database connections?

### Q: What happens if a service goes down? How would you know?

**A:** The `up` metric drops to 0. Prometheus detects this on the next scrape. In production, you'd configure alerting rules — e.g., "if patient-service is down for >1 minute, send PagerDuty/Slack alert." Our Grafana dashboard shows health status with red/green indicators.

### Q: How does Spring Boot Actuator integrate with Prometheus?

**A:** Spring Boot Actuator is a built-in module that exposes operational endpoints. Adding `micrometer-registry-prometheus` dependency automatically formats all Micrometer metrics into Prometheus exposition format at `/actuator/prometheus`. No custom code needed — it's convention over configuration.

### Q: What is Micrometer?

**A:** Micrometer is a metrics facade for JVM-based applications (like SLF4J is for logging). It provides a vendor-neutral API for recording metrics, and then backends (Prometheus, Datadog, New Relic, etc.) can consume them. This means you can switch monitoring systems without changing application code.

### Q: How would you set up alerting?

**A:** Two options:

1. **Prometheus Alertmanager**: Define alert rules in `prometheus.yml`, e.g., "alert if error rate > 5% for 5 minutes." Forward to Slack/email/PagerDuty.
2. **Grafana Alerts**: Create alerts directly on dashboard panels with notification channels.

### Q: How does monitoring relate to DevOps?

**A:** Monitoring is a pillar of DevOps observability. The DevOps lifecycle is: Plan → Code → Build → Test → Release → Deploy → Operate → **Monitor**. Monitoring feeds back into planning (identify bottlenecks, capacity planning, SLA compliance). Without monitoring, you can't do effective incident response, capacity planning, or continuous improvement.

### Q: What are the Four Golden Signals (Google SRE)?

**A:**

1. **Latency** — How long requests take (our p95 panel)
2. **Traffic** — Request volume (our request rate panel)
3. **Errors** — Failed request rate (our error rate panel)
4. **Saturation** — How full your resources are (CPU, memory, connections panels)

### Q: "Isn't this just one container?" / "How are these separate microservices?"

**A:** No — `docker compose ps` shows **11 separate containers**, each with its own:

- **Process**: Each container runs one Java process (or one PostgreSQL/Kafka instance)
- **Port**: auth=4005, patient=4000, billing=4001, analytics=4002, gateway=4004
- **Filesystem**: Completely isolated from each other
- **Network identity**: Each has its own IP on the Docker bridge network
- **Memory/CPU limits**: Can be set independently per container

Docker Compose is just the **orchestrator** that starts them together — it's equivalent to Kubernetes managing pods or ECS managing tasks. You can prove they're independent by stopping one service and showing the others still work:

```bash
docker compose stop patient-service
# Auth service still works:
curl -X POST http://localhost:4004/auth/login ...
# Restart it:
docker compose start patient-service
```

### Q: Why EC2 + Docker Compose instead of ECS Fargate or Kubernetes?

**A:** For a production system, we'd use ECS Fargate or Kubernetes for auto-scaling, load balancing, and managed infrastructure. For this demo, EC2 + Docker Compose achieves the same containerization and monitoring result with much simpler setup. The containers are identical — only the orchestrator differs. The key point is that our services are running on AWS infrastructure, accessible via a public IP, with CI/CD deploying automatically on every push.

### Q: Explain the architecture's communication patterns

**A:**

- **Synchronous REST**: Client → Gateway → Services (for CRUD operations needing immediate response)
- **Synchronous gRPC**: Patient → Billing (efficient binary protocol for internal service-to-service calls, ~10x faster than REST for this use case because of binary serialization)
- **Asynchronous Kafka**: Patient → Analytics (fire-and-forget event publishing, decouples services, analytics can be down without affecting patient creation)

### Q: Why is database-per-service pattern used?

**A:** Each service (auth, patient) has its own PostgreSQL database. This ensures:

- **Loose coupling**: Services can evolve independently
- **Independent scaling**: Patient DB can be scaled separately from auth DB
- **Fault isolation**: Auth DB crash doesn't affect patient data
- **Technology freedom**: Each service could use a different DB if needed

---

## 7. LIVE DEMO SCRIPT

### Step 1: Show architecture (2 min)

- Open the architecture diagram above
- Explain the 5 services and their roles
- Highlight the 3 communication patterns (REST, gRPC, Kafka)

### Step 2: Start the stack (already running)

```bash
docker compose ps  # Show all containers running
```

### Step 3: Show Prometheus (3 min)

- Open <http://localhost:9090>
- Go to **Status → Targets** — show all 5 services being scraped
- Run query: `up` — show all services are healthy
- Run query: `jvm_memory_used_bytes{area="heap"}` — show memory metrics
- Explain the pull model and scrape interval

### Step 4: Generate traffic (1 min)

```bash
# Login
TOKEN=$(curl -s -X POST http://localhost:4004/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@test.com","password":"password123"}' | python3 -c "import sys,json;print(json.load(sys.stdin)['token'])")

# Create patients to generate metrics
for i in $(seq 1 10); do
  curl -s -X POST http://localhost:4004/api/patients \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    -d "{\"name\":\"Demo Patient $i\",\"email\":\"demo${i}@test.com\",\"address\":\"$i Main St\",\"dateOfBirth\":\"1990-01-01\",\"registeredDate\":\"2025-01-01\"}" > /dev/null
done
echo "Created 10 patients"

# Get all patients
curl -s http://localhost:4004/api/patients \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool | head -20
```

### Step 5: Show Grafana (3 min)

- Open <http://localhost:3000> (admin/admin)
- Navigate to "Patient Management System" → "Patient Management - Microservices Monitoring"
- Show each panel and explain what it means
- Point out the HTTP request spikes from the traffic you just generated
- Show JVM memory, CPU, and thread counts

### Step 6: Simulate failure (2 min)

```bash
# Kill the patient service
docker compose stop patient-service

# Show in Prometheus: up{job="patient-service"} = 0
# Show in Grafana: Health panel turns RED

# Restart it
docker compose start patient-service
# Watch it recover to green
```

---

## 8. COST ANALYSIS

### Running Locally (FREE — Recommended for Defense)

**Docker Desktop** runs everything on your Mac. Cost: **$0**.

- Requires: ~8GB RAM free, ~10GB disk space
- All 5 services + 2 databases + Kafka + Prometheus + Grafana
- Perfect for demo, development, and your defense

### Why Local Docker IS Still DevOps

Running Docker Compose locally **is** DevOps because:

- You're using **containerization** (Docker)
- You're using **Infrastructure as Code** (docker-compose.yml, prometheus.yml)
- You're using **declarative configuration** (Grafana provisioning)
- You're doing **monitoring/observability** (Prometheus + Grafana)
- The containers are *identical* to what would run in production (ECS/Kubernetes)
- The only difference from "real" cloud is the orchestrator (Compose vs ECS/K8s)

### AWS Cloud Deployment (Active)

We deploy to an **EC2 t3.large** instance with Docker Compose — same setup as local, but on AWS.

| Resource | Cost | Notes |
|----------|------|-------|
| EC2 t3.large (8GB RAM) | ~$0.08/hr (~$2/day) | Needed for 11 containers |
| EBS 30GB gp3 storage | ~$2.40/month | Docker images + data |
| Data transfer | ~free for demo | < 1GB |
| **Total for defense** | **~$2-4** | Stop instance after defense! |

**IMPORTANT**: Stop or terminate the EC2 instance after your defense to avoid charges.

---

---

## 9. AWS DEPLOYMENT

Our system is deployed on **AWS EC2** — the same `docker-compose.yml` runs in the cloud.

### What's Running on AWS

| Component | AWS Resource | Details |
|-----------|-------------|----------|
| 5 microservices | EC2 t3.large | Docker containers |
| 2 PostgreSQL DBs | EC2 (Docker) | In-container databases |
| Kafka | EC2 (Docker) | In-container message broker |
| Prometheus | EC2 (Docker) | Accessible at `<EC2_IP>:9090` |
| Grafana | EC2 (Docker) | Accessible at `<EC2_IP>:3000` |
| API Gateway | EC2 (Docker) | Accessible at `<EC2_IP>:4004` |

### How to Access (Replace `<EC2_IP>` with actual IP)

```
API Gateway:  http://<EC2_IP>:4004
Prometheus:   http://<EC2_IP>:9090
Grafana:      http://<EC2_IP>:3000  (admin/admin)
```

### SSH into the Instance

```bash
ssh -i ~/.ssh/patient-mgmt-key.pem ec2-user@<EC2_IP>
cd java-spring-microservices
docker compose ps    # show all 11 containers
```

### Proving Microservices Are Separate (Defense Demo)

```bash
# Show 11 separate containers
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}"

# Stop ONE service — others keep running
docker compose stop patient-service
curl -X POST http://localhost:4004/auth/login ...   # Still works!
docker compose start patient-service                # Restore it
```

For the full step-by-step setup guide, see: `java-spring-microservices/AWS-DEPLOYMENT-GUIDE.md`

---

## 10. CI/CD PIPELINE (GitHub Actions → AWS)

### Pipeline Flow

```
Developer pushes to main
       │
       ▼
GitHub Actions:
  1. Build all 5 services (parallel Maven builds)
  2. Run unit tests
  3. Build Docker images
  4. SSH into EC2 → git pull → docker compose up --build
       │
       ▼
EC2 rebuilds & restarts containers
       │
       ▼
Prometheus auto-detects new services
Grafana shows updated metrics
```

### Workflow File

Location: `.github/workflows/ci.yml`

**3 stages**:

1. **Build & Test** — 5 parallel jobs compile each microservice with Maven and run tests
2. **Docker Build** — Builds all 5 Docker images, verifies they exist
3. **Deploy to AWS** — Uses `appleboy/ssh-action` to SSH into EC2, pull latest code, and redeploy

### GitHub Secrets Required

| Secret | Value |
|--------|-------|
| `EC2_HOST` | EC2 public IP address |
| `EC2_USERNAME` | `ec2-user` |
| `EC2_SSH_KEY` | Contents of the `.pem` key file |

### What to Say About CI/CD During Defense

> "Every push to our main branch triggers a GitHub Actions pipeline. It builds all 5 microservices in parallel, runs the tests, builds Docker images, then automatically deploys to our AWS EC2 instance via SSH. After deployment, Prometheus immediately starts scraping the new containers, and Grafana reflects the updated metrics — completing the DevOps feedback loop."

---

## QUICK REFERENCE CARD

```
── LOCAL ──
START:    docker compose up --build -d
STOP:     docker compose down
LOGS:     docker compose logs -f [service-name]
STATUS:   docker compose ps

GATEWAY:    http://localhost:4004
PROMETHEUS: http://localhost:9090  (→ Status → Targets)
GRAFANA:    http://localhost:3000  (admin/admin)

── AWS ──
SSH:      ssh -i ~/.ssh/patient-mgmt-key.pem ec2-user@<EC2_IP>
GATEWAY:  http://<EC2_IP>:4004
PROMETH:  http://<EC2_IP>:9090
GRAFANA:  http://<EC2_IP>:3000  (admin/admin)

── BOTH ──
LOGIN:  curl -X POST http://<HOST>:4004/auth/login \
        -H "Content-Type: application/json" \
        -d '{"email":"testuser@test.com","password":"password123"}'

── AFTER DEFENSE ──
AWS Console → EC2 → Select instance → Stop/Terminate (save money!)
```
