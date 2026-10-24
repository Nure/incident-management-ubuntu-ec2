# bongoDev — Student submission checklist (Ubuntu EC2 manual deployment)

Read [step-by-step.md](step-by-step.md) and deploy the app **without Docker**.

- [ ] Ubuntu 24.04 EC2 running, SSH access from my own IP only
- [ ] Security group: SSH 22 from my IP; HTTP 80 from everyone; ports 5000/5432 not public
- [ ] Node.js 24, PostgreSQL and Nginx installed natively
- [ ] PostgreSQL application DB and role created using `scripts/setup-database.sh`
- [ ] Backend managed by `incident-backend` systemd service
- [ ] Public URL loads Incident Command Center
- [ ] I created an incident, changed its status and confirmed persistence after refresh
- [ ] I changed the UI heading and redeployed manually using `scripts/deploy-ubuntu.sh`
- [ ] Screenshots and architecture explanation submitted (no secrets)
- [ ] EC2 instance terminated when homework is complete

**Optional follow-up:** Use the included `.github/workflows/deploy.yml` and [docs/GITHUB_ACTIONS.md](docs/GITHUB_ACTIONS.md) to deploy later application changes to the existing Ubuntu server on a Git push.
