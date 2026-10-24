# bongoDev Incident Management — Ubuntu EC2 (No Docker)

**Start here: [step-by-step.md](step-by-step.md)** — complete student instructions from EC2 creation to manual app deployment, verification, redeployment, and cleanup.

## Lab goal

Deploy a React + Express + PostgreSQL incident-management application **natively on Ubuntu Server 24.04 LTS EC2**. Install Node.js 24 LTS, PostgreSQL and Nginx on the machine; run the API with systemd; serve the frontend with Nginx. **Docker is not needed or used.** Once you understand the manual process, optionally automate updates with GitHub Actions.

## Quick commands (after you copy and extract the ZIP onto Ubuntu EC2)

```bash
cd ~/incident-management-ubuntu-ec2
bash scripts/install-ubuntu.sh
bash scripts/setup-database.sh
bash scripts/deploy-ubuntu.sh
bash scripts/verify.sh http://127.0.0.1
```

Then open **`http://YOUR_EC2_PUBLIC_IP/`** from your browser. The first installation requires network access and an appropriately configured Ubuntu EC2 instance (see the full student guide).

| File | Purpose |
|---|---|
| `step-by-step.md` | Student lab from start to finish |
| `frontend/` | React + Vite client |
| `backend/` | Node.js + Express + PostgreSQL REST API |
| `scripts/install-nodejs-ubuntu.sh` | Install Node.js 24 on Ubuntu 24.04 |
| `scripts/install-postgresql-ubuntu.sh` | Install PostgreSQL |
| `scripts/install-nginx-ubuntu.sh` | Install Nginx |
| `scripts/install-ubuntu.sh` | Install everything plus base utilities |
| `scripts/setup-database.sh` | Create database and local backend `.env` |
| `scripts/deploy-ubuntu.sh` | Build UI, install systemd service and Nginx site |
| `scripts/check-services.sh` | Display service and port diagnostics |
| `.github/workflows/deploy.yml` | Optional GitHub Actions homework deployment |
| `docs/GITHUB_ACTIONS.md` | Optional second-phase CI/CD walkthrough |

> Educational demo: app has no authentication and uses HTTP. Do not store sensitive incidents; do not expose Node port 5000 or PostgreSQL port 5432 publicly. Terminate demo resources when finished to avoid AWS charges.
