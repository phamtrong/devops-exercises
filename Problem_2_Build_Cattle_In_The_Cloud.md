Architecture Picture
https://github.com/phamtrong/devops-exercises/blob/main/Problem2.svg

# Frontend Flow Architecture with AWS WAF, CloudFront, and S3

## Overview

This document describes the architecture and data flow for serving a frontend web or mobile application on AWS, with security and performance optimizations using **AWS WAF**, **CloudFront**, and **S3**.

---

## Architecture Components

1. **Client (Browser / Mobile App)**  
   The end user accesses the frontend application through a web browser or mobile app.

2. **AWS Route 53**  
   Acts as the DNS service, directing user requests to the appropriate AWS resources based on latency or geo-location routing policies. This ensures users connect to the closest and fastest edge location.

3. **AWS WAF (Web Application Firewall)**  
   Provides protection at the application layer by filtering out malicious HTTP/S requests before they reach CloudFront or backend origins. It helps block common attacks such as SQL injection, cross-site scripting (XSS), and brute force attacks.

4. **AWS CloudFront**  
   A global Content Delivery Network (CDN) that caches and delivers static assets such as HTML, JavaScript, CSS, and images from edge locations close to users. This improves load times and reduces the load on backend origin servers.

5. **Amazon S3 Bucket (Static Files)**  
   Stores the static frontend assets, including compiled JavaScript bundles, HTML files, CSS stylesheets, and images. S3 serves as the origin for CloudFront caching.

---

## Data Flow

1. The client initiates a request to load the frontend application.

2. The request resolves via **AWS Route 53**, which routes the user to the appropriate AWS infrastructure based on geographic location or latency.

3. The request is first processed by **AWS WAF**, which inspects the traffic for malicious activity and blocks any suspicious requests, protecting the application from web attacks.

4. Clean requests are forwarded to **AWS CloudFront**, which attempts to serve the requested content from the nearest edge cache.

5. If the content is cached, CloudFront returns it directly to the client, minimizing latency and reducing backend load.

6. If the content is not cached (cache miss), CloudFront fetches the static files from the **Amazon S3 bucket** origin.

7. Finally, the content is delivered to the client, completing the flow.

---

## Benefits

- **Security:** AWS WAF provides a robust first line of defense against common web vulnerabilities and attacks.  
- **Performance:** CloudFront reduces latency by caching static assets closer to users worldwide.  
- **Scalability:** S3 can handle virtually unlimited requests for static assets without requiring server management.  
- **Cost-effectiveness:** Using managed services and caching reduces operational costs and infrastructure overhead.

---
# Frontend Flow Architecture with AWS WAF, CloudFront, and S3

## Overview

This document describes the architecture and data flow for serving a frontend web or mobile application on AWS, with security and performance optimizations using **AWS WAF**, **CloudFront**, and **S3**.

---

## Architecture Components

1. **Client (Browser / Mobile App)**  
   The end user accesses the frontend application through a web browser or mobile app.

2. **AWS Route 53**  
   Acts as the DNS service, directing user requests to the appropriate AWS resources based on latency or geo-location routing policies. This ensures users connect to the closest and fastest edge location.

3. **AWS WAF (Web Application Firewall)**  
   Provides protection at the application layer by filtering out malicious HTTP/S requests before they reach CloudFront or backend origins. It helps block common attacks such as SQL injection, cross-site scripting (XSS), and brute force attacks.

4. **AWS CloudFront**  
   A global Content Delivery Network (CDN) that caches and delivers static assets such as HTML, JavaScript, CSS, and images from edge locations close to users. This improves load times and reduces the load on backend origin servers.

5. **Amazon S3 Bucket (Static Files)**  
   Stores the static frontend assets, including compiled JavaScript bundles, HTML files, CSS stylesheets, and images. S3 serves as the origin for CloudFront caching.

---

## Data Flow

1. The client initiates a request to load the frontend application.

2. The request resolves via **AWS Route 53**, which routes the user to the appropriate AWS infrastructure based on geographic location or latency.

3. The request is first processed by **AWS WAF**, which inspects the traffic for malicious activity and blocks any suspicious requests, protecting the application from web attacks.

4. Clean requests are forwarded to **AWS CloudFront**, which attempts to serve the requested content from the nearest edge cache.

5. If the content is cached, CloudFront returns it directly to the client, minimizing latency and reducing backend load.

6. If the content is not cached (cache miss), CloudFront fetches the static files from the **Amazon S3 bucket** origin.

7. Finally, the content is delivered to the client, completing the flow.

---

## Benefits

- **Security:** AWS WAF provides a robust first line of defense against common web vulnerabilities and attacks.  
- **Performance:** CloudFront reduces latency by caching static assets closer to users worldwide.  
- **Scalability:** S3 can handle virtually unlimited requests for static assets without requiring server management.  
- **Cost-effectiveness:** Using managed services and caching reduces operational costs and infrastructure overhead.

---
# Backend API Flow Architecture with API Gateway, EKS Node Group, and Auto Scaling

 Backend API flow for a trading system deployed on AWS, highlighting the use of **API Gateway** as the entry point, **EKS Node Groups** running microservices, and the autoscaling strategies for maintaining resilience, performance, and cost efficiency.

---

## Architecture Components

1. **AWS API Gateway**  
   - Serves as the single entry point for all API requests (REST and WebSocket).  
   - Provides features such as request throttling, authentication integration, and TLS termination.  
   - Integrated with **AWS WAF** for protecting against malicious traffic and common web attacks.  
   - Supports WebSocket connections for real-time market data streaming.

2. **EKS Cluster with Node Groups (EC2 Instances)**  
   - Hosts containerized microservices such as Auth Service, Trading Service (matching engine), Wallet Service, and Market Data Service.  
   - Uses **Node Groups** consisting of EC2 instances, providing fine control over instance types and sizes.  
   - Node Groups are managed via **Auto Scaling Groups (ASG)** to dynamically adjust the number of EC2 instances based on workload demands.

3. **Supporting Services**  
   - **Amazon MSK (Kafka):** Event streaming and messaging between microservices.  
   - **ElastiCache Redis:** In-memory caching and order book data store for low-latency operations.  
   - **Amazon Aurora PostgreSQL:** Persistent storage with strong consistency, deployed in Multi-AZ mode for high availability.

---

## Data Flow

1. Client applications send API requests to **API Gateway**, including REST calls for placing orders and WebSocket connections for live market data.

2. **API Gateway**, protected by **AWS WAF**, validates and forwards requests to the appropriate microservice pods running in the EKS cluster.

3. Microservices running on **EKS Node Groups** handle business logic:  
   - The **Trading Service** processes orders, interacts with the order book in Redis, and publishes trade events to Kafka.  
   - The **Wallet Service** updates user balances and ensures transaction consistency in Aurora.  
   - The **Market Data Service** streams real-time data back to clients via WebSocket.

4. Kafka acts as the event bus ensuring decoupling and asynchronous communication between services.

---

## Auto Scaling Strategies

### 1. **EKS Node Group Auto Scaling**

- The EC2 instances in node groups are part of **Auto Scaling Groups (ASG)**.  
- ASGs monitor metrics such as CPU utilization, memory usage, or custom CloudWatch metrics to automatically scale the number of nodes up or down.  
- Scaling policy is configured to maintain enough capacity to handle pods scheduled by Kubernetes.

### 2. **Kubernetes Pod Auto Scaling**

- Within the EKS cluster, **Horizontal Pod Autoscaler (HPA)** scales microservice pods based on resource usage or custom metrics (e.g., request latency, queue length).  
- When pod demand increases, HPA creates more pods; when demand falls, it scales pods down to reduce costs.

### 3. **Scaling Kafka, Redis, and Aurora**

- **Kafka (MSK):**  
  - Kafka brokers can be added or removed manually or via scripts to handle increased event throughput.  
  - Partition count may be increased to parallelize workload.

- **Redis (ElastiCache):**  
  - Redis clusters support automatic failover and read replicas for scaling read-heavy workloads.  
  - Vertical scaling (instance size) and horizontal scaling (sharding) are used depending on demand.

- **Aurora PostgreSQL:**  
  - Supports read replicas to offload read queries from the primary instance.  
  - Multi-AZ deployments provide failover for high availability.  
  - Storage auto-scales transparently as data grows.

---

## Benefits of this Approach

- **High Availability:** Multi-AZ deployments and autoscaling prevent downtime during traffic spikes or failures.  
- **Scalability:** Both infrastructure and application layers scale independently and dynamically to meet demand.  
- **Cost Efficiency:** Resources scale down during low traffic to minimize cost, scaling up only when needed.  
- **Resilience:** Decoupled architecture with Kafka event bus minimizes cascading failures.

---
