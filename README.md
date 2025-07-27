# GitLab CI/CD Setup for Python Flask App with Docker Swarm and Kubernetes Deployments

This guide explains how to set up a GitLab CI/CD pipeline on a Debian-based Linux system to build, test, and deploy a Python Flask application to Docker Swarm (staging) and Kubernetes (production). While this guide focuses on GitLab CI/CD, the setup can be adapted for GitHub Actions with modifications.

## Table of Contents
- [Prerequisites](#prerequisites)
- [System Setup](#system-setup)
- [Install Dependencies](#install-dependencies)
- [Install GitLab Runner](#install-gitlab-runner)
- [Register GitLab Runner](#register-gitlab-runner)
- [Configure GitLab Runner](#configure-gitlab-runner)
- [Set Up Project Repository](#set-up-project-repository)
- [Configure CI/CD Variables](#configure-cicd-variables)
- [Set Up Deployment Environments](#set-up-deployment-environments)
- [Test the Pipeline](#test-the-pipeline)
- [Troubleshooting](#troubleshooting)
- [Optional Enhancements](#optional-enhancements)

## Prerequisites
- **System**: Debian-based Linux (e.g., Debian 11 or 12) with sudo access.
- **GitLab Account**: A project repository on GitLab (e.g., gitlab.com or self-hosted).
- **Application**: A Python Flask app with `requirements.txt` and tests (e.g., using `pytest`).
- **Deployment Targets**: Access to a Docker Swarm cluster and a Kubernetes cluster (or local setups like Minikube).
- **GitHub Repository**: Optionally, mirror this repo to GitHub for documentation or hybrid CI/CD workflows.

## System Setup
Update your Debian system to ensure compatibility.

```bash
sudo apt update && sudo apt upgrade -y
```

## Install Dependencies
Install Python, pip, Docker, and kubectl.

```bash
# Install Python and pip
sudo apt install -y python3 python3-pip python3-venv

# Install Docker
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

Verify installations:
```bash
python3 --version
pip3 --version
docker --version
kubectl version --client
```

## Install GitLab Runner
Add and install the GitLab Runner.

```bash
# Add GitLab Runner repository
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash

# Install GitLab Runner
sudo apt install -y gitlab-runner

# Verify
gitlab-runner --version
```

## Register GitLab Runner
Register the runner with your GitLab project using the Docker executor.

1. Get the registration token from **Settings > CI/CD > Runners** in your GitLab project.
2. Register the runner:
   ```bash
   sudo gitlab-runner register
   ```
   - **URL**: e.g., `https://gitlab.com/`.
   - **Token**: Paste the project’s token (e.g., `glrt-XXXXXXXXXXXXXXXXXXXX`).
   - **Description**: e.g., `debian-docker-runner`.
   - **Tags**: e.g., `debian,python,docker`.
   - **Executor**: `docker`.
   - **Default Docker image**: `python:3.9`.

3. Verify the runner appears online in GitLab.

## Configure GitLab Runner
Ensure the runner uses the Docker executor.

1. Edit the configuration:
   ```bash
   sudo nano /etc/gitlab-runner/config.toml
   ```

2. Example configuration:
   ```toml
   [[runners]]
     name = "debian-docker-runner"
     url = "https://gitlab.com/"
     token = "YOUR_RUNNER_TOKEN"
     executor = "docker"
     [runners.docker]
       image = "python:3.9"
       privileged = false
       disable_cache = false
     [runners.cache]
   ```

3. Restart the runner:
   ```bash
   sudo gitlab-runner restart
   ```

## Set Up Project Repository
Configure your GitLab project with the necessary files. These can be mirrored to GitHub for documentation.

1. **`.gitlab-ci.yml`**:
   ```yaml
   image: python:3.9

   cache:
     paths:
       - .venv/

   stages:
     - build
     - test
     - deploy

   build_job:
     stage: build
     script:
       - python3 -m venv .venv
       - . .venv/bin/activate
       - pip install -r requirements.txt
     artifacts:
       paths:
         - .venv/

   test_job:
     stage: test
     script:
       - . .venv/bin/activate
       - pytest
     dependencies:
       - build_job

   build_docker_image:
     stage: deploy
     image: docker:20.10
     services:
       - docker:dind
     script:
       - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
       - docker build -t $CI_REGISTRY_IMAGE:latest .
       - docker push $CI_REGISTRY_IMAGE:latest
     only:
       - main
     dependencies:
       - build_job

   deploy_swarm:
     stage: deploy
     image: docker:20.10
     services:
       - docker:dind
     script:
       - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
       - docker pull $CI_REGISTRY_IMAGE:latest
       - ssh $SWARM_HOST "docker service update --image $CI_REGISTRY_IMAGE:latest flask_app || docker service create --name flask_app -p 5000:5000 $CI_REGISTRY_IMAGE:latest"
     environment:
       name: staging
       url: http://swarm.example.com:5000
     only:
       - main
     dependencies:
       - build_docker_image

   deploy_kubernetes:
     stage: deploy
     image: bitnami/kubectl:latest
     script:
       - kubectl config use-context $KUBE_CONTEXT
       - kubectl apply -f k8s-deployment.yml
       - kubectl set image deployment/flask-app flask-app=$CI_REGISTRY_IMAGE:latest
       - kubectl rollout status deployment/flask-app
     environment:
       name: production
       url: http://k8s.example.com
     only:
       - main
     when: manual
     dependencies:
       - build_docker_image
   ```

2. **`Dockerfile`**:
   ```dockerfile
   FROM python:3.9-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY . .
   EXPOSE 5000
   CMD ["python", "app.py"]
   ```

3. **`k8s-deployment.yml`**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: flask-app
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: flask-app
     template:
       metadata:
         labels:
           app: flask-app
       spec:
         containers:
         - name: flask-app
           image: $CI_REGISTRY_IMAGE:latest
           ports:
           - containerPort: 5000
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: flask-app-service
   spec:
     selector:
       app: flask-app
     ports:
     - protocol: TCP
       port: 80
       targetPort: 5000
     type: LoadBalancer
   ```

4. **`requirements.txt`**:
   ```text
   flask==2.3.3
   pytest==7.4.0
   ```

5. **`app.py`**:
   ```python
   from flask import Flask
   app = Flask(__name__)

   @app.route('/')
   def hello():
       return 'Hello from Flask!'

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

6. **`test_app.py`**:
   ```python
   from app import app
   import pytest

   @pytest.fixture
   def client():
       app.config['TESTING'] = True
       with app.test_client() as client:
           yield client

   def test_hello(client):
       response = client.get('/')
       assert response.status_code == 200
       assert b'Hello from Flask!' in response.data
   ```

7. Push to GitLab (and optionally mirror to GitHub):
   ```bash
   git add .gitlab-ci.yml Dockerfile k8s-deployment.yml requirements.txt app.py test_app.py
   git commit -m "Add CI/CD for Flask with Docker Swarm and Kubernetes"
   git push origin main
   ```

## Configure CI/CD Variables
In GitLab, add variables in **Settings > CI/CD > Variables**:
- `CI_REGISTRY_USER`: GitLab username.
- `CI_REGISTRY_PASSWORD`: GitLab personal access token (`api`, `write_repository` scopes).
- `CI_REGISTRY`: e.g., `registry.gitlab.com`.
- `CI_REGISTRY_IMAGE`: e.g., `registry.gitlab.com/your-username/your-project`.
- `SWARM_HOST`: SSH host for Docker Swarm (e.g., `user@swarm-server`).
- `KUBE_CONTEXT`: Kubernetes context name.
- `SWARM_SSH_KEY`: SSH private key for Swarm host.
- `KUBE_CONFIG`: Base64-encoded kubeconfig.

For SSH:
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/gitlab-runner
# Add public key to Swarm host’s ~/.ssh/authorized_keys
```

For Kubernetes:
```bash
cat ~/.kube/config | base64
# Add output to KUBE_CONFIG variable
```

## Set Up Deployment Environments
1. **Docker Swarm**:
   ```bash
   docker swarm init
   ```
   Ensure SSH access for the runner.

2. **Kubernetes**:
   Set up a cluster (e.g., Minikube or cloud provider) and configure `kubectl` with the correct context.

## Test the Pipeline
1. In GitLab, go to **CI/CD > Pipelines**.
2. Verify:
   - `build_job`: Installs dependencies.
   - `test_job`: Runs tests.
   - `build_docker_image`: Builds/pushes Docker image.
   - `deploy_swarm`: Deploys to Swarm (staging).
   - `deploy_kubernetes`: Deploys to Kubernetes (production, manual).

## Troubleshooting
- **Runner offline**: Check `sudo gitlab-runner status` and Docker service.
- **Docker login issues**: Verify `CI_REGISTRY_*` variables.
- **SSH failures**: Check `SWARM_SSH_KEY` and permissions.
- **Kubernetes errors**: Validate `KUBE_CONFIG` and context.
- **Test failures**: Ensure `pytest` setup in `test_app.py`.

## Optional Enhancements
- Add health checks to `k8s-deployment.yml` or Swarm services.
- Configure Kubernetes autoscaling:
  ```yaml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: flask-app-hpa
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: flask-app
    minReplicas: 2
    maxReplicas: 10
    metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  ```
- For GitHub Actions, adapt `.gitlab-ci.yml` to a `.github/workflows/main.yml` file.

## Notes
- This guide uses GitLab CI/CD, but you can mirror the repo to GitHub for documentation or hybrid workflows.
- Replace placeholder URLs (`swarm.example.com`, `k8s.example.com`) with your own actual deployment endpoints.
- Regularly update GitLab Runner and dependencies:
  ```bash
  sudo apt update && sudo apt upgrade gitlab-runner
  ```
