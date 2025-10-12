### Project Overview
This project outlines the development and deployment of a simple two-tier web application using Apache and PHP for the frontend, and MySQL for the backend database. The application is containerized with Docker and orchestrated on Azure Kubernetes Service (AKS), Microsoft's managed Kubernetes platform. To automate the build and deployment process, GitHub Actions is utilized for Continuous Integration/Continuous Deployment (CI/CD), specifically to build the frontend Docker image and push it to Docker Hub. Portainer serves as a user-friendly graphical interface for managing Kubernetes resources and deploying the application to the AKS cluster.
The project demonstrates key DevOps practices, including containerization, orchestration, and automation, making it suitable for scalable web applications. All resources are isolated in a dedicated Kubernetes namespace called dev for better organization and security. This setup ensures the frontend can securely connect to the MySQL backend, allowing for basic operations like data retrieval and manipulation through a simple PHP interface

### Key technologies involved:
- Frontend: Apache HTTP Server with PHP for dynamic content.
- Backend: MySQL database for data persistence.
- Containerization: Docker for packaging the application.
- Orchestration: AKS for managing containers at scale.
- CI/CD: GitHub Actions for automated image builds.
- Management Tool: Portainer for simplified cluster operations.

### Problem Statement
Traditional web applications often face challenges in scalability, portability, and deployment consistency. Deploying a two-tier application (frontend and backend) manually on virtual machines or bare metal can lead to issues such as:
- Inconsistent environments between development, testing, and production.
- Manual scaling and management of resources, leading to downtime during updates.
- Lack of automation in building and deploying code changes.
- Security risks from exposed credentials and unisolated components.
- Difficulty in managing database persistence in distributed systems.
- This project addresses these by leveraging containerization and Kubernetes orchestration on AKS. Specifically, it solves the need for a reliable, automated deployment pipeline for a PHP-based web app with a MySQL backend, ensuring high availability, easy scaling, and secure configuration management.
- 
### Problem Overview
The core problem revolves around efficiently deploying and managing a two-tier application in a cloud environment. Observations from similar setups include:
- Development Overhead: Without containers, developers must replicate exact server configurations, leading to "it works on my machine" issues.
- Deployment Complexity: Manual uploads to servers or clusters are error-prone and time-consuming.
- CI/CD Gaps: Code changes require manual builds and pushes, delaying releases.
- Resource Isolation: Mixing resources in a shared cluster can cause conflicts or security breaches.
- Persistence and Connectivity: Ensuring the backend database remains stateful while the frontend statelessly connects to it is crucial.
- Management Tools: Command-line tools like kubectl are powerful but intimidating for beginners; a GUI like Portainer simplifies this.
- By using Docker for packaging, GitHub Actions for automation, AKS for orchestration, and Portainer for management, this project provides a streamlined solution. It focuses on a simple application (e.g., a PHP form interacting with MySQL) but scales to real-world scenarios like e-commerce or content management systems.
![Data archetitural design]()

### Prerequisites and Setup
Install Required Tools:
- Azure CLI: Download from the official Azure website and install.
- kubectl: Install via Azure CLI (az aks install-cli).
- Docker: Install Desktop version for local testing.
- Helm: For installing Portainer (curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash).
- Git: For repository management.
Create Azure Resources:
- Log in to Azure: az login.
- Create a resource group: az group create --name rg-aks-project --location eastus.
- Create AKS cluster: az aks create --resource-group rg-aks-project --name aks-cluster --node-count 2 --enable-addons monitoring --generate-ssh-keys.
- Get cluster credentials: az aks get-credentials --resource-group rg-aks-project --name aks-cluster.


Set Up GitHub Repository:

- Create a new repo on GitHub with your PHP application code (e.g., index.php for frontend logic connecting to MySQL).
- Add a Dockerfile for the frontend in the repo root:
 ```
version: "3.8"

services:
  mysql:
    image: mysql:8.0
    container_name: mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: webapp_db
      MYSQL_USER: webuser
      MYSQL_PASSWORD: webpass
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  webapp:
    build: .
    image: rukevweubio/twotier-application-deployment-image
    container_name: php-webapp
    restart: always
    ports:
      - "8080:80"
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: webuser
      MYSQL_PASSWORD: webpass
    depends_on:
      - mysql

volumes:
  mysql_data:
```
### Install Portainer on AKS:
- Add Helm repo: helm repo add portainer https://portainer.github.io/k8s/.
- Create namespace: kubectl create namespace portainer.
- install the portainer agent  on teh cluster
-  connect the  portainer agent service ip to the  portainer  ui 
- Install Portainer: helm install portainer portainer/portainer --namespace portainer.
- Expose Portainer: kubectl port-forward svc/portainer -n portainer 9000:9000 (access at http://localhost:9000).
  
### Build docker image  with gitaction 
- create a gitaction  workflow file
- create  docker secret and docker username  on git action  secret
-  commit  and push  code  for the gitation to be trigger
-  build and test docker image  with gitaction  and deploy image to docker hub
```
name: Build, Scan, and Deploy Docker Image  update 

on:
  push:
    branches:
      - main

env:
  IMAGE_NAME: rukevweubio/twotier-application-deployment-image
  DOCKERHUB_USERNAME: ${{ secrets.DOCKER_USERNAME }}
  DOCKERHUB_TOKEN: ${{ secrets.DOCKER_PASSWORD }}

jobs:
  build-scan-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ env.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build Docker image for webapp
        run: |
          docker compose build webapp
          docker images | grep twotier
          
      - name: Scan Docker image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:latest
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          severity: 'CRITICAL,HIGH'

      - name: Push Docker image to Docker Hub
        if: success()
        run: |
          docker push $IMAGE_NAME:latest
```
![Gitaction workflow](https://github.com/rukevweubio/Gitops-Portainer-AzureAks/blob/main/picture/Screenshot%20(2599).png)
  
###Containerization
Frontend Container:
- Test locally: docker build -t frontend-app:latest . and docker run -p 8080:80 frontend-app:latest.
- Ensure PHP code connects to MySQL using environment variables (e.g., via PDO):
-  used the submit.php file to connect the  front end to backend  and test locally
  
```
php$host = getenv('MYSQL_HOST') ?: 'localhost';
$db = getenv('MYSQL_DATABASE') ?: 'myappdb';
$user = getenv('MYSQL_USER') ?: 'root';
$pass = getenv('MYSQL_PASSWORD') ?: 'rootpass';
$pdo = new PDO("mysql:host=$host;dbname=$db", $user, $pass);
```

### Kubernates cluster 
- login into azure cloud
- create azure kubernates cluster
- login  with azure cli  using az login
-  connect  the azure  account to login
- create namespace  portainer and dev
- deploy  portainer  on teh  portainer namespace
- expose the ui of portainer  using  loadbalancer 
 ```  
az login
az login --tenant <TENANT_ID_OR_DOMAIN>
az account list --output table
az account show --output table
az account set --subscription "<SUBSCRIPTION_ID_OR_NAME>"
az account show
az ad signed-in-user show
az role assignment list --assignee $(az ad signed-in-user show --query objectId -o tsv) --output table
az aks show --resource-group <RESOURCE_GROUP> --name <AKS_CLUSTER_NAME> --output table
az aks show --resource-group <RESOURCE_GROUP> --name <AKS_CLUSTER_NAME> --output json
az aks get-credentials --resource-group <RESOURCE_GROUP> --name <AKS_CLUSTER_NAME>
az aks get-credentials --resource-group <RESOURCE_GROUP> --name <AKS_CLUSTER_NAME> --admin --overwrite-
```
