# Trading Platform Architecture

## Overview

This doc covers the full architecture of our trading platform on AWS. There are two main flows: the Frontend Flow that serves static assets through CDN, and the Backend API Flow that handles trading logic, real-time market data, and order management through microservices running on Kubernetes.

Everything runs on AWS managed services, but at the application layer we intentionally use Kubernetes-native tools (Kong, Istio, Helm, ArgoCD) so that if we ever need to move to another cloud, we only swap out the infrastructure layer — no changes to application code or deployment pipelines.

---

## Frontend Flow — WAF, CloudFront, S3

### How it works

The frontend is a SPA built into static files, hosted on S3, and distributed through CloudFront CDN. Every request goes through WAF before reaching CloudFront.

### Components

**Client (Browser / Mobile App)**

Users access the app through browser or mobile. The client talks to the backend via REST API for things like placing orders, authentication, wallet management — and via WebSocket for real-time market data streaming.

**Route 53**

DNS service that routes users to the nearest AWS resource based on latency or geo-location. Route 53 also handles multi-region failover — it runs health checks on the primary region's endpoint and automatically switches traffic to the secondary region if the primary goes down.

**AWS WAF**

Sits in front of CloudFront and API Gateway, filtering requests at the application layer. Blocks common attacks: SQL injection, XSS, brute force, bot traffic. WAF rules are configured as Web ACLs and attached to both the CloudFront distribution and the API Gateway endpoint, so frontend and backend share the same security rules.

**CloudFront**

CDN that distributes static assets (HTML, JS, CSS, images) from edge locations closest to the user. We configure it with:

- Origin Access Control (OAC) so the S3 bucket is not publicly accessible — all traffic must go through CloudFront.
- Separate cache behaviors per file type — long TTL for hashed assets (JS/CSS bundles), short TTL for `index.html` so users always get the latest version.
- Custom error pages that return `index.html` on 403/404 to make SPA client-side routing work properly.
- HTTPS enforced everywhere, SSL certs managed through ACM.

**S3 Bucket**

Stores all the static files after build. The bucket has public access completely blocked — only CloudFront can read from it via OAC. Versioning is enabled for rollback, and lifecycle policies auto-clean old build artifacts.

### Data Flow

Client sends a request to `https://app.example.com`. Route 53 resolves DNS and points to the nearest CloudFront edge. Before reaching CloudFront, the request passes through WAF — if it looks malicious, it gets blocked right there.

Clean requests hit CloudFront. If the content is already cached (cache hit), it's returned immediately. If not (cache miss), CloudFront fetches from S3, caches it at the edge, then returns it. Next time the same asset is requested, it's served from cache.

---

## Backend API Flow

### How it works

Backend runs microservices on EKS, communicating internally through Istio service mesh with mTLS. The entry point is Kong Gateway running on K8s instead of AWS API Gateway — this is a deliberate decision for portability. The system is deployed multi-region active-passive, Region A as primary and Region B as DR standby.

### API Gateway — Kong on Kubernetes

We chose Kong over AWS API Gateway for one reason: Kong runs on K8s, so it's portable. You configure it once and it works anywhere. AWS API Gateway locks you into AWS — if you move clouds, you have to rewrite all your routing, auth, and rate limiting from scratch.

Kong is deployed as a Kubernetes Ingress Controller inside the EKS cluster, sitting behind an AWS NLB (Network Load Balancer) that K8s provisions automatically. Route 53 points DNS to this NLB. WAF is attached at the NLB level to filter traffic before it reaches Kong.

What Kong handles:
- Routes requests to the right microservice pod based on URL path and headers.
- Rate limiting and throttling to protect backend services.
- JWT validation, API key management, OAuth2 through its plugin ecosystem.
- TLS termination.
- WebSocket support for market data streaming.

If we move to another cloud later, only the NLB part changes (Terraform handles that). Kong config stays exactly the same.

### EKS Cluster

**Node Groups**

The EKS cluster runs managed node groups on EC2 instances. We split into multiple node groups for different workload profiles — compute-optimized for the Trading Engine (needs strong CPU and low latency), memory-optimized for Order Book service (heavy Redis interaction), and general-purpose for Auth and other services.

For node scaling we use Karpenter instead of the traditional Cluster Autoscaler. Karpenter is faster because it looks directly at pending pods and provisions exactly the right instance type, rather than relying on pre-configured node group settings.

**Istio Service Mesh**

All traffic between pods goes through Istio. Every connection is mTLS — no service trusts another without certificate verification. Beyond security, Istio gives us:

- Traffic splitting for canary deployments — new version gets 5% of traffic first, then gradually ramps up.
- Circuit breaking — if a service keeps failing, Istio cuts the connection so it doesn't cascade.
- Automatic distributed trace and metrics collection for observability.
- Per-route retry policies and timeout configuration.

Istio is a CNCF project, runs on any K8s cluster. The entire config (VirtualService, DestinationRule, PeerAuthentication) is portable as-is — take it to another cloud and it just works.

### Microservices

**Auth Service** — Handles user authentication, issues and validates JWT tokens. Integrates with external identity providers via OAuth2/OIDC.

**Trading Engine (Matching Engine)** — Core order matching logic. Reads/writes the order book from Redis for the lowest possible latency. When a match happens, it publishes trade events to Kafka for downstream services. This one needs to run on compute-optimized nodes.

**Order Book Service** — Manages the real-time order book state in Redis. Exposes REST endpoints for querying order book depth and spread. Subscribes to Kafka topics for order updates and trade confirmations.

**Market Data Service** — Streams real-time market data (price ticks, trade history, order book changes) to clients via WebSocket. Subscribes to Kafka topics to aggregate data. Also supports historical data queries from PostgreSQL for charting.

**Wallet Service** — Manages user balances (deposits, withdrawals, trade settlements). Ensures consistency through database transactions and idempotent operations on PostgreSQL. Publishes balance update events to Kafka for audit trail.

All services are packaged as Helm charts with Kustomize overlays for each environment (dev/staging/production). This packaging doesn't depend on any specific cloud — the same charts deploy to any K8s cluster.

### Event Streaming — Amazon MSK (Kafka)

Kafka is the central event bus. It keeps services decoupled and communicating asynchronously. Key topics:

- `orders.placed` — New orders from Trading Engine.
- `trades.executed` — Match results.
- `balances.updated` — Wallet balance changes.
- `market.ticks` — Price updates for Market Data Service.

MSK is configured multi-AZ (3 AZs), replication factor 3, storage auto-scaling. We use MSK Connect to archive events to S3 when needed.

For cross-region DR, Kafka MirrorMaker2 replicates topics from Region A to Region B with a few seconds of lag.

If we ever want to self-host Kafka for full portability, Strimzi Kafka Operator on K8s is a drop-in replacement. Application code and topic config stay the same — only the connection endpoint changes.

### Cache — ElastiCache Redis

Redis is used for a few things: storing the live order book state (sub-millisecond reads/writes for Trading Engine), caching session tokens for Auth, rate limiting counters for Kong, and internal pub/sub when we need lower latency than Kafka.

Configured with cluster mode and sharding, multi-AZ auto failover, read replicas for read-heavy workloads, encryption in-transit and at-rest.

If we need portability, swap it with a Redis Operator running on K8s. Application code doesn't change — just the connection string.

### Database — Aurora PostgreSQL

Aurora PostgreSQL is the primary database for all persistent data: user accounts, KYC info, trade history, order logs, wallet balances, market data history for charting.

Configured with multi-AZ and auto failover, read replicas for analytical queries, storage auto-scaling (up to 128 TiB), point-in-time recovery, and IAM database authentication so EKS pods connect without passwords.

For cross-region DR, Aurora Global Database replicates to Region B with RPO under 1 second.

If we move off AWS, CloudNativePG operator handles PostgreSQL natively on K8s (automated failover, continuous backup, rolling updates). Or we can use Crossplane to provision whatever managed database the target cloud offers, all from K8s manifests.

---

## Multi-Region Disaster Recovery

The system runs active-passive: Region A takes all live traffic, Region B is a warm standby that continuously receives replicated data from Region A.

| Component | Replication method | RPO |
|-----------|--------------------|-----|
| Kafka | MirrorMaker2 cross-region | A few seconds |
| PostgreSQL | Aurora Global Database | < 1 second |
| Redis | ElastiCache Global Datastore | Sub-second |
| Static Assets | S3 Cross-Region Replication | A few minutes |

When Region A goes down, Route 53 health checks detect the failure and switch DNS to Region B. Aurora promotes the read replica to primary writer, Redis promotes its replica to primary, Kafka consumers in Region B start processing from replicated topics. The whole thing takes a few minutes, with minimal data loss.

---

## Auto Scaling

**Node level** — Karpenter watches for pending pods, provisions the right EC2 instance type on the spot. When load drops, it drains and terminates nodes. Much faster than Cluster Autoscaler because it's not constrained by static node group configs.

**Pod level** — HPA scales pods based on CPU, memory, and custom metrics (request latency, Kafka consumer lag, order queue depth via Prometheus Adapter or KEDA). Each service has its own HPA thresholds tuned to its workload.

**Kafka** — Add brokers for more throughput, increase partitions to parallelize consumers. Storage auto-scales.

**Redis** — Add shards for write-heavy workloads, add read replicas for read-heavy, or upgrade instance size.

**Aurora** — Add read replicas (up to 15), upgrade instance class, storage auto-scales.

---

## Observability

The monitoring stack is fully cloud-agnostic. We don't rely on CloudWatch as the primary monitoring tool.

**Metrics** — Prometheus scrapes metrics from all pods, Istio proxies, and Kong. Grafana displays dashboards for trading performance, Kafka consumer lag, Redis hit/miss ratios, Aurora query latency, and Istio mesh traffic.

**Traces** — OpenTelemetry Collector runs as a DaemonSet, collects distributed traces and exports to Jaeger or Grafana Tempo. This gives us full visibility of a request flow: Client → Kong → Auth → Trading Engine → Kafka → Order Book → PostgreSQL.

**Logs** — Loki aggregates logs from all pods through Promtail or Fluent Bit. Logs are labeled by namespace, service, and pod for easy filtering and correlation with metrics and traces.

**Alerting** — Alertmanager routes alerts by severity: PagerDuty for P1 incidents (service down, data loss risk), Slack for P2/P3 (high latency, scaling events).

---

## Infrastructure as Code — Terraform

All infrastructure is provisioned through Terraform, organized into modules per concern: networking, EKS, RDS, ElastiCache, MSK, CDN, WAF, DNS. Each environment (dev/staging/production) and each region has its own tfvars.

The module structure deliberately isolates cloud-specific resources. If we need to support another cloud later, we write new modules with the same output interface. The K8s layer (Helm releases, ArgoCD config) stays untouched.

---

## Deployment — ArgoCD GitOps

Deployment follows GitOps: developers merge code → CI builds the image, pushes to ECR, updates the image tag in the manifest repo → ArgoCD detects the change in Git and auto-syncs to the EKS cluster. Canary deployments go through Istio traffic splitting. If health checks fail, ArgoCD rolls back automatically.

Nobody runs `kubectl apply` manually. The Git repo is the single source of truth for cluster state. ArgoCD, Helm, and Kustomize are all K8s-native — they work on any cluster without modification.

---

## Portability Summary

| Layer | What we use | Portable? | What changes if we move clouds |
|-------|-------------|-----------|-------------------------------|
| DNS | Route 53 | No | ExternalDNS + another DNS provider |
| WAF | AWS WAF | No | ModSecurity / Coraza on K8s |
| CDN | CloudFront + S3 | No | Whatever CDN + object storage the target cloud has |
| API Gateway | Kong on K8s | **Yes** | Nothing |
| Service Mesh | Istio | **Yes** | Nothing |
| Microservices | Helm + K8s | **Yes** | Nothing |
| Event Streaming | MSK (Kafka) | No | Strimzi Operator on K8s |
| Cache | ElastiCache Redis | No | Redis Operator on K8s |
| Database | Aurora PostgreSQL | No | CloudNativePG or Crossplane |
| Monitoring | Prometheus + Grafana | **Yes** | Nothing |
| IaC | Terraform | Partial | Add modules for the new cloud |
| Deployment | ArgoCD | **Yes** | Nothing |

The application layer (Kong, Istio, microservices, monitoring, ArgoCD) is already fully portable. The infrastructure layer (DNS, CDN, database, cache, Kafka) uses AWS managed services for operational efficiency, but each one has a clear migration path to a K8s-native alternative when we need it.
