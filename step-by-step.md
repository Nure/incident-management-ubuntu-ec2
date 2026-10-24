# bongoDev — Student Lab: Manually Deploy Incident Management on Ubuntu EC2

**Platform:** AWS EC2 running **Ubuntu Server 24.04 LTS**  
**App:** React + Vite frontend, Node.js + Express REST API, PostgreSQL database  
**Production-style processes:** Nginx reverse proxy + systemd; **no Docker or Docker Compose**  
**Target duration:** 40–65 minutes (AWS installation and npm downloads may take longer)  

## 1. Purpose of this lab

In this lab you will deploy a real three-tier application **manually** on an Ubuntu EC2 machine. The objective is to understand exactly what servers, package managers, databases, operating-system services and web proxies do **before** automating deployments in GitHub Actions.

By completing this lab you should understand:

- How to launch Ubuntu EC2 and connect securely over SSH.
- Why Ubuntu uses `apt` and how to install **Node.js 24**, **PostgreSQL** and **Nginx**.
- How to set up a local PostgreSQL database and keep its credentials private.
- How to compile a React app, run an Express backend under `systemd`, and use Nginx to serve the UI and proxy `/api` requests.
- How to verify real data persistence and update the live app manually.
- Which steps a future GitHub Actions workflow should automate.

**This lab creates resources that can incur AWS charges.** Use a disposable classroom account/instance, keep track of its resources, and terminate the EC2 instance after you submit your evidence. The demo uses HTTP and has no application login; do not enter real personal or company information.

## 2. Architecture — know what each component does

```text
Your browser
    |
    | HTTP :80
    v
Ubuntu EC2: Nginx
    |-- / and static assets ---> React production files (/var/www/incident-management)
    |
    `-- /api/* and /health ---> Express/Node.js (127.0.0.1:5000)
                                    |
                                    `-- PostgreSQL (127.0.0.1:5432)

systemd keeps the Node.js backend service alive.
```

**Publicly expose only port 80 for this classroom website and port 22 from your own IP for SSH. Do NOT open 5000 or 5432 to the Internet.**

## 3. Prerequisites

- An AWS account and permissions to launch EC2 and edit security groups.
- The attached `incident-management-ubuntu-ec2.zip` file on your own computer.
- A terminal with `ssh` and `scp` (Mac Terminal, Linux terminal, or Windows PowerShell with OpenSSH).
- Internet access from EC2 to reach Ubuntu repositories, NodeSource and the npm registry.
- Basic knowledge of terminal commands (`cd`, `ls`, `cat`).

## 4. Launch an Ubuntu EC2 instance in AWS

1. Sign in to the AWS Management Console, choose a region (example: **us-east-1**) and open **EC2 → Instances → Launch instances**.
2. Enter the name **`bongodev-incident-ubuntu`**.
3. Choose the official **Ubuntu Server 24.04 LTS (64-bit x86)** AMI from Canonical. Do **not** use Amazon Linux for these `apt` scripts.
4. Choose a suitable size; **t3.small (2 GiB RAM)** or larger is a practical teaching starting point. It may not qualify for free-tier benefits. If an npm build runs out of memory, choose a larger instance rather than exposing additional ports.
5. Under **Key pair**, create an RSA/ED25519 SSH key pair (example file **`bongodev-ec2.pem`**) or select one you already own. Download the private key and store it securely. **Never upload the key to GitHub.**
6. Use a VPC with a **public subnet, Internet Gateway route and auto-assigned public IPv4 address**. A suitable default VPC is fine.
7. Create/choose a **security group** with these inbound rules:

   | Type | Port | Source | Reason |
   |---|---:|---|---|
   | SSH | 22 | **My IP** (`your-public-ip/32`) | You connect to the server |
   | HTTP | 80 | `0.0.0.0/0` | Students can open the demo website |

   Leave PostgreSQL 5432, backend 5000 and Vite 5173 **closed**. Outbound rules must allow package downloads.
8. Choose **20 GiB gp3 EBS** storage for comfortable lab space (storage costs may apply).
9. Click **Launch instance**. Wait for its instance state to become **Running**, then copy the **Public IPv4 address** from EC2 instance details.

For the remaining examples, replace `EC2_PUBLIC_IP` with your *real* address. Do not type the angle brackets or placeholder text.

## 5. Connect to EC2 over SSH

On **your computer**, open Terminal (Mac/Linux) or PowerShell (Windows with OpenSSH). If the `.pem` is saved in `~/Downloads`:

```bash
chmod 400 ~/Downloads/bongodev-ec2.pem
ssh -i ~/Downloads/bongodev-ec2.pem ubuntu@EC2_PUBLIC_IP
```

`chmod` applies on Mac/Linux. On Windows, keep your key readable only to your Windows account and use the Windows path with `ssh -i`. Ubuntu EC2 normally uses username **`ubuntu`**.

Now you are inside EC2. Verify the OS:

```bash
cat /etc/os-release
whoami
uname -m
```

You should see Ubuntu 24.04 and user `ubuntu`.

## 6. Transfer the project ZIP to the Ubuntu machine

**On your own computer (not inside the SSH window):** open a second terminal where the downloaded ZIP is stored. Run:

```bash
scp -i ~/Downloads/bongodev-ec2.pem \
  ~/Downloads/incident-management-ubuntu-ec2.zip \
  ubuntu@EC2_PUBLIC_IP:/home/ubuntu/
```

For Windows PowerShell, use the equivalent key/ZIP paths, such as `$HOME\Downloads\bongodev-ec2.pem` and `$HOME\Downloads\incident-management-ubuntu-ec2.zip`.

**Back in your EC2 SSH session:** extract the source:

```bash
sudo apt-get update
sudo apt-get install -y unzip
cd /home/ubuntu
unzip incident-management-ubuntu-ec2.zip
cd incident-management-ubuntu-ec2
ls -la
```

The folder contains `frontend/`, `backend/`, `scripts/`, `deploy/` and this `step-by-step.md`.

**Alternative:** Once the source is in your own GitHub repository, you may use `git clone` instead of `scp`. The first manual deployment does not require GitHub.

## 7. Install Node.js, PostgreSQL and Nginx on Ubuntu

You have **two choices**. For learning, run the individual installation scripts one at a time. **Do not run both choices; that would repeat the same installations.**

### Choice A — Learn each installation separately (recommended)

Run the following **on EC2** from the project root:

```bash
bash scripts/install-nodejs-ubuntu.sh
node --version
npm --version
```

Expected: Node.js **v24.x**. The script configures the signed NodeSource Ubuntu `apt` repository. `node` runs JavaScript and `npm` installs JavaScript dependencies.

```bash
bash scripts/install-postgresql-ubuntu.sh
psql --version
sudo systemctl status postgresql --no-pager
```

Ubuntu 24.04 commonly installs PostgreSQL 16 from its standard repository. PostgreSQL stores incidents so they survive page refreshes and Node restarts.

```bash
bash scripts/install-nginx-ubuntu.sh
nginx -v
sudo systemctl status nginx --no-pager
```

Nginx serves the React website on port 80 and forwards `/api` requests to the local Node service.

Install the few remaining helper tools:

```bash
sudo apt-get install -y git curl openssl rsync
```

### Choice B — Install the entire stack using one command

Instead of Choice A, run:

```bash
bash scripts/install-ubuntu.sh
```

This invokes all Ubuntu installers and installs the utilities. The end result is the same.

**Check:**

```bash
node -v
npm -v
psql --version
nginx -v
```

## 8. Create the PostgreSQL database and application user

Run:

```bash
bash scripts/setup-database.sh
```

The script will:

1. Create a PostgreSQL user called `incident_user` with a generated password.
2. Create the database `incidentdb` owned by that user.
3. Save the local API configuration in `backend/.env` with restrictive file permissions.
4. Test the username/password connection.

**Never commit or upload `backend/.env` or share its password.** It is intentionally ignored by `.gitignore`.

Check PostgreSQL without exposing the application password:

```bash
sudo -u postgres psql -c '\l'
sudo -u postgres psql -c '\du'
```

The application creates its `incidents` table automatically on first startup, then inserts two sample incidents if the table is empty.

## 9. Deploy the application manually

This is the key step. Run:

```bash
bash scripts/deploy-ubuntu.sh
```

Read the messages carefully. The script:

1. Runs `npm install` (or `npm ci` if lockfiles exist) for frontend and backend.
2. Builds the React production bundle with `npm run build`.
3. Copies the compiled site into `/var/www/incident-management/`.
4. Configures `incident-backend.service` and uses **systemd** to start/restart Express at `127.0.0.1:5000`.
5. Configures **Nginx** to serve the UI and proxy `/api/` and `/health` to Express.
6. Validates that the backend reports a healthy database connection.

**Why this is a manual deployment:** you logged in over SSH and ran the steps yourself. The next GitHub Actions exercise will automate those steps.

If installation succeeds, the site is reachable through the EC2 public IPv4 address.

## 10. Verify that everything works

**On EC2**, run these commands:

```bash
bash scripts/verify.sh http://127.0.0.1
bash scripts/check-services.sh
sudo systemctl status incident-backend --no-pager
sudo nginx -t
curl http://127.0.0.1/health
curl http://127.0.0.1/api/incidents
```

Health should report `"status":"healthy"` and `"database":"connected"`. The incidents API returns JSON.

Now open **your own computer's web browser** and visit:

```text
http://EC2_PUBLIC_IP/
```

Use **HTTP** here, not HTTPS, because the classroom example does not configure a TLS certificate.

From the UI, **create an incident → change its status → refresh the page**. It should remain in PostgreSQL. Then try deleting an incident.

For an API-only test, on EC2:

```bash
curl -i -X POST http://127.0.0.1/api/incidents \
  -H 'Content-Type: application/json' \
  -d '{"title":"EC2 deployment test","description":"Created using curl","severity":"low"}'
```

**Database proof:** list persisted records directly from PostgreSQL:

```bash
sudo -u postgres psql -d incidentdb -c 'SELECT id, title, severity, status FROM incidents ORDER BY id DESC;'
```

## 11. Show a manual redeployment

Edit only the React UI heading on EC2:

```bash
nano frontend/src/main.jsx
```

Find `Incident Command Center` and change it to `Incident Command Center — Updated Manually`. Save the file (**Ctrl+O**, **Enter**, **Ctrl+X** in nano).

Now rerun the same deployment script:

```bash
bash scripts/deploy-ubuntu.sh
```

Refresh `http://EC2_PUBLIC_IP/` and verify the new heading. Your existing incident records should remain because the script **does not drop the database**.

**Important:** This is an app-code redeployment, not EC2 reprovisioning. A future GitHub Actions workflow will run this deployment remotely after Git pushes.

## 12. Understand systemd and Nginx

```bash
sudo systemctl status incident-backend --no-pager
sudo journalctl -u incident-backend -n 80 --no-pager
sudo systemctl restart incident-backend
sudo systemctl status nginx --no-pager
sudo nginx -t
cat deploy/nginx-incident-management.conf
```

Ask yourself: If the backend crashes, what restarts it? Why does the backend bind to `127.0.0.1:5000`? Why is PostgreSQL not accessible from the Internet?

## 13. Troubleshooting

| Symptom | How to investigate |
|---|---|
| `ssh: connection timed out` | Correct public IP? Running instance? TCP 22 permitted **from your IP**? Public subnet and route? |
| `Permission denied (publickey)` | Wrong `.pem`, username (`ubuntu`), key permissions or EC2 key pair |
| `node: command not found` | Run `bash scripts/install-nodejs-ubuntu.sh`; verify Ubuntu 24.04 |
| `npm install` fails | EC2 needs outbound Internet; inspect npm error; verify disk space with `df -h` |
| `Could not connect to database` | `sudo systemctl status postgresql`; check DB setup, `backend/.env` and backend logs |
| `502 Bad Gateway` | `sudo journalctl -u incident-backend -n 100 --no-pager` and `curl http://127.0.0.1:5000/health` |
| Public website won't open | EC2 security group **HTTP 80**, public IP, route, `sudo systemctl status nginx` |
| Frontend loads but no incidents | Open browser developer console; test `curl http://127.0.0.1/api/incidents` from EC2 |
| Nginx configuration error | `sudo nginx -t` and `cat /etc/nginx/sites-available/incident-management` |
| Node build is killed | Check `free -h`; try a larger EC2 instance with enough RAM |

## 14. Student assignment — what to submit

- Screenshot: **Ubuntu EC2 Running** (hide/blur sensitive account information).
- Screenshot: `node -v`, `psql --version`, `nginx -v` terminal results.
- Screenshot: `sudo systemctl status incident-backend` and `curl .../health`.
- Screenshot: running **Incident Command Center** opened through the public EC2 IP.
- Screenshot: **a newly created incident** and its changed status.
- Screenshot: **the updated UI heading after manual redeployment**.
- A short explanation (5–8 lines) of how Browser → Nginx → Express → PostgreSQL work together, and which steps GitHub Actions could automate.

**Do not share passwords, `.env` contents or SSH private keys in screenshots or repositories.**

## 15. Next homework: automate deployment using GitHub Actions

After the manual deployment works, follow [`docs/GITHUB_ACTIONS.md`](docs/GITHUB_ACTIONS.md) for the optional follow-up lesson. The included `.github/workflows/deploy.yml` can build the app and deploy to **this same Ubuntu EC2** (after configuring secure access). It does **not** automatically create an EC2 instance. The Ubuntu machine, DB and Nginx must be installed first.

The deployment script is intentionally repeatable: do the manual steps first, then automate that script.

## 16. Mandatory cleanup — avoid AWS charges

1. Take the required screenshots and copy any homework notes.
2. Delete demo app secrets and SSH deployment keys from GitHub if you configured the optional automation.
3. AWS Console → **EC2 → Instances** → select `bongodev-incident-ubuntu` → **Instance state → Terminate instance**.
4. Confirm the instance is terminated and review attached EBS volumes, Elastic IPs (if any) and other billable resources. Termination may delete the root volume depending on its delete-on-termination setting.

**Never terminate an instance that stores important data.** The demo database is intentionally disposable and is lost when the attached storage is deleted.

---

### Extra credit questions

1. Why is PostgreSQL not exposed on public port 5432?
2. What is the difference between an Ubuntu service (`systemd`) and an Nginx reverse proxy?
3. Why do we build React using `npm run build` before deploying it?
4. Why does changing React code require a build/redeploy, but creating an incident does not?
5. Which actions could GitHub Actions perform automatically after `git push`?
