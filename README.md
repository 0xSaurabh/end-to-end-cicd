# end-to-end-cicd

<img width="1600" alt="image" src="https://github.com/user-attachments/assets/719c9179-5d03-4cb4-b38c-74724cc618e8" />

<img width="1600" alt="image" src="https://github.com/user-attachments/assets/50391987-53f6-47e9-9db2-c2074d8f031e" />

<img width="1600" alt="image" src="https://github.com/user-attachments/assets/3a21c8ec-2208-40f0-a341-13f29ab2ff2d" />

## INFRA SETUP

**EC2 (Jenkins)** : AMI: `Ubuntu 22.04` + Type: `t3.large` / `t2.large` *(RAM-heavy stack: Jenkins + Docker + Maven + kubectl + Helm)*

**EKS** : `eksctl` + 2 Nodes + Type: `t3.small` / `t3.medium`

**ECR** : to store Docker images

**GitHub** : for containing app + infra configs

## GITHUB STRUCTURE

```bash
.
├── src/
├── pom.xml
├── Dockerfile
├── Jenkinsfile
└── mychart/
```

NOTE: Push **everything before pipeline run**

## EC2 (JENKINS) SETUP

**Security Group**: `22` (SSH) + `8080` (Jenkins UI) + `30000–32767` (optional NodePort)

NOTE: Attach IAM role later w Permissions: `EKS` + `ECR` + `EC2 read` + `IAM (cluster ops)`. Or just simply give `AdminAccess`.

## SSH CONNECT

SSH using PuTTY(Requires `.ppk`) or OpenSSH(Uses `.pem` directly, `ssh -i your-key.pem ubuntu@your-ec2-public-ip`)

## INSTALLATIONS (ON EC2)

### 1. System Update

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Java (Jenkins req.)

```bash
sudo apt install fontconfig openjdk-21-jre -y
java -version
```

### 3. AWS CLI v2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### 4. Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod +x get_helm.sh
sudo ./get_helm.sh
helm version
```

### 5. kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

### 6. eksctl

```bash
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/
eksctl version
```

### 7. Docker

```bash
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
newgrp docker
docker --version
```

### 8. Jenkins

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

Note: **Docker Permission Fix**

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart docker
sudo systemctl restart jenkins
```

## JENKINS UI

URL: `http://<IP>:8080`

Initial password: `cat /var/lib/jenkins/secrets/initialAdminPassword`

## JENKINS PLUGINS

`Docker` + `Docker Pipeline` + `Pipeline` + `Git` + `GitHub` + `Kubernetes CLI` *(optional)*

## MAVEN SETUP

UI → Tools → Maven → Add

eg.
```bash
sudo apt install maven -y
mvn -version
```

## ECR

create a private ECR repo

## EKS CLUSTER

```bash
sudo su - jenkins

eksctl create cluster --name demo-eks --region ap-south-1 --nodegroup-name my-nodes --node-type t3.small --nodes 2 --managed
```

**kubeconfig**
```bash
aws eks update-kubeconfig --region ap-south-1 --name demo-eks
kubectl get nodes
```

## NAMESPACE

```bash
kubectl create namespace helm-deployment
kubectl get namespaces
```

## HELM CHART

```bash
helm create mychart
```

**values.yaml**
```bash
image:
  repository: <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/my-ecr-repo
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 80
```

**deployment.yaml**
```bash
containerPort: 8080
```

NOTE: `LoadBalancer` → creates external AWS URL

## DOCKERFILE

eg.
```bash
FROM eclipse-temurin:17-jdk-jammy
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

## IAM STRATEGY

Attach IAM roles(`EKS` and `ECR`) to your EC2 so that no static creds needed and your Jenkins can use `AWS CLI` + `kubectl` from the same EC2 instance.

NOTE: `kubeconfig` required
```bash
aws eks update-kubeconfig --region ap-south-1 --name demo-eks
```

## JENKINS PIPELINE JOB

Type: `Pipeline` + Source: `SCM → Git` + Script: `Jenkinsfile`

## JENKINSFILE

eg.
```bash
pipeline {
    agent any

    tools {
        maven 'Maven3'
    }

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO   = '208976686989.dkr.ecr.ap-south-1.amazonaws.com/my-ecr-repo'
        APP_NAME   = 'my-ecr-repo'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    dockerImage = docker.build("${ECR_REPO}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Docker Push') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $ECR_REPO
                '''
                sh "docker push ${ECR_REPO}:${BUILD_NUMBER}"
            }
        }

        stage('Deploy with Helm') {
            steps {
                sh """
                    helm upgrade --install myapp ./mychart \
                    --namespace helm-deployment --create-namespace \
                    --set image.repository=${ECR_REPO} \
                    --set image.tag=${BUILD_NUMBER} \
                    --set service.type=LoadBalancer
                """
            }
        }
    }
}
```

## PIPELINE RUN

Build Now and Check Console Output

Stages: `Checkout` → `Build` → `Test` → `Docker Build` → `Docker Push` → `Helm Deploy`

## VERIFY DEPLOYMENT

```bash
kubectl get pods -n helm-deployment
kubectl get svc -n helm-deployment
```

Expect: Running pod + `LoadBalancer` svc + External URL

## FLOW SUMMARY

Push → GitHub, Jenkins pulls, Maven build + Test, Docker build + Push → ECR, Helm deploy → EKS + LoadBalancer → URL

## TROUBLESHOOTING

If Jenkins not opening, check `8080` SG

If Docker permission issue, add `jenkins` to `docker group`

If kubectl fail, update `kubeconfig`

If Pods fail,
```bash
kubectl describe pod
kubectl logs
```

If Helm fail, check: `values.yaml`, `image tag`, `namespace`
