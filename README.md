# Node.js on Kubernetes with Prometheus & Grafana Monitoring

A reference implementation demonstrating how to containerize a Node.js application, deploy it to a local Kubernetes cluster using Minikube, and integrate it with an existing host-level Prometheus and Grafana monitoring stack.

This project illustrates the end-to-end workflow that is standard in modern cloud-native operations: application instrumentation, containerization, orchestration, metrics scraping, and visualization.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Detailed Walkthrough](#detailed-walkthrough)
  - [1. Application Layer](#1-application-layer)
  - [2. Containerization](#2-containerization)
  - [3. Kubernetes Deployment](#3-kubernetes-deployment)
  - [4. Prometheus Integration](#4-prometheus-integration)
  - [5. Grafana Dashboard](#5-grafana-dashboard)
- [Verification](#verification)
- [Screenshots](#screenshots)
- [Scaling](#scaling)
- [Adapting for Your Own Application](#adapting-for-your-own-application)
- [Troubleshooting](#troubleshooting)
- [Project Scope and Limitations](#project-scope-and-limitations)
- [Further Reading](#further-reading)
- [License](#license)

---

## Overview

This repository contains a minimal Node.js (Express) application instrumented with the Prometheus client library, packaged as a Docker image, and deployed to a Kubernetes cluster running under Minikube. The application exposes a `/metrics` endpoint which is scraped by a host-level Prometheus instance and visualized through Grafana dashboards.

The configuration assumes that **Prometheus and Grafana are already installed and running on the host machine** (outside the Kubernetes cluster), which is a common pattern when an organization has established monitoring infrastructure that supervises multiple environments.

### Objectives

- Demonstrate application-level instrumentation using `prom-client`
- Show the correct workflow for building Docker images within the Minikube Docker daemon
- Provide working Kubernetes manifests with health probes and resource constraints
- Document the Prometheus scrape configuration required to monitor a `NodePort`-exposed service from a host-level Prometheus
- Serve as a starting template for production-bound microservices


Architecture

 Ubuntu Host                                                                           
   │  Prometheus    │────────▶    Grafana                
   │  (systemd)     │  query  │    (systemd)                
   │   :9090        │         │     :3000                  
              
│ scrape every 15s                               
 http://<minikube-ip>:30001/metrics             

Minikube Cluster
   Service (NodePort 30001)                       
├──▶ Pod 1 (Node.js app, /metrics)        
└──▶ Pod 2 (Node.js app, /metrics)          


Prometheus reaches the application through the Minikube node IP and the `NodePort` (`30001`). The `NodePort` Service load-balances incoming traffic across the available replicas.



Technology Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| Node.js | 20 LTS (Alpine) | Application runtime |
| Express | ^4.19.2 | HTTP framework |
| prom-client | ^15.1.3 | Prometheus metrics library for Node.js |
| Docker | 20.10+ | Container runtime |
| Minikube | 1.36+ | Local Kubernetes cluster |
| Kubernetes | 1.33 | Container orchestration |
| Prometheus | 2.x | Metrics collection (host-installed) |
| Grafana | 10.x+ | Metrics visualization (host-installed) |



 Prerequisites

The following must be installed and operational on the host machine:

- **Ubuntu 22.04 LTS or 24.04 LTS** (any modern Linux distribution should work)
- **Docker Engine** with the current user in the `docker` group
- **Minikube** with the `docker` driver
- **kubectl** (any version compatible with the cluster)
- **Prometheus** running as a `systemd` service with a writable `/etc/prometheus/prometheus.yml`
- **Grafana** running as a `systemd` service, accessible on port `3000`
- A minimum of 4 GB RAM and 2 CPU cores available for Minikube

Verify the environment with:


docker --version
minikube version
kubectl version --client
sudo systemctl is-active prometheus
sudo systemctl is-active grafana-server

## Project Structure

.
├── index.js                  # Express application with /metrics endpoint
├── package.json              # Node.js dependencies
├── Dockerfile                # Multi-stage container build
├── .dockerignore             # Excluded paths from build context
├── .gitignore                # Excluded paths from version control
└── k8s/
    ├── deployment.yaml       # Kubernetes Deployment (2 replicas, health probes)
    └── service.yaml          # NodePort Service exposing port 30001


## Quick Start

For users familiar with the underlying technologies, the following sequence brings the application online:


# 1. Start Minikube
minikube start --driver=docker --memory=4096 --cpus=2

# 2. Build the image inside the Minikube Docker daemon
eval $(minikube docker-env)
docker build -t node-k8s-demo:1.0 .

# 3. Deploy to Kubernetes
kubectl apply -f k8s/

# 4. Verify availability
curl http://$(minikube ip):30001/
curl http://$(minikube ip):30001/metrics

# 5. Add the scrape job to Prometheus (see "Prometheus Integration" below)



## Detailed Walkthrough

### 1. Application Layer

The application (`index.js`) is a standard Express server with three endpoints:

| Endpoint | Purpose |
|----------|---------|
| `GET /` | Returns a greeting and the pod hostname (useful for verifying load balancing) |
| `GET /healthz` | Returns service health for Kubernetes liveness and readiness probes |
| `GET /metrics` | Exposes Prometheus metrics in the standard text exposition format |

#### Metrics Exposed

Two categories of metrics are registered:

**
Default Node.js metrics** (provided by `prom-client.collectDefaultMetrics()`):

- `process_cpu_user_seconds_total`, `process_cpu_system_seconds_total`
- `process_resident_memory_bytes`, `process_heap_bytes`
- `nodejs_heap_size_total_bytes`, `nodejs_heap_size_used_bytes`
- `nodejs_eventloop_lag_seconds`
- `nodejs_active_handles_total`, `nodejs_active_requests_total`

**Custom application metrics** (defined explicitly in `index.js`):

- `http_requests_total` — counter, labeled by `method`, `route`, and `status`
- `http_request_duration_seconds` — histogram, labeled by `method`, `route`, and `status`, with buckets at 5ms, 10ms, 50ms, 100ms, 500ms, 1s, 2s, and 5s

Custom metrics are populated by Express middleware that wraps every incoming request.

### 2. Containerization

The `Dockerfile` follows standard practices for production Node.js images:

- Uses `node:20-alpine` as the base image (small attack surface, ~150 MB total)
- Installs production dependencies only (`npm install --omit=dev`)
- Copies `package*.json` before the application source to maximize Docker layer caching

**Critical:** Minikube operates its own internal Docker daemon, which is independent of the host's Docker daemon. Images built on the host are not visible inside Minikube. The image must therefore be built within the Minikube Docker environment:

eval $(minikube docker-env)
docker build -t node-k8s-demo:1.0 .
```

The `eval` command modifies the current shell's environment variables so that subsequent `docker` commands target the Minikube daemon. This modification persists only within the current shell session.

If the image is built on the host instead, pods will fail with `ErrImagePull` because Kubernetes cannot locate the image in the cluster's runtime.

### 3. Kubernetes Deployment

#### `k8s/deployment.yaml`

The Deployment defines two replicas with the following configuration:

- **`imagePullPolicy: Never`** — prevents Kubernetes from attempting to pull the image from a remote registry. The image is expected to be present locally.
- **Resource requests and limits** — 64 MiB / 50m CPU minimum, 256 MiB / 500m CPU maximum, ensuring fair scheduling and protection against runaway processes.
- **Liveness and readiness probes** — both hit `/healthz` on port 3000. Liveness probes restart the container if it stops responding; readiness probes determine whether the pod should receive traffic.

#### `k8s/service.yaml`

The Service is of type `NodePort`, exposing the application on port `30001` of the Minikube node. This makes the application reachable from the host via `http://<minikube-ip>:30001`.

`NodePort` is suitable for development and host-level integration. Production deployments typically use `Ingress` controllers with TLS termination.

#### Applying the manifests

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl wait --for=condition=ready pod -l app=node-app --timeout=120s
```

### 4. Prometheus Integration

Because Prometheus runs on the host (outside the cluster), it must be configured to scrape the application via the Minikube `NodePort`.

#### Determining the Minikube IP

minikube ip  192.168.1.xxx

This returns the IP address assigned to the Minikube node (typically `192.168.49.2` when using the Docker driver).

#### Adding the Scrape Job

Edit `/etc/prometheus/prometheus.yml` and append the following entry under `scrape_configs`:

yaml
  - job_name: 'node-k8s-app'
    metrics_path: '/metrics'
    scrape_interval: 15s
    static_configs:
      - targets: ['192.168.49.x:30001']
        labels:
          app: 'node-k8s-demo'
          environment: 'minikube'

Replace `192.168.49.x` with the actual Minikube IP returned by the previous command.

**Indentation matters.** The `- job_name:` line must align with other entries already present under `scrape_configs:` (typically two spaces from the left margin).

#### Validating the Configuration

Before restarting Prometheus, validate the configuration file:


sudo promtool check config /etc/prometheus/prometheus.yml


The command must return `SUCCESS`. A failed validation indicates a syntax error and Prometheus will refuse to start if the existing service is restarted with an invalid configuration.

#### Applying the Changes


sudo systemctl restart prometheus
sudo systemctl status prometheus --no-pager


The service must report `active (running)`. Verify the new target is being scraped by navigating to **Status → Target health** in the Prometheus UI (typically `http://<host>:9090`) and locating the `node-k8s-app` job. Its state should be `UP` within one scrape interval.

### 5. Grafana Dashboard

#### Configuring the Data Source

In Grafana, navigate to **Connections → Data sources** and confirm that a Prometheus data source exists, pointing to `http://localhost:9090` (assuming Prometheus runs on the same host).

If no data source is present, add one and click **Save & test** to confirm connectivity.

#### Importing the Dashboard

A community-maintained dashboard is available which works out of the box with the metrics exported by this application:

1. Navigate to **Dashboards → New → Import**
2. Enter dashboard ID `11159` ([NodeJS Application Dashboard](https://grafana.com/grafana/dashboards/11159))
3. Click **Load**
4. Select the Prometheus data source
5. Click **Import**

The dashboard visualizes CPU usage, memory consumption, event loop lag, active handles, and heap statistics.

For HTTP-specific dashboards leveraging the custom metrics (`http_requests_total`, `http_request_duration_seconds`), consider dashboard IDs `12230` or `14584`.


## Verification

After completing all steps, the following commands should succeed:

# Pods are running
kubectl get pods
# Expected: 2 pods with status "Running" and "1/1" ready

# Service is exposed
kubectl get svc node-app-service
# Expected: TYPE NodePort, PORT(S) 3000:30001/TCP

# Application responds
curl http://$(minikube ip):30001/
# Expected: "Hello from Node.js running in Kubernetes! Hostname: ..."

# Metrics endpoint returns Prometheus-formatted output
curl http://$(minikube ip):30001/metrics | head
# Expected: lines beginning with "# HELP" and "# TYPE"

# Prometheus is scraping the target
curl -s "http://localhost:9090/api/v1/query?query=up{job=\"node-k8s-app\"}" | grep -o '"value":\[[^]]*\]'
# Expected: a value of 1 (indicating UP)



## Screenshots

### Prometheus Target Health

The `node-k8s-app` job appears in the Prometheus targets list with state `UP`, confirming successful scrape configuration.

![Prometheus Targets](docs/prometheus-targets.png)

### Grafana Dashboard

The imported dashboard (`11159`) displays live application metrics including CPU usage, memory consumption, event loop lag, and heap statistics.

![Grafana Dashboard](docs/grafana-dashboard.png)



## Scaling

Horizontal scaling is achieved by adjusting the replica count:


kubectl scale deployment node-app-deployment --replicas=5
kubectl get pods


The `NodePort` Service automatically distributes incoming traffic across all available replicas. This can be verified by sending repeated requests to the root endpoint and observing the returned hostname:


for i in {1..10}; do curl -s http://$(minikube ip):30001/ ; done

Each response reports the hostname of the pod that served the request.

---

## Adapting for Your Own Application

This project is intended to serve as a template. To adapt it to a production-bound Node.js application:

1. Replace the contents of `index.js` with your application code, preserving the `prom-client` integration block.
2. Update `package.json` with the actual application name, version, and dependencies.
3. Adjust resource requests and limits in `k8s/deployment.yaml` based on the application's profiled requirements.
4. Modify the readiness and liveness probe paths if `/healthz` is not appropriate.
5. Increment the image tag (e.g., `node-k8s-demo:1.1`) when rebuilding to ensure Kubernetes detects the new version on `kubectl rollout restart deployment`.

For non-Node.js applications, the same pattern applies — only the language and the metrics client library change. Comparable libraries exist for Python (`prometheus_client`), Go (`prometheus/client_golang`), Java (`micrometer` or `simpleclient`), and most other major runtimes.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| Pod stuck in `ErrImagePull` or `ImagePullBackOff` | Image was built against the host Docker daemon instead of the Minikube daemon | Run `eval $(minikube docker-env)` and rebuild, then `kubectl rollout restart deployment node-app-deployment` |
| Prometheus target shows `DOWN` | Incorrect Minikube IP or unreachable port | Re-run `minikube ip`, update `prometheus.yml`, validate, and restart Prometheus |
| `curl` from host to NodePort returns "connection refused" | Pods not ready, or Service misconfigured | Run `kubectl get pods` and `kubectl get svc` to inspect state |
| Grafana dashboard panels show "No data" | Variables (instance/job) not selected, or insufficient time since deployment | Wait one scrape interval (15s); select the correct instance from the dashboard variables |
| `promtool check config` reports YAML errors | Indentation inconsistency in `scrape_configs` | Ensure all `- job_name:` entries are aligned at the same column |
| Minikube IP changes after `minikube delete && minikube start` | Minikube re-creates its network on each cluster recreation | Update the target IP in `prometheus.yml` and restart Prometheus |

---

## Project Scope and Limitations

This project is intended as a learning and reference exercise. It deliberately omits several elements that would be required for production use:

- **Single-node cluster.** Minikube provides a single Kubernetes node, suitable for development only. Production environments use multi-node clusters such as those provided by managed services (EKS, GKE, AKS) or self-managed solutions (kubeadm, k3s).
- **No TLS.** All traffic between Prometheus and the application is plain HTTP. Production traffic should be encrypted, typically via an Ingress controller with cert-manager.
- **`NodePort` exposure.** Direct `NodePort` access is acceptable for development; production deployments should use an `Ingress` controller.
- **No CI/CD pipeline.** Image builds and deployments are manual. Production workflows should automate builds and rollouts via GitHub Actions, GitLab CI, ArgoCD, or similar.
- **No persistent storage.** The application is stateless. Stateful workloads require `PersistentVolume` claims and a configured storage class.
- **No autoscaling.** The replica count is fixed. Production deployments commonly use `HorizontalPodAutoscaler` driven by CPU, memory, or custom metrics.
- **No alerting.** This deployment exposes metrics but does not define alerting rules. Prometheus Alertmanager integration would extend this with notification routing.


## Further Reading

- [Prometheus Documentation](https://prometheus.io/docs/)
- [prom-client GitHub Repository](https://github.com/siimon/prom-client)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Minikube Handbook](https://minikube.sigs.k8s.io/docs/handbook/)
- [Grafana Documentation](https://grafana.com/docs/grafana/latest/)
- [The RED Method for monitoring microservices](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)
- [The USE Method for system performance](https://www.brendangregg.com/usemethod.html)

<img width="1711" height="177" alt="image" src="https://github.com/user-attachments/assets/9aacca24-64fa-457c-92c2-06575d403882" />
<img width="1790" height="832" alt="image" src="https://github.com/user-attachments/assets/5cbc1069-fc53-41f0-b029-986ac4c489ca" />




