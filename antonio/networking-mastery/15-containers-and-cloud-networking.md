# 15. Containers & Cloud Networking

Goal: how everything in Parts 1–3 applies once your app runs inside containers and/or a cloud provider's network — including the single most common piece of confusion beginners hit with Docker, which this platform's own config makes concrete.

## 15.1 Docker networking — the essentials

Each Docker container gets its own **network namespace** — its own virtual network stack, complete with its own loopback interface, its own view of "localhost," and (on the default **bridge** network) its own private IP address and virtual Ethernet interface (chapter 2 §2.1's MAC/NIC concepts, implemented in software) connected to a virtual switch Docker manages.

```
Host machine
 │
 ├── Container A  (own network namespace, own "localhost")
 │      │
 │      └── listening on :3000 INSIDE the container
 │
 └── Container B  (own network namespace, own "localhost")
        │
        └── listening on :5432 INSIDE the container
```

**Port publishing** (`docker run -p 8080:3000 ...` / `ports: ["8080:3000"]` in Compose) is Docker's own **DNAT** (chapter 4 §4.2) — it maps a port on the *host* to a port *inside* the container's network namespace, so something outside Docker entirely can reach the containerized service. Without an explicit port publish, a container's listening ports are only reachable from *other containers on the same Docker network* — not from the host, and not from outside.

## 15.2 The pitfall that causes enormous, entirely predictable confusion

> `localhost` inside a container is not the same `localhost` as your host machine, and it is not the same `localhost` as any *other* container.

This is directly, concretely relevant to this workspace's actual configuration, and worth internalizing precisely because it will bite you (or already has) the first time you containerize something that currently works via `localhost`:

```json
"GeoApi": { "Address": "https://localhost:5005/geo-api" }
"ClamAv": { "tcp": "tcp://localhost:3310" }
```
(chapter 6 §6.3, `HRNS.WebApi/appsettings.json`.) These values are entirely correct **today**, when `HRNS.Platform.Server` and its sibling services (`HRNS.Gelocation.Server`, a locally-running ClamAV daemon) all run directly on the same developer machine, sharing one OS network namespace. **The instant any of these processes moves into its own Docker container, these exact same config values become wrong** — `localhost` from inside the platform server's container would resolve to *the platform server's own container*, not to a sibling container running the geolocation service or ClamAV, even if both containers are running on the same physical host and even on the same Docker network. This is precisely why containerized multi-service setups configure services to reach each other by **container/service name** instead:

```yaml
# illustrative docker-compose.yml — not a file that exists in this repo (../dotnet-mastery/ ch.11 §11.1)
services:
  platform-server:
    # ...
    environment:
      GeoApi__Address: "http://gelocation-server:8080/geo-api"   # service name, not "localhost"
  gelocation-server:
    # ...
  clamav:
    image: clamav/clamav
    # ...
    # platform-server would reach it as "tcp://clamav:3310", not "tcp://localhost:3310"
```

Docker's built-in **embedded DNS** resolves each service's container name to its current internal IP automatically on a user-defined bridge network (the default network Compose creates) — this is a real, working instance of the DNS concepts from chapter 7, just scoped to one Docker host instead of the public internet, and it's what makes `gelocation-server` resolvable as a hostname at all inside that network.

**Host networking** (`--network host`) is Docker's escape hatch from all of this — the container shares the host's network namespace entirely, so `localhost` inside the container genuinely *is* the host's `localhost`. Simpler in exactly the way this section's confusion suggests, at the cost of losing the network isolation between containers that's usually the whole point of containerizing in the first place — generally a deliberate, narrow-use-case choice, not a default.

## 15.3 Kubernetes networking — the concepts, briefly

You don't need this at the beginning, and this platform's repo has no Kubernetes manifests to ground it in (consistent with the honest "no committed CI/CD or infra" finding from `../dotnet-mastery/` chapter 11 §11.1) — but the vocabulary is worth recognizing, since it's the same underlying ideas from this whole guide applied at a much larger, automated scale:

- **Pod** — the smallest deployable unit; one or more containers that share a network namespace (so containers *within* one pod really do share `localhost`, unlike separate Docker containers — a deliberate, different design choice from plain Docker Compose).
- **Pod IP** — each pod gets its own IP, cluster-internal — conceptually similar to a container's bridge-network IP, but managed cluster-wide.
- **Service / ClusterIP** — a stable, virtual IP address load-balancing (chapter 12 §12.2) across a *set* of pods (which come and go, get rescheduled, change IPs constantly) — solving exactly the service-discovery problem from chapter 14 §14.2, but as a first-class platform feature rather than something an app has to implement itself (recall `FileServerHealthCheck`'s database-driven discovery — Kubernetes Services are the "someone else already solved this generically" alternative).
- **NodePort / LoadBalancer** — ways of exposing a Service outside the cluster — NodePort opens a port on every cluster node directly; LoadBalancer provisions an actual external load balancer (chapter 12) from the underlying cloud provider.
- **Ingress** (and the newer **Gateway API**) — an L7 (chapter 12 §12.2) HTTP(S) routing layer in front of multiple Services, handling TLS termination (chapter 9 §9.5) and path/host-based routing for a whole cluster's worth of applications through one entry point.
- **kube-proxy** / **CoreDNS** — the actual mechanisms implementing Service virtual IPs and in-cluster DNS resolution, respectively — the Kubernetes-native answer to exactly this chapter's §15.2 problem (pods reach each other by Service *name*, resolved via CoreDNS, never by `localhost`).
- **CNI** (Container Network Interface) — the plugin standard defining how pod networking actually gets wired up under the hood (there are several implementations — Calico, Cilium, and others — each with different trade-offs, chapter 18 territory).
- **Network policies** — Kubernetes-native firewall rules (chapter 13 §13.1's concepts) controlling which pods can talk to which other pods.

## 15.4 Cloud networking — VPCs, subnets, and the same primitives, provider-branded

Every major cloud provider's networking model is built from the same conceptual primitives you already know from Parts 1–3, wrapped in provider-specific naming:

```
VPC / Virtual Network                    ← your own private, isolated slice of the cloud's network (chapter 3's addressing, cloud-scale)
│
├── Public subnet                          ← chapter 3's subnetting, applied to cloud resource placement
│     └── has a route (chapter 4 §4.1) to an Internet Gateway
│
├── Private subnet                           ← no direct route to the internet
│     └── outbound-only internet access via a NAT Gateway (chapter 4 §4.2's NAT, as a managed cloud service)
│
├── Route table                                 ← chapter 4 §4.1's routing table, as a cloud resource
├── Security group                                ← chapter 13 §13.1's stateful firewall, scoped per-instance
├── Network ACL                                     ← chapter 13 §13.1's stateless firewall, scoped per-subnet
└── Load balancer                                     ← chapter 12 §12.2, as a managed cloud service
```

The pattern worth internalizing, rather than memorizing any one provider's specific console/terminology: a database typically lives in a **private subnet** (no direct internet route at all — reachable only from within the VPC, e.g. from your app servers) — this maps directly onto the "restrict inbound, only what's explicitly needed" firewall posture from chapter 13 §13.1, just expressed as network topology instead of firewall rules. An application server needing outbound internet access (to call SendGrid, Firebase, reCAPTCHA — this platform's real external dependencies) but not needing to accept unsolicited inbound connections from the public internet directly (because a load balancer/reverse proxy, chapter 12, is what actually faces the internet) is exactly the **private subnet + NAT gateway** pattern above — outbound works, unsolicited inbound doesn't.

## Checkpoint

1. If `HRNS.Gelocation.Server` were containerized separately from `HRNS.Platform.Server` and both were run via Docker Compose on the same developer machine, what specific config value in `appsettings.json` (chapter 6 §6.3) would need to change, and to what, per §15.2?
2. Explain why a database server typically belongs in a cloud VPC's **private** subnet rather than the public one, using chapter 13 §13.1's inbound/outbound firewall vocabulary.
3. `../dotnet-mastery/` chapter 11 §11.3 sketched an illustrative multi-stage Dockerfile for `HRNS.Platform.Server`. If you were also containerizing `HRNS.Gelocation.Server` and a ClamAV daemon alongside it via Docker Compose, sketch (in words, not YAML) what the `GeoApi:Address` and `ClamAv:tcp` config values would need to look like inside that compose file, per this chapter's service-name-not-localhost rule.
