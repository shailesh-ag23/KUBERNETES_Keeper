# 🟪 Keeper With Kubernetes 

Keeper is a production-inspired full-stack notes application built to demonstrate modern **Cloud and DevOps practices**.

The project follows the complete deployment journey of a multi-service application, starting from local development, moving through Docker containerization, and finally deploying the application on **AWS EC2 using a self-managed Kubernetes cluster**.

The application consists of:

- **React (Vite)** – Frontend application.
- **Node.js and Express** – Backend REST API.
- **PostgreSQL** – Relational database.
- **Nginx** – Serves the production frontend and reverse-proxies API requests.
- **Docker** – Containerizes the application services.
- **Amazon ECR** – Stores container images.
- **Kubernetes (kubeadm)** – Orchestrates application workloads.
- **Traefik** – Handles external application traffic through Kubernetes Ingress.

Rather than focusing only on application development, Keeper demonstrates how a multi-service application can be containerized, deployed, scaled, configured, and managed in a cloud environment.

✨ The project also explores Kubernetes networking, persistent storage, workload management, resource allocation, health checks, and automated scaling.

---

## ✨ Architecture Diagram

The application follows a multi-tier architecture where the frontend, backend, and database operate as independent services.

<img width="1400" height="786" alt="image" src="https://github.com/user-attachments/assets/be1daf6c-1345-4a90-9e13-1cf6c7c43edb" />

---

# 🐳 Containerizing the Application

After verifying that the application works correctly in the local development environment, the next step is to containerize each service.

Each service is packaged independently using its own Docker image.

The application consists of:

![React](https://img.shields.io/badge/React-61DAFB?logo=react&logoColor=black)
![Node](https://img.shields.io/badge/Node.js-339933?logo=node.js&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-336791?logo=postgresql&logoColor=white)

Each service can now be built, updated, and deployed independently.


---

# 🟧 Deploying the Containerized Application to AWS

After successfully containerizing and testing the application locally, the next step is to deploy it to AWS.

Amazon EC2 provides the virtual machines required to host the Kubernetes cluster.

Amazon ECR is used to store the Docker images.

The deployment workflow is:

1. Build application images.
2. Create ECR repositories.
3. Authenticate Docker with ECR.
4. Push images to ECR.
5. Provision EC2 instances.
6. Install Kubernetes dependencies.
7. Initialize the Kubernetes control plane.
8. Join worker nodes.
9. Deploy the application.

## 🚩 Step 1: Create Amazon ECR Repositories

Amazon Elastic Container Registry (ECR) is a managed container registry used to store Docker images.

Create separate repositories for the application services.

Example repository names:

```text
dockeper-react-nginx-image
dockeper-node-image
dockeper-postgresql-image
```

These repositories store the images that will later be pulled by Kubernetes.

## 🚩 Step 2: Build the Docker Images

Build the frontend image:

```bash
cd client

docker build -t dockeper-react-nginx-image:1.0 .
```

Build the backend image:

```bash
cd server

docker build -t dockeper-node-image:1.0 .
```

Build the PostgreSQL image:

```bash
docker build -t dockeper-postgresql-image:1.0 \
  -f Dockerfile.postgresql .
```


## 🚩 Step 3: Configure AWS CLI

Install AWS CLI on the machine used to build and push the images.

Configure AWS credentials:

```bash
aws configure
```

Provide:

- AWS Access Key ID.

- AWS Secret Access Key.
- Default AWS Region.
- Output format.

Use an IAM identity with the required ECR permissions.

Avoid embedding access keys in scripts, source code, or GitHub repositories.

## 🚩 Step 4: Authenticate Docker with Amazon ECR

Authenticate Docker:

```bash
aws ecr get-login-password \
  --region "$AWS_REGION" |
docker login \
  --username AWS \
  --password-stdin "$ECR_REGISTRY"
```

A successful authentication produces:

```text
Login Succeeded
```

<img width="1029" height="231" alt="Screenshot 2026-11-14 074149" src="https://github.com/user-attachments/assets/837d49e4-7246-4d1d-ab3e-50199de74478" />


## 🚩 Step 5: Tag the Images

Tag the frontend image:

```bash
docker tag dockeper-react-nginx-image:1.0 \
  "$ECR_REGISTRY/dockeper-react-nginx-image:1.0"
```

Tag the backend image:

```bash
docker tag dockeper-node-image:1.0 \
  "$ECR_REGISTRY/dockeper-node-image:1.0"
```

Tag the PostgreSQL image:

```bash
docker tag dockeper-postgresql-image:1.0 \
  "$ECR_REGISTRY/dockeper-postgresql-image:1.0"
```

## 🚩 Step 6: Push the Images to ECR

Push the frontend image:

```bash
docker push \
  "$ECR_REGISTRY/dockeper-react-nginx-image:1.0"
```

Push the backend image:

```bash
docker push \
  "$ECR_REGISTRY/dockeper-node-image:1.0"
```

Push the PostgreSQL image:

```bash
docker push \
  "$ECR_REGISTRY/dockeper-postgresql-image:1.0"
```

Once the images are pushed, they are available in Amazon ECR.

<img width="1919" height="724" alt="Screenshot 2026-11-14 074558" src="https://github.com/user-attachments/assets/20d15506-ba53-4c51-92f7-9312e5cd8585" />

Kubernetes will pull these images when creating the application Pods.

---

# 🟪 Provisioning the Kubernetes Infrastructure on AWS EC2

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Instead of using a managed Kubernetes service, Keeper uses a self-managed Kubernetes cluster created with kubeadm. This provides practical experience with Kubernetes cluster architecture, control-plane components, worker nodes, container runtimes, and cluster networking.

⚒️ Our approach: Self-managed Kubernetes

We provisioned EC2 instances and used kubeadm to build the cluster. We installed and configured the control-plane components, container runtime, networking plugin, and worker node ourselves.

🟧 Alternative: Amazon EKS

AWS operates the Kubernetes control plane for us. We focus mainly on provisioning or managing worker nodes and deploying our applications. 

AWS runs at least two Kubernetes API server instances and three etcd instances across three Availability Zones. It monitors them and replaces unhealthy control-plane instances. We don't have to provision EC2 instances for the control plane, install Kubernetes on them, or manage etcd ourselves.

✨ Why we chose kubeadm for Dockeper

Our main objective was to understand Kubernetes infrastructure, not just deploy an application onto a managed Kubernetes service.

Using kubeadm gave us hands-on experience with:
- Setting up the Kubernetes control plane and worker nodes.
- Understanding the API Server, Scheduler, Controller Manager, and etcd.
- Installing and configuring containerd.
- Configuring kubelet and bootstrapping nodes.
- Connecting nodes using kubeadm join.
- Installing Flannel as the Container Network Interface (CNI).
- Understanding node networking, Pod networking, taints, and scheduling.
- Troubleshooting cluster-level issues instead of relying on AWS to manage that layer.

💲There was also a cost consideration. EKS has an additional per-cluster fee, whereas our kubeadm cluster does not have an EKS control-plane charge.

## 🚩 Step 1: Create EC2 Instances

Create two Ubuntu EC2 instances.

| Instance | Purpose |
|---|---|
| dockeper-k8s-master | Kubernetes control plane |
| dockeper-k8s-worker | Application workloads |

Both instances should be deployed in the same VPC and should be able to communicate over their private IP addresses.

AMI - Ubuntu Server 24.04 LTS.

Select instance types with sufficient CPU and memory for Kubernetes and the application workloads.

Instance Type - c7i-flex.large

Attach suitable EBS storage to both instances.

Storage - 25 GiB gp3

The worker node requires enough disk space for:

- Container images.
- Container writable layers.
- PostgreSQL persistent data.
- Kubernetes workload data.

### ⚙️ Security group configuration

Kubernetes requires communication between the control plane and worker nodes.

Configure the security group to allow the required traffic.

<img width="1916" height="532" alt="image" src="https://github.com/user-attachments/assets/1bfb682b-24c2-48c5-a688-753b7e62a918" />

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH administration |
| All | All | Communication between Master and Worker nodes in the same security group |
| 32080 | TCP | Traefik ingress |

Control-plane and node-to-node ports should be restricted to the cluster's private network or security-group references.

⚠️ Do not expose etcd, kubelet, or control-plane management ports publicly.

The application should only expose the traffic required by the chosen ingress or service configuration.

## 🚩 Step 2: Connect to the EC2 Instances

Connect to the master:

```bash
ssh -i <YOUR_KEY.pem> ubuntu@<MASTER_PUBLIC_IP>
```

Connect to the worker:

```bash
ssh -i <YOUR_KEY.pem> ubuntu@<WORKER_PUBLIC_IP>
```

The username depends on the AMI being used.

## 🚩 Step 3: Update Both EC2 Instances

Run these commands on both nodes:

```bash
sudo apt update

sudo apt upgrade -y
```

Install basic utilities:

```bash
sudo apt install -y \
  curl \
  wget \
  apt-transport-https \
  ca-certificates \
  gpg
```


## 🚩 Step 4: Configure the Kubernetes Prerequisites

Kubernetes requires a compatible container runtime.

Keeper uses containerd, and the Kubernetes nodes also require swap to be disabled for the standard kubeadm configuration.

Run on both nodes:

```bash
sudo swapoff -a
```

Disable swap permanently by updating `/etc/fstab` and commenting out the swap entry.

```bash
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Load the required kernel modules:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure the required network parameters:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply the settings:

```bash
sudo sysctl --system
```

## 🚩 Step 5: Install containerd

Install containerd on both nodes.

Use the official containerd package source appropriate for your Ubuntu version, then install:

```bash
sudo apt install -y containerd
```

Generate the default configuration:

```bash
sudo mkdir -p /etc/containerd

containerd config default | \
sudo tee /etc/containerd/config.toml
```

Configure systemd cgroup management in containerd.

In `/etc/containerd/config.toml`, ensure:

```toml
SystemdCgroup = true
```

Restart containerd:

```bash
sudo systemctl restart containerd

sudo systemctl enable containerd
```

Verify:

```bash
sudo systemctl status containerd
```

<img width="674" height="90" alt="image" src="https://github.com/user-attachments/assets/4e4c90ec-f45a-460c-be88-0b854c9c4ae2" />


## 🚩 Step 6: Install kubeadm, kubelet and kubectl

Install the Kubernetes packages on both nodes.

Add the official Kubernetes package repository for the selected Kubernetes minor version.

The repository and package versions must match the Kubernetes release being installed.

After configuring the repository:

```bash
sudo apt update

sudo apt install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl
```

Enable kubelet:

```bash
sudo systemctl enable kubelet
```

Verify:

```bash
kubeadm version

kubectl version --client

kubelet --version
```

<img width="1549" height="139" alt="image" src="https://github.com/user-attachments/assets/f37e229c-aec0-4d1f-93fd-d54b18651e7c" />  


✅ At this stage, both EC2 instances have the basic Kubernetes components installed.

---

# ☑️ Setting Up the Kubernetes Cluster

## 🚩 Step 1: Initialize the Control Plane

Run the following command only on the master node.

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<MASTER_PRIVATE_IP>
```

Explanation:

- `kubeadm init` initializes the Kubernetes control plane.
- `--pod-network-cidr` specifies the Pod network range.
- `--apiserver-advertise-address` specifies the private IP address used by the control plane.

☑️ Kubeadm initializes the control-plane components, including:

- API Server.
- Scheduler.
- Controller Manager.
- etcd.

✅ After successful initialization, kubeadm displays a command for joining worker nodes.

Save the generated `kubeadm join` command securely.

## 🚩 Step 2: Configure kubectl

Configure kubectl for the current user on the master node:

```bash
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf \
  $HOME/.kube/config

sudo chown "$(id -u):$(id -g)" \
  $HOME/.kube/config
```

Verify the cluster:

```bash
kubectl get nodes
```

Initially, the master node may show `NotReady` because the Pod network plugin has not yet been installed.

## 🚩 Step 3: Install Flannel CNI

Kubernetes requires a Container Network Interface (CNI) plugin to provide networking between Pods.

Dockeper uses Flannel with the Pod CIDR:

```text
10.244.0.0/16
```

Apply the Flannel manifest from the official Flannel project:

```bash
kubectl apply -f \
  https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

✨ The CNI plugin creates the networking infrastructure required for Pod-to-Pod communication.

Wait for the Flannel Pods:

```bash
kubectl get pods -n kube-flannel
```

<img width="467" height="133" alt="image" src="https://github.com/user-attachments/assets/78df109f-87af-4d06-bb5e-1123972e67e6" />


Check the nodes:

```bash
kubectl get nodes
```

The master node should eventually become `Ready`.

## 🚩 Step 4: Join the Worker Node

Connect to the worker node.

Execute the `kubeadm join` command generated during control-plane initialization.

It follows this structure:

```bash
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

The token and CA certificate hash must be taken from the actual output of `kubeadm init`.

Verify the worker from the master:

```bash
kubectl get nodes -o wide
```

Both nodes should eventually report:

```text
Ready
```

<img width="1162" height="105" alt="image" src="https://github.com/user-attachments/assets/5579dcb8-6d26-4823-aa10-1b83939a5bc1" />

☑️ The Kubernetes cluster is now operational.

## 🚩 Step 5: Understand the Control Plane and Worker Node

The master node manages the Kubernetes cluster.

✨ Its major components are:

- **API Server:** Entry point for Kubernetes API requests.
- **etcd:** Stores cluster state and configuration.
- **Scheduler:** Assigns Pods to suitable worker nodes.
- **Controller Manager:** Runs controllers that maintain the desired cluster state.

The worker node runs application workloads.

✨ Its major components include:

- **Kubelet:** Ensures that assigned Pods are running.
- **Container Runtime:** Runs containers using containerd.
- **Kube-proxy:** Implements Service networking rules.

Verify the system components:

```bash
kubectl get pods -n kube-system
```

Check the nodes:

```bash
kubectl describe nodes
```

The control-plane node uses a `NoSchedule` taint by default, preventing normal application workloads from being scheduled there.

The Keeper application workloads run on the worker node.

---

# 🚀 Deploying Keeper on Kubernetes

After configuring the cluster, the next step is to deploy the application services.

Kubernetes manages the frontend, backend, and PostgreSQL components as independent workloads.

✨ The deployment uses:

- Namespace.
- Deployments.
- StatefulSet.
- Services.
- Persistent Volumes.
- Persistent Volume Claims.
- Kubernetes Secrets.
- Ingress.
- Horizontal Pod Autoscaler.

## 🚩 Step 1: Create a Namespace

Namespaces provide logical separation between Kubernetes resources.

Create the Dockeper namespace:

```bash
kubectl create namespace dockeper
```

Verify:

```bash
kubectl get namespaces
```

All application resources are deployed inside this namespace.

To work with the namespace more conveniently:

```bash
kubectl config set-context --current \
  --namespace=dockeper
```

## 🚩 Step 2: Configure Amazon ECR Authentication

Kubernetes needs permission to pull private container images from Amazon ECR.

ECR authentication tokens expire after approximately 12 hours.

✅ Create an image pull secret:

```bash
kubectl create secret docker-registry ecr-registry \
  --docker-server=<YOUR_ECR_REGISTRY> \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region us-east-1)" \
  --namespace=dockeper \
  --dry-run=client -o yaml | kubectl apply -f -
```

The secret is referenced by application Pods through `imagePullSecrets`.

Example:

```yaml
imagePullSecrets:
  - name: ecr-registry
```

Whenever the ECR authentication token expires, refresh the secret using the same command.

The AWS CLI must be configured with credentials that have the required ECR permissions on the machine where this command is executed.

## 🚩 Step 3: Configure PostgreSQL Credentials

Kubernetes Secrets are used to provide sensitive application configuration.

Create the PostgreSQL credentials:

```bash
kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_USER=<YOUR_DB_USER> \
  --from-literal=POSTGRES_PASSWORD=<YOUR_DB_PASSWORD> \
  --from-literal=POSTGRES_DB=dockeper \
  -n dockeper
```

Verify:

```bash
kubectl get secrets -n dockeper
```

<img width="475" height="96" alt="image" src="https://github.com/user-attachments/assets/0ab81b4e-eabc-4d6e-8896-395fae7c94ce" />


The PostgreSQL and backend workloads reference the secret to obtain their required database configuration.

⚠️ Secrets are base64-encoded by default, not automatically encrypted as a security guarantee. Production environments should also configure suitable encryption at rest and access restrictions.

## 🚩 Step 4: Configure PostgreSQL Persistent Storage

PostgreSQL requires persistent storage to retain database files when a Pod is recreated.

Kubernetes provides two main storage abstractions:

- **Persistent Volume (PV):** Represents the actual storage resource.
- **Persistent Volume Claim (PVC):** Represents an application's request for storage.

Keeper uses local storage on the worker node.

The PostgreSQL data directory is:

```text
/mnt/data/postgres
```

Prepare the directory on the worker:

```bash
sudo mkdir -p /mnt/data/postgres
```

Configure the required ownership and permissions according to the PostgreSQL container image's runtime UID and GID.

The Persistent Volume references the worker's local storage path and uses node affinity to ensure the PostgreSQL Pod is scheduled on the node that contains the data.

The PVC requests the required storage and binds to the Persistent Volume.

Verify:

```bash
kubectl get pv

kubectl get pvc -n dockeper
```

The PVC should eventually show:

```text
STATUS: Bound
```

<img width="1070" height="122" alt="image" src="https://github.com/user-attachments/assets/ccffbe87-43b8-494a-86c8-f8c2fd7be5f8" />


### 🤔 Why persistent storage matters

Without persistent storage, data stored only inside the container's writable layer can be lost when the container is replaced.

With a PV and PVC, PostgreSQL can use storage independently of the lifecycle of its Pod.

⚠️ **Important:** This implementation uses node-local storage. It provides persistence across Pod recreation on the same node, but it is not highly available. If the worker node or its EBS volume becomes unavailable, the database cannot automatically move to another node with the same local data.

## 🚩 Step 5: Deploy PostgreSQL

PostgreSQL is deployed using a StatefulSet.

A StatefulSet is useful for applications that require stable identity and persistent storage.

The PostgreSQL workload uses:

- The PostgreSQL container image stored in ECR.
- Database credentials from Kubernetes Secrets.
- The `postgres-pvc` Persistent Volume Claim.
- Port `5432`.
- A persistent data mount at the PostgreSQL data directory.

The application database is exposed internally through a Kubernetes Service named:

```text
postgres
```

The backend connects using:

```env
DB_HOST=postgres
DB_PORT=5432
```

Apply the PostgreSQL configuration:

```bash
kubectl apply -f <POSTGRES_MANIFEST>
```

Verify the StatefulSet:

```bash
kubectl get statefulsets -n dockeper
```

Verify the Pod:

```bash
kubectl get pods -n dockeper
```

Check the logs:

```bash
kubectl logs -f <POSTGRES_POD_NAME> -n dockeper
```

<img width="969" height="214" alt="image" src="https://github.com/user-attachments/assets/aff25651-1e5d-4db1-a4b2-f8ab8b4a83d5" />


Verify the Service:

```bash
kubectl get services -n dockeper
```

## 🚩 Step 6: Deploy the Backend

The backend is deployed using a Kubernetes Deployment.

The Deployment defines the desired number of replicas and ensures that the required Pods are maintained.

The backend configuration includes:

- Backend image from Amazon ECR.
- Container port `3000`.
- PostgreSQL connection environment variables.
- Kubernetes Secret references.
- ECR image pull secret.
- CPU and memory requests.
- CPU and memory limits.
- Liveness and readiness probes.

The backend connects to PostgreSQL through the internal Service:

```text
postgres:5432
```

The backend is exposed internally through a ClusterIP Service named:

```text
backend
```

This allows other Pods to access the backend without exposing it directly to the internet.

Apply the backend configuration:

```bash
kubectl apply -f <BACKEND_MANIFEST>
```

Verify:

```bash
kubectl get deployments -n dockeper

kubectl get pods -n dockeper

kubectl get services -n dockeper
```

View backend logs:

```bash
kubectl logs -f deployment/backend -n dockeper
```

<img width="679" height="508" alt="image" src="https://github.com/user-attachments/assets/903738c8-6c83-4fc8-b329-9bd98ed080f9" />

The backend should become Ready once its application process and configured readiness probe are successful.

## 🚩 Step 7: Deploy the Frontend

The frontend is deployed using a Kubernetes Deployment.

The frontend container uses the production React build served by Nginx.

The Deployment includes:

- Frontend image from Amazon ECR.
- Nginx container port.
- Resource requests and limits.
- Liveness and readiness probes.
- ECR image pull secret.

The frontend is exposed internally using a ClusterIP Service named:

```text
frontend
```

Apply the frontend configuration:

```bash
kubectl apply -f <FRONTEND_MANIFEST>
```

Verify:

```bash
kubectl get deployments -n dockeper

kubectl get pods -n dockeper

kubectl get services -n dockeper
```

View logs:

```bash
kubectl logs -f deployment/frontend -n dockeper
```

<img width="773" height="382" alt="image" src="https://github.com/user-attachments/assets/04bd4a9c-ae92-4318-815a-e7a13089f17d" />

At this point, the frontend, backend, and PostgreSQL workloads should be running inside Kubernetes.

<img width="607" height="445" alt="image" src="https://github.com/user-attachments/assets/b42e3e1a-bb21-4b59-a0b7-339a0db89814" />


## 🚩 Step 8: Understand Kubernetes Services

Kubernetes Services provide stable networking endpoints for application workloads.

Pods are temporary and can be replaced with new IP addresses.

A Service provides a stable endpoint that continues to route traffic to matching Pods.

✨ Keeper uses ClusterIP Services for internal communication.

| Service | Purpose |
|---|---|
| frontend | Exposes the frontend Pods internally |
| backend | Exposes the backend Pods internally |
| postgres | Exposes PostgreSQL internally |


The backend connects to PostgreSQL through the Service name rather than using a Pod IP address.

Similarly, the frontend's Nginx reverse proxy communicates with the backend Service.

✅ This removes the need to hardcode Pod IP addresses.

---

# 🌐 Kubernetes Ingress with Traefik

Kubernetes Ingress provides a way to manage external HTTP and HTTPS access to application Services.

Instead of exposing every application Service individually, traffic can enter through a common ingress controller.

Keeper uses Traefik as its ingress controller.

## 🚩 Step 1: Install Helm

Helm is a Kubernetes package manager.

It simplifies the installation and management of Kubernetes applications through charts.

Install Helm using the official installation instructions for your operating system.

Verify:

```bash
helm version
```

## 🚩 Step 2: Add the Traefik Helm Repository

Add the official Traefik Helm repository:

```bash
helm repo add traefik https://traefik.github.io/charts
```

Update the repository:

```bash
helm repo update
```

## 🚩 Step 3: Install Traefik

Install Traefik using Helm:

```bash
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace
```

Verify:

```bash
kubectl get pods -n traefik

kubectl get services -n traefik
```

<img width="626" height="221" alt="image" src="https://github.com/user-attachments/assets/3c00d52a-0ccd-4807-84d7-460b295f18d9" />

Traefik watches Kubernetes resources and configures its routing rules according to the Ingress resources.

## 🚩 Step 4: Create an Ingress Resource

The Ingress resource defines how incoming HTTP requests are routed to the application Services.

For Dockeper:

- Incoming application traffic reaches Traefik.
- Traefik routes requests to the frontend Service.
- Nginx serves the React application.
- Nginx forwards `/api/` requests to the backend Service.

Apply the Ingress manifest:

```bash
kubectl apply -f <INGRESS_MANIFEST>
```

Verify:

```bash
kubectl get ingress -n dockeper

kubectl describe ingress -n dockeper
```
<img width="458" height="321" alt="image" src="https://github.com/user-attachments/assets/4d1ac966-9308-40a5-bc02-6f1e35a8593e" />

## 🚩 Step 5: Access the Application

Use the configured NodePort or ingress endpoint to access the application.

For a NodePort configuration, the URL follows this structure:

```text
http://<WORKER_PUBLIC_IP>:<NODE_PORT>
```

Ensure the relevant security-group rule permits the required traffic.

| 32080 | TCP | Traefik ingress |

At this point, external HTTP requests can reach the Keeper application through Traefik.

<img width="1912" height="986" alt="image" src="https://github.com/user-attachments/assets/40fa8e65-cb38-4615-ac51-67148a75591f" />

---

# ❤️‍🩹 Kubernetes Health Management

Kubernetes continuously monitors the state of application Pods.

A container being in the Running state does not necessarily mean that the application inside it is ready to serve requests.

Kubernetes provides health probes to manage this.

## ✨ Liveness Probe

A liveness probe determines whether a container is still functioning. If the liveness probe repeatedly fails, Kubernetes restarts the container.

For Keeper, the backend health endpoint can be used to verify application health.

Example:

```yaml
livenessProbe:
  httpGet:
    path: /api/health
    port: 3000
  initialDelaySeconds: 15
  periodSeconds: 10
```

## ✨ Readiness Probe

A readiness probe determines whether the application is ready to receive traffic.

If the readiness probe fails, Kubernetes removes the Pod from the Service's ready endpoints until it becomes healthy again.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /api/health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
```

The frontend and PostgreSQL workloads also use suitable health checks for their respective containers.

## ✨ Verify Pod Health

Check the Pod status:

```bash
kubectl get pods -n dockeper
```

View detailed information:

```bash
kubectl describe pod <POD_NAME> -n dockeper
```

View container logs:

```bash
kubectl logs <POD_NAME> -n dockeper
```

✅ These commands help identify:

- Image pull failures.
- Failed health probes.
- Container crashes.
- Scheduling issues.
- Resource constraints.
- Application startup errors.

---

# 📊 Kubernetes Resource Management

Kubernetes allows CPU and memory requirements to be defined for individual containers.

This helps the scheduler allocate resources and prevents workloads from consuming unlimited resources.

## ⏳ Resource Requests

Requests specify the minimum amount of CPU and memory that Kubernetes should account for when scheduling a Pod.

For example:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

Here:

- `100m` represents 0.1 CPU cores.
- `128Mi` represents 128 mebibytes of memory.

## ⏳ Resource Limits

Limits define the maximum resources a container can consume.

Example:

```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
```

📉 If a container exceeds its CPU limit, it may be throttled.

❌ If it exceeds its memory limit, it may be terminated with an OOMKilled status.

Resource requests and limits are configured for Keeper's frontend, backend, and PostgreSQL workloads.

## 3. Verify Resource Consumption

Kubernetes resource metrics can be viewed using:

```bash
kubectl top nodes
```

View Pod resource consumption:

```bash
kubectl top pods -n dockeper
```

View individual containers:

```bash
kubectl top pods -n dockeper --containers
```

<img width="519" height="216" alt="image" src="https://github.com/user-attachments/assets/1888f3e8-42e2-404a-8b7e-834e115e93b4" />


These commands require Metrics Server to be installed and functioning.

---

# 📈 Horizontal Pod Autoscaling

One of Kubernetes' important features is Horizontal Pod Autoscaling (HPA).

HPA automatically adjusts the number of Pod replicas based on observed resource utilization.

Keeper uses HPA to scale the backend and frontend workloads based on CPU utilization.

## 🚩 Step 1: Install Metrics Server

Metrics Server collects resource utilization metrics from Kubernetes nodes and exposes them through the Kubernetes Metrics API.

These metrics are used by HPA to make scaling decisions.

Install Metrics Server using its official deployment manifest:

```bash
kubectl apply -f \
  https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

In this self-managed EC2 cluster, kubelet serving certificates may not contain the IP addresses required for normal certificate verification.

For this learning environment, Metrics Server was configured with:

```text
--kubelet-insecure-tls
```

This disables kubelet certificate verification and is a temporary workaround, not a recommended production security configuration.

Verify:

```bash
kubectl get pods -n kube-system
```

Check whether metrics are available:

```bash
kubectl top nodes
```

## 🚩 Step 2: Configure Backend HPA

The backend HPA automatically adjusts the number of backend replicas according to CPU utilization.

Its configuration includes:

- Minimum replicas: 1.
- Maximum replicas: 4.
- CPU utilization target: 60%.

The HPA uses the backend Deployment as its scaling target.

Verify:

```bash
kubectl get hpa -n dockeper
```

## 🚩 Step 3: Configure Frontend HPA

The frontend is also configured with Horizontal Pod Autoscaling.

Its configuration includes:

- Minimum replicas: 1.
- Maximum replicas: 3.
- CPU utilization target: 60%.

Verify:

```bash
kubectl get hpa -n dockeper
```

<img width="689" height="125" alt="image" src="https://github.com/user-attachments/assets/ffd81cd1-fd90-4619-8335-2a7744ce7eca" />


## 🚩 Step 4: Monitor Scaling

Check the HPA status:

```bash
kubectl describe hpa backend-hpa -n dockeper
```
<img width="1159" height="349" alt="image" src="https://github.com/user-attachments/assets/7598bd01-688b-4d09-9dc3-7d2996e475ef" />

View current replicas:

```bash
kubectl get deployments -n dockeper
```

Monitor CPU consumption:

```bash
kubectl top pods -n dockeper
```

✨ When CPU utilization exceeds the configured target and the HPA has sufficient metrics, Kubernetes can increase the number of replicas.

✨ When utilization decreases, Kubernetes can scale the replicas down according to its scaling behavior and stabilization settings.

### 🧪 Test HPA manually

A temporary load-generating workload can be used to generate traffic against the application.

Monitor the HPA while the workload is running:

```bash
kubectl get hpa -n dockeper -w
```

Watch the backend replicas:

```bash
kubectl get deployment backend -n dockeper -w
```

After testing, remove the temporary load generator and allow the HPA to return the Deployment to its minimum replica count.

---

# 🔍 Kubernetes Monitoring and Troubleshooting

Kubernetes provides several commands to inspect the state of the cluster and diagnose application problems.

### View all nodes

```bash
kubectl get nodes -o wide
```

### View all Pods in Dockeper

```bash
kubectl get pods -n dockeper -o wide
```

### View Deployments

```bash
kubectl get deployments -n dockeper
```

### View Services

```bash
kubectl get services -n dockeper
```

### View StatefulSets

```bash
kubectl get statefulsets -n dockeper
```

### View Persistent Volumes

```bash
kubectl get pv
```

### View Persistent Volume Claims

```bash
kubectl get pvc -n dockeper
```

### View Ingress resources

```bash
kubectl get ingress -n dockeper
```

### View HPA

```bash
kubectl get hpa -n dockeper
```

### Describe a Pod

```bash
kubectl describe pod <POD_NAME> -n dockeper
```

### View application logs

```bash
kubectl logs <POD_NAME> -n dockeper
```

Follow logs continuously:

```bash
kubectl logs -f <POD_NAME> -n dockeper
```

### View recent events

```bash
kubectl get events -n dockeper \
  --sort-by=.metadata.creationTimestamp
```

These commands help identify problems involving scheduling, networking, storage, image authentication, resource limits, and application health.

---

# 🛠️ Common Kubernetes Operations

The following commands are useful when managing the Dockeper application.

### Restart a Deployment

```bash
kubectl rollout restart deployment/backend -n dockeper
```

### Check rollout status

```bash
kubectl rollout status deployment/backend -n dockeper
```

### Scale a Deployment manually

```bash
kubectl scale deployment backend \
  --replicas=2 \
  -n dockeper
```

When an HPA manages the Deployment, the HPA may later adjust the replica count according to its configured policy.

### View application configuration

```bash
kubectl get configmaps -n dockeper

kubectl get secrets -n dockeper
```

### Execute a command inside a container

```bash
kubectl exec -it <POD_NAME> \
  -n dockeper -- <COMMAND>
```

### Delete a temporary Pod

```bash
kubectl delete pod <POD_NAME> -n dockeper
```

If the Pod is managed by a Deployment or StatefulSet, Kubernetes creates a replacement to maintain the desired state.

---

# 🟧 Managing AWS Infrastructure and Costs

Since the Kubernetes cluster runs on EC2 instances, infrastructure costs must be considered.

The EC2 instances can be stopped when the cluster is not being used.

Before stopping the instances:

- Confirm that no important application operation is in progress.
- Ensure the PostgreSQL data is stored on persistent EBS-backed storage.
- Understand that local PV data remains tied to the worker node and its volume.
- Remember that stopping EC2 instances does not eliminate EBS storage charges.

After starting the instances again, verify:

```bash
kubectl get nodes

kubectl get pods -A
```

Check whether the application workloads have recovered. If necessary, inspect the kubelet, containerd, and Kubernetes Pod events.

---

# 🧰 Project Technologies

| Category | Technologies |
|---|---|
| Frontend | React, Vite, JavaScript |
| Backend | Node.js, Express |
| Database | PostgreSQL |
| Reverse Proxy | Nginx |
| Containerization | Docker |
| Cloud Provider | AWS |
| Virtual Machines | Amazon EC2 |
| Container Registry | Amazon ECR |
| Container Runtime | containerd |
| Orchestration | Kubernetes |
| Cluster Setup | kubeadm |
| Networking | Flannel CNI, Kubernetes Services |
| Ingress | Traefik |
| Package Management | Helm |
| Storage | EBS-backed local Persistent Volume |
| Scaling | Horizontal Pod Autoscaler |
| Metrics | Metrics Server |
| Operating System | Ubuntu Linux |

---

# 🔐 Security Considerations

The project follows several practices to avoid exposing sensitive configuration.

- Database credentials are supplied through Kubernetes Secrets in the Kubernetes deployment.
- AWS credentials should not be committed to GitHub.
- Kubernetes kubeconfig files should never be published.
- `.env` files containing credentials must be excluded from version control.
- Private ECR repositories require authentication.
- Kubernetes control-plane ports should not be exposed publicly.
- EC2 security groups should allow only required traffic.
- Application Services should remain internal unless external access is required.

Before publishing the repository, verify that no credentials, private keys, tokens, or sensitive configuration files have been committed.

---

# ❇️ Future Enhancements

Dockeper can be extended further by introducing additional DevOps practices.

- **Monitoring and Observability:** Integrate Prometheus and Grafana to visualize cluster and application metrics.
- **CI/CD:** Automate image building, testing, ECR pushes, and Kubernetes deployments using GitHub Actions or Jenkins.
- **Infrastructure as Code:** Provision AWS infrastructure using Terraform.
- **Centralized Logging:** Collect and analyze application and Kubernetes logs.
- **Custom Domain and HTTPS:** Configure a domain and secure application traffic using TLS.
- **High Availability:** Introduce multiple worker nodes and more resilient storage.
- **Secrets Management:** Integrate an external secrets manager for production credentials.

---

# 🎯 Conclusion

Keeper demonstrates the complete journey of taking a multi-service application from local development to deployment on AWS using Kubernetes.

The project begins with a React/Vite frontend, Node.js backend, and PostgreSQL database running locally.

The services are then containerized independently using Docker.

Nginx is introduced to serve the optimized React production build and reverse-proxy API requests to the backend.

The resulting container images are pushed to Amazon ECR and deployed on AWS EC2 using a self-managed Kubernetes cluster provisioned with kubeadm.

The Kubernetes deployment incorporates:

- Independent application Deployments and StatefulSets.
- ClusterIP Services for internal communication.
- Traefik Ingress for external traffic routing.
- Persistent Volumes and Persistent Volume Claims for PostgreSQL data.
- Kubernetes Secrets for database credentials.
- Liveness and readiness probes for workload health.
- CPU and memory requests and limits.
- Horizontal Pod Autoscaling for dynamic workload scaling.
- Metrics Server for resource utilization metrics.

The main objective of Keeper is not simply to run an application inside containers, but to understand how a multi-service application can be packaged, configured, deployed, scaled, and managed in a cloud environment.

The project will continue to evolve by introducing observability, CI/CD automation, infrastructure as code, and additional production-oriented security practices.

---

## 👨‍💻 Author

**Shailesh AG**  
Computer Science Student | Cloud & DevOps Enthusiast

