Here is my Full plan Link :

https://drive.google.com/file/d/1b1NKhpKPH3VJm3bXdThc1COqUIk-AOKU/view?usp=sharing

Due to time issue I can explain it properly but I shall try my best to explain. Hopefully my plan image will be helpfull for understanding the issue.

Infrastracture Create with Teffaform :

sudo apt install -y curl gnupg software-properties-common

wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update

sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys B7B3B788A8D3785C

sudo apt update

sudo apt install terraform

terraform --version
terraform -install-autocomplete
My Plan is to cleate two VM for Jenkins and K8s:
We need to setup and configure awscli:
AWS Configure


add config file here

Main.tf :(For Terraform)

provider "aws" {
  region = "us-east-1"  # Change to your preferred region
}

resource "aws_instance" "vm" {
  count         = 2
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.medium"  # 2 vCPUs, 4 GB RAM

  tags = {
    Name = "ubuntu-vm-${count.index + 1}"
  }
}

data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"]  # Canonical's AWS account ID
}

output "instance_public_ips" {
  value = aws_instance.vm[*].public_ip
}


For Jenkins Install:

Jenkins Installation and login:
======================
  sudo apt update
 sudo apt upgrade
 sudo apt update
 sudo apt install fontconfig openjdk-17-jre
 java -version
openjdk version "17.0.13" 2024-10-15
OpenJDK Runtime Environment (build 17.0.13+11-Debian-2)
OpenJDK 64-Bit Server VM (build 17.0.13+11-Debian-2, mixed mode, sharing)
java -version
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc   https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]"   https://pkg.jenkins.io/debian-stable binary/ | sudo tee   /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
cat /var/lib/jenkins/secrets/initialAdminPassword
  sudo nano  /var/lib/jenkins/secrets/initialAdminPassword
  
Agent Connect with secretfile:
   
For Frontend :
DockerFile:
FROM node:18
WORKDIR /app
COPY package.json .
RUN npm install
RUN npm install -g npm@11.1.0
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]


Runs the app in the development mode.\
Open [http://localhost:3000](http://localhost:3000) to view it in your browser.

For Backend
FROM node:18
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
EXPOSE 5000
CMD ["node", "src/index.js"]

Docker-compose.yml
version: "3.8"

services:
  db:
    image: mysql:8.37
    container_name: mysql_db
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: mydb
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  backend:
    build: ./backend
    container_name: node_backend
    ports:
      - "5000:5000"
    depends_on:
      - db
    environment:
      DB_HOST: db
      DB_USER: root
      DB_PASSWORD: password
      DB_NAME: mydb

  frontend:
    build: ./frontend
    container_name: react_frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend

volumes:
  mysql_data:

Push Docker Image:
Docker Login First then 
docker push <myusername>/backend:latest
docker push <myusername>/frontend:latest
For Check locally 
Build First:
sudo docker-compose build --no-cache Then
docker-compose up -d 
Docker Pull :
docker pull <myusername>/backend:latest
docker pull <myusername>/frontend:latest
docker pull <myusername>/mysql-db:8.37

Deploy In k8s:

K8s Install Through Terraform:
==============================
touch minikube.tf
provider "kubernetes" {
  host                   = "https://${minikube_ip}:8443"
  cluster_ca_certificate = base64decode(local.kube_ca)
  client_certificate     = base64decode(local.client_cert)
  client_key             = base64decode(local.client_key)
}

resource "kubernetes_pod" "example" {
  metadata {
    name = "example"
  }

  spec {
    container {
      name  = "nginx"
      image = "nginx"
    }
  }
}
minikube ip
kubectl config view --raw
terraform apply

First Deploy backend:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: myusername/node-backend:latest
          ports:
            - containerPort: 5000
          env:
            - name: DB_HOST
              value: db
            - name: DB_USER
              value: root
            - name: DB_PASSWORD
              value: password
            - name: DB_NAME
              value: mydb
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
  type: ClusterIP

Then Deploy ForntEnd:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: myusername/react-frontend:latest
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
  type: ClusterIP
kubectl apply -f mysql-deployment.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f frontend-deployment.yaml
First I deploy DB then Backend Then Frontend As Backend has dependency on DB and Frontend has dependency on Frontend.


For Cheking :
kubectl Get pods
kubectl Get Services
For Specific pod logs
kubectl logs <pod-name>

Fornend Load Balancer:

apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: LoadBalancer
  
IMPLEMENT k8S WITH cicd:

Ensure Kubeconfig file from your Minikube setup in cicd
Ensure that your CI/CD server has network access to the Kubernetes cluster, especially if you're running Minikube locally and need to expose it externally.
Install Kubernetes CLI (kubectl) on Your CI/CD Server:
Example to connectL
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
mv kubectl /usr/local/bin/
Configure kubectl with Minikube's Kubeconfig: Minikube provides a Kubeconfig file that your CI/CD tool can use. Make sure your CI/CD system has access to the Kubeconfig file. Here's how to get it from Minikube:

From your Minikube machine
minikube kubeconfig > ~/.kube/config
For Connect CICD and K8s securely

Jenkins Integration

Install Kubernetes Plugin in Jenkins:

Go to Manage Jenkins → Manage Plugins.
Install the Kubernetes plugin, which enables Jenkins to communicate with your Kubernetes cluster.

Configure Kubernetes in Jenkins:
 Manage Jenkins → Configure System.
Scroll down to the Cloud section and add a Kubernetes Cloud.
In the Kubernetes URL, provide the path to the Minikube cluster or the API server URL from the Kubeconfig.
Define Jenkins Pipeline for Kubernetes Deployment: In your Jenkinsfile, define stages to deploy to Kubernetes. For example:

pipeline {
    agent "test"   // As my agent Name is test
    
    environment {
        DB_HOST = 'db'
        DB_USER = 'root'
        DB_PASSWORD = 'password'
        DB_NAME = 'mydb'
    }
    
    stages {
        stage('Build') {
            steps {
                script {
                    sh 'npm install && npm run build'
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Use kubectl to deploy the app to Kubernetes
                    sh 'kubectl apply -f k8s/deployment.yaml'
                }
            }
        }
    }
}
Configure Webhook to Trigger Jenkins Pipeline:
We can do it with two way:
From Github repository add github-webhok >> Jenkinsurl>>
Run Jenkins Job: Once Jenkins is configured and the job is triggered (either manually or via webhook), Jenkins will deploy the application to Kubernetes using the configurations in your pipeline.

With GitLab CI/CD Integration

Set up GitLab CI/CD Configuration: In my repository, define the .gitlab-ci.yml file for your CI/CD pipeline. This YAML file will have stages for build, test, and deployment.

Examplev of  .gitlab-ci.yml to deploy to Kubernetes:
stages:
  - build
  - deploy

build:
  stage: build
  script:
    - npm install
    - npm run build
  only:
    - master

deploy:
  stage: deploy
  script:
    - kubectl apply -f k8s/deployment.yaml
  only:
    - master
This is the process how I can Complete My CICD and K8s implementation. For security we can use sonarqube, Trivy and OWASP. Thus we can Complete the full path securly.
