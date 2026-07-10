PHASE 3: AWS Production Provisioning & Configuration
Technical Implementation Runbook & Troubleshooting Manual
Project Name: Tailor Inventory System
Lead DevOps Engineer: Poorna Bhagya
Status: 100% Completed

Section 1: Local Environment Preparation & SSH Security
Step 1.1: Local Key File Path Selection
Before connecting from the local Windows machine to the AWS EC2 server, navigate to the folder where the private key file (.pem file) is located using PowerShell.

PowerShell
cd ~\Downloads
Troubleshooting: WARNING: UNPROTECTED PRIVATE KEY FILE!
Issue: The SSH connection is rejected due to broad permissions when running the command: ssh -i tailor-inventory-key.pem ubuntu@54.254.235.155.

Cause: Unlike Linux, Windows permits other local users and groups (such as CodexSandboxUsers) to read files inside the Downloads folder by default. The OpenSSH system enforces strict security policies and rejects authentication using private keys that are accessible to other users.

Solution (PoLP Principle): Disable inheritance on the file via Windows PowerShell and grant exclusive full control permissions only to the current user (Porna Bhagya).

Remediation Commands (Line-by-Line):

Disable permission inheritance on the file:

PowerShell
icacls .\tailor-inventory-key.pem /inheritance:r
Grant full control (Full Access - F) exclusively to the current user ($env:USERNAME):

PowerShell
icacls .\tailor-inventory-key.pem /grant:r "$($env:USERNAME):(F)"
Verify the Access Control List (ACL) configuration:

PowerShell
icacls .\tailor-inventory-key.pem
(The output must show only Poorna Bhagya:(F)).

Step 1.2: Establish Secure SSH Connection
After restricting the key file permissions, execute the following command to securely connect to the AWS EC2 instance:

Bash
ssh -i tailor-inventory-key.pem ubuntu@54.254.235.155
Section 2: Server Core OS Updates & Kernel Optimization
Step 2.1: Base Operating System Upgrade
Once authenticated to the server, update the local package index and upgrade the system to apply the latest security patches:

Bash
sudo apt update && sudo apt upgrade -y
(Note: If a pink or blue configuration prompt appears during installation, retain the default options and press Enter to proceed).

Step 2.2: System Reboot
Reboot the instance to apply kernel updates and initialize a clean server environment:

Bash
sudo reboot
(The reboot command terminates the active SSH session, returning you to the local command prompt. Wait approximately 30 seconds before executing the SSH connection command from Step 1.2 again).

Section 3: Storage & Swap Space (Virtual RAM) Optimization
Troubleshooting: Unzip Write Error (Disk Full)
Issue: The installation process fails with a write error (disk full?) when extracting installation assets like the AWS CLI.

Initial Assessment: Checking the disk space allocation with df -h shows that the root directory (/dev/root) is utilizing 97% of its available capacity.

Cause: Under the AWS Free Tier tier, standard EC2 instances are provisioned with an 8GB root volume. Allocating a 4GB swap space compromised the remaining space required by the operating system and dependencies.

Strategic Resolution: Clean package management cache files and clear old system logs using sudo apt-get clean and sudo journalctl --vacuum-time=1d. Reduce the swap file size from 4GB to 2GB to complement the 1GB physical RAM (resulting in 3GB total virtual memory). This remediation instantly frees up 2GB of storage capacity on the root drive.

Step 3.1: Swap Space Implementation (Revised to 2GB)
Execute the following commands to safely configure a 2GB virtual memory allocation:

Bash
# 1. Deactivate the active swap file space
sudo swapoff /swapfile

# 2. Remove the original 4GB swap allocation from the file system
sudo rm /swapfile

# 3. Pre-allocate a new 2GB continuous block file on the root drive
sudo fallocate -l 2G /swapfile

# 4. Restrict file permissions to the root user for enhanced host security
sudo chmod 600 /swapfile

# 5. Format the designated file block into Linux swap space
sudo mkswap /swapfile

# 6. Enable the newly initialized 2GB swap partition
sudo swapon /swapfile

# 7. Append the block parameters to /etc/fstab to preserve modifications across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
Verification
Analyze storage and memory metrics to confirm successful optimization:

Bash
df -h   # Use% should decrease to around 70%, verifying 2GB+ of free space.
free -h  # The Swap row must display an allocation metric of 2.0Gi.

# --- [Swap Commands Deep Architecture Explanation] ---
# * fallocate: Dynamically reserves a dedicated continuous storage block on the physical disk.
# * chmod 600: Restricts data access exclusively to the root account to prevent system users or processes from extracting sensitive RAM runtime state data like secrets or tokens.
# * mkswap: Configures the raw system file into a dedicated architecture that the Linux kernel recognizes as virtual memory.
# * swapon: Instructs the core OS kernel to expand physical runtime capacity from 1GB to a combined 3GB memory space.
# * fstab injection: The fstab configuration file registers active mount systems during host boot. Neglecting this script causes the swap system to drop when the AWS host restarts.
Section 4: Installation of Prerequisites (Docker, Nginx & AWS CLI v2)
Step 4.1: Install Docker Engine & Docker Compose v2
Provision the host server with the core runtime environments required to deploy isolated container workloads:

Bash
sudo apt install docker.io docker-compose-v2 -y
Docker Permission Management (Best Practice):
Configure the platform to run container workloads without invoking root privileges (sudo) by binding the default ubuntu system user account to the docker security group:

Bash
sudo usermod -aG docker $USER
(Note: Terminate the active connection with exit and reconnect to evaluate this environment change. Executing docker ps without sudo should return an empty table status, verifying successful permissions).

Step 4.2: Install Nginx Web Server
Install Nginx to act as the primary reverse proxy handler for incoming port 80 traffic:

Bash
sudo apt install nginx -y
Step 4.3: Install AWS CLI v2
Install the official AWS CLI tools to manage secure deployment pulls from private Amazon ECR registries:

Bash
sudo apt install unzip -y && \
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && \
unzip awscliv2.zip && \
sudo ./aws/install && \
rm -rf awscliv2.zip aws
Validation: Confirm the deployment status by running aws --version to output the runtime version.

Section 5: Project Isolation & Architecture Setup (Step 3.8)
Concept: Why Project Isolation?
Maintaining isolated project directory structures on a Linux instance prevents resource dependencies from overlapping when hosting multiple web applications. This practice isolates application runtimes, environment configurations, and continuous deployment bindings. Editing these architecture blueprints is performed inside the terminal via the Nano editor utility.

Troubleshooting: Connection Timed Out (Port 22)
Issue: The SSH client fails to connect and drops with a Connection timed out status while the AWS Management Console displays the instance as Running.

Cause: Host access rules configured in Step 3.6 restrict ingress SSH traffic (Port 22) to a whitelisted local IP address. Local internet service providers often cycle user connection points using dynamic IP addressing, which changes your public IP profile and causes AWS to block entry.

Solution: Navigate to the AWS Console, locate the Security Groups configuration for the instance, click Edit Inbound Rules, modify the source parameter on Port 22 back to My IP, and save the configuration rule.

1️⃣ Part A: Production Docker Compose Specification
Establish the primary system directory and write the core deployment configuration tracking both the web service (Next.js + Payload) and database engine (PostgreSQL 16).

Bash
mkdir -p ~/tailor-inventory && cd ~/tailor-inventory
nano docker-compose.yml
Production Configuration Code (docker-compose.yml):

YAML
version: '3.8'

services:
  # 1. PostgreSQL Database Service
  db:
    image: postgres:16-alpine
    container_name: tailor-postgres-db
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-tailor_admin}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-TailorSecurePass2026}
      POSTGRES_DB: ${POSTGRES_DB:-tailor_inventory_db}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - tailor-network
    expose:
      - "5432"

  # 2. Next.js + Payload CMS Web Application Service
  web:
    image: tailor-inventory-app:latest
    container_name: tailor-nextjs-app
    restart: always
    ports:
      - "3001:3000"
    depends_on:
      - db
    env_file:
      - .env
    networks:
      - tailor-network

# Isolated Internal Network
networks:
  tailor-network:
    driver: bridge

# Persistent Storage Volume for DB
volumes:
  postgres_data:
    driver: local
(Note: Save modifications inside Nano by pressing Ctrl + O -> Enter -> Ctrl + X. Validate configuration text correctness using cat docker-compose.yml).

Docker Compose — Line-by-Line Architecture Deep Dive:
version: '3.8': References composition specification standard release 3.8.

image: postgres:16-alpine: Leverages an ultra-lightweight Alpine Linux distribution containing the Postgres 16 stack to preserve resource memory.

restart: always: Enforces container automatic restarts if processes fault out or if the parent hardware reboots.

${POSTGRES_USER:-tailor_admin}: Declares fallback context properties if variables do not resolve inside the home environment file.

postgres_data:/var/lib/postgresql/data: Maps system paths into persistent host resources, ensuring database state data survives container re-creation.

expose: "5432": Limits accessibility on port 5432 exclusively to containers sharing the backend engine bridge network (Strict Public Isolation).

ports: "3001:3000": Forwards port 3001 of the host network card straight to internal target port 3000 inside the app node container.

depends_on: db: Evaluates target service health before initializing dependent runtimes (Order Control).

driver: bridge: Establishes a private isolated overlay network on the server instance.

2️⃣ Part B: Environment Security Layer (.env)
Isolates production environment credentials locally on the target instance to prevent sensitive application values from leaking into git repositories.

Bash
nano .env
Production Environment Properties (.env):

Ini, TOML
# Database Configuration (Internal Docker Network Routing)
POSTGRES_USER=tailor_admin
POSTGRES_PASSWORD=TailorSecurePass2026
POSTGRES_DB=tailor_inventory_db
DATABASE_URL=postgresql://tailor_admin:TailorSecurePass2026@db:5432/tailor_inventory_db

# Payload CMS / Next.js Production Secrets
PAYLOAD_SECRET=SuperSecretPayloadKey2026ChangeMe
NEXT_PUBLIC_SERVER_URL=http://54.254.235.155

# Amazon S3 Decoupled Media Storage Settings
S3_BUCKET=tailor-inventory-assets
S3_REGION=ap-southeast-1
.env — Line-by-Line Architecture Deep Dive:
DATABASE_URL: Application database connection definition. Setting the host string to @db routes connection requests internally through Docker DNS resolution parameters directly to the Postgres instance.

NEXT_PUBLIC_SERVER_URL: Registers the active AWS public network identifier. Next.js and Payload CMS rely on this parameter to handle absolute domain compilation and coordinate second-layer routing engines.

PAYLOAD_SECRET: Represents the primary encryption cipher seed needed to securely track system login cookies and serialize cryptographic JWT parameters.

S3_BUCKET & S3_REGION: Provisions access destinations pointing toward targeted Amazon S3 buckets operating out of the Singapore cloud sector (ap-southeast-1) for application asset storage. (An IAM instance policy profile handles instance authorization, removing the requirement to explicitly maintain permanent AWS root developer credentials inside code blocks).

3️⃣ Part C: Nginx Reverse Proxy & Traffic Routing Configuration
Maps incoming traffic targeted at default public web ports (Port 80 HTTP) directly to the application container executing privately on Port 3001.

Bash
# Initialize application server definitions within the sites-available directory tree
sudo nano /etc/nginx/sites-available/tailor-inventory
Nginx Configuration Rules:

Nginx
server {
    listen 80;
    listen [::]:80;

    server_name 54.254.235.155;

    client_max_body_size 50M;

    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
Link & Activate Config
Nginx architecture conventions dictate that virtual site rules recorded within sites-available become functional only after generating a relative tracking symlink into the live sites-enabled system directory path.

Bash
# Instantiate symlink binding
sudo ln -s /etc/nginx/sites-available/tailor-inventory /etc/nginx/sites-enabled/

# Verify structural health of the Nginx declarative code rules
sudo nginx -t
Expected Output:
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

Bash
# Restart Nginx to load modifications into production runtime space
sudo systemctl restart nginx
Nginx Architecture — Line-by-Line Deep Dive:
listen 80 & listen [::]:80: Directs Nginx to bind and evaluate communication strings over standard IPv4 and IPv6 network adapters utilizing port 80.

server_name: Filters and captures browser requests directed at this public address destination (this parameter shifts to catch actual values once custom web domains wrap over the IP address).

client_max_body_size 50M: Increases file payload ingestion limits to 50MB to accommodate image uploads inside Payload CMS (the default value is 1MB, which rejects asset uploads exceeding that volume).

proxy_pass [http://127.0.0.1:3001](http://127.0.0.1:3001): The Core Reverse Proxy Magic. Rewrites and passes incoming requests down to internal localhost port 3001, where the application container runs.

Upgrade $http_upgrade & Connection 'upgrade': Stabilizes socket upgrade hooks to protect real-time application synchronization behaviors and maintain hot reloading protocols.

proxy_set_header X-Real-IP $remote_addr: Encapsulates and bubbles original network user IP profiles through to Next.js backends to preserve logging and telemetry collection transparency.

proxy_set_header X-Forwarded-Proto $scheme: Passes client communications parameters downstream to app instances to track whether incoming operations evaluate under HTTP or HTTPS layer definitions.

Final Verification: How to know Nginx is 100% working?
Terminal Diagnostics Validation:

Bash
sudo systemctl status nginx
(Verify the runtime tracker is highlighted in green displaying an active (running) state. Press q to exit).

Real-world Browser Test:
Input the active public tracking address ([http://54.254.235.155](http://54.254.235.155)) inside your local web browser.

Observed Result: The viewport must render a 502 Bad Gateway system page flag.

DevOps Logic: Throwing a 502 condition at this point confirms Nginx is operating perfectly. It means Nginx successfully intercepts traffic on public Port 80 and attempts to route the instruction chain down to inner Port 3001. Because the main application container hasn't been instantiated yet, the reverse proxy returns a 502 error, which confirms the network routing engine is working correctly.

PHASE 3 IS OFFICIALLY COMPLETE AND SIGNED OFF!