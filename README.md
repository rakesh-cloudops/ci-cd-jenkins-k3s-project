# End-to-End CI/CD Pipeline with Jenkins, Docker, and K3s on AWS

An end-to-end DevOps lab project demonstrating how to build, test, containerize, and deploy a Java application to a multi-node Kubernetes cluster using Jenkins, Docker, Docker Hub, and K3s on AWS EC2.

## Project Overview

This project automates the application delivery process, from source code checkout to deployment on Kubernetes.

The application is built with Java 21 and Maven. Jenkins executes the pipeline on a dedicated worker agent, builds and tests the application, creates a Docker image, pushes the image to Docker Hub, and deploys the updated version to the K3s cluster when publishing is enabled.

The project was implemented step by step to gain practical experience with AWS infrastructure, Linux administration, CI/CD automation, containerization, Kubernetes orchestration, and troubleshooting.

## Architecture

```text
Developer
    |
    v
GitHub Repository
    |
    v
Jenkins Controller (AWS EC2)
    |
    v
Jenkins Agent (K3s Worker)
    |
    +--> Checkout Source Code
    |
    +--> Maven Build and Unit Tests
    |
    +--> Docker Image Build
    |
    +--> Push Image to Docker Hub
    |
    v
K3s Kubernetes Cluster
    |
    v
Kubernetes Deployment
    |
    +--> Application Pod 1
    |
    +--> Application Pod 2
    |
    v
Kubernetes Service
    |
    v
Java Application on Port 8080
    |
    v
/health Endpoint
```

## Technology Stack

| Component | Technology |
|---|---|
| Cloud Infrastructure | AWS EC2 |
| Operating System | Ubuntu Linux |
| Source Control | Git and GitHub |
| CI/CD | Jenkins |
| Build Tool | Apache Maven |
| Programming Language | Java 21 |
| Containerization | Docker |
| Container Registry | Docker Hub |
| Container Orchestration | K3s / Kubernetes |
| Application Deployment | Kubernetes Deployment and Service |
| Code Quality Integration | SonarQube pipeline stage available; server integration pending verification |
| Monitoring | Prometheus and Grafana lab setup; complete cluster monitoring pending verification |

## Infrastructure Setup

The lab uses three EC2 instances with separate responsibilities.

| Node | Role |
|---|---|
| `k3s-master` | K3s control plane and Jenkins controller |
| `k3s-worker1` | K3s worker and Jenkins build agent |
| `k3s-worker2` | K3s worker and Jenkins build agent |

The nodes communicate over the private network within the AWS VPC. Kubernetes administration is performed through the control-plane node, while application workloads are scheduled on worker nodes.

> Private IP addresses, public IP addresses, SSH keys, passwords, and tokens are environment-specific and should not be committed to this repository.

## Prerequisites

Before setting up the project, prepare:

- An AWS account with permission to create EC2 instances and security groups.
- Three Ubuntu EC2 instances connected to the same VPC.
- SSH access to the instances.
- Git and GitHub access.
- Java 21 and Maven on the Jenkins build agent.
- Docker on the build agent.
- A Docker Hub account.
- Jenkins with Pipeline and Git support.
- K3s installed and the worker nodes joined to the cluster.

## Step 1: Create the AWS EC2 Instances

1. Launch three Ubuntu EC2 instances.
2. Configure the instances to communicate over the same VPC.
3. Configure SSH access using an EC2 key pair.
4. Allow only the required ports in the security groups.
5. Assign the roles `k3s-master`, `k3s-worker1`, and `k3s-worker2`.

Use private IP addresses for internal cluster communication wherever possible.

## Step 2: Install K3s on the Master

Connect to the master instance over SSH and install K3s:

```bash
curl -sfL https://get.k3s.io | sh -
```

Check the service:

```bash
sudo systemctl status k3s --no-pager
```

Check the Kubernetes nodes:

```bash
sudo kubectl get nodes -o wide
```

The master should become Ready before joining the worker nodes.

## Step 3: Join the Worker Nodes

On the master, retrieve the K3s node token:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

On each worker, join the cluster using the master’s private IP and token:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://MASTER_PRIVATE_IP:6443 \
  K3S_TOKEN='YOUR_K3S_NODE_TOKEN' sh -
```

Replace the placeholders with the actual private IP and token. Keep the token secret.

Verify both workers from the master:

```bash
sudo kubectl get nodes -o wide
```

All three nodes should show `Ready`.

## Step 4: Install and Configure Jenkins

Install Jenkins on the designated controller instance and open the Jenkins UI through the configured access method.

Configure the Jenkins worker agents to connect to the controller. The pipeline uses this label:

```groovy
agent {
    label 'k3s-worker'
}
```

The worker agent needs the tools required by the pipeline:

```bash
java -version
mvn -version
docker --version
git --version
kubectl version --client
```

The Jenkins runtime user must have permission to use Docker and the correct Kubernetes credentials and cluster context for deployment.

Verify these commands as the Jenkins agent user, not only as the normal Ubuntu login user.

## Step 5: Configure Docker Hub Credentials

In Jenkins, navigate to:

**Manage Jenkins → Credentials → System → Global credentials**

Create a username/password credential with this ID:

```text
registry-credentials
```

Use your Docker Hub username and an appropriate Docker Hub access token as the password.

The pipeline uses this credential to authenticate before pushing the image. Never hardcode the Docker Hub password or token in the Jenkinsfile.

## Step 6: Configure the Kubernetes Manifests

The `k8s/` directory contains the deployment and service manifests.

### Deployment

`k8s/deployment.yaml` defines the application workload:

- Two application replicas.
- Container port `8080`.
- Readiness and liveness probes using `/health`.
- CPU and memory requests and limits.
- A non-root container user.
- A read-only root filesystem with a writable temporary volume.

The image is represented by a placeholder:

```yaml
image: __IMAGE__
```

Jenkins replaces this placeholder with the selected image name and build-number tag during publishing.

### Service

`k8s/service.yaml` defines a Kubernetes Service that exposes port `80` and forwards traffic to application port `8080`.

This allows traffic to reach the healthy application pods through the Kubernetes Service.

## Step 7: Configure the Jenkins Pipeline

Create a Jenkins Pipeline job using **Pipeline script from SCM**.

Configure:

- Repository URL: `https://github.com/rakesh-cloudops/ci-cd-jenkins-k3s-project.git`
- Branch: `*/main`
- Script Path: `Jenkinsfile`
- Credentials: select Git credentials only if required by repository access.

The pipeline has two optional parameters:

| Parameter | Purpose |
|---|---|
| `ANALYZE` | Runs the SonarQube analysis and waits for the quality gate when the SonarQube server is configured in Jenkins. |
| `PUBLISH` | Builds the Docker image, pushes it to the registry, and deploys the updated image to Kubernetes. |

The `IMAGE` parameter must contain the actual image repository, for example:

```text
YOUR_DOCKERHUB_USERNAME/my-app
```

The registry parameter defaults to Docker Hub.

## Step 8: Understand the CI/CD Stages

The Jenkinsfile executes the following stages:

1. **Checkout** — retrieves the source code from GitHub.
2. **Test** — runs Maven verification and unit tests.
3. **SonarQube** — runs only when `ANALYZE` is enabled and the server is configured.
4. **Quality Gate** — waits for the SonarQube quality gate when analysis is enabled.
5. **Docker Build** — builds the image when `PUBLISH` is enabled.
6. **Docker Login** — authenticates using the Jenkins credential.
7. **Docker Push** — pushes the build-number-tagged image to Docker Hub.
8. **Deploy** — applies the Kubernetes Deployment and Service manifests.

The default pipeline can run the build and tests without publishing a new image. Publishing must be explicitly enabled.

## Step 9: Run and Verify the Pipeline

Run the Jenkins job with:

- `ANALYZE` disabled unless SonarQube has been configured.
- `PUBLISH` enabled to build, push, and deploy the application.
- `IMAGE` set to your Docker Hub repository.

After a successful run, verify the Kubernetes deployment from the master:

```bash
sudo kubectl get nodes -o wide
sudo kubectl get deployments
sudo kubectl get pods -o wide
sudo kubectl get services
```

Check that the application deployment has two available replicas and that both pods become Ready.

Inspect deployment details if necessary:

```bash
sudo kubectl describe deployment my-app
sudo kubectl logs deployment/my-app
```

## Step 10: Test the Application Health Endpoint

The application exposes a health endpoint at `/health`.

To test it through a temporary local port-forward, run this command on the master:

```bash
sudo kubectl port-forward svc/my-app 18080:80
```

Keep that terminal running. Open another terminal on the master and execute:

```bash
curl --max-time 10 -v http://localhost:18080/health
```

Expected response:

```text
ok
```

A successful HTTP response confirms that the Kubernetes Service can route the request to the application.

## Repository Structure

```text
ci-cd-jenkins-k3s-project/
├── .github/
│   └── workflows/
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── src/
├── .dockerignore
├── .gitignore
├── Dockerfile
├── docker-compose.yml
├── Jenkinsfile
├── pom.xml
├── License.md
└── README.md
```

## Troubleshooting Notes

### Jenkins cannot access Docker

Check Docker as the Jenkins agent user:

```bash
whoami
id
docker version
docker ps
```

Verify the agent user's Docker permissions and service status.

### Kubernetes commands fail on the agent

Check whether the Jenkins agent has a valid kubeconfig and the correct cluster context:

```bash
kubectl config current-context
kubectl get nodes
```

The Kubernetes API endpoint must be reachable from the agent, and the configured credentials must have the required permissions.

### Application pods are not Ready

Inspect pod status, events, and logs:

```bash
sudo kubectl get pods -o wide
sudo kubectl describe pod POD_NAME
sudo kubectl logs POD_NAME
```

### Docker image is not found

Verify the `IMAGE` parameter, the Docker Hub repository, the build-number tag, and the Jenkins registry credentials.

## Security Notes

- Never commit private SSH keys, access tokens, passwords, or kubeconfig files.
- Do not expose the Kubernetes API or Jenkins UI publicly without appropriate access controls.
- Restrict security-group rules to trusted sources.
- Use Jenkins credentials for secrets.
- Use private IP addresses for communication between EC2 instances.
- Rotate credentials if they are accidentally exposed.

## Current Project Status

The CI/CD build and Kubernetes deployment flow has been tested in the lab. Maven tests passed, a Docker image was pushed to Docker Hub, two application replicas became Ready, and the `/health` endpoint returned `ok`.

SonarQube quality-gate integration and full Prometheus/Grafana cluster monitoring remain optional follow-up tasks until they are fully configured and verified.

## Learning Outcomes

This project provided hands-on experience with:

- AWS EC2 provisioning and private networking.
- Linux administration and SSH troubleshooting.
- K3s cluster setup and worker-node management.
- Jenkins controller and agent configuration.
- Maven builds and automated testing.
- Docker image creation and registry authentication.
- Kubernetes Deployments, Services, probes, and resource limits.
- CI/CD troubleshooting and end-to-end deployment verification.

## License

See [License.md](License.md) for the license included with this repository.
