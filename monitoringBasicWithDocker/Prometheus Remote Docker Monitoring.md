# Prometheus Remote Docker Monitoring

## Docker on VM1 and Prometheus on VM2

## 1. Objective

In this practical, we will build a monitoring environment where:

- **VM1** runs Docker containers
- **cAdvisor** runs on VM1 and exposes Docker container metrics
- **VM2** runs Prometheus
- Prometheus remotely scrapes cAdvisor running on VM1
- We monitor an Nginx Docker container running on VM1
- We generate traffic and observe the metrics from VM2

---

# 2. Architecture

```text
                         VM2
              ┌────────────────────────┐
              │                        │
              │      Prometheus        │
              │                        │
              │       :9090            │
              │                        │
              └───────────┬────────────┘
                          │
                          │ HTTP
                          │
                          │ scrape
                          │
                          ▼
                         VM1
              ┌────────────────────────┐
              │                        │
              │       cAdvisor         │
              │        :8080           │
              │                        │
              │    Docker Metrics      │
              │          │             │
              │          ▼             │
              │    Docker Engine       │
              │          │             │
              │     ┌────┴────┐        │
              │     │         │        │
              │  ┌───────┐ ┌───────┐  │
              │  │ Nginx │ │  App  │  │
              │  └───────┘ └───────┘  │
              │                        │
              └────────────────────────┘
```

---

# 3. VM Details

We will use two VMs.

| VM | Purpose | Required Port |
|---|---|---:|
| VM1 | Docker + cAdvisor + Nginx | 8080 |
| VM2 | Prometheus | 9090 |

Example:

```text
VM1 Private IP = 10.0.1.4
VM2 Private IP = 10.0.1.5
```

Your IP addresses will be different.

Check VM1 IP:

```bash
hostname -I
```

Check VM2 IP:

```bash
hostname -I
```

---

# 4. Network Requirement

The most important requirement is:

```text
VM2
 |
 | TCP 8080
 |
 ↓
VM1
```

Prometheus on VM2 must be able to reach:

```text
VM1_PRIVATE_IP:8080
```

Test from VM2:

```bash
curl http://<VM1_PRIVATE_IP>:8080/metrics
```

If this works, Prometheus can scrape cAdvisor.

---

# PART 1 — VM1

# 5. Install Docker on VM1

Verify Docker:

```bash
docker --version
```

If Docker is not installed, install Docker using your normal Docker installation procedure.

Verify:

```bash
docker info
```

---

# 6. Run Nginx on VM1

Run:

```bash
docker run -d \
  --name monitored-nginx \
  -p 8081:80 \
  nginx:alpine
```

Check:

```bash
docker ps
```

Expected:

```text
CONTAINER ID   IMAGE          NAME
xxxx           nginx:alpine   monitored-nginx
```

Test:

```bash
curl http://localhost:8081
```

---

# 7. Install cAdvisor on VM1

Run:

```bash
docker run -d \
  --name cadvisor \
  --restart unless-stopped \
  -p 8080:8080 \
  -v /:/rootfs:ro \
  -v /var/run:/var/run:ro \
  -v /sys:/sys:ro \
  -v /var/lib/docker:/var/lib/docker:ro \
  gcr.io/cadvisor/cadvisor:latest
```

Check:

```bash
docker ps
```

You should see:

```text
monitored-nginx
cadvisor
```

---

# 8. Verify cAdvisor on VM1

Run:

```bash
curl http://localhost:8080/metrics
```

You should see Prometheus metrics.

For example:

```text
container_cpu_usage_seconds_total
container_memory_usage_bytes
container_network_receive_bytes_total
container_network_transmit_bytes_total
```

This proves that cAdvisor is working.

---

# 9. Test cAdvisor from VM2

Now move to **VM2**.

Run:

```bash
curl http://<VM1_PRIVATE_IP>:8080/metrics
```

Example:

```bash
curl http://10.0.1.4:8080/metrics
```

You should receive metrics.

This is an important test.

If this command fails, **do not continue to Prometheus configuration**.

First fix the network connectivity.

---

# 10. Azure NSG / Firewall

If the VMs are Azure VMs, make sure VM1 allows:

```text
Source: VM2 private IP/subnet
Destination port: TCP 8080
```

Prefer:

```text
VM2 subnet → VM1 TCP 8080
```

rather than:

```text
Internet → VM1 TCP 8080
```

There is normally no reason to expose cAdvisor publicly.

---

# PART 2 — VM2

# 11. Install Prometheus on VM2

Create a directory:

```bash
mkdir -p ~/prometheus
cd ~/prometheus
```

Create configuration:

```bash
nano prometheus.yml
```

Add:

```yaml
global:
  scrape_interval: 5s

scrape_configs:

  - job_name: "vm1-docker"

    static_configs:
      - targets:
          - "<VM1_PRIVATE_IP>:8080"
```

For example:

```yaml
global:
  scrape_interval: 5s

scrape_configs:

  - job_name: "vm1-docker"

    static_configs:
      - targets:
          - "10.0.1.4:8080"
```

---

# 12. Run Prometheus on VM2

Run:

```bash
docker run -d \
  --name prometheus \
  --restart unless-stopped \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
  prom/prometheus:latest
```

Check:

```bash
docker ps
```

Expected:

```text
CONTAINER ID   IMAGE                    NAME
xxxx           prom/prometheus:latest   prometheus
```

---

# 13. Open Prometheus

From your laptop/browser:

```text
http://<VM2_IP>:9090
```

For example:

```text
http://20.x.x.x:9090
```

---

# 14. Check Prometheus Target

In Prometheus:

```text
Status
   ↓
Targets
```

You should see:

```text
ENDPOINT
10.0.1.4:8080

STATE
UP
```

The important part is:

```text
UP
```

This means:

```text
Prometheus VM2
      |
      | HTTP scrape
      ↓
cAdvisor VM1
      |
      ↓
Docker metrics
```

---

# 15. Understand the Remote Monitoring Flow

The complete flow is:

```text
                    VM2
             ┌───────────────┐
             │  Prometheus   │
             │               │
             │    :9090      │
             └───────┬───────┘
                     │
                     │
                     │ HTTP GET
                     │
                     ↓
                    VM1
             ┌───────────────┐
             │   cAdvisor    │
             │    :8080      │
             └───────┬───────┘
                     │
                     ↓
                Docker Engine
                     │
             ┌───────┴───────┐
             ↓               ↓
          Nginx             App
```

Prometheus does **not** need to be installed on VM1.

Only cAdvisor needs to run on VM1.

---

# 16. Query Container Memory

In Prometheus on VM2, go to:

```text
Graph
```

Run:

```promql
container_memory_usage_bytes
```

You should get metrics from VM1.

---

# 17. Find the Nginx Container

Run:

```promql
container_memory_usage_bytes{name="monitored-nginx"}
```

This should show the memory used by the Nginx container on VM1.

---

# 18. Query CPU

Run:

```promql
container_cpu_usage_seconds_total{name="monitored-nginx"}
```

This is a cumulative CPU counter.

For CPU usage rate:

```promql
rate(container_cpu_usage_seconds_total{name="monitored-nginx"}[1m]) * 100
```

---

# 19. Query Network Traffic

Receive:

```promql
container_network_receive_bytes_total{name="monitored-nginx"}
```

Transmit:

```promql
container_network_transmit_bytes_total{name="monitored-nginx"}
```

---

# 20. Generate Traffic on VM1

On VM1:

```bash
while true
do
    curl -s http://localhost:8081 > /dev/null
done
```

Leave it running.

---

# 21. Monitor Docker on VM1

Open another terminal on VM1:

```bash
docker stats
```

You should see:

```text
CONTAINER          CPU %      MEM USAGE
monitored-nginx    ...        ...
cadvisor           ...        ...
```

---

# 22. Observe the Same Metrics from VM2

Now go to Prometheus on VM2.

Run:

```promql
rate(container_cpu_usage_seconds_total{name="monitored-nginx"}[1m]) * 100
```

You can see the Nginx CPU activity remotely.

This demonstrates:

```text
VM1 Docker
     ↓
cAdvisor
     ↓
Network
     ↓
Prometheus VM2
     ↓
PromQL
     ↓
Graph
```

---

# 23. Verify Prometheus Configuration

On VM2:

```bash
docker exec -it prometheus sh
```

Then:

```bash
cat /etc/prometheus/prometheus.yml
```

Expected:

```yaml
global:
  scrape_interval: 5s

scrape_configs:

  - job_name: "vm1-docker"

    static_configs:
      - targets:
          - "10.0.1.4:8080"
```

Exit:

```bash
exit
```

---

# 24. Check Prometheus Logs

On VM2:

```bash
docker logs prometheus
```

Follow logs:

```bash
docker logs -f prometheus
```

Press:

```text
CTRL + C
```

to stop.

---

# 25. Check cAdvisor Logs

On VM1:

```bash
docker logs cadvisor
```

---

# 26. Troubleshooting

## Problem 1 — Target is DOWN

Prometheus shows:

```text
10.0.1.4:8080 DOWN
```

First test from VM2:

```bash
curl http://10.0.1.4:8080/metrics
```

If it fails, troubleshoot networking.

---

## Problem 2 — cAdvisor works on VM1 but not VM2

On VM1:

```bash
curl http://localhost:8080/metrics
```

If this works:

```text
cAdvisor = working
```

Now on VM2:

```bash
curl http://<VM1_IP>:8080/metrics
```

If this fails:

```text
cAdvisor = working
Network = problem
```

Check:

```text
Azure NSG
Linux firewall
Routing
Private IP
Port 8080
```

---

# 27. Test Linux Firewall

On VM1:

```bash
sudo ufw status
```

If UFW is enabled, allow only VM2/subnet as appropriate.

Example:

```bash
sudo ufw allow from <VM2_PRIVATE_IP> to any port 8080 proto tcp
```

Then test again from VM2:

```bash
curl http://<VM1_PRIVATE_IP>:8080/metrics
```

---

# 28. Check Listening Port on VM1

Run:

```bash
sudo ss -lntp | grep 8080
```

You should see port `8080` listening.

Also:

```bash
docker port cadvisor
```

Expected:

```text
8080/tcp -> 0.0.0.0:8080
```

---

# 29. Important Security Point

Do **not** unnecessarily expose cAdvisor to the Internet.

Bad:

```text
Internet
   |
   ↓
VM1:8080
   |
cAdvisor
```

Better:

```text
VM2
 |
 | Private Network
 | TCP 8080
 ↓
VM1
 |
cAdvisor
```

Only Prometheus needs to reach cAdvisor.

---

# 30. Useful Commands

## VM1

Check Docker:

```bash
docker ps
```

Check Docker resource usage:

```bash
docker stats
```

Check cAdvisor:

```bash
docker ps | grep cadvisor
```

Check cAdvisor metrics:

```bash
curl http://localhost:8080/metrics
```

Check Nginx:

```bash
curl http://localhost:8081
```

Check port:

```bash
sudo ss -lntp | grep 8080
```

---

## VM2

Check Prometheus:

```bash
docker ps
```

Check Prometheus logs:

```bash
docker logs prometheus
```

Test VM1:

```bash
curl http://<VM1_PRIVATE_IP>:8080/metrics
```

Open Prometheus:

```text
http://<VM2_IP>:9090
```

---

# 31. Important PromQL Queries

### All container memory

```promql
container_memory_usage_bytes
```

### Nginx memory

```promql
container_memory_usage_bytes{name="monitored-nginx"}
```

### Nginx CPU

```promql
rate(container_cpu_usage_seconds_total{name="monitored-nginx"}[1m]) * 100
```

### Network received

```promql
container_network_receive_bytes_total{name="monitored-nginx"}
```

### Network transmitted

```promql
container_network_transmit_bytes_total{name="monitored-nginx"}
```

---

# 32. Practical Exercise

Perform the following tasks.

### Task 1

Run Nginx on VM1.

```bash
docker run -d \
  --name monitored-nginx \
  -p 8081:80 \
  nginx:alpine
```

### Task 2

Run cAdvisor on VM1.

### Task 3

Verify:

```bash
curl http://localhost:8080/metrics
```

### Task 4

From VM2:

```bash
curl http://<VM1_PRIVATE_IP>:8080/metrics
```

### Task 5

Install Prometheus on VM2.

### Task 6

Configure:

```yaml
targets:
  - "<VM1_PRIVATE_IP>:8080"
```

### Task 7

Verify:

```text
Prometheus
   ↓
Status
   ↓
Targets
   ↓
UP
```

### Task 8

Query:

```promql
container_memory_usage_bytes
```

### Task 9

Generate traffic against Nginx.

### Task 10

Observe CPU/network metrics in Prometheus.

---

# 33. Final Architecture

```text
                 PRIVATE NETWORK
        ───────────────────────────────────

                  VM2
        ┌───────────────────────┐
        │                       │
        │     Prometheus        │
        │                       │
        │       :9090           │
        │                       │
        └───────────┬───────────┘
                    │
                    │
                    │ TCP 8080
                    │
                    │ /metrics
                    ↓
                  VM1
        ┌───────────────────────┐
        │                       │
        │       cAdvisor        │
        │                       │
        │       :8080           │
        │           │           │
        │           ↓           │
        │    Docker Engine      │
        │           │           │
        │      ┌────┴────┐      │
        │      ↓         ↓      │
        │   Nginx       App     │
        │   :8081               │
        │                       │
        └───────────────────────┘
```

---

# 34. Key Learning

The most important concept from this lab is:

```text
                 VM2
           ┌─────────────┐
           │ Prometheus  │
           └──────┬──────┘
                  │
                  │ Scrape
                  │
                  ↓
                 VM1
           ┌─────────────┐
           │  cAdvisor   │
           └──────┬──────┘
                  │
                  ↓
              Docker Host
                  │
          ┌───────┼───────┐
          ↓       ↓       ↓
        Nginx     App    MySQL
```

### Remember:

```text
cAdvisor
   =
Container Metrics Exporter

Prometheus
   =
Metrics Collector + Time-Series Database

PromQL
   =
Query Language

VM1
   =
Monitored Docker Host

VM2
   =
Monitoring Server
```

The key networking requirement is simply:

```text
VM2:Prometheus
      |
      | TCP 8080
      ↓
VM1:cAdvisor
```

Prometheus does not need to run on the same machine as Docker.