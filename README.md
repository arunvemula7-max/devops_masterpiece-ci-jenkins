# DevOps-MasterPiece

In this project, I have implemented an end-to-end production-grade CI-CD pipeline while adhering to security best practices and DevSecOps principles. This project utilizes a comprehensive toolset including Git, GitHub, Jenkins, Maven, JUnit, SonarQube, JFrog Artifactory, Docker, Trivy, AWS S3, Docker Hub, GitHub CLI, EKS, ArgoCD, Prometheus, Grafana, Slack, and Hashicorp Vault to achieve a robust automation workflow.

#### Jenkins is utilized for Continuous Integration and ArgoCD for Continuous Deployment.

## Project Architecture
![](https://github.com/arunkumarreddyvemula/DevOps_MasterPiece-CI-with-Jenkins/blob/main/images/pipeline.png)

## Pipeline Flow:
1. When a commit event occurs in the application code GitHub repository, the GitHub webhook triggers Jenkins to start the build process.

2. Maven builds the code. If the build fails, the pipeline stops, and Jenkins notifies the user via Slack.

3. JUnit performs unit testing. If the application passes the test cases, it proceeds to the next step; otherwise, the pipeline fails and a Slack notification is sent.

4. The SonarQube scanner analyzes the code and sends the report to the SonarQube server. The report is evaluated against defined quality gates, and results are displayed on the web dashboard.

5. Quality gates define specific conditions, such as the maximum allowable number of bugs, vulnerabilities, or code smells. A webhook sends the quality gate status back to Jenkins. If the status is "failed", the pipeline stops and notifies the user.

6. Once the quality gate passes, artifacts are sent to JFrog Artifactory. Successful artifact storage is required to proceed to the next stage.

7. Following the artifact push, Docker builds the container image. Any failure at this stage triggers a notification and stops the pipeline.

8. Trivy scans the Docker image for vulnerabilities. If any are found, the pipeline fails, the generated report is uploaded to AWS S3 for review, and a notification is sent.

9. After a successful Trivy scan, the Docker image is pushed to Docker Hub.

10. Jenkins clones the Kubernetes manifest repository from the feature branch (or pulls changes if already present).

11. Jenkins updates the image tag in the deployment manifest to reflect the newly built version.

12. Jenkins commits and pushes the updated manifest back to the feature branch.

13. Jenkins creates a pull request against the main branch for review.

14. After a team member reviews and merges the pull request, ArgoCD detects the changes in the main branch and deploys the application to the EKS cluster.

### PreRequisites
1. JDK
2. Git
3. GitHub
4. GitHub CLI
5. Jenkins
6. SonarQube
7. JFrog Artifactory
8. Docker
9. Trivy
10. AWS Account
11. AWS CLI
12. Docker Hub Account
13. Terraform
14. EKS Cluster
15. kubectl
16. ArgoCD
17. Helm
18. Prometheus and Grafana
19. Hashicorp Vault
20. Slack

## Server Configuration
1. 2 t2.medium (Ubuntu) EC2 Instances:
   - Instance 1: SonarQube and Hashicorp Vault server
   - Instance 2: JFrog Artifactory
2. 1 t2.large (Ubuntu) EC2 Instance: Jenkins, Docker, Trivy, AWS CLI, GitHub CLI, Terraform
3. EKS Cluster with t3.medium nodes

# Implementation Guide

## Step 1: Installation

### Stage-01: Install JDK and Create a Java Springboot application
Push all application code files to the GitHub repository.

![](https://github.com/arunkumarreddyvemula/DevOps_MasterPiece-CI-with-Jenkins/blob/main/images/Screenshot%20from%202023-06-23%2022-45-05.png)

### Stage-02: Install Jenkins, Docker, Trivy, AWS CLI, GitHub CLI, Terraform
#### Jenkins Installation
1. Follow the official Jenkins installation guide for Linux.
2. Install suggested plugins upon initial setup.
3. Install additional plugins: SonarQube Scanner, Quality Gates, Artifactory, Hashicorp Vault, Slack, and Open Blue Ocean.

#### Docker Installation
1. Install Docker:
```sh
sudo apt update
sudo apt install docker.io
```
2. Add the current user and Jenkins user to the Docker group:
```sh
sudo usermod -aG docker $USER
sudo usermod -aG docker jenkins
```

#### Trivy Installation
1. Install Trivy:
```sh 
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

#### AWS CLI Installation
1. Install AWS CLI:
```sh 
sudo apt install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

#### GitHub CLI Installation
1. Install GitHub CLI:
```sh 
type -p curl >/dev/null || (sudo apt update && sudo apt install curl -y)
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
&& sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg \
&& echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
&& sudo apt update \
&& sudo apt install gh -y
```

#### Terraform Installation
1. Install Terraform:
```sh 
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update
sudo apt-get install terraform
```

### Stage-03: Install SonarQube and Hashicorp Vault
#### SonarQube Installation
1. Run SonarQube as a Docker container:
```sh
sudo docker run -d -p 9000:9000 --name sonarqube sonarqube  
```

#### Hashicorp Vault Installation
1. Install Vault:
```sh
sudo curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt update
sudo apt install vault -y
```

### Stage-04: Install JFrog Artifactory
1. Run JFrog Artifactory as a Docker container:
```sh 
sudo docker pull docker.bintray.io/jfrog/artifactory-oss:latest
sudo mkdir -p /jfrog/artifactory
sudo chown -R 1030 /jfrog/
sudo docker run --name artifactory -d -p 8081:8081 -p 8082:8082 \
	-v /jfrog/artifactory:/var/opt/jfrog/artifactory \
	docker.bintray.io/jfrog/artifactory-oss:latest
```

### Stage-05: Slack Setup
Configure a Slack workspace and channel (e.g., #cicd-pipeline) for receiving notifications.

### Stage-06: EKS Cluster Creation using Terraform
The Terraform configuration for the EKS cluster is maintained within the project repository.
Note: Ensure AWS CLI is configured with appropriate permissions before running Terraform.
After the cluster is created, update the local kubeconfig:
```sh
aws eks --region your-region-name update-kubeconfig --name cluster-name
```

### Stage-07: Install ArgoCD in EKS
1. Deploy ArgoCD:
```sh
kubectl create namespace argocd 
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
2. Change the service type to NodePort for UI access:
```sh
kubectl -n argocd edit svc argocd-server
```

### Stage-08: Install Helm
1. Install Helm:
```sh
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Stage-09: Install Prometheus and Grafana
1. Use Helm to deploy the monitoring stack:
```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
kubectl create namespace prometheus
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
```

## Step 2: Tool Configuration

### Stage-01: Jenkins Configuration
1. Configure Maven and SonarQube Scanner under "Manage Jenkins" > "Global Tool Configuration".

### Stage-02: Hashicorp Vault Configuration
1. Update `/etc/vault.d/vault.hcl` with the appropriate listener and storage settings.
2. Initialize and unseal the Vault.
3. Enable AppRole authentication and create a role for Jenkins:
```sh 
vault auth enable approle
vault write auth/approle/role/jenkins-role token_num_uses=0 secret_id_num_uses=0 policies="jenkins"
vault read auth/approle/role/jenkins-role/role-id
vault write -f auth/approle/role/jenkins-role/secret-id
```

### Stage-03: SonarQube Server Configuration
1. Create a project manually and generate an analysis token.
2. Configure Quality Gates and Webhooks to point back to the Jenkins server.

### Stage-04: JFrog Artifactory Configuration
1. Create a Maven repository and a dedicated user for Jenkins integration.

### Stage-05: S3 Bucket Creation
1. Create a unique S3 bucket in the AWS console to store Trivy vulnerability reports.

### Stage-06: DockerHub Account
1. Ensure a DockerHub account is active for image storage.

### Stage-07: Slack Configuration for Jenkins
1. Add the Jenkins CI app to your Slack workspace and obtain the Integration Token.

### Stage-08: Slack Configuration for ArgoCD
1. Create a Slack App, configure Bot Token Scopes (chat:write), and install it to the workspace.

## Step 3: Store Credentials in Vault
1. Enable the KV secrets engine and store credentials:
```sh
vault secrets enable -path=secrets kv
vault write secrets/creds/docker username=your_user password=your_password
```
2. Define a policy allowing Jenkins to read these secrets.

## Step 4: Integrate Tools with Jenkins
1. Add Vault AppRole credentials to Jenkins.
2. Configure the SonarQube server, JFrog Artifactory, and Slack within the Jenkins System Configuration.
3. Use the Vault plugin to retrieve DockerHub credentials dynamically during the pipeline execution.

## Step 5: Integrate ArgoCD
1. Connect ArgoCD to the GitHub repository containing the Kubernetes manifests.
2. Configure ArgoCD notifications to send sync status updates to Slack.

## Step 6: Pipeline Creation
The pipeline is defined using a declarative Jenkinsfile. Key stages include:
- Git Checkout
- Build and JUnit Test
- SonarQube Analysis
- Quality Gate Check
- JFrog Artifact Upload
- Docker Build and Trivy Scan
- S3 Report Upload
- Docker Push
- Manifest Update and PR Creation

## Step 7: Project Output

### Jenkins Pipeline View
The pipeline executes all stages from code checkout to PR creation.
![](https://github.com/arunkumarreddyvemula/DevOps_MasterPiece-CI-with-Jenkins/blob/main/images/jenkins.png)

### SonarQube Analysis
Detailed code quality reports are generated for every build.
![](https://github.com/arunkumarreddyvemula/DevOps_MasterPiece-CI-with-Jenkins/blob/main/images/sonarqube.png)

### Quality Gate Status
The pipeline automatically halts if the code does not meet the defined quality standards.
![](https://github.com/arunkumarreddyvemula/DevOps_MasterPiece-CI-with-Jenkins/blob/main/images/quality-gate.png)

### Trivy Security Reports
Vulnerability reports are archived in S3 for compliance and auditing.
![](https://github.com/arunkumarreddyvemula/DevOps_MasterPiece-CI-with-Jenkins/blob/main/images/trivy-report.png)

### ArgoCD Deployment
ArgoCD ensures the EKS cluster state matches the Git repository.
![](https://github.com/arunkumarreddyvemula/DevOps_MasterPiece-CI-with-Jenkins/blob/main/images/argocd.png)

### Monitoring with Grafana
Real-time metrics are visualized via Prometheus and Grafana.
![](https://github.com/arunkumarreddyvemula/DevOps_MasterPiece-CI-with-Jenkins/blob/main/images/Screenshot%20from%202023-06-22%2023-19-50.png)

---

### Maintainer
**Arun Kumar Reddy Vemula**
AI/ML Engineer

I am an AI/ML Engineer with over 5 years of experience developing practical machine learning solutions across various domains, including autonomous vehicles and industrial IoT. My expertise extends to building robust DevSecOps pipelines that ensure scalable and secure deployment of high-performance models. I am passionate about automating complex workflows and maintaining high standards in software delivery.

- Email: arunkumarreddy952@gmail.com
- GitHub: arunkumarreddyvemula
- LinkedIn: Arun Kumar Reddy Vemula

This project is actively maintained to reflect current best practices in DevOps and CI/CD automation. Acknowledgments to the original contributors for the foundational architecture.