# Docker Swarm & Orchestration

**Clustering, Scaling, and Managing Containerised Workloads**

> Native Docker clustering -- turning a pool of Docker hosts into a single virtual system for high-availability, load balancing, and zero-downtime deployments.

`Init` -> `Join` -> `Deploy` -> `Scale` -> `Monitor`

Swarm - Orchestration - Clustering - Services

---

## Table of Contents

1. [Topics](#topics)
2. [What Is Container Orchestration?](#what-is-container-orchestration)
3. [Docker Swarm Architecture](#docker-swarm-architecture)
4. [Initialising a Swarm Cluster](#initialising-a-swarm-cluster)
5. [Services, Tasks, and Replicas](#services-tasks-and-replicas)
6. [Service Discovery and DNS](#service-discovery-and-dns)
7. [Overlay Networking in Swarm](#overlay-networking-in-swarm)
8. [Load Balancing & Routing Mesh](#load-balancing--routing-mesh)
9. [Rolling Updates and Rollbacks](#rolling-updates-and-rollbacks)
10. [Docker Stacks and Stack Files](#docker-stacks-and-stack-files)
11. [Secrets and Configs in Swarm](#secrets-and-configs-in-swarm)
12. [Placement Constraints and Preferences](#placement-constraints-and-preferences)
13. [Health Checks and Self-Healing](#health-checks-and-self-healing)
14. [Scaling Strategies](#scaling-strategies)
15. [Monitoring Swarm Clusters](#monitoring-swarm-clusters)
16. [Swarm vs Kubernetes](#swarm-vs-kubernetes)
17. [When to Use Swarm vs Kubernetes](#when-to-use-swarm-vs-kubernetes)
18. [Summary & Further Reading](#summary--further-reading)

---

## Topics

### Swarm Fundamentals

- What is container orchestration
- Swarm architecture & Raft consensus
- Initialising a Swarm cluster
- Services, tasks, and replicas

### Networking & Discovery

- Service discovery and DNS
- Overlay networking
- Load balancing & routing mesh
- Ingress network

### Deployments & Security

- Rolling updates and rollbacks
- Docker Stacks & stack files
- Secrets and configs
- Health checks & self-healing

### Operations & Comparison

- Placement constraints & preferences
- Scaling strategies
- Monitoring Swarm clusters
- Swarm vs Kubernetes

---

## What Is Container Orchestration?

Container orchestration **automates the deployment, scaling, networking, and management** of containerised applications across a cluster of machines.

### Without Orchestration

- Manually start/stop containers on each host
- No automatic failover when a node dies
- Manual load balancing and port management
- Scaling means SSH-ing into servers
- Rolling updates are error-prone scripts

### With Orchestration

- Declare desired state, orchestrator enforces it
- Automatic rescheduling on node failure
- Built-in service discovery & load balancing
- Scale with a single command
- Zero-downtime rolling updates

### Key Orchestrators

- **Docker Swarm** -- built into Docker Engine, simple to set up
- **Kubernetes** -- industry standard, highly extensible
- **Nomad** -- HashiCorp, multi-workload scheduler

---

## Docker Swarm Architecture

A Swarm cluster consists of **manager nodes** (control plane) and **worker nodes** (data plane). Managers use the **Raft consensus algorithm** to maintain a consistent cluster state.

### Manager Nodes

- Maintain cluster state via Raft
- Schedule services onto workers
- Serve the Swarm API
- Odd number recommended (3 or 5)
- Can also run workloads

### Worker Nodes

- Execute containers (tasks)
- Report task state to managers
- No access to cluster state
- Can be promoted to manager
- Horizontally scalable

### Raft Consensus

- Leader election among managers
- Tolerates (N-1)/2 failures
- 3 managers -> tolerates 1 failure
- 5 managers -> tolerates 2 failures
- Consistent distributed log

```
Manager 1 (Leader) <-> Manager 2 <-> Manager 3
       |                   |               |
    Worker 1           Worker 2        Worker 3
```

---

## Initialising a Swarm Cluster

Swarm mode is **built into Docker Engine** -- no extra software needed. One command creates the cluster.

### Create the Swarm (on manager)

```bash
# Initialise the swarm on the first manager
docker swarm init --advertise-addr 192.168.1.10

# Output includes a join token for workers
# Swarm initialized: current node is now a manager
```

### Join as Worker

```bash
# Run on each worker node
docker swarm join \
  --token SWMTKN-1-xxx...xxx \
  192.168.1.10:2377
```

### Join as Manager

```bash
# Get the manager join token
docker swarm join-token manager

# Run on additional manager nodes
docker swarm join \
  --token SWMTKN-1-yyy...yyy \
  192.168.1.10:2377
```

### Verify the Cluster

```bash
# List all nodes
docker node ls
# ID          HOSTNAME   STATUS  AVAILABILITY  ROLE
# abc123 *   manager1   Ready   Active        Leader
# def456     worker1    Ready   Active
# ghi789     worker2    Ready   Active
```

---

## Services, Tasks, and Replicas

In Swarm, you deploy **services** (not individual containers). Each service spawns one or more **tasks**, and each task runs exactly one container.

### Service Types

- **Replicated** -- run N copies, Swarm distributes them across nodes
- **Global** -- run exactly one task on every node (e.g., monitoring agents)

```bash
# Create a replicated service
docker service create \
  --name web \
  --replicas 3 \
  -p 80:80 \
  nginx:alpine

# Create a global service
docker service create \
  --name node-exporter \
  --mode global \
  prom/node-exporter
```

### Inspect and Manage

```bash
# List services
docker service ls

# See task placement
docker service ps web
# ID       NAME    NODE      STATE
# a1b2c3   web.1   worker1   Running
# d4e5f6   web.2   worker2   Running
# g7h8i9   web.3   manager1  Running

# View service details
docker service inspect --pretty web

# View logs across all replicas
docker service logs web
```

### Task Lifecycle

`New -> Pending -> Assigned -> Accepted -> Preparing -> Starting -> Running -> Complete`

---

## Service Discovery and DNS

Swarm has a **built-in DNS server** that automatically assigns each service a DNS entry. Containers can reach other services simply by name.

### How It Works

- Each service gets a **Virtual IP (VIP)** on the overlay network
- DNS resolves the service name to the VIP
- VIP load-balances across all healthy tasks
- No external service registry required

### DNS Round-Robin (alternative)

- Use `--endpoint-mode dnsrr`
- DNS returns all task IPs directly
- Useful when external LB handles distribution
- No VIP -- client decides which IP to use

```bash
# Services on the same overlay network can resolve each other by name
docker service create --name api \
  --network backend \
  myapp/api

docker service create --name db \
  --network backend \
  postgres:16

# Inside "api" container:
# ping db        => resolves to VIP 10.0.1.5
# nslookup db    => returns 10.0.1.5 (VIP)

# With DNS round-robin:
docker service create --name api \
  --network backend \
  --endpoint-mode dnsrr \
  myapp/api
# nslookup api => returns all task IPs
```

---

## Overlay Networking in Swarm

Overlay networks create a **distributed network across all Swarm nodes**, enabling containers on different hosts to communicate as if they were on the same LAN.

### Key Concepts

- Uses VXLAN encapsulation under the hood
- Encrypted by default with `--opt encrypted`
- Scoped to services attached to the network
- Built-in **ingress** network for published ports

```bash
# Create an overlay network
docker network create \
  --driver overlay \
  --subnet 10.0.9.0/24 \
  --opt encrypted \
  my-overlay

# Attach services to the network
docker service create --name web \
  --network my-overlay \
  nginx:alpine
```

### Built-in Networks

| Network | Purpose |
|---------|---------|
| **ingress** | Handles published port routing mesh |
| **docker_gwbridge** | Connects overlay to host network |
| **user-defined overlay** | Service-to-service communication |

### Network Isolation

- Services on different overlays are isolated
- A service can join multiple overlay networks
- Use networks to segment frontend / backend / data tiers

---

## Load Balancing & Routing Mesh

Swarm provides **two layers of load balancing**: an external routing mesh (ingress) and internal VIP-based balancing.

### Routing Mesh (Ingress)

- Published ports are available on **every node**
- Hit any node on port 80 -> reaches a running task
- Even nodes not running the service route traffic
- Uses IPVS (IP Virtual Server) in the kernel

```
Client :80 -> Any Node -> Task (any node)
```

### Internal Load Balancing

- Service VIP distributes among healthy tasks
- Transparent to the application
- Automatic health-aware routing

### Bypassing the Mesh

- Use `--publish mode=host` for direct host binding
- Needed for source-IP preservation
- Only reaches task on that specific node

```bash
# Default: routing mesh
docker service create -p 80:80 web

# Host mode: bypass mesh
docker service create \
  --publish mode=host,target=80,published=80 \
  web
```

---

## Rolling Updates and Rollbacks

Swarm performs **rolling updates** by incrementally replacing tasks with the new version, ensuring zero downtime.

### Update Configuration

```bash
# Update image with rolling strategy
docker service update \
  --image nginx:1.27 \
  --update-parallelism 2 \
  --update-delay 10s \
  --update-failure-action rollback \
  --update-max-failure-ratio 0.25 \
  --update-order start-first \
  web
```

### Update Parameters

- **parallelism** -- tasks updated simultaneously
- **delay** -- wait between batches
- **failure-action** -- pause, continue, or rollback
- **order** -- stop-first (default) or start-first

### Automatic Rollback

```bash
# Rollback to the previous version
docker service rollback web

# Or configure auto-rollback
docker service create \
  --name web \
  --replicas 6 \
  --update-failure-action rollback \
  --rollback-parallelism 2 \
  --rollback-delay 5s \
  --rollback-max-failure-ratio 0.1 \
  nginx:1.26
```

### Update Lifecycle

`v1 Running -> Drain v1 -> Start v2 -> v2 Healthy`

---

## Docker Stacks and Stack Files

A **stack** is a group of related services defined in a Compose file and deployed to a Swarm cluster. Think of it as **docker-compose for production**.

### Stack File Example

```yaml
# docker-stack.yml
version: "3.8"
services:
  web:
    image: myapp/web:2.1
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
    ports:
      - "80:80"
    networks:
      - frontend
      - backend

  api:
    image: myapp/api:2.1
    deploy:
      replicas: 2
    networks:
      - backend

  db:
    image: postgres:16
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay

volumes:
  db-data:
```

### Stack Commands

```bash
# Deploy a stack
docker stack deploy -c docker-stack.yml myapp

# List stacks
docker stack ls

# List services in a stack
docker stack services myapp

# List tasks in a stack
docker stack ps myapp

# Remove a stack
docker stack rm myapp
```

### Stack vs Compose

- Stacks use the same YAML format as Compose
- The `deploy:` key is only used in Swarm mode
- `build:` is ignored -- stacks require pre-built images
- Stacks manage services, networks, volumes, secrets
- Re-deploying a stack performs a rolling update

---

## Secrets and Configs in Swarm

Swarm provides **first-class secret management** -- secrets are encrypted at rest, transmitted only to nodes running tasks that need them, and mounted as in-memory files.

### Secrets

- Encrypted in the Raft log at rest
- Transmitted over TLS to assigned nodes only
- Mounted as files in `/run/secrets/`
- Never stored on disk in the container
- Immutable -- update by rotating

```bash
# Create a secret
echo "s3cureP@ss" | docker secret create db_pass -

# Use in a service
docker service create --name api \
  --secret db_pass \
  myapp/api

# Inside container: cat /run/secrets/db_pass

# Rotate a secret
echo "newP@ss" | docker secret create db_pass_v2 -
docker service update \
  --secret-rm db_pass \
  --secret-add db_pass_v2 api
```

### Configs

- Similar to secrets but for non-sensitive data
- Not encrypted at rest
- Mounted at any path in the container
- Great for config files (nginx.conf, etc.)

```bash
# Create a config
docker config create nginx_conf ./nginx.conf

# Use in a service
docker service create --name web \
  --config source=nginx_conf,target=/etc/nginx/nginx.conf \
  nginx:alpine
```

### In Stack Files

```yaml
secrets:
  db_password:
    external: true    # pre-created
  api_key:
    file: ./api_key.txt  # from file

services:
  api:
    secrets:
      - db_password
      - api_key
```

---

## Placement Constraints and Preferences

Control **where tasks are scheduled** using constraints (hard rules) and preferences (soft rules / spread strategies).

### Constraints (Hard Rules)

- Task will **only** run on nodes matching the constraint
- Based on node labels, role, hostname, engine labels

```bash
# Add labels to nodes
docker node update --label-add zone=eu-west worker1
docker node update --label-add ssd=true worker2

# Constrain to specific nodes
docker service create --name db \
  --constraint 'node.labels.ssd == true' \
  --constraint 'node.role == worker' \
  postgres:16

# Constrain to a specific hostname
docker service create --name monitoring \
  --constraint 'node.hostname == manager1' \
  grafana/grafana
```

### Preferences (Soft Rules)

- Swarm **tries** to spread evenly across label values
- Best-effort -- tasks still schedule if preference cannot be met

```bash
# Spread replicas across availability zones
docker service create --name web \
  --replicas 6 \
  --placement-pref 'spread=node.labels.zone' \
  nginx:alpine
# 2 tasks in zone=eu-west
# 2 tasks in zone=eu-central
# 2 tasks in zone=us-east
```

### In Stack Files

```yaml
deploy:
  placement:
    constraints:
      - node.labels.ssd == true
    preferences:
      - spread: node.labels.zone
```

---

## Health Checks and Self-Healing

Swarm continuously monitors task health. When a task becomes **unhealthy** or a node goes down, the orchestrator **automatically reschedules** tasks to maintain desired state.

### Health Check Config

```bash
# Define health check on service
docker service create --name web \
  --health-cmd "curl -f http://localhost/ || exit 1" \
  --health-interval 30s \
  --health-timeout 10s \
  --health-retries 3 \
  --health-start-period 60s \
  nginx:alpine
```

```dockerfile
# Or in the Dockerfile
HEALTHCHECK --interval=30s --timeout=10s \
  --retries=3 --start-period=60s \
  CMD curl -f http://localhost/ || exit 1
```

### Self-Healing Behaviour

- Unhealthy task -> stopped and replaced
- Node failure -> tasks rescheduled to healthy nodes
- Manager failure -> Raft elects a new leader
- Desired state always reconciled by the orchestrator

### Restart Policies

```yaml
deploy:
  restart_policy:
    condition: on-failure  # none | on-failure | any
    delay: 5s
    max_attempts: 3
    window: 120s
```

`Running -> Unhealthy -> Stopped -> New Task`

---

## Scaling Strategies

Scale services up or down with a **single command**. Swarm distributes new tasks across available nodes automatically.

### Manual Scaling

```bash
# Scale a single service
docker service scale web=10

# Scale multiple services at once
docker service scale web=10 api=5 worker=8

# Scale down
docker service scale web=2

# Update replicas (alternative)
docker service update --replicas 10 web
```

### Resource Limits

```yaml
deploy:
  resources:
    limits:
      cpus: '0.50'
      memory: 512M
    reservations:
      cpus: '0.25'
      memory: 256M
```

### Scaling Patterns

- **Horizontal** -- increase replicas across nodes
- **Vertical** -- increase CPU/memory limits
- **Node scaling** -- add more worker nodes to the swarm
- **Global services** -- auto-scale with cluster size

### Best Practices

- Set resource reservations so the scheduler can bin-pack
- Use placement preferences to spread across zones
- Monitor with `docker service ps` to verify distribution
- Combine with external autoscalers (Orbiter, Docker Flow) for automatic scaling
- Use global mode for per-node agents (log shippers, exporters)

---

## Monitoring Swarm Clusters

Effective monitoring covers **cluster health**, **service metrics**, and **container-level resource usage**.

### Built-in Commands

```bash
# Cluster overview
docker node ls
docker service ls

# Task health and placement
docker service ps web --no-trunc

# Container stats (per node)
docker stats

# System-wide info
docker system df
docker info
```

### Prometheus Stack

- **cAdvisor** -- container metrics (CPU, mem, net)
- **Node Exporter** -- host-level metrics
- **Prometheus** -- scrapes & stores time series
- **Grafana** -- dashboards & alerting

### Monitoring Stack (as Swarm services)

```yaml
services:
  prometheus:
    image: prom/prometheus
    deploy:
      placement:
        constraints: [node.role == manager]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports: ["9090:9090"]

  grafana:
    image: grafana/grafana
    ports: ["3000:3000"]

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    deploy:
      mode: global   # one per node
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro

  node-exporter:
    image: prom/node-exporter
    deploy:
      mode: global
```

---

## Swarm vs Kubernetes

| Aspect | Docker Swarm | Kubernetes |
|--------|-------------|------------|
| Setup complexity | One command (`docker swarm init`) | Multiple components (API server, etcd, kubelet, etc.) |
| Learning curve | Gentle -- extends familiar Docker CLI | Steep -- new concepts (Pods, Deployments, Ingress...) |
| Auto-scaling | Manual or external tools | Built-in HPA/VPA, KEDA |
| Networking | Simple overlay + routing mesh | CNI plugins (Calico, Cilium, Flannel...) |
| Service mesh | Not built-in | Istio, Linkerd, Cilium mesh |
| Storage | Docker volumes, NFS plugins | CSI drivers, PVs/PVCs, StorageClasses |
| Ecosystem | Smaller, Docker-focused | Massive (Helm, Operators, CRDs, GitOps...) |
| Production adoption | Niche / small-medium workloads | Industry standard, all cloud providers |
| Configuration | Compose YAML (familiar) | K8s manifests (verbose) |
| Community | Smaller, less active | Huge, CNCF backed |

---

## When to Use Swarm vs Kubernetes

### Choose Docker Swarm When...

- You want the **simplest path to orchestration**
- Your team already knows Docker and Docker Compose
- Small to medium cluster (3-20 nodes)
- You need something production-ready in hours, not weeks
- Internal / non-critical workloads
- No dedicated platform / DevOps team
- Limited budget -- no managed K8s costs

### Choose Kubernetes When...

- You need **auto-scaling and advanced scheduling**
- Large-scale deployments (50+ nodes, 100s of services)
- Multi-cloud or hybrid-cloud strategy
- You need a service mesh, advanced ingress, CRDs
- Strong ecosystem requirements (Helm charts, Operators)
- You have a dedicated platform team
- Managed K8s available (EKS, GKE, AKS)

### The Middle Ground

Start with **Swarm** to learn orchestration concepts (services, replicas, rolling updates, overlay networks). These concepts transfer directly to Kubernetes. Many teams start with Swarm and migrate to K8s when they outgrow it.

---

## Summary & Further Reading

### Key Takeaways

- Docker Swarm is the **simplest container orchestrator**
- Built into Docker Engine -- no extra installs
- Manager/worker architecture with Raft consensus
- Declarative services with automatic reconciliation
- Built-in DNS, overlay networking, routing mesh
- Rolling updates and automatic rollback
- Secrets management and config injection
- Self-healing: failed tasks are rescheduled

### Essential Commands

- `docker swarm init` -- create a cluster
- `docker service create` -- deploy a service
- `docker service scale` -- scale replicas
- `docker service update` -- rolling update
- `docker stack deploy` -- deploy from Compose
- `docker node ls` -- inspect cluster

### Further Reading

- [Docker Swarm Official Docs](https://docs.docker.com/engine/swarm/)
- [Swarm Getting Started Tutorial](https://docs.docker.com/engine/swarm/swarm-tutorial/)
- [Administer a Swarm](https://docs.docker.com/engine/swarm/admin_guide/)
- [Example Voting App (Swarm Stack)](https://github.com/dockersamples/example-voting-app)
