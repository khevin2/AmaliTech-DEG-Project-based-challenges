# Deployment Guide

This guide documents the AWS EC2 deployment used by the DeployReady CI/CD pipeline.

## Target Platform

- Cloud provider: AWS
- Compute service: EC2
- Operating system: Ubuntu Server LTS
- Runtime: Docker Engine
- Public entrypoint: `http://<ec2-public-ip>/health`

EC2 is sufficient for this challenge because the API is a single stateless service and can run as one Docker container.

## EC2 Setup

1. Launch an EC2 instance.
   - Recommended size for the challenge: `t2.micro` or `t3.micro`.
   - Recommended OS: Ubuntu Server LTS.
   - Attach a key pair that is used only for deployment access.

2. Configure the security group.

| Type | Protocol | Port | Source | Reason |
| --- | --- | --- | --- | --- |
| SSH | TCP | 22 | `<your-public-ip>/32` | Deployment and administration. Do not use `0.0.0.0/0`. |
| HTTP | TCP | 80 | `0.0.0.0/0` | Public health/API access for the challenge. |
| HTTP | TCP | 80 | `::/0` | Optional only if IPv6 is enabled. |

3. Create or choose a deployment user.

The workflow assumes `EC2_USER` can run Docker without an interactive password prompt. A dedicated `deploy` user is preferred.

Example:

```bash
sudo adduser --disabled-password --gecos "" deploy
```

Copy the deployment public key into `/home/deploy/.ssh/authorized_keys`, then set permissions:

```bash
sudo mkdir -p /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo nano /home/deploy/.ssh/authorized_keys
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh
```

Private keys must remain private and must never be committed to Git.

## Docker Installation

Connect to the EC2 instance:

```bash
ssh -i <private-key.pem> ubuntu@<ec2-public-ip>
```

Install Docker Engine on Ubuntu:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

Allow the deployment user to run Docker:

```bash
sudo usermod -aG docker deploy
```

Log out and back in so the group membership is refreshed, then verify:

```bash
docker version
docker ps
```

## GitHub Secrets

Create these repository secrets before the first deployment:

| Secret | Example | Notes |
| --- | --- | --- |
| `EC2_HOST` | `203.0.113.10` | EC2 public IP or DNS name. |
| `EC2_USER` | `deploy` | User with Docker access. |
| `EC2_SSH_KEY` | Private key content | Use the private key matching `authorized_keys`. |
| `EC2_KNOWN_HOSTS` | Output of `ssh-keyscan -H <host>` | Pins the EC2 host key. |
| `GHCR_USERNAME` | `your-github-user` | Account used for GHCR pull access. |
| `GHCR_TOKEN` | GitHub package token | Minimum permission: read packages for the image. |
| `APP_PORT` | `3000` | Port the Node.js process listens on inside the container. |

Generate `EC2_KNOWN_HOSTS` from your workstation:

```bash
ssh-keyscan -H <ec2-public-ip>
```

The real `.env`, private SSH keys, `.pem` files, and tokens must never be committed.

## How the Image Is Pulled

The workflow builds and pushes:

```text
ghcr.io/<github-owner>/<repository>:<commit-sha>
```

During deployment, the EC2 server logs in to GHCR and pulls that immutable SHA tag:

```bash
echo "<GHCR_TOKEN>" | docker login ghcr.io -u "<GHCR_USERNAME>" --password-stdin
docker pull ghcr.io/<github-owner>/<repository>:<commit-sha>
```

Do not paste real tokens into shell history on shared machines.

## How the Container Is Started

The workflow replaces the current container with:

```bash
docker stop deployready-api 2>/dev/null || true
docker rm deployready-api 2>/dev/null || true

docker run -d \
  --name deployready-api \
  --restart unless-stopped \
  -p 80:3000 \
  -e PORT=3000 \
  ghcr.io/<github-owner>/<repository>:<commit-sha>
```

The host listens on port `80`. The container listens on `PORT=3000`.

## Check Running Containers

```bash
docker ps
docker ps --filter name=deployready-api
```

Useful fields:

- `STATUS` should show the container as running.
- `PORTS` should include `0.0.0.0:80->3000/tcp`.

## View Logs

```bash
docker logs deployready-api
docker logs --tail 100 -f deployready-api
```

The app should log:

```text
Server running on port 3000
```

## Test `/health`

From the EC2 instance:

```bash
curl -fsS http://127.0.0.1/health
```

From your workstation:

```bash
curl -fsS http://<ec2-public-ip>/health
```

Expected response:

```json
{"status":"ok"}
```

## Operational Notes

- Keep SSH open only to your current public IP address.
- Rotate the deployment key if it is exposed.
- Use a GHCR token with only the permissions required to pull this package.
- Keep package publishing in GitHub Actions limited to `packages: write`.
- Do not commit `.env`, `.pem`, private keys, tokens, `node_modules`, or build artifacts.
- If the container restarts repeatedly, inspect `docker logs deployready-api` before redeploying.
