# Kube-Essentials

A learning project focused on working with **Kubernetes**. The goal is to deploy a full-stack application using the [fastapi/full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template), which includes a **React** frontend, **FastAPI** backend, and **PostgreSQL** database.

## Prerequisites

Before you begin, ensure you have the following tools installed on your machine:

- **Docker**: Required for building and running container images. Follow the installation instructions for your platform:
  - [Docker installation guide](https://docs.docker.com/get-docker/)

- **Kind** (Kubernetes in Docker): Used to create local Kubernetes clusters using Docker. Install Kind by following the official documentation:
  - [Kind installation guide](https://kind.sigs.k8s.io/docs/user/quick-start/)

- **kubectl**: The command-line tool for interacting with your Kubernetes cluster. Install it using the official Kubernetes documentation:
  - [kubectl installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

- **Helm**: A package manager for Kubernetes that simplifies the deployment of applications. Install Helm by following the instructions here:
  - [Helm installation guide](https://helm.sh/docs/intro/install/)

Make sure each of these tools is installed and properly configured before proceeding with the setup of **kube-essentials**.

## Recommendations

For those looking to deepen their understanding of Kubernetes, it’s recommended to take the **CKAD (Certified Kubernetes Application Developer)** course by **Mumshad Mannambeth** if you have an opportunity. This course covers all the essential concepts and provides hands-on labs, which are crucial for mastering Kubernetes.

Additionally, the [official Kubernetes documentation](https://kubernetes.io/docs/home/) is an invaluable resource for learning and referencing all Kubernetes concepts. It provides thorough guides, examples, and best practices for deploying and managing applications in Kubernetes.

Since you will be working frequently with `kubectl`, it's recommended to create an alias for easier command execution. Setting up an alias like `k` can significantly simplify your workflow. You can find the instructions on how to create this alias in the [official Kubernetes documentation](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-autocomplete).

## Cluster Setup

To set up your Kubernetes cluster with **kind** and configure the necessary components, follow the steps below:

### Step 1: Create the Kind Cluster

The **kind** configuration is already prepared with 1 control-plane node and 3 worker nodes: `node-basic`, `node-fast`, and `node-db`. To create the cluster, run:

```bash
kind create cluster --config kind-config.yaml
```

This command will create a local Kubernetes cluster using the `kind-config.yaml` configuration file, which includes the desired node setup.

### Step 2: Install the Nginx Ingress Controller

The Nginx Ingress Controller is required to manage external access to your services within the cluster. To install the Nginx Ingress Controller, apply the following manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

This command deploys the Nginx Ingress Controller, allowing you to define ingress rules for routing traffic to your applications within the cluster.

Make sure to verify the setup by checking the status of the ingress controller pods:

```bash
kubectl get pods -n ingress-nginx
```

The controller pod should be in a Running state before you proceed.

### Step 3: PostgreSQL Provisioning

To set up the **PostgreSQL** database in your Kubernetes cluster, follow the steps below:

**Add the Bitnami Helm repository**:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

**Install PostgreSQL using the Bitnami Helm chart**:

```bash
helm install postgres bitnami/postgresql --version 16.0.3 -n postgres --create-namespace \
  --set architecture=standalone \
  --set "primary.tolerations[0].key=postgres-only" \
  --set "primary.tolerations[0].operator=Exists" \
  --set "primary.tolerations[0].effect=NoSchedule" \
  --set "primary.nodeSelector.kubernetes\.io/hostname=node-db" \
  --set primary.persistence.size=2Gi \
  --set primary.resourcesPreset=none \
  --set primary.resources.requests.cpu=125m \
  --set primary.resources.requests.memory=256Mi \
  --set primary.resources.limits.cpu=250m \
  --set primary.resources.limits.memory=512Mi
```

This command deploys a standalone PostgreSQL instance in the postgres namespace, with resources tailored for local development and a node selector to ensure it runs on the `node-db` node.

> **Note**: While deploying databases inside a Kubernetes cluster can be complex and challenging for production environments, this setup is suitable for learning purposes. In production, it is recommended to use a managed database service or run the database outside the Kubernetes cluster to ensure better data availability, backup, and disaster recovery capabilities.

### Step 4: Prepare the Frontend and Backend Images

To prepare the Docker images for the frontend and backend components and load them into your **kind** cluster, follow these steps:

**Build the Docker images** using the source code from the `0.7.1` release of the FastAPI full-stack template:

```bash
docker build -t frontend:0.7.1 https://github.com/fastapi/full-stack-fastapi-template.git#0.7.1:frontend
```

```bash
docker build -t backend:0.7.1 https://github.com/fastapi/full-stack-fastapi-template.git#0.7.1:backend
```

**Load the Docker images** into the kind cluster. This step ensures that the images are available to your Kubernetes cluster without needing to push them to a remote registry:

```bash
kind load docker-image frontend:0.7.1
```

```bash
kind load docker-image backend:0.7.1
```

### Step 5: Create a Development Namespace

To keep your resources organized and avoid using the default namespace, create a `dev` namespace for deploying your application components:

```bash
kubectl create ns dev
```

### Step 6 (Optional): Rename the Containers

```bash
docker rename kind-control-plane control-plane
```

```bash
docker rename kind-worker node-basic
```

```bash
docker rename kind-worker2 node-fast
```

```bash
docker rename kind-worker3 node-db
```

## Tasks

All you need to do is add the required configuration into **8 files** inside the `manifests` folder. You can add configuration to one file and apply it right away, or you can add multiple configurations and apply them all at once. You can also make changes to existing configurations and reapply them as needed. To apply your changes to the **dev** environment, run the following command:

```bash
kubectl apply -f manifests --recursive -n dev
```

- Create a Deployment named "backend" in `manifests/backend/deployment.yaml`. It should have 1 replica, and the container should use the image `backend:0.7.1`, and expose port `8000`.

- Create a Secret in `manifests/backend/secret.yaml` named `backend-secrets`. Some values are predefined, but you need to generate a `FIRST_SUPERUSER_PASSWORD` and find the `POSTGRES_PASSWORD` from your cluster. Refer to the comments inside the `secret.yaml` file for detailed instructions.

- Update the "backend" Deployment in `manifests/backend/deployment.yaml` to use environment variables from `backend-secrets` so that they are available to the backend container.

- Add an init container to the "backend" Deployment in `manifests/backend/deployment.yaml` to ensure that the backend waits for PostgreSQL to be ready before starting. Use the `cgr.dev/chainguard/wait-for-it` image and configure it correctly to wait for the PostgreSQL service. Research how to use this image and adjust the settings as needed.

- Add a second init container to the "backend" Deployment in `manifests/backend/deployment.yaml` to perform database migrations before the main application starts. Use the `backend:0.7.1` image for this init container, and ensure it uses the same secret that is mounted to the backend container for accessing environment variables. Configure the init container to run the following command: `python app/backend_pre_start.py && alembic upgrade head && python app/initial_data.py`

- Create a Service in `manifests/backend/service.yaml` to expose the "backend" Deployment. You already have all the necessary information to configure this service correctly.

- Apply the configuration you have created so far to the `dev` namespace.

- Verify that the following resources have been created successfully:
  - Pod
  - ReplicaSet
  - Deployment
  - Secret
  - Service

  Use appropriate commands to check the status of each resource and ensure that they are in the expected state before proceeding.

- Create a Deployment named "frontend" in `manifests/frontend/deployment.yaml`. It should have 1 replica, and the container should use the image `frontend:0.7.1`, and expose port `80`.

- Create a Service in `manifests/frontend/service.yaml` to expose the "frontend" Deployment. You already have all the necessary information to configure this service correctly.

- Create an Ingress resource in `manifests/ingress.yaml` with the following specifications:
  - Set the `ingressClassName` to `nginx`.
  - Use `localhost` as the host.
  - Configure the path `/docs` to redirect traffic to the `backend` service.
  - Configure the path `/api` to also redirect traffic to the `backend` service.
  - Configure the root path `/` to redirect traffic to the `frontend` service.

- If you have configured everything properly, you can open `localhost` in your browser and see the application. Try logging in using the credentials from the secret you created and explore the app. Also, check if `localhost/docs` is working to access the API documentation.

  If you encounter any issues, don't worry — troubleshooting is a crucial skill, and solving these challenges will help you learn even more.

- In real-world scenarios, we often need to control where applications are deployed. To practice this, let's establish the following rule for our setup:

  The `backend` should only be deployed on the `fast-node`, while the `frontend` can be deployed on either the `fast-node` or `basic-node` (since the `db-node` is exclusively reserved for PostgreSQL).

  Find a way to update the backend deployment to make it true.

- Understand **resource requests** and **limits** in Kubernetes, which manage how much CPU and memory a container can use. Then, add the following settings to your deployments:

  **Backend**: Requests `100m` CPU, `256Mi` memory; Limits `500m` CPU, `1Gi` memory
  **Frontend**: Requests `75m` CPU, `128Mi` memory; Limits `150m` CPU, `256Mi` memory

- Add **liveness** and **readiness** probes to the `backend` and `frontend` deployments to monitor the health of your services:

  **Frontend**:
  - Liveness: Check `path: /` on `port: 80`, with a 15-second initial delay and a 20-second interval.
  - Readiness: Check `path: /` on `port: 80`, with a 7-second initial delay and a 10-second interval.

  **Backend**:
  - Liveness: Check `path: /api/v1/utils/health-check/` on `port: 8000`, with a 15-second initial delay and a 20-second interval.
  - Readiness: Check `path: /api/v1/utils/health-check/` on `port: 8000`, with a 15-second initial delay and a 10-second interval.

- We'll set up a periodic job to back up our PostgreSQL database, ensuring regular data protection. Start by creating a PersistentVolumeClaim (PVC) in `manifests/pvc.yaml` named `postgres-backup` with `ReadWriteOnce` access, `0.5Gi` of storage, and `standard` as the storage class.

- Create a CronJob in `manifests/cronjob.yaml` named `postgres-backup` to automate daily backups of the PostgreSQL database. The job should:
  - Run every day at midnight (`"0 0 * * *"`).
  - Use the `postgres:17.0` image.
  - Use environment variables from the `backend-secrets` Secret.
  - Store the backups in a volume mounted at `/backups` using the `postgres-backup` PVC.
  - Ensure the job runs on `node-basic` using a node selector.
  - Use the command bellow to make the backup with timespamps

  ```bash
  export TIMESTAMP=$(date +"%Y%m%d%H%M%S") && \
  export PGPASSWORD=$POSTGRES_PASSWORD && \
  pg_dump -h $POSTGRES_SERVER -U $POSTGRES_USER -f /backups/backup-$TIMESTAMP.sql
  ```

- To test the `postgres-backup` CronJob without waiting until midnight, create a one-time Job from the CronJob. This will allow you to trigger a backup immediately and verify that the setup is working correctly. Use `kubectl` to create a job from the existing `postgres-backup` CronJob, ensuring it runs with the same configuration.

- And the last one: try to find the backup file directly on the `node-basic` node, It won't be easy)))

## Final thoughts

Congratulations on making it through this challenging project! Whether you completed all the tasks or tackled just a few, you've taken significant steps toward mastering Kubernetes. Deep diving into a project like this isn't the easy path — it's the right path. It’s the way to truly understand the concepts, work through real-world scenarios, and see the results of your efforts.

This project has introduced you to essential Kubernetes concepts, from managing deployments and secrets to handling persistence and automating tasks with CronJobs. Along the way, you’ve faced real challenges and gained practical skills that will serve you well in future projects.

## CleanUp

After completing the project and testing your setup, make sure to clean up all resources to avoid unnecessary use of local resources:

**Delete the kind cluster** to remove all Kubernetes resources and the cluster itself:
```bash
kind delete cluster
```
