
# Project Documentation: Deployment of a Two-Tier Application on Azure Kubernetes Service (AKS) Using Portainer

## Introduction

This documentation outlines the deployment of a two-tier web application consisting of:
- **Front-end**: An Apache HTTP Server hosting static or dynamic web content.
- **Back-end**: A MySQL database for data storage and management.

The deployment leverages Azure Kubernetes Service (AKS) for orchestrating the back-end, Docker for containerizing the front-end, Docker Hub for image storage, and Portainer for simplified management and deployment of Kubernetes resources on the AKS cluster.

The architecture assumes the front-end connects to the back-end MySQL instance via a Kubernetes Service. This setup provides scalability, high availability, and ease of management in a cloud-native environment.

**Key Technologies Used**:
- Azure Kubernetes Service (AKS)
- Kubernetes (for orchestration)
- Docker (for containerization)
- Docker Hub (for image registry)
- Portainer (for Kubernetes management UI)
- Apache HTTP Server (front-end)
- MySQL (back-end)

**Assumptions**:
- You have an Azure subscription.
- The front-end Apache application is configured to connect to the MySQL database (e.g., via environment variables or config files).
- Basic knowledge of command-line tools like `az` (Azure CLI), `kubectl`, and `docker`.

## Prerequisites

Before starting, ensure the following:
1. **Azure Account**: Active Azure subscription with permissions to create AKS clusters.
2. **Tools Installed**:
   - Azure CLI (`az`): Install from [Microsoft Docs](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).
   - kubectl: Install via Azure CLI or from [Kubernetes Docs](https://kubernetes.io/docs/tasks/tools/).
   - Docker: Install from [Docker Docs](https://docs.docker.com/get-docker/).
   - Git (optional, for cloning repositories).
3. **Docker Hub Account**: Create a free account at [hub.docker.com](https://hub.docker.com).
4. **Portainer**: Will be deployed on the cluster as part of the process.
5. **Hardware/Software**:
   - A machine with at least 4GB RAM for local Docker builds.
   - Access to Azure Portal for monitoring.

## Step-by-Step Implementation Process

### Step 1: Set Up Azure Kubernetes Service (AKS) Cluster

1. **Log in to Azure CLI**:
   ```
   az login
   ```

2. **Create a Resource Group** (if not existing):
   ```
   az group create --name myResourceGroup --location eastus
   ```

3. **Create AKS Cluster**:
   Use the Azure CLI to provision an AKS cluster. This example creates a basic cluster with 2 nodes.
   ```
   az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 2 --enable-addons monitoring 
   ```

4. **Get Cluster Credentials**:
   Connect your local `kubectl` to the AKS cluster.
   ```
   az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
   ```

5. **Verify Cluster**:
   ```
   kubectl get nodes
   ```
   This should list the nodes in your cluster.

### Step 2: Deploy the Back-End (MySQL) on AKS

The back-end MySQL will be deployed as a StatefulSet in Kubernetes for persistent data storage. Use a PersistentVolumeClaim (PVC) for data persistence.

1. **Create a Namespace** (optional, for organization):
   ```
   kubectl create namespace app-namespace
   ```

2. **Create MySQL Deployment YAML**:
   Create a file named `mysql-deployment.yaml` with the following content:

   ```yaml
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: mysql
     namespace: app-namespace
   spec:
     serviceName: mysql
     replicas: 1
     selector:
       matchLabels:
         app: mysql
     template:
       metadata:
         labels:
           app: mysql
       spec:
         containers:
         - name: mysql
           image: mysql:8.0
           env:
           - name: MYSQL_ROOT_PASSWORD
             value: "your-root-password"  # Change to a secure password
           - name: MYSQL_DATABASE
             value: "your-database-name"
           ports:
           - containerPort: 3306
           volumeMounts:
           - name: mysql-persistent-storage
             mountPath: /var/lib/mysql
     volumeClaimTemplates:
     - metadata:
         name: mysql-persistent-storage
       spec:
         accessModes: [ "ReadWriteOnce" ]
         resources:
           requests:
             storage: 10Gi  # Adjust size as needed
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: mysql-service
     namespace: app-namespace
   spec:
     selector:
       app: mysql
     ports:
       - protocol: TCP
         port: 3306
         targetPort: 3306
     type: ClusterIP  # Internal service
   ```

3. **Apply the YAML**:
   ```
   kubectl apply -f mysql-deployment.yaml
   ```

4. **Verify Deployment**:
   ```
   kubectl get pods -n app-namespace
   kubectl get services -n app-namespace
   ```
   Wait for the MySQL pod to be in "Running" state.

### Step 3: Build and Push Front-End (Apache) Docker Image

Assume your front-end is a simple Apache server with custom web content (e.g., HTML/PHP files connecting to MySQL).

1. **Create Dockerfile** for Front-End:
   Create a file named `Dockerfile` in your project directory:

   ```dockerfile
   FROM httpd:2.4  # Official Apache image
   COPY ./your-web-content/ /usr/local/apache2/htdocs/  # Copy your web files
   # If using PHP, use a PHP-enabled Apache image like php:8.2-apache and install extensions as needed
   ```

2. **Build the Docker Image**:
   ```
   docker build -t yourusername/frontend-app:latest .
   ```

3. **Log in to Docker Hub**:
   ```
   docker login
   ```

4. **Push to Docker Hub**:
   ```
   docker push yourusername/frontend-app:latest
   ```

### Step 4: Deploy Front-End on AKS Using Kubernetes

Create a Deployment for the front-end that pulls the image from Docker Hub.

1. **Create Front-End Deployment YAML**:
   Create a file named `frontend-deployment.yaml`:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: frontend
     namespace: app-namespace
   spec:
     replicas: 2  # Scale as needed
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
           image: yourusername/frontend-app:latest
           ports:
           - containerPort: 80
           env:
           - name: DB_HOST
             value: "mysql-service"  # Points to MySQL service
           - name: DB_USER
             value: "root"
           - name: DB_PASSWORD
             value: "your-root-password"
           - name: DB_NAME
             value: "your-database-name"
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: frontend-service
     namespace: app-namespace
   spec:
     selector:
       app: frontend
     ports:
       - protocol: TCP
         port: 80
         targetPort: 80
     type: LoadBalancer  # Exposes externally
   ```

2. **Apply the YAML**:
   ```
   kubectl apply -f frontend-deployment.yaml
   ```

3. **Get External IP**:
   ```
   kubectl get services -n app-namespace
   ```
   Access the app via the LoadBalancer IP.

### Step 5: Install and Use Portainer for Management and Deployment

Portainer provides a web UI to manage Kubernetes resources, including deploying images.

1. **Deploy Portainer on AKS**:
   Create `portainer.yaml`:

   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: portainer
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: portainer
     namespace: portainer
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: portainer
     template:
       metadata:
         labels:
           app: portainer
       spec:
         containers:
         - name: portainer
           image: portainer/portainer-ce:latest
           args:
           - --admin-password=$2y$05$yourhashedpassword  # Generate hash via 'htpasswd -nb -B admin yourpassword'
           ports:
           - containerPort: 9000
           volumeMounts:
           - mountPath: /data
             name: portainer-data
         volumes:
         - name: portainer-data
           emptyDir: {}
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: portainer-service
     namespace: portainer
   spec:
     selector:
       app: portainer
     ports:
       - protocol: TCP
         port: 9000
         targetPort: 9000
     type: LoadBalancer
   ```

   Apply:
   ```
   kubectl apply -f portainer.yaml
   ```

2. **Access Portainer**:
   Get the service IP:
   ```
   kubectl get services -n portainer
   ```
   Open `http://<external-ip>:9000` in a browser. Log in with admin credentials.

3. **Connect Portainer to AKS**:
   In Portainer UI:
   - Go to "Environments" > Add Environment > Kubernetes.
   - Use the kubeconfig from Step 1 (or upload it).

4. **Deploy/Manage Resources via Portainer**:
   - For front-end deployment: Use Portainer's "Stacks" or "Applications" to deploy YAML files or pull images directly from Docker Hub.
   - Monitor pods, services, and deployments.
   - For the front-end image: In Portainer, go to "Images" > Pull from Docker Hub, then create a new Deployment using the pulled image.
   - Manage back-end similarly by viewing/editing existing resources.

## Testing and Verification

1. **Test Back-End**: Exec into MySQL pod and verify database:
   ```
   kubectl exec -it mysql-0 -n app-namespace -- mysql -u root -p
   ```

2. **Test Front-End**: Curl or browse the front-end service IP to ensure it loads and connects to MySQL.

3. **Scale and Monitor**: Use `kubectl scale` or Portainer to adjust replicas. Monitor via Azure Portal or `kubectl top`.

## Troubleshooting

- **Pod Crashes**: Check logs with `kubectl logs <pod-name> -n app-namespace`.
- **Connection Issues**: Ensure environment variables match and services are correctly named.
- **Portainer Access**: If LoadBalancer doesn't provision, use NodePort temporarily.
- **Azure Costs**: Monitor resource usage to avoid unexpected bills; delete the cluster when not in use with `az aks delete`.

## Cleanup

To remove resources:
```
kubectl delete namespace app-namespace portainer
az aks delete --resource-group myResourceGroup --name myAKSCluster --yes --no-wait
az group delete --name myResourceGroup --yes --no-wait
```

This documentation provides a formal, step-by-step guide to replicate the project. Adjust passwords, names, and sizes for production use. For security, use Secrets for sensitive data instead of plain env vars.