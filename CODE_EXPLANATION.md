# 🚀 Web Monitoring Project — Code Explanation

> A Dockerized Flask app with GitHub Actions CI/CD pipeline and Prometheus + Grafana monitoring.

---

## 📁 Project Structure

```
Web_Monitoring/
├── app/
│   ├── app.py                        # Flask web application
│   └── requirements.txt              # Python dependencies
├── monitoring/
│   └── prometheus.yml                # Prometheus scrape config
├── Dockerfile                        # Container build instructions
├── docker-compose.yml                # Multi-container orchestration
├── architecture.svg                  # Architecture diagram
└── .github/
    └── workflows/
        └── deploy.yml                # GitHub Actions CI/CD pipeline
```

---

## 🐍 `app/app.py` — The Flask Application

```python
from flask import Flask, Response
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)
REQUEST_COUNT = Counter("app_requests_total", "Total HTTP requests")

@app.route("/")
def home():
    REQUEST_COUNT.inc()
    return "Hello from Docker! 🚀"

@app.route("/metrics")
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5001)
```

### What each part does

| Code | Purpose |
|------|---------|
| `Counter("app_requests_total", ...)` | Creates a counter metric that tracks how many times the app is hit |
| `REQUEST_COUNT.inc()` | Increments the counter by 1 on every visit to `/` |
| `@app.route("/metrics")` | Exposes all metrics as plain text — Prometheus scrapes this URL |
| `generate_latest()` | Converts all metrics to Prometheus text format |
| `CONTENT_TYPE_LATEST` | Sets correct `text/plain` content type so Prometheus accepts it |
| `host="0.0.0.0"` | Makes Flask accessible from outside the container |

---

## 📦 `Dockerfile` — Container Build Instructions

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app/ .
RUN pip install -r requirements.txt
EXPOSE 5001
CMD ["python", "app.py"]
```

### What each line does

| Line | Purpose |
|------|---------|
| `FROM python:3.12-slim` | Starts from a lightweight Python base image |
| `WORKDIR /app` | Sets the working directory inside the container |
| `COPY app/ .` | Copies your app files into the container |
| `RUN pip install -r requirements.txt` | Installs Flask and prometheus_client |
| `EXPOSE 5001` | Documents that the container listens on port 5001 |
| `CMD ["python", "app.py"]` | Command that runs when the container starts |

---

## 🐙 `docker-compose.yml` — Multi-Container Orchestration

```yaml
services:
  app:
    build: .
    ports:
      - "5001:5001"
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```

### What each section does

| Section | Purpose |
|---------|---------|
| `app` service | Builds your Flask app from the Dockerfile and runs it |
| `prometheus` service | Pulls the official Prometheus image and mounts your config |
| `grafana` service | Pulls Grafana image, sets admin password |
| `networks: monitoring` | Puts all 3 containers on the same virtual network so they can talk to each other by name (e.g. `app:5001`) |
| `volumes` | Mounts your local `prometheus.yml` into the Prometheus container |

---

## 🔥 `monitoring/prometheus.yml` — Scrape Configuration

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: "flask-app"
    static_configs:
      - targets: ["app:5001"]
```

### What each part does

| Config | Purpose |
|--------|---------|
| `scrape_interval: 5s` | Prometheus collects metrics every 5 seconds |
| `job_name: "flask-app"` | Labels this job in Prometheus UI and Grafana |
| `targets: ["app:5001"]` | Points to your Flask app — `app` is the service name in docker-compose, `5001` is the port |

---

## ⚙️ `.github/workflows/deploy.yml` — CI/CD Pipeline

```yaml
name: Build & Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    env:
      FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

    steps:
      - name: Checkout code
        uses: actions/checkout@v4.2.2

      - name: Build Docker image
        run: docker build -t flask-app .

      - name: Login to Docker Hub
        uses: docker/login-action@v3.3.0
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Push image
        run: |
          docker tag flask-app ${{ secrets.DOCKER_USERNAME }}/flask-app:latest
          docker push ${{ secrets.DOCKER_USERNAME }}/flask-app:latest
```

### What each step does

| Step | Purpose |
|------|---------|
| `on: push: branches: [main]` | Triggers the pipeline automatically on every push to main branch |
| `runs-on: ubuntu-latest` | Runs the pipeline on a fresh Ubuntu virtual machine provided by GitHub |
| `actions/checkout@v4.2.2` | Downloads your code into the pipeline runner |
| `docker build -t flask-app .` | Builds a Docker image from your Dockerfile |
| `docker/login-action` | Logs into Docker Hub using your secrets |
| `docker tag` | Tags the image with your Docker Hub username |
| `docker push` | Uploads the image to Docker Hub registry |
| `secrets.DOCKER_USERNAME` | Secure variable stored in GitHub → Settings → Secrets |

---

## 📊 Grafana Queries Explained

| Query | What it shows |
|-------|--------------|
| `app_requests_total` | Running total of all HTTP requests ever made |
| `rate(app_requests_total[1m])` | How many requests per second in the last 1 minute |
| `process_resident_memory_bytes` | How much RAM your Flask app is using |
| `process_cpu_seconds_total` | Total CPU time used by the app |
| `up{job="flask-app"}` | Is the app alive? `1` = yes, `0` = down |

---

## 🔄 How Everything Connects

```
You push code to GitHub
        ↓
GitHub Actions triggers automatically
        ↓
Pipeline builds Docker image
        ↓
Image pushed to Docker Hub
        ↓
You run: docker-compose up --build
        ↓
3 containers start on the same network:
  ┌─────────────┐     ┌──────────────┐     ┌─────────┐
  │  Flask App  │────▶│  Prometheus  │────▶│ Grafana │
  │  :5001      │     │  :9090       │     │ :3000   │
  └─────────────┘     └──────────────┘     └─────────┘
  Counts requests      Scrapes /metrics     Shows graphs
  every page visit     every 5 seconds      in real time
```

---

## 🚀 How to Run

```bash
# Start all services
docker-compose up --build

# Stop all services
docker-compose down

# View logs
docker-compose logs -f
```

### URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| Flask App | http://localhost:5001 | — |
| Prometheus | http://localhost:9090 | — |
| Grafana | http://localhost:3000 | admin / admin |
| Metrics (raw) | http://localhost:5001/metrics | — |

---

## 🛠️ Tech Stack

| Technology | Role |
|-----------|------|
| Python + Flask | Web application framework |
| prometheus_client | Exposes metrics from Python app |
| Docker + Docker Desktop | Containerizes and runs the app locally |
| Docker Compose | Orchestrates multiple containers together |
| Prometheus | Collects and stores time-series metrics |
| Grafana | Visualizes metrics as dashboards |
| GitHub Actions | Automates build and push on code changes |
| Docker Hub | Stores and distributes Docker images |

---

*Project built for learning Infrastructure Engineering fundamentals: containerization, CI/CD, and observability.*
