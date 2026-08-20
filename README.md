# Student Registration App — CI/CD Pipeline

A Python Flask + MongoDB student registration application, containerized with Docker and deployed through a fully automated CI/CD pipeline built on **GitHub Actions**, **Amazon ECR**, **Amazon EC2**, and **Amazon SES**.

Every push to `main` is automatically tested, containerized, pushed to a private registry, deployed to a live server, health-checked, and reported on by email — success or failure — with zero manual steps.

---

## Table of Contents

- [1. Project Overview](#1-project-overview)
- [2. Architecture](#2-architecture)
- [3. Tech Stack](#3-tech-stack)
- [4. Project Structure](#4-project-structure)
- [5. Application Details](#5-application-details)
- [6. Prerequisites](#6-prerequisites)
- [7. Local Development Setup](#7-local-development-setup)
- [8. Running with Docker Compose](#8-running-with-docker-compose)
- [9. Testing](#9-testing)
- [10. AWS Setup](#10-aws-setup)
- [11. CI/CD Pipeline — Step by Step](#11-cicd-pipeline--step-by-step)
- [12. Deployment to EC2](#12-deployment-to-ec2)
- [13. Email Notifications (Amazon SES)](#13-email-notifications-amazon-ses)
- [14. Required Secrets](#14-required-secrets)
- [15. Manual Deployment (Pipeline Unavailable)](#15-manual-deployment-pipeline-unavailable)
- [16. Troubleshooting Log](#16-troubleshooting-log)
- [17. Security Notes](#17-security-notes)
- [18. Production Improvements](#18-production-improvements)
- [19. Screenshots](#19-screenshots)

---

## 1. Project Overview

This project implements the core DevOps skill set of a modern CI/CD workflow: source control, automated testing, containerization, a container registry, a real compute deployment target, and operational feedback via email.

The application itself is intentionally simple — a student registration system — so the focus stays on the pipeline mechanics:

- View, add, update, and delete student records
- A `/health` endpoint used as the deployment verification gate
- MongoDB-backed persistence via Flask-PyMongo

The pipeline that ships this application does the following on every push to `main`:

1. Runs the automated test suite
2. Builds a Docker image tagged with the Git commit SHA
3. Pushes that image to Amazon ECR
4. Deploys it to an EC2 instance by replacing the running container
5. Verifies the new deployment with a health check
6. Emails a success or failure report via Amazon SES

---

## 2. Architecture

### 2.1 CI/CD Pipeline Flow

```
Developer
    |
    | git push (main)
    v
GitHub Repository
    |
    | workflow trigger
    v
GitHub Actions Runner (Ubuntu 24.04)
    |
    +----------------------+
    |                      |
    v                      v
Install Dependencies   Start MongoDB Service
    |                      |
    +----------+-----------+
               |
               v
          Run Pytest (5 tests)
               |
        Tests Passed? ------ No ----> STOP (pipeline fails, failure email sent)
               |
              Yes
               |
               v
     Build Docker Image
     (tagged: <commit-sha> and latest)
               |
               v
     Authenticate to Amazon ECR
               |
               v
     Push Image to Amazon ECR
               |
               v
     SSH into EC2 Instance
               |
               v
     Authenticate EC2 Docker with ECR
               |
               v
     Verify MongoDB container is running  ---- No ----> STOP (protects DB, failure email)
               |
              Yes
               v
     Verify Docker network exists  ---- No ----> STOP (failure email)
               |
              Yes
               v
     Stop & Remove Old App Container
               |
               v
     Pull & Start New App Container
               |
               v
     /health Verification (up to 10 retries)
               |
        +------+------+
        |             |
      Pass          Fail
        |             |
        v             v
     Success        Failure
        |             |
        +------+------+
               |
               v
     Amazon SES Email Notification
```

### 2.2 Runtime Architecture (on EC2)

```
                    Internet
                       |
                       v
          EC2 Security Group (ports 5000, 27017)
                       |
                       v
        +-----------------------------------+
        |     Docker Bridge Network          |
        |     student-app-network            |
        |                                     |
        |   +-------------------------+       |
        |   | Flask Application       |       |
        |   | student-registration-app|       |
        |   | Port: 5000              |       |
        |   +------------+------------+       |
        |                |                     |
        |                | mongodb:27017       |
        |                v                     |
        |   +-------------------------+       |
        |   | MongoDB Container        |       |
        |   | Image: mongo:7           |       |
        |   | Port: 27017              |       |
        |   +-------------------------+       |
        |                                     |
        +-----------------------------------+
                       |
                       v
                  student_db.students
```

Instead of connecting over `localhost` or `host.docker.internal`, the Flask container talks to MongoDB using Docker's built-in DNS and the **container name**:

```
mongodb://mongodb:27017/student_db
```

Both containers are attached to the same user-defined bridge network (`student-app-network`), which is what makes name-based resolution possible.

---

## 3. Tech Stack

| Category                   | Technology                     |
|------------------------------|----------------------------------|
| Language                       | Python 3.14                     |
| Web Framework                    | Flask 3.1                       |
| Database                          | MongoDB 7 (via Flask-PyMongo)   |
| Testing                             | Pytest                          |
| Containerization                      | Docker, Docker Compose          |
| CI/CD                                   | GitHub Actions                  |
| Container Registry                        | Amazon ECR                     |
| Compute (Deploy Target)                     | Amazon EC2 (Ubuntu)           |
| Access Control                                | AWS IAM                        |
| Email Notifications                             | Amazon SES                    |

---

## 4. Project Structure

```
Student-Registration-App/
├── .github/
│   └── workflows/
│       └── ci_local.yml       # CI/CD pipeline definition
├── app.py                     # Flask application entry point
├── test_app.py                 # Pytest test suite
├── requirements.txt            # Python dependencies
├── Dockerfile                  # Application container definition
├── docker-compose.yml          # Local multi-container environment
├── .dockerignore                # Files excluded from the Docker build context
├── .gitignore
├── templates/                  # HTML templates
├── Images/                      # Pipeline/deployment evidence screenshots
└── README.md
```

---

## 5. Application Details

| Item                     | Value |
|----------------------------|-------|
| Framework                    | Flask |
| Database driver               | Flask-PyMongo / PyMongo |
| Health endpoint                 | `GET /health` → `{"status": "healthy"}` |
| Application port                  | `5000` |
| Database                            | `student_db` |
| Collection                            | `students` |

The `/health` endpoint exists specifically so the deployment step can confirm the container **actually started and is serving traffic** — not just that `docker run` exited without error. A container that starts and then immediately crashes is still reported as a failed deployment because the health check will fail.

---

## 6. Prerequisites

Before the pipeline can run, the following must be set up **manually, once**:

1. **An Amazon ECR repository** to store built Docker images.
2. **An EC2 instance** (Ubuntu) with:
   - Docker installed and running
   - A Docker bridge network created for the app and database to communicate
   - A security group allowing inbound traffic on port `5000` (application) and, if using SSH-based deploys, port `22`
3. **AWS IAM credentials** with permission to push/pull from ECR and (if used) assume any deployment role.
4. **A verified Amazon SES sender identity** (and recipient identity, while the SES account is in sandbox mode) for email notifications.
5. **GitHub repository secrets** configured with all the values in [Section 14](#14-required-secrets).

---

## 7. Local Development Setup

**Step 1 — Clone the repository**
```bash
git clone https://github.com/<your-username>/Student-Registration-App-CICD-GitHub-Actions.git
cd Student-Registration-App-CICD-GitHub-Actions
```

**Step 2 — Create and activate a virtual environment**
```bash
python3 -m venv venv
source venv/bin/activate
```
Keeps project dependencies isolated from the system Python installation.

**Step 3 — Install dependencies**
```bash
pip install -r requirements.txt
```
Installs Flask, Flask-PyMongo, PyMongo, python-dotenv, certifi, and pytest — the minimal runtime and testing dependencies (deliberately trimmed down from a full `pip freeze`, which would otherwise pull in unrelated dev tools).

**Step 4 — Create a `.env` file**
```env
MONGO_URI=mongodb://localhost:27017/student_db
SECRET_KEY=mysecretkey
```
`.env` is excluded from Git (`.gitignore`) and from the Docker build context (`.dockerignore`) so secrets never leave the local machine.

**Step 5 — Run the application**
```bash
python app.py
```
Starts the Flask development server on port `5000`.

---

## 8. Running with Docker Compose

Running the app and MongoDB together locally, exactly as they run on EC2, is done through Docker Compose.

**Step 1 — Build and start both services**
```bash
docker compose up -d
```
This builds the Flask image from the `Dockerfile` and pulls the MongoDB image, then starts both containers.

**Step 2 — Confirm both containers are running**
```bash
docker ps
```
Expect to see `student-registration-app` and `student-mongodb`, both `Up`.

**Step 3 — Verify the health endpoint**
```bash
curl http://localhost:5001/health
# {"status": "healthy"}
```

**Step 4 — Verify data is being persisted**
```bash
docker exec -it student-mongodb mongosh student_db --eval 'db.students.find().pretty()'
```
Confirms the Flask container can actually write to and read from MongoDB, not just serve the UI.

**Step 5 — Tear down**
```bash
docker compose down
```
Stops and removes the containers and network while leaving the built images intact.

| Service                     | Image           | Host Port | Container Port |
|-------------------------------|-----------------|-----------|------------------|
| `student-registration-app`      | built locally   | 5001      | 5000             |
| `mongodb`                        | `mongo:8`       | 27018     | 27017            |

> Non-standard host ports (5001, 27018) were chosen because ports 5000 and 27017 were already occupied by other local processes — this is purely a host-side mapping and does not affect container-to-container communication.

---

## 9. Testing

**Run the suite locally:**
```bash
pytest -v
```

| Test                    | What it verifies                     |
|---------------------------|----------------------------------------|
| `test_home_page`          | The home page renders successfully     |
| `test_add_student`         | A student record can be created        |
| `test_update_student`      | A student record can be updated        |
| `test_delete_student`      | A student record can be deleted        |
| `test_health`               | `/health` returns a healthy status     |

These same five tests run inside GitHub Actions against a MongoDB **service container**, so a pass in CI genuinely proves the code works outside of the developer's local machine — not just on the developer's own laptop with its own local MongoDB setup.

**Why tests gate the build:** if `pytest -v` exits with a non-zero status, GitHub Actions marks the step as failed and the workflow stops before Docker image build/push/deploy ever run. This prevents broken code from ever reaching ECR or EC2.

---

## 10. AWS Setup

### 10.1 ECR Repository

**Step 1 — Verify the configured AWS region**
```bash
aws configure get region
```
All resources in this project live in `ap-south-1` (Mumbai). Using a mismatched region is a common cause of "repository not found" errors — always confirm this first.

**Step 2 — Check if the repository already exists**
```bash
aws ecr describe-repositories --repository-names student-registration-app --region ap-south-1
```

**Step 3 — Create the repository (if it doesn't exist)**
```bash
aws ecr create-repository --repository-name student-registration-app --region ap-south-1
```
Produces a repository URI in the form:
```
<account-id>.dkr.ecr.ap-south-1.amazonaws.com/student-registration-app
```

### 10.2 EC2 Instance

**Step 1 — Verify Docker is installed and running**
```bash
docker --version
sudo systemctl status docker
```

**Step 2 — Create a dedicated Docker network**
```bash
docker network create student-app-network
```
This lets the application and MongoDB containers discover each other by name instead of relying on IP addresses, which can change whenever a container restarts.

**Step 3 — Start the MongoDB container**
```bash
docker run -d \
  --name mongodb \
  --network student-app-network \
  -p 27017:27017 \
  mongo:7
```

**Step 4 — Configure the EC2 Security Group**

| Type       | Port | Source              | Purpose                     |
|------------|------|----------------------|-------------------------------|
| Custom TCP | 5000 | Your IP / 0.0.0.0/0*  | Application access             |
| Custom TCP | 22   | Your IP               | SSH-based deployment           |
| Custom TCP | 27017 | Your IP only (never 0.0.0.0/0) | Optional direct MongoDB access for debugging |

\* `0.0.0.0/0` is acceptable only for temporary demo/testing purposes — see [Security Notes](#17-security-notes).

**Step 5 — Attach an IAM role to the instance (or configure credentials) with ECR pull permissions**, e.g. `AmazonEC2ContainerRegistryReadOnly` or a scoped equivalent, so the instance can authenticate to ECR without long-lived static keys stored on the box.

---

## 11. CI/CD Pipeline — Step by Step

The full pipeline is defined in [`.github/workflows/ci_local.yml`](.github/workflows/ci_local.yml) and runs on every push to `main` (plus manual `workflow_dispatch` runs).

| # | Stage                          | What happens                                                                                          |
|---|-----------------------------------|--------------------------------------------------------------------------------------------------------|
| 1 | **Checkout**                      | `actions/checkout@v4` pulls the exact commit that triggered the workflow onto the runner.               |
| 2 | **Set up Python**                  | `actions/setup-python@v5` installs Python 3.14 on the Ubuntu 24.04 runner for a reproducible environment. |
| 3 | **Configure AWS credentials**       | `aws-actions/configure-aws-credentials@v4` authenticates the runner using the `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` secrets, scoped to `ap-south-1`. |
| 4 | **Install dependencies**             | `pip install -r requirements.txt` installs the exact runtime/test dependency set used by the application. |
| 5 | **Start MongoDB service**            | GitHub Actions spins up a MongoDB 7 service container so the test suite has a real database to run against. |
| 6 | **Run tests**                         | `pytest -v` runs all five tests. **If any test fails, the workflow stops here** — no image is ever built from unvalidated code. |
| 7 | **Build Docker image**                 | Builds the image from the `Dockerfile`, tagged with both the Git commit SHA and `latest`, so every deployed artifact is traceable to a specific commit. |
| 8 | **Login to Amazon ECR**                 | `aws ecr get-login-password` authenticates Docker with the private ECR registry. |
| 9 | **Tag image for ECR**                     | Re-tags the local image with the full ECR repository URI. |
| 10 | **Push image to ECR**                      | Pushes both the commit-SHA-tagged and `latest`-tagged images to the registry. |
| 11 | **Prepare SSH configuration**               | Writes the EC2 private key (from the `EC2_SSH_KEY` secret) to disk with restricted permissions (`chmod 600`) and adds the EC2 host to `known_hosts` via `ssh-keyscan`. |
| 12 | **Generate ECR login password for EC2**       | Produces a fresh ECR auth token for the remote Docker daemon to use. |
| 13 | **Login to ECR from EC2**                       | Pipes the token over SSH so the EC2 Docker daemon can authenticate to the private registry. |
| 14 | **Deploy to EC2**                                | Runs a remote script (see below) that pulls the new image and replaces the running container. |
| 15 | **Verify deployment**                              | Runs `docker ps` on EC2 one more time to confirm the final container state before the job ends. |
| 16 | **Send success/failure email**                      | Amazon SES sends a customized email depending on whether the job succeeded (`if: success()`) or failed (`if: failure()`). |

### 11.1 What the Remote Deployment Script Does

Executed over SSH on the EC2 instance as part of Stage 14:

1. **Pull the new image** — `docker pull <ecr-image>:<commit-sha>` retrieves the exact image built in Stage 7.
2. **Verify MongoDB is running** — checks `docker ps` for a container named `mongodb`; if it's missing, the script exits immediately **without touching the application container**, protecting the database from an unsafe deployment.
3. **Verify the Docker network exists** — checks `docker network inspect student-app-network`; if missing, the deployment stops.
4. **Remove the old application container** — `docker rm -f student-registration-app` (a no-op, safely ignored, if no container exists yet).
5. **Start the new container** — runs the freshly pulled image on the `student-app-network`, mapping port 5000 and passing `MONGO_URI` and `SECRET_KEY` as environment variables.
6. **Wait for startup** — a short `sleep 10` gives Flask time to initialize before health-checking it.
7. **Health-check loop** — calls `curl -fsS http://localhost:5000/health` up to 10 times, 3 seconds apart. The first successful response marks the deployment successful; if all 10 attempts fail, the script prints the last 100 lines of application logs and exits with a failure code.

This health-check loop is the actual **deploy-verification gate** — a container that starts but crashes on its first request is caught here and reported as a failed deployment, not a successful one.

---

## 12. Deployment to EC2

**Connection method: SSH**

The pipeline connects to EC2 over SSH using a private key stored in the `EC2_SSH_KEY` GitHub secret, rather than an SSM-based approach. SSH was chosen because:

- It requires no additional AWS-side session management setup beyond a security group rule.
- The deployment logic (pull, safety checks, replace container, health-check) is simple enough to run as a single remote bash script over a standard SSH session.
- It keeps the pipeline portable — the same deploy script can be run manually from a developer's machine if the pipeline is ever unavailable (see [Section 15](#15-manual-deployment-pipeline-unavailable)).

**Why the container is replaced, not reinstalled from scratch:** the deployment script always removes only the `student-registration-app` container by name and leaves `mongodb` and the Docker network untouched, so each deploy is a true in-place replacement of the application layer rather than a fresh environment rebuild.

---

## 13. Email Notifications (Amazon SES)

Amazon SES was chosen over Gmail/SMTP because it integrates directly with the AWS credentials already configured in the pipeline — no separate email password or app password needs to be generated or stored.

### 13.1 One-Time SES Setup

**Step 1 — Verify a sender identity**
AWS Console → Amazon SES → Verified identities → Create identity → enter an email address → click the verification link sent to that inbox.

**Step 2 — (While in sandbox mode) verify the recipient identity too**
SES sandbox accounts can only send to verified addresses; request production access to lift this restriction.

**Step 3 — Store the SES configuration as GitHub secrets**
`SES_FROM_EMAIL`, `SES_TO_EMAIL`, `SES_REGION` — see [Section 14](#14-required-secrets).

### 13.2 Success Email

Triggered by `if: success()`. Includes:

- Clear success indicator in the subject line
- Application name and environment
- Git commit SHA and branch deployed
- Docker image tag pushed to ECR

### 13.3 Failure Email

Triggered by `if: failure()`. Includes:

- Clear failure indicator in the subject line
- Git commit SHA and branch
- The GitHub Actions **workflow name and run ID**, so the exact failed run can be opened directly for log investigation

> Note: the email content does not currently name *which specific stage* failed (test / build / push / deploy) — only that the run failed, along with the run ID needed to look it up. If your grading criteria require the failed stage to be named explicitly in the email body, that can be added by passing `${{ job.status }}` / step outcome context into the SES message.

### 13.4 Validating the Failure Path

The failure email was deliberately tested by temporarily appending `exit 1` after the `pytest -v` step, forcing the workflow into a failed state, confirming the failure email arrived correctly, then removing the line and restoring the normal pipeline.

---

## 14. Required Secrets

All sensitive values are stored in **GitHub Repository → Settings → Secrets and variables → Actions**, never committed to the repository or hardcoded in the workflow file.

| Secret                  | Purpose                                          |
|---------------------------|-----------------------------------------------------|
| `AWS_ACCESS_KEY_ID`         | Authenticates the runner with AWS                    |
| `AWS_SECRET_ACCESS_KEY`      | Authenticates the runner with AWS                    |
| `EC2_SSH_KEY`                 | Private key used to SSH into the EC2 instance         |
| `EC2_HOST`                     | EC2 public IP or hostname                             |
| `EC2_USER`                       | SSH username for the EC2 instance                     |
| `SES_FROM_EMAIL`                   | Verified SES sender address                         |
| `SES_TO_EMAIL`                       | Notification recipient address                       |
| `SES_REGION`                           | AWS region where the SES identity is verified       |

---

## 15. Manual Deployment (Pipeline Unavailable)

If GitHub Actions is unavailable, the same deployment can be reproduced by hand:

**Step 1 — Authenticate Docker with ECR**
```bash
aws ecr get-login-password --region ap-south-1 | \
docker login --username AWS --password-stdin \
<account-id>.dkr.ecr.ap-south-1.amazonaws.com
```

**Step 2 — Pull the desired image**
```bash
docker pull <account-id>.dkr.ecr.ap-south-1.amazonaws.com/student-registration-app:<commit-sha-or-latest>
```

**Step 3 — Confirm MongoDB and the network are healthy**
```bash
docker ps --format '{{.Names}}' | grep -q '^mongodb$'
docker network inspect student-app-network
```

**Step 4 — Replace the running container**
```bash
docker rm -f student-registration-app 2>/dev/null || true

docker run -d \
  --name student-registration-app \
  --network student-app-network \
  -p 5000:5000 \
  -e MONGO_URI=mongodb://mongodb:27017/student_db \
  -e SECRET_KEY=<secret-key> \
  <account-id>.dkr.ecr.ap-south-1.amazonaws.com/student-registration-app:<tag>
```

**Step 5 — Verify the health endpoint**
```bash
curl -fsS http://localhost:5000/health
```

**Step 6 — Check logs if anything looks wrong**
```bash
docker logs --tail 100 student-registration-app
```

---

## 16. Troubleshooting Log

Real issues hit during development, kept here for reference since they're likely to recur:

| Issue                                       | Cause                                              | Resolution |
|-----------------------------------------------|-------------------------------------------------------|--------------|
| Port `5000`/`27017` already in use locally       | Another local process (e.g. macOS ControlCenter, local MongoDB) already bound the port | Mapped to alternate host ports (`5001`, `27018`) in Docker Compose |
| `ValueError: You must specify a URI or set the MONGO_URI Flask config variable` | `.env` isn't copied into the container automatically | Pass `MONGO_URI` explicitly via `-e` flag or Compose `environment:` block |
| MongoDB container repeatedly exits with code `139` (segfault) on EC2 | `mongo:8` was unstable on a small (~1 GB RAM) EC2 instance | Switched to `mongo:7`, which ran stably |
| `RepositoryNotFoundException` from `aws ecr describe-repositories` | Wrong repository name or wrong AWS region configured | Confirmed the correct name (`student-registration-app`) and region (`ap-south-1`) |
| Docker container name conflict on `docker compose up` | An old manually-created container with the same name already existed | `docker rm <container>` before restarting Compose |
| Application can't reach MongoDB using `host.docker.internal` | Both containers are on the same Docker network — should resolve by container name, not host-internal DNS | Use `mongodb://mongodb:27017/student_db` instead |

---

## 17. Security Notes

- `.env` is excluded from both Git (`.gitignore`) and the Docker build context (`.dockerignore`).
- No AWS, SSH, or SMTP credentials are hardcoded anywhere in the repository or workflow file — everything sensitive lives in GitHub Actions Secrets.
- MongoDB should not be exposed to `0.0.0.0/0` in the EC2 security group; where direct external access is needed for tools like MongoDB Compass, restrict the source to a specific IP or use an SSH tunnel instead.
- Opening application port `5000` to `0.0.0.0/0` is acceptable only for temporary demo/testing purposes, not for a production deployment.

---

## 18. Production Improvements

The current setup is appropriate for a learning/assignment environment. For production use, the following are recommended:

- Serve the application with Gunicorn instead of the Flask development server, and disable debug mode.
- Enable HTTPS in front of the application.
- Enable MongoDB authentication and TLS/SSL.
- Move MongoDB off the public-facing instance entirely — private networking or a managed service such as Amazon DocumentDB.
- Use AWS Secrets Manager or Parameter Store instead of plain environment variables for runtime secrets.
- Add persistent Docker volumes for MongoDB data so records survive container recreation.
- Add monitoring/alerting (e.g. CloudWatch) and automatic container restart policies.
- Replace broad IAM permissions with least-privilege, scoped policies.
- Put the application behind a load balancer / Elastic IP so the public address doesn't change on instance restart.

---

## 19. Screenshots

Evidence of the working pipeline and deployment is kept in [`Images/`](./Images), including:

- Passing Pytest suite (local and CI runs)
- Docker health checks and container logs
- ECR repository creation and successful image push/pull
- EC2 instance, IAM role, and IAM policy configuration
- Docker image pulled and running on EC2
- Application UI and database record verification, both locally and on EC2
- Full pipeline run deploying to EC2 via GitHub Actions
- Docker network connecting the application to MongoDB
- Successful `/health` check after pipeline deployment
- Success and failure email notifications delivered via Amazon SES


```Images```

### 1. Pytest Cases Passed

![Pytest Cases Passed](Images/1.PytestCassesPassed.png)

### 2. Docker Health Check and Logs – Local

![Docker Health Check and Logs – Local](Images/2.DockerHealthCheck&Log_local.png)

### 3. Application UI – Local

![Application UI – Local](Images/3.AppUi_OnLoca.png)

### 4. Database Created – Local

![Database Created – Local](Images/4.DBCreated-OnLocal.png)

### 5. Data Insertion Details – Local

![Data Insertion Details – Local](Images/5.InsertionDetails-OnLoca.png)

### 6. Data UI Details – Local

![Data UI Details – Local](Images/6.DataUIDetails-OnLocal.png)

### 7. Student Details Deleted

![Student Details Deleted](Images/7.DetailsDeleted.png)

### 8. Docker Logs

![Docker Logs](Images/8.DockerLogs.png)

### 9. ECR Created From Pipeline

![ECR Created From Pipeline](Images/9.ECRCreated_FromPipeline.png)

### 10. ECR Pull Success

![ECR Pull Success](Images/10.ECRPullSuccess.png)

### 11. Test After Pulling Image From ECR

![Test After Pulling Image From ECR](Images/11.TestAfterPullingImageFromECR.png)

### 12. Docker Logs After Pulling Image From ECR

![Docker Logs After Pulling Image From ECR](Images/12.DockerLogTestAfterPullingImageFromECR.png)

### 13. EC2 Instance Created

![EC2 Instance Created](Images/13.EC2InstanceCreated.png)

### 14. IAM Role Attached

![IAM Role Attached](Images/14.IAMRoleAttached.png)

### 15. IAM Policy Attached

![IAM Policy Attached](Images/15.IAMPolicyAttached.png)

### 16. Docker Image Pulled From ECR On EC2

![Docker Image Pulled From ECR On EC2](Images/16.Pulled-DockerImageFrom-ECR-On-EC2.png)

### 17. Testing The App Locally From EC2

![Testing The App Locally From EC2](Images/17.TestingTheAppLocallyFromEC2.png)

### 18. App Launch UI From EC2

![App Launch UI From EC2](Images/18.AppLaunchUIFromEC2.png)

### 19. Student Details Added On EC2 App

![Student Details Added On EC2 App](Images/19.StudentDetailsAddedOnEc2App.png)

### 20. Student Details Added In DB From EC2

![Student Details Added In DB From EC2](Images/20.StudentDetailsAddedInDbfromEc2.png)

### 21. Image Pushed To ECR Through Pipeline

![Image Pushed To ECR Through Pipeline](Images/21.ImagePushedToECRThroughPipeline.png)

### 22. App Deployed On EC2 Through Pipeline

![App Deployed On EC2 Through Pipeline](Images/22.AppDeployedOnEc2ThroughPipeline.png)

### 23. Docker Network Created To Connect With MongoDB

![Docker Network Created To Connect With MongoDB](Images/23.CreatedDockerNetworkToConnecWithMongoDB.png)

### 24. Health Check Is Fine After Pipeline Deployment

![Health Check Is Fine After Pipeline Deployment](Images/24.HealthCheckIsFineAfterPipelineDeployment.png)

### 25. Deployment Successful Email Through Amazon SES

![Deployment Successful Email Through Amazon SES](Images/25.DeploymentSuccessFullEmailThroughAmazonSES.png)

### 26. Deployment Failure Email Through Amazon SES

![Deployment Failure Email Through Amazon SES](Images/26.DeploymentFailureEmailThroughAmazonSES.png)
