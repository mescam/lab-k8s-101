# Kubernetes 101 lab

This self-paced lab provides an introduction to Kubernetes using Kind (Kubernetes in Docker), which allows you to run Kubernetes clusters directly within Docker containers. You'll set up a local Kubernetes cluster, deploy a WordPress site using Deployments and Services, and configure persistent storage with Persistent Volume Claims (PVCs).

## Objectives
- Install and configure Kind to run a local Kubernetes cluster.
- Learn to define and manage Kubernetes resources with YAML configuration files:
- - **PersistentVolumeClaim** (PVC): Reserves storage space in a cluster for persistent data.
- - **Deployment**: Manages stateless applications, maintaining app instances and updating them as defined.
- - **Service**: Provides a static address for accessing pods, balancing traffic among them.
- Deploy a multi-container application (WordPress) using a MySQL database.
- Access WordPress from a local machine.

## Prerequisites
- Docker installed on your local machine
- kubectl installed on your local machine

## Section 1: Installing Kind and Creating a Cluster
### Step 1.1: Install Kind
Kind can be installed via binary release from https://github.com/kubernetes-sigs/kind/releases

```
chmod +x ./kind
mv ./kind /usr/local/bin/kind
```

### Step 1.2: Create a Cluster

Create a simple configuration file `kind-config.yaml`

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: tcp # Optional, defaults to tcp
```

Run the following command to create the cluster:

```
kind create cluster --name my-cluster --config kind-config.yaml
```

### Step 1.3: Configure Kubectl
Set kubectl to use my-cluster:

```
kubectl cluster-info --context kind-my-cluster
```

## Section 2: Deploying MySQL
### Step 2.1: Persistent Volume Claim for MySQL

Create a file `mysql-pvc.yaml` to define a PVC:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

This PVC requests 1 GB of disk space and ensures the disk can be written and read by a single node.

### Step 2.2: MySQL Deployment

Create a file `mysql-deployment.yaml`:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:8
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: wordpress
        - name: MYSQL_DATABASE
          value: wordpress
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

This deployment configures a MySQL container, sets environment variables for the root password and database name, and mounts the PVC at /var/lib/mysql.

### Step 2.3: MySQL Service
Create a file `mysql-service.yaml`
```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  type: ClusterIP
```

This service exposes MySQL on port 3306 within the cluster, allowing other applications (like WordPress) to connect to it using the service name mysql.

## Section 3: Deploying Wordpress
### Step 3.1: Persistent Volume Claim for Wordpress
Create a file `wordpress-pvc.yaml` for the WordPress PVC:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
Similar to the MySQL PVC, it reserves 1 GB of persistent storage for WordPress data.

### Step 3.2: WordPress Deployment
Create `wordpress-deployment.yaml`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - image: wordpress:latest
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_PASSWORD
          value: wordpress
        - name: WORDPRESS_DB_USER
          value: root
        - name: WORDPRESS_DB_NAME
          value: wordpress
        ports:
        - containerPort: 80
        volumeMounts:
        - name: wordpress-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-storage
        persistentVolumeClaim:
          claimName: wordpress-pvc
```
This deployment sets up WordPress, linking it to the MySQL service for the database, and mounts the WordPress PVC for web content storage.

### Step 3.3: WordPress Service
Create `wordpress-service.yaml`:

```
apiVersion: v1
kind: Service
metadata:
  name: wordpress
spec:
  type: NodePort
  ports:
    - port: 80
      nodePort: 30000
  selector:
    app: wordpress
```
This service makes WordPress accessible via <Node IP>:30000 and balances traffic across WordPress pods.

## Section 4: Accessing WordPress
### Step 4.1: Apply All Configurations
Apply all the YAML configurations:

```
kubectl apply -f mysql-pvc.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f wordpress-pvc.yaml
kubectl apply -f wordpress-deployment.yaml
kubectl apply -f wordpress-service.yaml
```

### Step 4.2: Access WordPress
Get the IP address of your Kind node:
```
kubectl get nodes -o wide
```
Open a web browser and navigate to http://NODE_IP:30000.
You should now see the WordPress installation page. Complete the installation by creating a username and password.

# Conclusion
In this lab, you have learned how to use Kind to create a local Kubernetes cluster, and you have used Deployments, PVCs, and Services to deploy a WordPress site with a MySQL backend. These components are essential for managing applications and their storage needs efficiently in Kubernetes.





