## Section 0.3: Non-Functional Requirements (NFR) Formulation

### 1. User Scale and Traffic Load Targets

- **Concurrent Users:** Target max of 10-20 concurrent internal users (Tailors, inventory managers, and shop staff).
- **Monthly Active Users (MAU):** Expected ~500 to 1,000 monthly active sessions initially.
- **DevOps Verdict:** Since the traffic load is strictly internal and low, a single container deployment is highly sufficient, requiring no complex auto-scaling groups at this MVP stage.

### 2. Availability and Downtime Tolerance (SLA)

- **Downtime Tolerance:** Allowed 1-2 minutes of expected downtime during production deployments or updates.
- **Architecture Decision:** Given that this is an MVP phase, a Single AWS EC2 instance deployment is designated as standard. High Availability (Multi-AZ) or zero-downtime rolling updates are out of scope to prioritize strict cost savings.

### 3. Data Retention and Durability (Data Protection and Backups)

- **Data Criticality:** Extremely High. Financial records, customer measurements, and garment inventory logs cannot be lost under any circumstance.
- **Backup Policy:**
  - **Automated Daily Snapshots:** AWS RDS (or Neon.tech) automated daily snapshots will be enabled with a strict 7-day retention period.
  - **Manual Milestone Backups:** A manual DB snapshot will be triggered before any major database schema migration or production deployment.
  - **Backup Cost Optimization:** Utilizing AWS RDS Free Tier which includes up to 20GB of backup storage allocation for free; Expected Backup Cost = $0.00.
- **Disaster Recovery (DR):**
  - **Recovery Point Objective (RPO):** Maximum 24 hours (data will be restored up to the last automated daily snapshot).
  - **Recovery Time Objective (RTO):** Under 30 minutes.
  - **DR Strategy:** The application layer inside EC2 is completely stateless. In the event of a total EC2 server crash, data durability is safely isolated inside the managed database. A new EC2 instance can be instantly reprovisioned via the GitHub Actions CI/CD pipeline, and connected back to the database with zero production data loss.

### 4. CI/CD Pipeline Security and Automation Triggers

- **Automation Trigger:** The pipeline must trigger automatically ONLY upon a successful git push to the main branch.
- **Security Guardrails:**
  - Branch protection must be enabled on GitHub to prevent direct pushes to main without successful local compilation.
  - All cloud secrets (AWS Keys, Database URLs) must be stored inside GitHub Actions Secrets; no raw keys are allowed inside the code repository.

### 5. Identity and Access Management (IAM) and Cloud Governance

- **Principle of Least Privilege (PoLP):** Root AWS accounts will never be used for daily operations or CI/CD pipelines.
- **Developer Access Control:** Developers will not have direct SSH access to the production EC2 instance. Infrastructure configuration and deployment management are strictly restricted to the DevOps role.
- **IAM Registry:**

| Identity / Role     | Type            | Who / What uses it?                 | Access Level (Permission)                                                                                                           |
| :------------------ | :-------------- | :---------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------- |
| AWS Root Account    | Root Account    | Emergency Only                      | Full Access (Credentials changed, 2FA enabled, and secured safely).                                                                 |
| DevOps Admin        | IAM User        | Poorna Bhagya                       | Full Administrator Access (For complete cloud infrastructure setup).                                                                |
| Backend Dev         | IAM User        | Backend Developer                   | Database Access, S3 Access, and CloudWatch Logs (No deletion permissions for core resources).                                       |
| Frontend Dev        | IAM User        | Frontend Developer                  | Limited S3/Hosting Access (Read-Only preferred).                                                                                    |
| Management / Client | IAM User        | Project Manager / Client / Auditors | ReadOnlyAccess + Billing (Strictly for viewing infrastructure status and billing invoices. No configuration modifications allowed). |
| GitHub-CI-CD        | IAM Role / User | GitHub Actions Pipeline             | Strictly restricted to Amazon ECR (Push) and EC2 (Deploy) via secure secrets.                                                       |
| EC2-App-Role        | IAM Role        | EC2 Server Instance                 | Restricted to AWS S3 Read/Write (For user uploads and media assets).                                                                |

### 6. Network Architecture and Firewall Security (VPC and Security Groups)

- **VPC Strategy:** To maintain strict cost optimization ($0.00 infrastructure overhead), the deployment will utilize the AWS Default VPC and its pre-configured Public Subnets. No custom private subnets or expensive NAT Gateways will be provisioned.
- **Port Restriction:** The AWS EC2 Security Group will lock down all incoming traffic except:
  - **Port 80 (HTTP) & Port 443 (HTTPS):** Open to the public for global web application access.
  - **Port 22 (SSH):** Restricted strictly to the DevOps administrator's specific IP address (IP White-listing) to prevent external brute-force attacks.
- **Database and Container Isolation:** * Even though the managed database (RDS/Neon) resides within a public subnet, its Security Group rules will strictly block all public ingress traffic, whitelisting access only from the production EC2 instance IP.
  - The application container running on port 3001 will remain isolated inside an internal Docker bridge network, accessible strictly via the Nginx Reverse Proxy.

### 7. Data-in-Transit Encryption (SSL/TLS)

- **Compliance Requirement:** 100% of external web traffic must be encrypted.
- **Implementation:** Standard HTTP traffic (Port 80) will automatically redirect to HTTPS (Port 443) using modern TLS 1.3 encryption protocols. SSL certificates will be managed via Certbot (Let's Encrypt) with automated 90-day renewal cron jobs.

### 8. System Performance and Logging Metrics

- **Response Time Target:** App server response time (TTFB) should remain under 500ms for core inventory tables under normal load.
- **Log Retention:** Container engine stdout logs and Nginx access logs will be rotated weekly to prevent the EC2 disk (gp3 volume) from filling up unexpectedly.
