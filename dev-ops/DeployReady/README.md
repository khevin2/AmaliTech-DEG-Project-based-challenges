# DeployReady DevOps Challenge

## Project Overview

DeployReady is a DevOps-ready packaging and deployment setup for the Kora Analytics Node.js API in `app/`.

The API exposes:

| Method | Route | Purpose |
| --- | --- | --- |
| GET | `/health` | Returns service health status. |
| GET | `/metrics` | Returns uptime, memory, and Node.js runtime information. |
| POST | `/data` | Accepts and echoes a non-empty JSON payload. |

The application logic is intentionally left unchanged. This solution adds the operational layer around the app: deterministic dependency installation, containerization, local Docker Compose support, CI testing, image publishing, and EC2 deployment automation.

## Architecture Overview

Editable architecture diagram: [DeployReady Architecture FigJam](https://www.figma.com/board/Rf2yopq6Sb7tjNQfluwduD/DeployReady-Architecture?node-id=0-1&p=f)

```text
Developer push to main
        |
        v
GitHub Actions
  - npm ci and npm test in app/
  - Docker build from dev-ops/DeployReady
  - Push image to GitHub Container Registry
  - SSH into EC2
        |
        v
AWS EC2 instance
  - Docker pulls ghcr.io/<owner>/<repo>:<commit-sha>
  - Existing deployready-api container is replaced
  - Health check failure rolls back to the previous image
  - Container listens on PORT=3000
  - Host port 80 forwards to container port 3000
        |
        v
GET http://<ec2-public-ip>/health
```

## Repository Layout

```text
dev-ops/DeployReady/
  app/
    index.js
    index.test.js
    package.json
    package-lock.json
  Dockerfile
  .dockerignore
  docker-compose.yml
  .env.example
  DEPLOYMENT.md
  README.md

.github/workflows/deploy.yml
```

The GitHub Actions workflow is stored at the repository root because GitHub only discovers workflows from `.github/workflows`.

## Local Setup

Prerequisites:

- Node.js 22 or a compatible recent Node.js version
- npm

Run the API locally:

```bash
cd dev-ops/DeployReady/app
npm ci
npm test
npm start
```

The app reads `PORT` from the environment and defaults to `3000` when it is not set.

PowerShell example:

```powershell
cd dev-ops/DeployReady/app
$env:PORT = "3000"
npm start
```

Health check:

```bash
curl http://localhost:3000/health
```

Expected response:

```json
{"status":"ok"}
```

## Docker Setup

Prerequisites:

- Docker Engine
- Docker Compose v2

Docker Compose automatically reads variables from `.env` when the file exists. Create a local environment file from the committed example when you want to set the port explicitly:

```bash
cd dev-ops/DeployReady
cp .env.example .env
```

PowerShell equivalent:

```powershell
cd dev-ops/DeployReady
Copy-Item .env.example .env
```

Start the API with Docker Compose:

```bash
docker compose up --build
```

Compose maps host port `3000` to the container port from `PORT` in `.env`. If `.env` is absent, Compose falls back to `PORT=3000` so a clean checkout can still start locally.

Test the container:

```bash
curl http://localhost:3000/health
```

Build the image directly:

```bash
docker build -t deployready-api:local .
```

Run the image directly:

```bash
docker run --rm -p 3000:3000 -e PORT=3000 deployready-api:local
```

## CI/CD Pipeline

The workflow is defined in `.github/workflows/deploy.yml` and runs on every push to `main`.

Pipeline stages:

1. Test: installs dependencies with `npm ci --prefix app` and runs `npm test --prefix app`.
2. Build: builds the Docker image from `dev-ops/DeployReady`.
3. Push: tags the image with the Git commit SHA and pushes it to GitHub Container Registry.
4. Deploy: connects to EC2 over SSH, pulls the SHA-tagged image, replaces the old container, starts the new one, verifies `/health`, and rolls back to the previous image if the new container cannot start or is unhealthy.

Published image format:

```text
ghcr.io/<github-owner>/<repository>:<commit-sha>
```

The workflow also pushes `latest` for convenience, but deployment uses the immutable commit SHA tag.

### Bonus: Rollback on failed health check

The deployment step captures the image used by the currently running `deployready-api` container before replacing it. After the new container starts, the workflow retries `GET /health` locally on the EC2 instance.

If the new container cannot start or fails health checks, the workflow:

1. Prints the last 100 log lines from the failed container.
2. Stops and removes the failed container.
3. Starts a replacement container from the previous image with the same name, port mapping, restart policy, and `PORT`.
4. Verifies `/health` again.
5. Fails the GitHub Actions run so the bad deployment is visible.

Rollback requires a previous container image to exist locally on the EC2 instance. If there is no previous deployment, the workflow fails clearly instead of hiding the problem.

## Required GitHub Secrets

Configure these in GitHub under `Settings -> Secrets and variables -> Actions`.

| Secret | Purpose |
| --- | --- |
| `EC2_HOST` | Public IP address or DNS name of the EC2 instance. |
| `EC2_USER` | Linux user used for deployment, for example `deploy` or `ubuntu`. |
| `EC2_SSH_KEY` | Private SSH key for the deployment user. |
| `EC2_KNOWN_HOSTS` | Pinned SSH host key line for the EC2 host, generated with `ssh-keyscan -H <host>`. |
| `GHCR_USERNAME` | GitHub username or organization account used by EC2 to pull from GHCR. |
| `GHCR_TOKEN` | GitHub token with the minimum package read permission needed to pull the image from GHCR. |
| `APP_PORT` | Container port used by the Node.js API. Use `3000` for this app. |

The workflow uses the built-in `GITHUB_TOKEN` for publishing to GHCR with `packages: write` permission. No AWS access keys are required for this SSH-based deployment approach.

## Deployment Approach

This solution targets a single AWS EC2 instance running Docker.

At deploy time, GitHub Actions:

1. Authenticates to GHCR on the EC2 instance.
2. Pulls the new image by commit SHA.
3. Stops and removes the existing `deployready-api` container if it exists.
4. Starts a replacement container with:

```bash
docker run -d \
  --name deployready-api \
  --restart unless-stopped \
  -p 80:3000 \
  -e PORT=3000 \
  ghcr.io/<github-owner>/<repository>:<commit-sha>
```

5. Verifies the deployment with:

```bash
curl -fsS http://127.0.0.1/health
```

If the container cannot start or the health check fails, GitHub Actions rolls the EC2 container back to the previous image and marks the workflow as failed.

The EC2 security group should allow HTTP on port `80` from the internet and SSH on port `22` only from your current public IP address.

## Security Decisions

- The Docker container runs as the non-root `node` user.
- The image is based on the lean `node:22-bookworm-slim` image.
- Only production dependencies are installed in the runtime image.
- `.dockerignore` excludes `.env`, private keys, logs, dependency folders, and build output.
- The real `.env`, private SSH keys, `.pem` files, and tokens must never be committed.
- GitHub Actions uses repository secrets for all deployment credentials.
- The workflow pins the EC2 host key through `EC2_KNOWN_HOSTS` instead of disabling SSH host verification.
- GitHub Actions permissions are limited to repository read access and package write access.
- The EC2 deployment user should have only the access required to run Docker.
- SSH must be restricted to your IP address in the EC2 security group.

## Troubleshooting

### `docker compose up --build` cannot find `.env`

Create it from the example:

```bash
cp .env.example .env
```

### Port `3000` is already in use locally

Stop the local process using port `3000`, or change the host-side port in `docker-compose.yml` for local testing only.

### `npm ci` fails

Make sure `app/package-lock.json` is committed and run the command from `dev-ops/DeployReady`:

```bash
npm ci --prefix app
```

### EC2 cannot pull from GHCR

Check:

- `GHCR_USERNAME` is correct.
- `GHCR_TOKEN` has package read access.
- The GHCR image visibility allows the token to read it.
- The image name is lowercase and matches the workflow output.

### SSH deployment fails

Check:

- `EC2_HOST`, `EC2_USER`, `EC2_SSH_KEY`, and `EC2_KNOWN_HOSTS` are configured.
- The EC2 security group allows SSH from your current public IP only.
- The deployment user has access to Docker.

### `/health` fails after deployment

Check the container state and logs on EC2:

```bash
docker ps --filter name=deployready-api
docker logs --tail 100 deployready-api
curl -v http://127.0.0.1/health
```

If GitHub Actions attempted a rollback, inspect the workflow logs for the failed container logs and the rollback result.
