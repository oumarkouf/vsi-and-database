# Lab: Deploy a Web App on a VM (IaaS) with a Managed Database (PaaS)

## Objectives

By the end of this lab you will be able to:

- Provision cloud infrastructure as code with **Terraform**
- Deploy a **VM (IaaS)** and a **managed SQL database (PaaS)** on AWS or GCP
- Secure network access (Security Group / firewall, SSH keys)
- Configure a server with **Ansible** (or shell commands as a fallback)
- Deploy and test a Python/Flask application connected to PostgreSQL

## Target architecture

```mermaid
flowchart LR
    U[User / Browser] -->|HTTPS 443| LB[HTTPS Load Balancer<br/>ALB or Cloud Load Balancing<br/>TLS terminated here]
    LB -->|HTTP 8080<br/>health check /health| VM1[VM 1<br/>Flask app]
    LB -->|HTTP 8080<br/>health check /health| VM2[VM 2<br/>Flask app]
    A[Admin] -->|SSH 22 - your IP only| VM1
    A -->|SSH 22 - your IP only| VM2
    VM1 -->|PostgreSQL 5432<br/>private access only| DB[(Managed PostgreSQL<br/>RDS or Cloud SQL)]
    VM2 -->|PostgreSQL 5432<br/>private access only| DB
```

| Component | AWS | GCP |
|---|---|---|
| VM (IaaS) x2 | EC2 (Amazon Linux / Ubuntu) | Compute Engine (Ubuntu) |
| HTTPS load balancer | Application Load Balancer (ALB) + ACM certificate | External Application Load Balancer + SSL certificate |
| Database (PaaS) | RDS for PostgreSQL | Cloud SQL for PostgreSQL |
| Network filtering | Security Groups | VPC firewall rules |
| CLI | AWS CLI | gcloud CLI |

> Choose **one** provider. The sample app uses PostgreSQL (`psycopg2`), so the database must be PostgreSQL.

## Prerequisites

- A cloud account (AWS or GCP) with billing/free tier enabled and a budget alert set
- Git, and a terminal
- Docs: [gcloud install](https://cloud.google.com/sdk/docs/install-sdk) -
  [AWS CLI install](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

## Steps

### 1. Prepare your tools
1. Install the CLI of your provider and authenticate (`aws configure` or `gcloud auth login`).
2. Install [Terraform](https://developer.hashicorp.com/terraform/install) and check with `terraform version`.
3. Generate an SSH key pair if you do not have one: `ssh-keygen -t ed25519`.

### 2. Provision the infrastructure with Terraform
Write Terraform code (`main.tf`, `variables.tf`, `outputs.tf`) that creates:

- A VM (Amazon Linux or Ubuntu) with your SSH public key
- A managed PostgreSQL instance (smallest/free-tier size)
- Firewall rules:
  - SSH (22) restricted to **your IP** (not `0.0.0.0/0`)
  - App port (8080, or 80 if using Apache) open to the Internet
  - PostgreSQL (5432) reachable **only from the VM**, not from the Internet
- Outputs: VM public IP and database endpoint

Rules:
- Do **not** hardcode the DB password: use a sensitive variable (`TF_VAR_db_password`).
- Run `terraform fmt`, `terraform validate`, then `terraform plan` before `terraform apply`.

### 3. Configure SSH access
```bash
chmod 400 ~/.ssh/<your-key>
ssh -i ~/.ssh/<your-key> <user>@<vm-public-ip>
```
Default users: `ec2-user` (Amazon Linux), `ubuntu` (Ubuntu).

### 4. Install the software (Ansible recommended)
Write an Ansible playbook (inventory + `playbook.yml`) that installs Python 3, pip, Git and the app dependencies.
If you cannot use Ansible, run the commands manually:

| | Amazon Linux | Ubuntu |
|---|---|---|
| Update | `sudo yum update -y` | `sudo apt update && sudo apt upgrade -y` |
| Install | `sudo yum install -y python3 python3-pip git` | `sudo apt install -y python3 python3-pip python3-venv git` |

**Bonus:** install Apache HTTP (`httpd` / `apache2`) as a reverse proxy on port 80 forwarding to Flask on 8080.

### 5. Deploy the application
```bash
git clone <your-repository-url> && cd vsi-and-database
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env      # edit with your DB endpoint and password
set -a; source .env; set +a
python app.py
```

### 6. Test
- `curl http://<vm-public-ip>:8080/health` returns `{"status":"ok"}` (app is up)
- `curl http://<vm-public-ip>:8080/` returns the PostgreSQL version (database is reachable)

Troubleshooting:

| Symptom | Likely cause |
|---|---|
| Connection timeout from the browser | App port not open in the Security Group / firewall |
| `/health` OK but `/` fails with a timeout | DB firewall does not allow the VM, or wrong subnet/VPC |
| `password authentication failed` | Wrong `DB_USER` / `DB_PASS` |
| `Missing environment variable` | `.env` not loaded in the current shell |

### 7. Add an HTTPS load balancer in front of two VMs

Goal: users reach the app only through `https://<lb-dns>`; the load balancer terminates TLS and spreads requests across **two identical VMs**. The VMs are not exposed directly any more.

1. **Create a second VM** with Terraform (`count = 2` / two instances, ideally in **different zones/AZs**) and deploy the same app on both (steps 3 to 5, e.g. Ansible inventory with two hosts).
   Each VM answers `/whoami` with its hostname and adds an `X-Served-By` header, so you can see which VM served each request.
2. **Get a TLS certificate**
   - With a domain: AWS ACM / GCP managed certificate (free, DNS validation).
   - Without a domain (lab only): generate a self-signed certificate and import it (ACM import / GCP self-managed SSL certificate).
     ACM only accepts an **RSA 2048** certificate (X.509 v3, with a SAN) and an **unencrypted PKCS#1 private key** (`-----BEGIN RSA PRIVATE KEY-----`).
     OpenSSL 3.x (default on recent macOS/Linux) writes PKCS#8 (`-----BEGIN PRIVATE KEY-----`), which the load balancer rejects, so convert it:
     ```bash
     openssl req -x509 -newkey rsa:2048 -nodes -days 30 \
       -keyout lb.pkcs8.key -out lb.crt \
       -subj "/CN=lab-lb.example.com" \
       -addext "subjectAltName=DNS:lab-lb.example.com" \
       -addext "basicConstraints=critical,CA:FALSE" \
       -addext "keyUsage=digitalSignature,keyEncipherment" \
       -addext "extendedKeyUsage=serverAuth"

     # PKCS#8 -> PKCS#1 (the format ACM expects)
     openssl rsa -in lb.pkcs8.key -traditional -out lb.key

     head -1 lb.key        # must print: -----BEGIN RSA PRIVATE KEY-----
     ```
     Import it (AWS), using the converted `lb.key`:
     ```bash
     aws acm import-certificate \
       --certificate fileb://lb.crt \
       --private-key fileb://lb.key
     ```
     The name in the certificate is only a label: the browser/`curl -k` reaches the load balancer by its DNS name and ignores the mismatch.
     Never commit `lb.key` or `lb.pkcs8.key`.
3. **Create the load balancer with Terraform**

   | Element | AWS | GCP |
   |---|---|---|
   | Load balancer | `aws_lb` (type `application`, public subnets in 2 AZs) | `google_compute_global_forwarding_rule` + `google_compute_target_https_proxy` + `google_compute_url_map` |
   | Backends | `aws_lb_target_group` (HTTP 8080) + `aws_lb_target_group_attachment` for each VM | `google_compute_instance_group` (or unmanaged group per VM) + `google_compute_backend_service` |
   | Health check | `/health`, HTTP 8080 | `google_compute_health_check` on `/health`, port 8080 |
   | HTTPS listener | `aws_lb_listener` port 443, `certificate_arn` | target HTTPS proxy with `ssl_certificates` |
   | HTTP to HTTPS (optional) | listener 80 with a redirect action | separate URL map with `https_redirect = true` |

4. **Tighten the firewall rules**
   - Load balancer: 443 (and 80 for the redirect) open to the Internet.
   - VMs, port 8080: allowed **only from the load balancer** (AWS: source = the ALB security group; GCP: Google health-check/proxy ranges `130.211.0.0/22`, `35.191.0.0/16`).
   - SSH (22) still restricted to your IP; PostgreSQL (5432) still reachable only from the VMs.
5. Add the LB address to `outputs.tf` (`lb_dns_name` on AWS, forwarding rule IP on GCP).

### 8. Show that the load balancing works

1. **Check targets are healthy** (console, or `aws elbv2 describe-target-health --target-group-arn <arn>`): both VMs must be `healthy`.
2. **Send repeated requests** to the load balancer with the provided script [`test-lb.sh`](test-lb.sh):
   ```bash
   ./test-lb.sh <lb-dns-or-ip> 20
   ```
   Expected output: requests alternate between the two hostnames, e.g.
   ```text
   request 1 -> vm-1
   request 2 -> vm-2
   ...
   summary:  10 vm-1
   summary:  10 vm-2
   ```
   The same check by hand: `curl -sk -i https://<lb-dns>/whoami` shows `X-Served-By: <hostname>` and the JSON body. In a browser, refresh `https://<lb-dns>/whoami` several times.
   (`-k` is only needed with a self-signed certificate.)
3. **Check HTTPS**: `curl -vk https://<lb-dns>/health 2>&1 | grep -E "SSL connection|subject|HTTP/"` shows the TLS handshake and the certificate.
4. **Demonstrate high availability** (failover):
   ```bash
   # on VM 1
   sudo systemctl stop <app-service>      # or: pkill -f "python app.py"
   ```
   Wait for the health check to mark VM 1 `unhealthy` (about 10-30 s), then run `./test-lb.sh <lb-dns-or-ip> 20` again: **all** responses now come from VM 2 and the app stays available. Restart the app on VM 1 and observe that traffic is shared again.
5. Take screenshots: healthy targets, the `test-lb.sh` summary showing both VMs, and the failover test.

Troubleshooting:

| Symptom | Likely cause |
|---|---|
| Targets `unhealthy` | VM firewall does not allow the load balancer on 8080, or `/health` not reachable |
| `502` / `503` from the LB | No healthy target, or the app is not listening on `0.0.0.0:8080` |
| Always the same VM | Sticky sessions enabled, or `curl` reuses one connection (try more requests / `--no-keepalive`) |
| Certificate warning / `curl: (60)` | Self-signed certificate: use `-k` for the lab |
| ACM import fails (`ValidationException` / invalid private key) | Key is PKCS#8 (`BEGIN PRIVATE KEY`), encrypted, or not RSA 2048: convert with `openssl rsa -traditional` (see step 7.2) |

### 9. Clean up (mandatory)
```bash
terraform destroy
```
Verify in the console that no VM, load balancer, database, certificate or disk remains, to avoid unexpected costs (load balancers are billed per hour).

## Deliverables

- A Git repository containing: Terraform code (2 VMs, DB, HTTPS load balancer), Ansible playbook (if used), `app.py`, `requirements.txt`, `test-lb.sh`
- A short README (how to deploy) and screenshots of the working app, the healthy targets, the load-balancing output and the failover test
- No secrets in the repository (`.env`, `*.tfstate`, `*.tfvars`, keys must be listed in your `.gitignore`)

## Going further

- Run the app as a `systemd` service so it restarts automatically
- Use a real domain and a managed certificate (Let's Encrypt / ACM / GCP managed cert)
- Use an auto-scaling group / managed instance group instead of two fixed VMs
- Add a Terraform remote state backend (S3 / GCS)
