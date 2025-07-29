# User Management Application

## Project Overview

The User Management Application is a full-stack web application designed to manage user data with CRUD operations (Create, Read, Update, Delete). The front-end, built with HTML, CSS, and JavaScript, provides a responsive, user-friendly interface with smooth animations. The back-end, powered by Node.js, Express, and MySQL, handles data operations. The application is deployed on a Kubernetes cluster using a blue-green deployment strategy for zero-downtime updates, with MySQL as the persistent data store. Role-Based Access Control (RBAC) secures the Kubernetes environment, and a Jenkins pipeline automates the build, test, security scanning, and deployment processes.

### Features
- **Add Users**: Create users with name, email, and role (User or Admin).
- **View Users**: Display a list of all users with their details.
- **Edit Users**: Update user information via an intuitive interface.
- **Delete Users**: Remove users from the database.
- **Responsive UI**: Minimalistic design with smooth animations.
- **Blue-Green Deployment**: Ensures zero-downtime updates with blue and green environments.
- **Kubernetes RBAC**: Secures deployment with fine-grained access controls.
- **CI/CD Pipeline**: Automates code quality checks, security scans, and deployments using Jenkins.

## Architecture

The application is deployed on a Kubernetes cluster (specifically an EKS cluster, as indicated in the `Jenkinsfile`) with the following components:

1. **Node.js Application**:
   - **Blue Deployment** (`app-deployment-blue.yml`): Runs 2 replicas of the `zookl0/nodejs-mysql:blue` image on port 5000 with resource limits (CPU: 500m, Memory: 512Mi) and a liveness probe.
   - **Green Deployment** (`app-deployment-green.yml`): Runs 2 replicas of the `zookl0/nodejs-mysql:green` image for rolling updates.
   - **Service** (`app-service.yml`): A LoadBalancer service routes external traffic from port 80 to the application’s port 5000, initially targeting the blue deployment.
   - **Environment Configuration**:
     - **ConfigMap** (`configmap.yml`): Specifies the MySQL host (`mysql`).
     - **Secret** (`secret.yml`): Stores base64-encoded credentials (`DB_USER: root`, `DB_PASSWORD: Test@123`, `DB_NAME: test_db`).

2. **MySQL Database**:
   - **Deployment** (`mysql-deployment.yml`): Runs a single MySQL 5.7 instance with a headless ClusterIP service on port 3306 and a liveness probe.
   - **Persistent Storage**:
     - **PersistentVolume** (`pv.yml`): A 1Gi volume using `hostPath` at `/mnt/data`.
     - **PersistentVolumeClaim** (`pvc.yml`): Requests 1Gi of storage, bound to the PV via the `mysql-storage` label.

3. **RBAC Configurations**:
   - **ServiceAccount** (`sa.yml`): Defines `nodejs-app-sa` in the `webapps` namespace.
   - **Role** (`role.yml`): Grants permissions in the `webapps` namespace for pods, secrets, configmaps, deployments, ingresses, and more.
   - **ClusterRole** (`cluster-role.yml`): Grants cluster-wide permissions for managing persistent volumes.
   - **RoleBinding** (`role-binding.yml`): Binds the `nodejs-app-role` to the `nodejs-app-sa` ServiceAccount.
   - **ClusterRoleBinding** (`cluster-role-binding.yml`): Binds the `nodejs-app-cluster-role` to the `nodejs-app-sa` ServiceAccount.
   - **Secret** (`secret.yml`): Provides a non-expiring token for `nodejs-app-sa`.

4. **Dockerfile** (`Dockerfile-custom`): Builds the Node.js application image, producing `zookl0/nodejs-mysql:blue` and `zookl0/nodejs-mysql:green` for blue-green deployments.

## CI/CD Pipeline

The Jenkins pipeline, defined in the `Jenkinsfile`, automates the build, test, security scanning, and deployment processes. It runs on a Jenkins agent labeled `Agent-1` and uses the following parameters:
- `DEPLOY_ENV`: Choice between `blue` or `green` to select the deployment environment.
- `DOCKER_TAG`: Choice between `blue` or `green` to select the Docker image tag.
- `SWITCH_TRAFFIC`: Boolean to toggle traffic switching between blue and green environments.

### Environment Variables
- `SCANNER_HOME`: Path to the SonarQube scanner tool (`sonar-scanner`).
- `IMAGE_NAME`: Docker image name (`zookl0/nodejs-mysql`).
- `KUBE_NAMESPACE`: Kubernetes namespace (`webapps`).
- `TAG`: Docker image tag, derived from `DOCKER_TAG` parameter.

### Pipeline Stages
1. **GitCheckOut**:
   - Clones the `main` branch from `https://github.com/tajroshith/Nodejs-Mysql.git` using `Github_Creds` credentials.
2. **SonarQube-Analysis**:
   - Runs SonarQube analysis with `sonar-scanner` for code quality, using the `sonar-server` environment.
   - Configures `sonar.projectKey` and `sonar.projectName` as `Nodejs-Mysql`.
3. **Wait-for-Quality-Gate**:
   - Waits up to 5 minutes for the SonarQube Quality Gate result, without aborting the pipeline on failure.
4. **Trivy-fs-scan**:
   - Performs a filesystem scan with Trivy, generating a report (`trivy-fs-report.html`).
5. **DockerBuild**:
   - Logs into Docker Hub using `Docker_hub_pwd` credentials.
   - Builds the Docker image using `Dockerfile-custom`, tagged as `zookl0/nodejs-mysql:<TAG>`.
6. **Trivy-image-scan**:
   - Scans the built Docker image with Trivy, generating a report (`trivy-image-report.html`).
7. **DockerPush**:
   - Pushes the Docker image to Docker Hub using `Docker_hub_pwd` credentials.
8. **Deploy to k8s - app-service**:
   - Applies `app-service.yml` to the `webapps` namespace if the service doesn’t exist, using `K8-token` credentials for the EKS cluster.
9. **Deploy to k8s ALL**:
   - Applies Kubernetes manifests (`secret.yml`, `configmap.yml`, `pv.yml`, `pvc.yml`, `mysql-deployment.yml`, and either `app-deployment-blue.yml` or `app-deployment-green.yml` based on `DEPLOY_ENV`) to the `webapps` namespace.
10. **Switch Traffic**:
    - If `SWITCH_TRAFFIC` is true, updates the `app-service` selector to route traffic to the specified `DEPLOY_ENV` (blue or green).
11. **Verify Deployments**:
    - Checks the status of pods with the specified `version` label and the `app-service` service in the `webapps` namespace.

The pipeline connects to an EKS cluster (`<Cluster-Name>`) at `<Cluster-Endpoint>` using `service-token` credentials.

## Prerequisites

- **Node.js**: Version 12.x or higher.
- **MySQL**: Version 5.7 or higher.
- **Docker**: For building and running containers.
- **Kubernetes**: An EKS cluster or equivalent.
- **kubectl**: Kubernetes command-line tool.
- **Jenkins**: With plugins for Git, Docker, Kubernetes, SonarQube, and Trivy.
- **Git**: To clone the repository.
- **SonarQube**: For code quality analysis.
- **Trivy**: For security scanning.
- **Docker Hub**: Account for pushing images.
- **EKS Cluster**: Configured with `K8-token` credentials.

## Setup Instructions

### 1. Clone the Repository
```bash
git clone https://github.com/tajroshith/Nodejs-Mysql.git
cd Nodejs-Mysql
```

### 2. Set Up MySQL Locally (Optional for Development)
For local testing without Kubernetes:
1. Update the package index:
   ```bash
   sudo apt update
   ```
2. Install MySQL:
   ```bash
   sudo apt install mysql-server
   ```
3. Log in to MySQL as root:
   ```bash
   sudo mysql -u root
   ```
4. Set a password:
   ```sql
   ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Test@123';
   FLUSH PRIVILEGES;
   exit;
   ```
5. Log in again:
   ```bash
   sudo mysql -u root -p
   ```
6. Create the database and table:
   ```sql
   CREATE DATABASE test_db;
   USE test_db;
   CREATE TABLE users (
       id INT AUTO_INCREMENT PRIMARY KEY,
       name VARCHAR(255) NOT NULL,
       email VARCHAR(255) NOT NULL UNIQUE,
       role ENUM('Admin', 'User') NOT NULL
   );
   ```

### 3. Configure and Run the Client
1. Navigate to the client folder:
   ```bash
   cd client
   ```
2. Install dependencies:
   ```bash
   npm install
   ```
3. Build the client:
   ```bash
   npm run build
   ```

### 4. Configure and Run the Server
1. Navigate to the server folder:
   ```bash
   cd server
   ```
2. Install dependencies:
   ```bash
   npm install
   ```
3. Start the server:
   ```bash
   npm start
   ```
   The server runs on `http://localhost:5000`.

### 5. Set Up Jenkins
1. Install Jenkins and required plugins (Git, Docker, Kubernetes, SonarQube, Pipeline).
2. Configure credentials:
   - `Github_Creds`: For accessing the GitHub repository.
   - `Docker_hub_pwd`: For Docker Hub login.
   - `K8-token`: For EKS cluster access.
3. Set up SonarQube:
   - Configure a SonarQube server instance named `sonar-server`.
   - Install the `sonar-scanner` tool in Jenkins.
4. Install Trivy on the Jenkins agent (`Agent-1`).
5. Create a pipeline job pointing to the `Jenkinsfile` in the repository.
6. Run the pipeline, selecting `DEPLOY_ENV`, `DOCKER_TAG`, and `SWITCH_TRAFFIC` as needed.

### 6. Build the Docker Image (Manual, if not using Jenkins)
1. Build the blue image:
   ```bash
   docker build -t zookl0/nodejs-mysql:blue -f Dockerfile-custom .
   ```
2. Build the green image (for updates):
   ```bash
   docker build -t zookl0/nodejs-mysql:green -f Dockerfile-custom .
   ```
3. Push images to Docker Hub:
   ```bash
   docker push zookl0/nodejs-mysql:blue
   docker push zookl0/nodejs-mysql:green
   ```

### 7. Deploy to Kubernetes (Manual, if not using Jenkins)
1. Create the `webapps` namespace:
   ```bash
   kubectl create namespace webapps
   ```
2. Apply the manifests in order:
   ```bash
   kubectl apply -f pv.yml
   kubectl apply -f pvc.yml
   kubectl apply -f configmap.yml
   kubectl apply -f secret.yml
   kubectl apply -f mysql-deployment.yml
   kubectl apply -f sa.yml
   kubectl apply -f cluster-role.yml
   kubectl apply -f role.yml
   kubectl apply -f cluster-role-binding.yml
   kubectl apply -f role-binding.yml
   kubectl apply -f secret.yml
   kubectl apply -f app-deployment-blue.yml
   kubectl apply -f app-service.yml
   ```
3. Verify deployments:
   ```bash
   kubectl get pods -n webapps
   kubectl get svc -n webapps
   ```

### 8. Blue-Green Deployment
To switch to the green deployment:
1. Update `app-service.yml` to change `version: blue` to `version: green`.
2. Apply the updated service:
   ```bash
   kubectl apply -f app-service.yml
   ```
3. Verify the green deployment:
   ```bash
   kubectl get pods -n webapps
   ```
4. Scale down the blue deployment if stable:
   ```bash
   kubectl scale deployment app-blue-dep -n webapps --replicas=0
   ```

## RBAC Configuration

- **ServiceAccount** (`nodejs-app-sa`): Used by application pods to interact with the Kubernetes API.
- **Role** (`nodejs-app-role`): Grants permissions in the `webapps` namespace for pods, secrets, configmaps, deployments, ingresses, and more.
- **ClusterRole** (`nodejs-app-cluster-role`): Grants cluster-wide permissions for persistent volumes.
- **RoleBinding** and **ClusterRoleBinding**: Bind the ServiceAccount to the Role and ClusterRole.
- **Secret** (`mysecretname`): Provides a non-expiring token for the ServiceAccount.

## Usage

1. Access the application via the LoadBalancer service’s external IP:
   ```bash
   kubectl get svc app-service -n webapps -o wide
   ```
   Navigate to `http://<external-ip>` in a browser.
2. **Add a User**: Enter name, email, and role, then click "Add User."
3. **View Users**: View the user list below the form.
4. **Edit a User**: Click "Edit" to modify user details.
5. **Delete a User**: Click "Delete" to remove a user.

## Troubleshooting

- **Pod Failures**: Check logs:
  ```bash
  kubectl logs <pod-name> -n webapps
  ```
- **MySQL Issues**: Verify ConfigMap (`DB_HOST`) and Secret values.
- **Service Access**: Ensure the LoadBalancer has an external IP.
- **RBAC Errors**: Confirm Role and ClusterRole permissions.
- **Jenkins Failures**: Check pipeline logs for errors in SonarQube, Trivy, Docker, or Kubernetes stages.
- **SonarQube Issues**: Ensure the `sonar-server` is accessible and configured correctly.
- **Trivy Scan Failures**: Review `trivy-fs-report.html` or `trivy-image-report.html` for vulnerabilities.

## License

This project is licensed under the MIT License. See the `LICENSE` file in the repository.

## References

- Kubernetes ServiceAccount Token: [https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#:~:text=To%20create%20a%20non%2Dexpiring,with%20that%20generated%20token%20data.)
