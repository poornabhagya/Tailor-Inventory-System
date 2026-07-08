# Technical Stack Specification and DevOps Runbook

## Metadata

- **Project Name:** Tailor Inventory System
- **Version:** 1.0.0
- **Author:** Bhagya Ekanayake / Poorna Bhagya - DevOps and Cloud Engineering
- **Last Updated:** 2026-07-04
- **Status:** Approved

---

## 1. Purpose

This document defines the complete technology stack, environments, configurations, and operational requirements for the Tailor Inventory System. It serves as the definitive source of truth for CI/CD pipeline design, cloud infrastructure provisioning, and developer onboarding.

---

## 2. System Overview

A tailoring rental and inventory management system built as a unified Next.js web application with an embedded Payload CMS backend, utilizing PostgreSQL for persistence. The architecture follows a monolith structure combining both the user-facing interface and administrative capabilities into a single package deployed via containerized environments.

---

## 3. Core Technology Stack

| Layer                   | Technology        | Version              | Notes                                                    |
| :---------------------- | :---------------- | :------------------- | :------------------------------------------------------- |
| **Language**            | TypeScript        | 5.7.3                | Strict mode enabled                                      |
| **Framework**           | Next.js           | 16.2.6               | App Router architecture                                  |
| **CMS / Backend**       | Payload CMS       | 3.85.1               | Headless engine integrated via `@payloadcms/next`        |
| **UI Library**          | React / React DOM | 19.2.6               | Component-driven presentation layer                      |
| **Database**            | PostgreSQL        | 16                   | Managed via `@payloadcms/db-postgres` (Drizzle ORM)      |
| **Image Processing**    | sharp             | 0.34.2               | High-performance production media optimization           |
| **API Layer**           | GraphQL / REST    | ^16.8.1              | Auto-generated endpoints natively by Payload             |
| **Rich Text Editor**    | Lexical           | 3.85.1               | WYSIWYG editor via `@payloadcms/richtext-lexical`        |
| **Runtime Environment** | Node.js           | ^18.20.2 or >=20.9.0 | Core engine target for Docker base image                 |
| **Package Manager**     | pnpm              | ^9 or ^10            | Strict lockfile dependency management agent              |
| **E2E Testing Layer**   | Playwright        | 1.58.2               | Automated browser workflow simulations in CI pipeline    |
| **Unit Testing Layer**  | Vitest            | 4.0.18               | High-speed unit and integration test execution framework |

---

## 4. Dependency Inventory

### Production Dependencies

| Package Name                   | Version | Purpose                                                                       |
| :----------------------------- | :------ | :---------------------------------------------------------------------------- |
| `next`                         | 16.2.6  | Core web framework providing server-side rendering and routing.               |
| `react` & `react-dom`          | 19.2.6  | Core UI library for building the component-driven interface.                  |
| `payload`                      | 3.85.1  | The enterprise headless CMS engine providing the admin platform.              |
| `@payloadcms/next`             | 3.85.1  | Native integration plugin to embed Payload directly into Next.js.             |
| `@payloadcms/ui`               | 3.85.1  | Renders the built-in React UI components for the Admin Dashboard.             |
| `@payloadcms/db-postgres`      | 3.85.1  | PostgreSQL database adapter using Drizzle ORM to map schemas.                 |
| `@payloadcms/richtext-lexical` | 3.85.1  | Rich text editor plugin powered by Meta's Lexical engine.                     |
| `sharp`                        | 0.34.2  | High-performance image processing library for media asset optimization.       |
| `graphql`                      | ^16.8.1 | Enables GraphQL API layer querying support natively within Payload.           |
| `cross-env`                    | ^7.0.3  | Sets runtime environment variables safely across different OS platforms.      |
| `dotenv`                       | 16.4.7  | Loads sensitive configurations from environment files into execution context. |

### Development Dependencies

| Package Name                    | Version | Purpose                                                                       |
| :------------------------------ | :------ | :---------------------------------------------------------------------------- |
| `typescript`                    | 5.7.3   | Adds static typing and compiles TypeScript code down to JavaScript.           |
| `@types/node`, `@types/react`   | Various | Provides TypeScript definitions for editor auto-completion.                   |
| `vitest`                        | 4.0.18  | Fast unit/integration test runner to check logic in local and CI pipeline.    |
| `@testing-library/react`        | 16.3.0  | Provides utilities to simulate user events and test React components.         |
| `@playwright/test`              | 1.58.2  | Automated browser testing framework for end-to-end workflow checks.           |
| `jsdom`                         | 28.0.0  | Pure JavaScript implementation of web standards for headless unit testing.    |
| `eslint` & `eslint-config-next` | ^9.16.0 | Code quality tool scanning for syntax, bugs, and anti-patterns.               |
| `prettier`                      | ^3.4.2  | Code formatter ensuring uniform coding style guidelines across the team.      |
| `tsx`                           | 4.21.0  | CLI tool to execute TypeScript files directly without manual pre-compilation. |
| `vite-tsconfig-paths`           | 6.0.5   | Resolves path aliases dynamically during unit testing phases.                 |

---

## 5. Environments

| Environment                | Git Branch             | Target URL / Hosting                   | Status / Infrastructure Notes                                            |
| :------------------------- | :--------------------- | :------------------------------------- | :----------------------------------------------------------------------- |
| **Local Development**      | Any `feature/*` branch | `http://localhost:3001`                | Active: Runs locally using `pnpm dev`.                                   |
| **Continuous Integration** | All branches via PR    | GitHub Runners (No public URL)         | Planned: Automates execution of Vitest and Playwright on Pull Requests.  |
| **Staging**                | `develop`              | `https://staging.tailor-inventory.com` | Planned: Containerized AWS EC2 instance deployment for internal testing. |
| **Production**             | `main`                 | `https://tailor-inventory.com`         | Planned: Live production AWS cloud cluster environment for end users.    |

---

## 6. Network and Port Layout

| Service / Layer         | Internal Port | External / Host Port | Environment          | Purpose                                                              |
| :---------------------- | :------------ | :------------------- | :------------------- | :------------------------------------------------------------------- |
| **Next.js App (Local)** | 3001          | 3001                 | Local                | Local development and isolated feature testing.                      |
| **Next.js App (Cloud)** | 3000          | Hidden behind proxy  | Staging / Production | Runs inside isolated Docker bridge networks on cloud instances.      |
| **Postgres Database**   | 5432          | 5432                 | All Envs             | Target port mapped globally for database connection pooling.         |
| **Nginx Proxy (HTTP)**  | 80            | 80                   | Staging / Production | Handles incoming unencrypted web traffic to trigger HTTPS redirects. |
| **Nginx Proxy (HTTPS)** | 443           | 443                  | Staging / Production | Terminates public TLS traffic via verified SSL certificates.         |

---

## 7. Environment Variables

| Variable                 | Required | Example / Format                           | Secret? | Notes                                                                   |
| :----------------------- | :------- | :----------------------------------------- | :------ | :---------------------------------------------------------------------- |
| `DATABASE_URL`           | Yes      | `postgresql://user:pass@localhost:5432/db` | Yes     | Connection string used by Drizzle ORM to access PostgreSQL.             |
| `PAYLOAD_SECRET`         | Yes      | `a1b2c3d4e5f6g7h8i9j0...`                  | Yes     | Cryptographic hash key utilized to secure admin panel cookie sessions.  |
| `NEXT_PUBLIC_SERVER_URL` | Yes      | `https://tailor-inventory.com`             | No      | Base application entry URL used for internal and external API fetching. |
| `NODE_ENV`               | Yes      | `development` \| `production`              | No      | Toggles performance optimizations inside the framework build.           |
| `S3_BUCKET`              | No       | `tailor-inventory-assets`                  | No      | Defines target global AWS bucket name for binary object uploads.        |

---

## 8. Build and Run Commands

| Script               | Actual Command                   | Target Env           | Purpose                                                             |
| :------------------- | :------------------------------- | :------------------- | :------------------------------------------------------------------ |
| `dev`                | `next dev -p 3001`               | Local                | Starts local environment with automatic hot-reloading.              |
| `devsafer`           | `rm -rf .next && next dev...`    | Local                | Troubleshooting command to flush corrupt local cache layers.        |
| `payload`            | `payload`                        | Local / Dynamic      | Base Payload CLI tool used to execute core schema tasks.            |
| `test`               | `pnpm test:int && pnpm test:e2e` | Local / CI           | Automated sequential script aggregating integration and E2E checks. |
| `build`              | `next build`                     | CI / Production      | Production compiler configured via custom RAM memory flags.         |
| `start`              | `next start`                     | Staging / Production | Launches compiled production builds on target cloud infrastructure. |
| `generate:importmap` | `payload generate:importmap`     | Local / CI           | Resolves structural ESM modules and mappings ahead of compilation.  |
| `generate:types`     | `payload generate:types`         | Local                | Compiles TypeScript declarations from database collections.         |
| `lint`               | `eslint .`                       | Local / CI           | Static code analysis scanning for structural patterns or defects.   |
| `test:int`           | `vitest run ...`                 | Local / CI           | Executes isolated unit and integration suites using Vitest.         |
| `test:e2e`           | `playwright test ...`            | Local / CI           | Triggers continuous browser automation tests via Playwright.        |

---

## 9. Database Schema and Storage Strategy

| Collection          | Schema Fields                                                            | Actual Relations                               | DevOps Storage Strategy                                                                        |
| :------------------ | :----------------------------------------------------------------------- | :--------------------------------------------- | :--------------------------------------------------------------------------------------------- |
| **users**           | id, email, password, loginAttempts, lockUntil                            | Internal Authentication                        | High Security: Encrypted records persisted inside dedicated database engines.                  |
| **media**           | id, alt, url, filename, mimeType, filesize, width, height                | Global Reference                               | Cloud Object Storage: Binary payloads decoupled to AWS S3 to minimize host disk load.          |
| **customers**       | id, nic, name, phone                                                     | Referenced by rentals                          | High Durability: Audited tabular customer tracking. Automated backup rules applied.            |
| **inventory-items** | id, name, sku, category, size, color, status, replacementCost            | Referenced by rentals                          | High Velocity Transactional: Tracks operational asset lifecycles. Daily state snapshots.       |
| **rentals**         | id, customer, items, rentalStartDate, rentalEndDate, status, depositPaid | Many-to-One (customers) / Many-to-Many (items) | Financial Ledger: Highly critical audit trail. Real-time point-in-time state recovery enabled. |

---

## 10. Third-Party and Cloud Services Integrations

| Service Name         | Purpose                                                     | Pricing Plan                           | Responsibility                                    |
| :------------------- | :---------------------------------------------------------- | :------------------------------------- | :------------------------------------------------ |
| **AWS EC2**          | Hosts application runtime via Docker containers.            | Free Tier (`t2.micro` / `t3.micro`)    | DevOps Engineer (Provisioning & hardening)        |
| **AWS ECR**          | Secure cloud registry hosting versioned Docker images.      | Free Tier (5GB storage per month)      | DevOps Engineer (CI/CD push/pull automation)      |
| **AWS S3**           | Persists runtime application media and garment assets.      | Free Tier (Up to 5GB storage)          | DevOps Engineer (Bucket policies & authorization) |
| **AWS RDS / Neon**   | Managed transactional database infrastructure engine.       | Free Tier (Neon) / `db.t4g.micro`      | Lead Developer & DevOps                           |
| **Resend / AWS SES** | Dispenses transactional customer order email notifications. | Free Tier (3,000 emails per month)     | Software Engineer (API integration & config)      |
| **GitHub Actions**   | Orchestrates automated build and test runner sequences.     | Free (Public) / 2000 mins/mo (Private) | DevOps Engineer (Workflow architecture script)    |

---

## 11. Known Issues and Technical Debt

| Issue                                     | Impact                                                                                                             | Status | Next Action                                                                                       |
| :---------------------------------------- | :----------------------------------------------------------------------------------------------------------------- | :----- | :------------------------------------------------------------------------------------------------ |
| **ESLint Crash** (`ERR_MODULE_NOT_FOUND`) | **High:** Pipeline fails at the linting phase, entirely breaking the automated delivery chain.                     | Open   | Install `@eslint/eslintrc` explicitly into `devDependencies` configuration.                       |
| **Memory Heavy Compiler Flag**            | **Medium:** Compiling requests substantial resources. Small environments encounter Out-of-Memory (OOM) crashes.    | Open   | Delegate the build script execution entirely onto specialized GitHub Hosted Runners.              |
| **Missing Email Adapter Warning**         | **Low:** Logs output notifications. Functional communications fail to route externally in production environments. | Open   | Provision integration properties using verified Resend or AWS SES credentials.                    |
| **Hardcoded Ports**                       | **Low:** Harmless locally, but prevents standardized port binding parameterization on cloud environments.          | Open   | Externalize port assignments dynamically using runtime variable mappings inside the Docker setup. |

---

## 12. Deployment Requirements

### A. Minimum System Specifications

- **Cloud Provider:** AWS (Amazon Web Services)
- **Host Operating System:** Ubuntu Server 24.04 LTS (64-bit)
- **Target Production EC2 Type:** `t3.micro` or `t2.micro` (AWS Free Tier Eligible - 1GB RAM)

> **DevOps Deployment Strategy (Cost Optimization):**
> Next.js and Payload compilation processes require substantial resources, demanding up to 8GB RAM. To safely utilize the AWS Free Tier architecture (1GB RAM) without triggering server out-of-memory constraints, the production bundle phase (`pnpm build`) is externalized to external GitHub Actions Hosted Runners. The resulting production Docker image is pushed directly to Amazon ECR and later pulled down into the target EC2 node as a pre-built static container package, achieving absolute host-level resource optimization.

### B. Containerization and Base Environment

- **Docker Base Image:** `node:22.17.0-alpine` (Lightweight, security-hardened minimal OS layer running Node.js v22)
- **Application Runtime Port:** Parameterized to run securely via proxy configurations mapped to port `3001`.

### C. Required Pre-installed Host Tools

- **Docker Engine & Docker Compose:** Container abstraction layer running runtime app deployments.
- **AWS CLI (v2):** Embedded execution tool providing secure instance profile credential lookups with Amazon ECR.
- **Nginx:** Operates as a local high-performance reverse proxy routing inbound public connections into port `3001`.
- **Certbot (Let's Encrypt):** Background utility managing automatic renewals of web-facing SSL certificates.

### D. Database Migration Strategy

- **Mechanism:** Native Payload schema state synchronization.
- **Execution:** Database updates trigger systematically during startup phases within production containers by running `pnpm payload migrations:run` immediately prior to launch events.

---

## 13. Revision History

| Version    | Date       | Author        | Description of Changes                                                                                                                                                                                                     |
| :--------- | :--------- | :------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **v1.0.0** | 2026-07-04 | Poorna Bhagya | Initial release and full compilation of the DevOps Master Runbook (Sections 1-12). Validated with local code analysis (`payload-types.ts`), repository dependency checks, and successful production build test executions. |
