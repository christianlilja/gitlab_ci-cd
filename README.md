Guide to Setting Up GitLab CI/CD for a Python Flask App with Docker Swarm and Kubernetes Deployments on Debian

This guide explains how to set up a GitLab CI/CD pipeline on a Debian-based Linux system to build, test, and deploy a Python Flask application to Docker Swarm and Kubernetes. It includes installing and configuring a GitLab Runner and setting up the necessary tools for Docker and Kubernetes deployments.

Prerequisites





A Debian-based system (e.g., Debian 11 or 12) with sudo access.



A GitLab account and a project repository on GitLab (e.g., gitlab.com or self-hosted).



A Python Flask application with a requirements.txt file and a test suite (e.g., using pytest).



Access to a Docker Swarm cluster and a Kubernetes cluster (or local setups for testing).



The .gitlab-ci.yml file provided below, placed in the root of your GitLab project repository.

Step 1: Update the System

Ensure your Debian system is up to date.

sudo apt update && sudo apt upgrade -y

Step 2: Install Required Dependencies

Install Python, pip, Docker, and kubectl for building and deploying the Flask app.





Install Python and pip:

sudo apt install -y python3 python3-pip python3-venv



Install Docker:

sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER



Install kubectl (for Kubernetes):

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client



Verify installations:

python3 --version
pip3 --version
docker --version
kubectl version --client

Step 3: Install GitLab Runner

Add the GitLab Runner repository and install the runner.





Add the repository:

curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash



Install the runner:

sudo apt install -y gitlab-runner



Verify:

gitlab-runner --version

Step 4: Register the GitLab Runner

Register the runner with your GitLab project using the Docker executor for better isolation.





Obtain the registration token from Settings > CI/CD > Runners in your GitLab project.



Register the runner:

sudo gitlab-runner register

Prompts:





GitLab instance URL: e.g., https://gitlab.com/.



Registration token: Paste the token (e.g., glrt-XXXXXXXXXXXXXXXXXXXX).



Description: e.g., debian-docker-runner.



Tags: e.g., debian,python,docker.



Executor: Choose docker.



Default Docker image: python:3.9.



Verify the runner is online in Settings > CI/CD > Runners.

Step 5: Configure GitLab Runner

Ensure the runner uses the Docker executor.





Edit the configuration:

sudo nano /etc/gitlab-runner/config.toml



Example configuration:

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



Restart the runner:

sudo gitlab-runner restart

Step 6: Set Up the Project Repository

Configure your GitLab project with the necessary files.





Create or verify the .gitlab-ci.yml file:

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



Create a Dockerfile for the Flask app:

FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]



Create a k8s-deployment.yml for Kubernetes:

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



Ensure requirements.txt includes Flask and pytest:

flask==2.3.3
pytest==7.4.0



Example app.py for the Flask app:

from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello from Flask!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)



Example test_app.py for testing:

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



Push to GitLab:

git add .gitlab-ci.yml Dockerfile k8s-deployment.yml requirements.txt app.py test_app.py
git commit -m "Add CI/CD for Flask with Docker Swarm and Kubernetes"
git push origin main

Step 7: Configure CI/CD Variables

Add sensitive credentials in Settings > CI/CD > Variables:





CI_REGISTRY_USER: Your GitLab username.



CI_REGISTRY_PASSWORD: A GitLab personal access token with api and write_repository scopes.



CI_REGISTRY: Your GitLab registry (e.g., registry.gitlab.com).



CI_REGISTRY_IMAGE: Your image path (e.g., registry.gitlab.com/your-username/your-project).



SWARM_HOST: SSH host for Docker Swarm (e.g., user@swarm-server).



KUBE_CONTEXT: Kubernetes context name (from kubectl config get-contexts).

For SSH access to Docker Swarm:





Generate an SSH key:

ssh-keygen -t rsa -b 4096 -f ~/.ssh/gitlab-runner



Add the public key to the Swarm host’s ~/.ssh/authorized_keys.



Add the private key as a CI/CD variable (SWARM_SSH_KEY).

For Kubernetes:





Create a kubeconfig file and encode it:

cat ~/.kube/config | base64



Add the base64-encoded kubeconfig as a CI/CD variable (KUBE_CONFIG).

Step 8: Set Up Deployment Environments





Docker Swarm:





Initialize a Swarm (on the Swarm host):

docker swarm init



Ensure the GitLab Runner can SSH into the Swarm host.



Kubernetes:





Ensure your cluster is running (e.g., via Minikube for testing or a cloud provider).



Configure kubectl with the correct context and credentials.

Step 9: Test the Pipeline





Navigate to CI/CD > Pipelines in your GitLab project.



Verify the pipeline runs:





build_job: Installs Python dependencies.



test_job: Runs pytest.



build_docker_image: Builds and pushes the Docker image.



deploy_swarm: Deploys to Docker Swarm (staging).



deploy_kubernetes: Deploys to Kubernetes (production, manual trigger).

Step 10: Troubleshooting





Runner issues: Check sudo gitlab-runner status and ensure Docker is running.



Docker login failures: Verify CI_REGISTRY_* variables.



SSH errors: Ensure SWARM_SSH_KEY and host permissions are correct.



Kubernetes errors: Validate KUBE_CONFIG and context; check kubectl logs.



Test failures: Ensure pytest is configured correctly in test_app.py.

Step 11: Enhance the Setup (Optional)





Use Docker-in-Docker (dind) securely with privileged = true in config.toml if needed.



Add health checks in k8s-deployment.yml or Docker Swarm services.



Configure autoscaling in Kubernetes:

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

This setup provides a complete CI/CD pipeline for a Flask application with deployments to Docker Swarm (staging) and Kubernetes (production) on a Debian system.
