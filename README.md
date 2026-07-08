# Tailor Inventory System — Production-Grade Cloud Infrastructure & DevOps Blueprint

This repository houses the complete **Enterprise DevOps lifecycle, Non-Functional Requirements (NFR) formulation, and Automated Cloud Infrastructure blueprints** for the Tailor Inventory System. 

The architecture is explicitly designed to balance maximum data security, strict isolation patterns, and zero-cost cloud infrastructure overhead ($0.00 AWS usage targets) under constraints appropriate for a high-performance Minimum Viable Product (MVP).

---

##  System Architecture Diagram

Below is the concrete logical and network mapping for the production environment on AWS. It showcases the boundary separation between public web access, isolated internal docker runtimes, and secured backend storage subnets.

![AWS Cloud Architecture Diagram](./docs/project1_cloud_archi_diagram.drawio.png)

---

##  Deep Architectural Breakdown & Mechanics

### 1. Zero-Cost Compute & Smart Compilation Model
*   **The Hardware Constraint:** The production host is deployed on an **AWS EC2 Free-Tier instance (t2.micro/t3.micro)** with only 1GB of RAM.
*   **The Problem:** Modern Next.js optimization and Payload CMS compilation pipelines demand substantial temporary resource usage during the build phase (up to 8GB of RAM). Triggering `pnpm build` natively on the host results in immediate Out-of-Memory (OOM) kernel crashes.
*   **The DevOps Solution:** The building process is entirely externalized to **GitHub Actions Hosted Runners**. The runner compiles the strict TypeScript environment down into optimized static chunks, packages it inside a minimal security-hardened `node:22.17.0-alpine` base layer, and pushes the production image to **Amazon ECR**. The EC2 host merely orchestrates a lightweight container retrieval (`docker pull`), optimizing host-level memory utilization to absolute minimal idling configurations.

### 2. High-Durability Managed Database Separation
*   **Persistence Layer:** Relational schemas are managed via **PostgreSQL (v16)** using Drizzle ORM mappings powered by `@payloadcms/db-postgres`.
*   **Isolation and Whitelisting:** Although database nodes exist structurally within the default public routing subnets to eliminate expensive custom NAT Gateway expenses, public internet exposure is completely locked down. The database Security Group uses a strict single ingress restriction—**whitelisting inbound traffic on Port 5432 exclusively from the Production EC2 Instance Public IP address**.
*   **Data Protection & Backups:** To meet strict non-functional constraints regarding transactional financial entries and measurement logs, automated daily point-in-time snapshots are maintained with a strict 7-day retention period alongside mandatory manual pre-migration milestone snapshots.

### 3. Stateless Application Layer & Object Decoupling
*   **Ephemeral Compute Design:** The application tier running inside the EC2 environment is entirely stateless. 
*   **Media Offloading:** To prevent the local Elastic Block Store (gp3 SSD volume) from experiencing degradation or sector exhaustion due to media attachments (garment samples, customer receipts, styling updates), all raw upload assets handled by the high-performance `sharp` processing layer are natively decoupled and offloaded securely to an **Amazon S3 Bucket** utilizing restricted IAM instance profiles (`EC2-App-Role`).
*   **Disaster Recovery (DR):** In the event of a total server failure, a new instance can be reprovisioned instantly via the automated GitHub Actions delivery chain and re-attached to the immutable persistent storage layer with **zero production data loss** and an RTO of under 30 minutes.

### 4. Dual-Layer Firewall Hardening & SSL Termination
*   **Public Routing Subnets:** Inbound connections on web ports **80 (HTTP)** and **443 (HTTPS)** are open globally (`0.0.0.0/0`) to route consumer traffic cleanly.
*   **Strict SSH Whitelisting:** Port **22 (SSH)** is tied to a secure administrative IP Whitelist rule. Host control access is completely hidden from public visibility, blocking external brute-force scanning vectors entirely.
*   **Reverse Proxy Mechanics:** Public edge traffic hits an explicit **Nginx Reverse Proxy Instance**. Nginx handles unencrypted Port 80 queries, triggers immediate standard HTTP-to-HTTPS permanent redirections, terminates modern TLS 1.3 encryption paths locally via Let's Encrypt certificates, and proxies downstream calls into the internal Docker bridge network on port **3001**.

---

##  Core Infrastructure Specifications

### Technology Matrix & Software Inventory
| Layer | Core Selection | Target Version | Operational Notes |
| :--- | :--- | :--- | :--- |
| **Language Runtime** | TypeScript / Node.js | v5.7.3 / v22.x | Explicit typing configurations enforced. |
| **App Engine** | Next.js (App Router) | v16.2.6 | Uniform monorepo design enclosing admin/public code. |
| **Headless CMS** | Payload CMS | v3.85.1 | Embedded engine via `@payloadcms/next`. |
| **Database Engine** | PostgreSQL | v16 | Drizzle abstraction layers managing data models. |
| **Host OS Target** | Ubuntu Server | 24.04 LTS | Standard stable long-term support architecture. |
| **Edge Router** | Nginx Proxy | Latest Stable | Manages edge redirection and local cert streams. |

### Network Layout Mapping
*   **Port 80 / 443:** Open Publicly (`0.0.0.0/0`) -> Routed via Nginx Proxy.
*   **Port 22:** Restricted strictly to Administrator IP -> Secure SSH Management.
*   **Port 3001:** Bound internally within the Docker bridge network -> Application Layer.
*   **Port 5432:** Bound strictly to the EC2 Source Subnet Host IP -> PostgreSQL Database Engine.

---

##  Operational Runbook Status

Detailed installation execution, memory configuration strategies (such as local host **Swap Space Optimization** to handle memory spikes gracefully), and step-by-step infrastructure bootstrap flows can be reviewed inside the specialized deployment files:

1.  **System Non-Functional Requirements:** Located at [`/docs/NFR.md`](./docs/NFR.md)
2.  **Full Technology Stack Runbook Specification:** Located at [`/docs/Technical_Report.md`](./docs/Technical_Report.md)

---
Developed and maintained by **Poorna Bhagya — DevOps & Cloud Engineering** (2026).