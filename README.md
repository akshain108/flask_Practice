# Flask Application -- CI/CD Deployment Pipeline

This repository contains a Flask application, Docker configuration,
automated tests, and a CI/CD pipeline that builds, tests, and deploys
the application to an AWS EC2 instance.

The pipeline is designed to stop immediately when a required stage fails
and to send a success or failure notification email with the relevant
stage information.

## Repository Structure

``` text
.
├── app.py
├── Dockerfile
├── requirements.txt
├── tests/
│   └── test_app.py
├── Jenkinsfile
└── README.md
```

> If this project uses GitHub Actions instead of Jenkins, replace
> `Jenkinsfile` with the appropriate file under `.github/workflows/`.

------------------------------------------------------------------------

## Prerequisites

### 1. AWS Resources

The deployment requires:

-   An AWS account.
-   A running EC2 instance for the application.
-   A VPC/subnet with network connectivity.
-   A security group allowing the required application traffic.
-   Docker installed on the EC2 instance.
-   Git installed on the EC2 instance if the deployment script uses Git.
-   An IAM role for the EC2 instance if AWS APIs are used by the
    application or deployment process.
-   An Amazon ECR repository if the pipeline pushes a Docker image to
    ECR.

Example AWS components:

``` text
GitHub Actions
     |
     | Build + Test
     v
Docker Image
     |
     | Push
     v
Amazon ECR
     |
     | Deploy
     v
EC2 Instance
     |
     v
Flask Application Container
```

### 2. IAM Permissions

Use least-privilege IAM permissions.

If the pipeline pushes images to Amazon ECR, the CI/CD identity needs
permissions such as:
AmazonEC2ContainerRegistryReadOnly


### 3. EC2 Setup

The EC2 instance should have:

1.  A supported Linux distribution.
2.  Docker installed and running.
3.  The EC2 instance registered as an SSM managed node if SSM deployment
    is used.
4.  An IAM instance role with the required permissions.
5.  The application port allowed by the security group if the
    application must be accessed directly.
6.  Sufficient disk space for Docker images and containers.

Verify Docker:

``` bash
docker --version
sudo systemctl status docker
```

------------------------------------------------------------------------

## Pipeline Overview

The pipeline performs the following logical stages:

1.  **Checkout** -- retrieves the source code.
2.  **Install Dependencies** -- installs Python dependencies required
    for testing.
3.  **Test** -- runs the pytest suite.
4.  **Build Docker Image** -- builds the application image.
5.  **Push Image** -- pushes the image to the configured registry, if
    applicable.
6.  **Deploy to EC2** -- updates the application running on EC2.
7.  **Post/Notification** -- sends a success email when all stages pass
    or a failure email when a stage fails.

A failure in the test or build stages prevents the deployment stage from
running.

------------------------------------------------------------------------

## Required Pipeline Secrets

Never store credentials directly in workflow YAML, or
application source code.

### GitHub Actions

Repository uses GitHub Actions, configure secrets under:

**Repository → Settings → Secrets and variables → Actions**

Typical secrets are:

```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION
ECR_REPOSITORY
EC2_HOST
EC2_USER
EC2_SSH_PRIVATE_KEY
SMTP_USERNAME
SMTP_PASSWORD
```

Only create the secrets that are actually required by the selected
deployment method.
------------------------------------------------------------------------

## EC2 Deployment Method


The pipeline will connect to:

``` text
EC2_PUBLIC_IP:22
```

using an SSH private key stored securely in GitHub Actions
secrets.

The EC2 security group must allow SSH from the CI/CD runner's IP range.

Do **not** place the private key inside the Git repository.

------------------------------------------------------------------------

## Manual Deployment

If the CI/CD pipeline is unavailable, the deployment can be reproduced
manually.

### Option 1 -- Deploy an Existing Docker Image

Log in to the EC2 instance using SSM or SSH.

``` bash
docker pull <registry>/<repository>:<tag>
```

Stop the existing container:

``` bash
docker stop flask-app || true
docker rm flask-app || true
```

Start the new version:

``` bash
docker run -d \
  --name flask-app \
  -p 5000:5000 \
  <registry>/<repository>:<tag>
```

Check the container:

``` bash
docker ps
docker logs flask-app
```

Verify the application:

``` bash
curl http://localhost:5000/
```

### Option 2 -- Build Directly on EC2

If no container image is available:

``` bash
git clone <repository-url>
cd <repository-directory>
```

Build the image:

``` bash
docker build -t flask-app:manual .
```

Run the application:

``` bash
docker stop flask-app || true
docker rm flask-app || true

docker run -d \
  --name flask-app \
  -p 5000:5000 \
  flask-app:manual
```

Verify:

``` bash
docker ps
docker logs flask-app
curl http://localhost:5000/
```

------------------------------------------------------------------------

## Running Tests Manually

Install dependencies:

``` bash
pip install -r requirements.txt
```

Run the test suite:

``` bash
pytest -v
```

A deployment should not be performed if the test suite fails.

------------------------------------------------------------------------

## Building the Docker Image Manually

Build:

``` bash
docker build -t flask-app:latest .
```

Run locally:

``` bash
docker run --rm -p 5000:5000 flask-app:latest
```

Test the application:

``` bash
curl http://localhost:5000/
```

------------------------------------------------------------------------

## Failure Handling

The pipeline is intentionally configured so that a failed stage prevents
later deployment stages from executing.

For example:

``` text
Checkout       ✓
Install        ✓
Test           ✗
Build          SKIPPED
Push           SKIPPED
Deploy         SKIPPED
Failure Email  ✓
```

This prevents an application with failing tests from being deployed to
EC2.

The failure notification should identify:

-   Pipeline/job name
-   Build number
-   Failed stage
-   Failure reason
-   Link to the build logs

------------------------------------------------------------------------

## Successful Deployment

A successful pipeline should look similar to:

``` text
Checkout       ✓
Install        ✓
Test           ✓
Build          ✓
Push           ✓
Deploy         ✓
Success Email  ✓
```

The deployment stage should verify that the new container is running
before reporting success.

------------------------------------------------------------------------

## Required Evidence / Screenshots

For the project submission, capture the following evidence.

### Added Essential Snapshots in /SnapShots
