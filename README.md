# 🐝 Docker Swarm and Orchestration

An interactive Reveal.js presentation on Docker Swarm — cluster setup, services, overlay networking, rolling updates, stacks, secrets, scaling, and Swarm vs Kubernetes.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Docker_Swarm_and_Orchestration/)

## 📄 [Markdown Version](presentation.md)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Topics | Fundamentals, networking, deployments, operations |
| 02 | What Is Container Orchestration? | Automating deployment, scaling, and self-healing |
| 03 | Docker Swarm Architecture | Managers, workers, and Raft consensus |
| 04 | Initialising a Swarm Cluster | swarm init, join tokens, node ls |
| 05 | Services, Tasks, and Replicas | Replicated and global service modes |
| 06 | Service Discovery and DNS | Virtual IPs and DNS round-robin |
| 07 | Overlay Networking in Swarm | VXLAN, encryption, built-in networks |
| 08 | Load Balancing & Routing Mesh | Ingress mesh and internal VIP balancing |
| 09 | Rolling Updates and Rollbacks | Parallelism, delay, failure actions |
| 10 | Docker Stacks and Stack Files | Compose-style deploys to Swarm |
| 11 | Secrets and Configs in Swarm | Encrypted secrets and config files |
| 12 | Placement Constraints and Preferences | Hard rules and spread strategies |
| 13 | Health Checks and Self-Healing | Automatic rescheduling on failure |
| 14 | Scaling Strategies | Horizontal, vertical, and node scaling |
| 15 | Monitoring Swarm Clusters | Prometheus, cAdvisor, Grafana stack |
| 16 | Swarm vs Kubernetes | Comparing complexity, features, ecosystem |
| 17 | When to Use Swarm vs Kubernetes | Deciding by team, scale, and requirements |
| 18 | Summary & Further Reading | Key takeaways and essential commands |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

[Docker Swarm Documentation](https://docs.docker.com/engine/swarm/) · [Swarm Getting Started Tutorial](https://docs.docker.com/engine/swarm/swarm-tutorial/) · [Administer a Swarm](https://docs.docker.com/engine/swarm/admin_guide/) · [Kubernetes Documentation](https://kubernetes.io/docs/)

## License

Educational use. Code examples provided as-is.
