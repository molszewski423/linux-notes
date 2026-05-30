# Hybrid Cloud — Podman Homelab to AWS EC2

How to split a single-server Podman pod across a home server and AWS EC2,
using Tailscale as the secure bridge. No VPC peering, no VPN gateway, no open ports.

## The Problem

Running public-facing services (web, API, webhooks) on a home server means:
- Home IP changes (dynamic DNS workarounds needed)
- Single point of failure
- Upload bandwidth limits hurt API response times
- Cloudflare Tunnel helps but latency still higher than a datacenter

The solution: move only the public-facing services to AWS, keep internal
automation and AI workloads at home where the GPU is.

## Architecture

```
Public traffic:
  Visitor → Cloudflare → EC2 (Tunnel) → [landing, API, billing]

Internal traffic:
  EC2 ↔ Tailscale ↔ Home server → [orchestrator, scrapers, n8n, bots]
                  ↕
             Home workstation → Ollama (GPU inference)
```

The key insight: **Cloudflare Tunnel runs on EC2, not at home.**
Tunnel uses outbound-only connections — EC2 needs no open inbound ports.
Security group can block all inbound except SSH and Tailscale.

## Why Tailscale Instead of VPC/VPN

- Zero config — install on both machines, they find each other
- Works through NAT on both sides (home router + EC2 security group)
- Encrypted WireGuard underneath
- Machines use stable Tailscale IPs regardless of public IP changes
- Free for personal use (up to 100 devices)

After `tailscale up` on EC2, the instance can reach any machine on your
tailnet by IP or MagicDNS name (`http://archbox:8109`).

## Shared Database Problem

**Don't share SQLite across services running on different machines.**

SQLite uses file locking — works fine when everything is on one server with
a shared volume. Falls apart the moment services are on separate hosts.

If you're splitting services across EC2 and home:
- Migrate to PostgreSQL before splitting
- Run PostgreSQL on EC2 (or RDS later)
- All services connect over the network to the same Postgres instance
- Home server services connect via Tailscale IP

## Cloudflare Tunnel — Moving From Home to EC2

Creating a new tunnel token and running it on EC2 is all that's needed.
No DNS changes — Cloudflare's Zero Trust dashboard routes hostnames to
whichever tunnel token is active.

Cutover is instant and reversible:
- Activate EC2 tunnel in dashboard → traffic moves to EC2
- Reactivate home tunnel → traffic moves back home
- No DNS propagation wait

## EC2 Instance Sizing

For a lightweight API + static site + PostgreSQL:
- `t3.micro` (1 vCPU, 1GB): free tier eligible, very tight
- `t3.small` (2 vCPU, 2GB): ~$17/month, comfortable for 3-4 services

If running Podman + PostgreSQL + 2 Python APIs, `t3.micro` will OOM.
Start with `t3.small`, downsize later if metrics show it's underutilized.

## Kubernetes Migration Path

Designing for this from the start costs almost nothing:
1. All service URLs and secrets in environment variables (no hardcoded IPs)
2. No shared filesystems between containers
3. Health checks on every service
4. Resource limits on every container

These four rules mean your `docker-compose.yml` translates to Kubernetes
manifests with minimal rework. The compose file IS the prototype for your
Deployment + Service objects.

## Lessons

- Move the Cloudflare Tunnel to EC2 before anything else — validates networking
- PostgreSQL migration before the cutover, not during
- Keep the home tunnel running in parallel until EC2 is stable for 24+ hours
- Rollback is just switching the tunnel in the Cloudflare dashboard — < 2 min
