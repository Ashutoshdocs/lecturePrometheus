# Prometheus Docker Container Monitoring Demo

## 1. Objective

In this practical, we will:

- Install and run **Prometheus**
- Install and run **cAdvisor**
- Run an **Nginx Docker container**
- Configure Prometheus to scrape cAdvisor
- Monitor Docker container CPU, memory, and network metrics
- Generate traffic and observe metric changes
- Query container metrics using **PromQL**

---

# 2. Architecture

```text
                         Prometheus
                            |
                            | Scrape /metrics
                            ↓
                         cAdvisor
                            |
                            | Container metrics
                            ↓
                      Docker Runtime
                            |
                 ┌──────────┼──────────┐
                 ↓          ↓          ↓
               Nginx       App        MySQL
```

### Components

| Component | Purpose | Port |
|---|---|---:|
| Prometheus | Collects and stores metrics | 9090 |
| cAdvisor | Exposes Docker container metrics | 8080 |
| Nginx | Container being monitored | 8081 |

---

# 3. Prerequisites

Make sure Docker is installed.

Verify:

```bash
docker --version
```

```bash
docker compose version
```

Example:

```text
Docker version 28.x.x
Docker Compose version v2.x.x
```

---

# 4. Create Project Directory

```bash
mkdir prometheus-docker-demo
cd prometheus-docker-demo
```

Create the Prometheus directory:

```bash
mkdir prometheus
```

Final structure:

```text
prometheus-docker-demo/
│
├── docker-compose.yml
│
└── prometheus/
    └── prometheus.yml
```

---

# 5. Create Prometheus Configuration

Create:

```bash
nano prometheus/prometheus.yml
```

Add:

```yaml
global:
  scrape_interval: 5s

scrape_configs:

  - job_name: "prometheus"
    static_configs:
      - targets:
          - "prometheus:9090"

  - job_name: "cadvisor"
    static_configs:
      - targets:
          - "cadvisor:8080"
```

---

# 6. Understand prometheus.yml

## global

```yaml
global:
  scrape_interval: 5s
```

This tells Prometheus:

> Collect metrics every 5 seconds.

---

## scrape_configs

```yaml
scrape_configs:
```

This section defines the targets that Prometheus should monitor.

---

## Prometheus Target

```yaml
- job_name: "prometheus"
  static_configs:
    - targets:
        - "prometheus:9090"
```

Prometheus monitors its own metrics.

---

## cAdvisor Target

```yaml
- job_name: "cadvisor"
  static_configs:
    - targets:
        - "cadvisor:8080"
```

Prometheus collects Docker container metrics from cAdvisor.

---

# 7. Create Docker Compose File

Create:

```bash
nano docker-compose.yml
```

Add:

```yaml
services:

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus

    ports:
      - "9090:9090"

    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro

    networks:
      - monitoring


  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor

    ports:
      - "8080:8080"

    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro

    networks:
      - monitoring


  nginx:
    image: nginx:alpine
    container_name: monitored-nginx

    ports:
      - "8081:80"

    networks:
      - monitoring


networks:
  monitoring:
    driver: bridge
```

---

# 8. Start the Environment

Run:

```bash
docker compose up -d
```

Expected output:

```text
[+] Running 3/3
 ✔ Container prometheus
 ✔ Container cadvisor
 ✔ Container monitored-nginx
```

---

# 9. Verify Containers

Run:

```bash
docker ps
```

Expected:

```text
CONTAINER ID   IMAGE                         PORTS
xxxx           prom/prometheus:latest       0.0.0.0:9090->9090
xxxx           gcr.io/cadvisor/cadvisor     0.0.0.0:8080->8080
xxxx           nginx:alpine                 0.0.0.0:8081->80
```

---

# 10. Test Nginx

Run:

```bash
curl http://localhost:8081
```

You should receive the Nginx HTML response.

If running this on a VM, open:

```text
http://<VM-IP>:8081
```

---

# 11. Open cAdvisor

Open in the browser:

```text
http://<VM-IP>:8080
```

cAdvisor displays information about Docker containers.

You can see information such as:

```text
CPU
Memory
Network
Filesystem
Container information
Container uptime
```

You should find containers such as:

```text
monitored-nginx
prometheus
cadvisor
```

---

# 12. Check cAdvisor Metrics

Run:

```bash
curl http://localhost:8080/metrics
```

You will see Prometheus-format metrics.

Examples:

```text
container_cpu_usage_seconds_total
```

```text
container_memory_usage_bytes
```

```text
container_network_receive_bytes_total
```

```text
container_network_transmit_bytes_total
```

This demonstrates that cAdvisor exposes container metrics through:

```text
/metrics
```

---

# 13. Open Prometheus

Open:

```text
http://<VM-IP>:9090
```

Prometheus UI will appear.

---

# 14. Check Prometheus Targets

Go to:

```text
Status
    ↓
Targets
```

You should see:

```text
prometheus    UP
cadvisor      UP
```

The important concept is:

```text
UP
```

means Prometheus can successfully scrape that target.

---

# 15. Prometheus Scraping Architecture

```text
                    Prometheus
                         |
                         |
                    HTTP GET
                         |
                         ↓
              http://cadvisor:8080/metrics
                         |
                         ↓
                      cAdvisor
                         |
                         ↓
                 Docker container
                    statistics
```

Prometheus periodically scrapes:

```text
cadvisor:8080
```

and stores the returned metrics.

---

# 16. Query Container Memory

In Prometheus, go to:

```text
Graph
```

Run:

```promql
container_memory_usage_bytes
```

This displays memory usage for containers.

---

# 17. Query Nginx Memory

Run:

```promql
container_memory_usage_bytes{name="monitored-nginx"}
```

This filters the result to the Nginx container.

---

# 18. Convert Memory to MB

Run:

```promql
container_memory_usage_bytes{name="monitored-nginx"} / 1024 / 1024
```

The result is approximately:

```text
Memory Usage
     ↓
   Bytes
     ↓
   /1024
     ↓
    KB
     ↓
   /1024
     ↓
    MB
```

---

# 19. Check Container CPU

Run:

```promql
container_cpu_usage_seconds_total
```

This is a cumulative CPU counter.

For Nginx:

```promql
container_cpu_usage_seconds_total{name="monitored-nginx"}
```

---

# 20. Calculate CPU Usage

Use:

```promql
rate(container_cpu_usage_seconds_total{name="monitored-nginx"}[1m]) * 100
```

### Explanation

```text
container_cpu_usage_seconds_total
                ↓
             rate()
                ↓
         CPU usage rate
                ↓
               *100
                ↓
          Percentage
```

---

# 21. Check Network Receive Traffic

Run:

```promql
container_network_receive_bytes_total
```

For Nginx:

```promql
container_network_receive_bytes_total{name="monitored-nginx"}
```

---

# 22. Check Network Transmit Traffic

Run:

```promql
container_network_transmit_bytes_total
```

For Nginx:

```promql
container_network_transmit_bytes_total{name="monitored-nginx"}
```

---

# 23. Generate Traffic

Open another terminal.

Run:

```bash
while true
do
  curl -s http://localhost:8081 > /dev/null
done
```

This continuously sends requests to Nginx.

Stop it using:

```text
CTRL + C
```

---

# 24. Observe Docker Statistics

While generating traffic, run:

```bash
docker stats
```

Example:

```text
CONTAINER          CPU %    MEM USAGE
monitored-nginx    1.2%     8.5MiB
prometheus         0.8%     45MiB
cadvisor           0.5%     30MiB
```

The exact values will vary.

---

# 25. Observe Metrics in Prometheus

Go back to Prometheus.

Run:

```promql
rate(container_cpu_usage_seconds_total{name="monitored-nginx"}[1m]) * 100
```

Switch to the graph view.

You can now observe CPU usage over time.

---

# 26. Important Concept

## Does Prometheus directly monitor Docker?

Not directly in this setup.

The architecture is:

```text
Docker
   |
   | Container statistics
   ↓
cAdvisor
   |
   | /metrics
   ↓
Prometheus
   |
   | PromQL
   ↓
Graph
```

cAdvisor acts as the metrics exporter for container-level information.

---

# 27. Prometheus Data Flow

```text
              Docker Containers
                      |
                      ↓
                  cAdvisor
                      |
                  /metrics
                      |
                      ↓
                 Prometheus
                      |
                  Time Series
                      |
                      ↓
                   PromQL
                      |
                      ↓
                 Visualization
```

---

# 28. Useful PromQL Queries

### All container memory

```promql
container_memory_usage_bytes
```

### Nginx memory

```promql
container_memory_usage_bytes{name="monitored-nginx"}
```

### All container CPU

```promql
container_cpu_usage_seconds_total
```

### Nginx CPU

```promql
container_cpu_usage_seconds_total{name="monitored-nginx"}
```

### Nginx CPU percentage

```promql
rate(container_cpu_usage_seconds_total{name="monitored-nginx"}[1m]) * 100
```

### Network received

```promql
container_network_receive_bytes_total
```

### Network transmitted

```promql
container_network_transmit_bytes_total
```

---

# 29. Check Prometheus Logs

Run:

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

to stop following the logs.

---

# 30. Check cAdvisor Logs

```bash
docker logs cadvisor
```

Follow:

```bash
docker logs -f cadvisor
```

---

# 31. Check Nginx Logs

```bash
docker logs monitored-nginx
```

---

# 32. Inspect Prometheus Configuration

Run:

```bash
docker exec -it prometheus sh
```

Inside the container:

```bash
cat /etc/prometheus/prometheus.yml
```

Exit:

```bash
exit
```

---

# 33. Useful Docker Commands

Check running containers:

```bash
docker ps
```

Check all containers:

```bash
docker ps -a
```

Check resource usage:

```bash
docker stats
```

Check networks:

```bash
docker network ls
```

Check the monitoring network:

```bash
docker network inspect prometheus-docker-demo_monitoring
```

---

# 34. Stop the Lab

```bash
docker compose down
```

This stops and removes the containers.

---

# 35. Start the Lab Again

```bash
docker compose up -d
```

---

# 36. Remove Everything

```bash
docker compose down -v
```

---

# 37. Troubleshooting

## Prometheus target is DOWN

Check:

```bash
docker ps
```

Then:

```bash
docker logs prometheus
```

Check cAdvisor:

```bash
docker logs cadvisor
```

---

## cAdvisor is not showing containers

Check:

```bash
docker logs cadvisor
```

Verify the Docker-related volume mounts in `docker-compose.yml`:

```yaml
volumes:
  - /:/rootfs:ro
  - /var/run:/var/run:ro
  - /sys:/sys:ro
  - /var/lib/docker:/var/lib/docker:ro
```

---

## Cannot access Prometheus from another machine

Check that port `9090` is allowed through:

```text
Cloud Firewall
       +
VM Firewall
       +
Docker port mapping
```

For Azure VM, ensure the NSG allows TCP:

```text
9090
```

Similarly, for cAdvisor:

```text
8080
```

and Nginx:

```text
8081
```

For a lab environment, you can alternatively use SSH port forwarding instead of exposing monitoring ports publicly.

---

# 38. Lab Questions

Try answering these after completing the practical.

### Question 1

Which component collects Docker container metrics?

```text
Answer: __________________
```

### Question 2

Which component stores the metrics?

```text
Answer: __________________
```

### Question 3

Which endpoint does Prometheus scrape?

```text
Answer: __________________
```

### Question 4

What does `UP` mean on the Prometheus Targets page?

```text
Answer: __________________
```

### Question 5

Which PromQL query shows container memory?

```text
Answer: __________________
```

### Question 6

How would you calculate Nginx CPU usage?

```text
Answer: __________________
```

---

# 39. Final Architecture

```text
                         ┌───────────────────┐
                         │    Prometheus     │
                         │                   │
                         │       :9090       │
                         └─────────┬─────────┘
                                   │
                              Scrape every
                                5 seconds
                                   │
                                   ↓
                         ┌───────────────────┐
                         │     cAdvisor      │
                         │                   │
                         │       :8080       │
                         └─────────┬─────────┘
                                   │
                            Container metrics
                                   │
                                   ↓
                  ┌────────────────────────────────┐
                  │         Docker Engine          │
                  │                                │
                  │   ┌────────┐  ┌────────────┐  │
                  │   │ Nginx  │  │ Prometheus │  │
                  │   └────────┘  └────────────┘  │
                  │                                │
                  └────────────────────────────────┘
```

---

# 40. Key Takeaway

Remember the three roles:

```text
cAdvisor
   ↓
EXPORTS container metrics

Prometheus
   ↓
SCRAPES + STORES metrics

PromQL
   ↓
QUERIES metrics
```

Therefore:

```text
Docker Container
       ↓
    cAdvisor
       ↓
   /metrics
       ↓
  Prometheus
       ↓
    PromQL
       ↓
    Graph
```

This is the foundation for the next practical:

```text
Docker
   ↓
cAdvisor
   ↓
Prometheus
   ↓
Grafana
   ↓
Docker Monitoring Dashboard
```