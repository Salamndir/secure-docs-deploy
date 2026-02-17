


# Secure Docs — Infrastructure & Deployment

> **Cloud-agnostic deployment template** for a containerized full-stack application.
> Manages orchestration, secrets injection, and CI/CD pipelines separately from application code.

## Strategy
The deployment pipeline follows a **"Build Once, Run Anywhere"** strategy using Docker Hub as the artifact registry.

- **Backend & Frontend Repos:** Strictly responsible for **CI** (Building Docker images and pushing them to the registry). They know nothing about the server.
- **Deployment Repo (This Repo):** Strictly responsible for **CD** (Runtime configuration, Docker Compose orchestration, and Server management).

## Architecture

The system runs on a **Docker Compose** orchestrated environment within an AWS EC2 instance. The entire stack is isolated inside a private bridge network, with Nginx acting as the only entry point.

```mermaid
graph TD
    subgraph CI_CD_Pipeline ["CI/CD Pipeline"]
        Dev[Developer] -->|Push Code| GH[GitHub Actions]
        GH -->|Build & Push| Hub[Docker Hub]
    end

    subgraph Runtime_Environment ["AWS EC2 (t2.micro)"]
        SSH[SSH Connection] -->|Inject Secrets| EnvFile[.env]
        Hub -.->|Pull Optimized Images| Compose[Docker Compose]
        EnvFile --> Compose

        subgraph Docker_Network ["Internal Bridge Network"]
            Compose --> Gateway[Nginx Gateway]
            Gateway -->|Reverse Proxy| Frontend[Angular Container]
            Gateway -->|Reverse Proxy| Backend[Spring Boot Container]
            Gateway -->|Reverse Proxy| Keycloak[Auth Container]
            Backend --> DB[(PostgreSQL)]
            Keycloak --> DB
        end
    end



Design Decisions
1. Why a separate deployment repo?
To achieve true Cloud Agnosticism.

If we decide to switch from AWS EC2 to Oracle Cloud or a VPS tomorrow, we only need to update this repository.

The application source code (Spring Boot / Angular) remains untouched, as it interacts only with Docker, not the underlying server.

2. Why offload builds to GitHub Actions?
While a t2.micro (1GB RAM) could build the project using swap memory, it would be slow and inefficient.
I adopted a Runtime-Only strategy:

GitHub Actions handles the heavy compilation (Maven/NPM) and pushes optimized images to Docker Hub.

The server simply pulls and runs the final artifacts via Docker Compose, ensuring zero-downtime deployments and minimal resource usage.

3. Secrets Handling
Secrets are injected via SSH at deploy time directly into a transient .env file used by Docker Compose. They are never stored in Git or Docker images.

Deployment Workflow
Manual trigger via GitHub Actions → SSH Connection → Inject Secrets → docker compose pull → docker compose up -d

Related Repositories
Backend: secure-docs-backend (https://github.com/Salamndir/secure-docs-backend) (Builds & Pushes Docker Image)

Frontend: secure-docs-frontend (https://github.com/Salamndir/secure-docs-frontend) (Builds & Pushes Docker Image)