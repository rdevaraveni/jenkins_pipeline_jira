# Jenkins Pipeline for Jira Deployment on Kubernetes

This guide provides step-by-step instructions to install and configure Docker, Kubernetes (kubeadm), Jenkins, and other required tools to execute the Jira deployment pipeline.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Install Docker](#install-docker)
3. [Install Kubernetes with kubeadm](#install-kubernetes-with-kubeadm)
4. [Install kubectl](#install-kubectl)
5. [Install Jenkins](#install-jenkins)
6. [Configure Jenkins for Kubernetes](#configure-jenkins-for-kubernetes)
7. [Add the Pipeline to Jenkins](#add-the-pipeline-to-jenkins)
8. [Execute the Pipeline](#execute-the-pipeline)
9. [Access Jira](#access-jira)

---

## Prerequisites

- A Linux machine (Ubuntu 20.04+ or CentOS 7+) with at least 4 CPU cores, 8GB RAM, and 50GB disk space
- Root or sudo access
- Internet connectivity

---

## Install Docker

### On Ubuntu/Debian

```bash
# Update package index
sudo apt-get update

# Install dependencies
sudo apt-get install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Enable and start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Add user to docker group (optional, to avoid sudo)
sudo usermod -aG docker $USER
newgrp docker
```

### On CentOS/RHEL

```bash
# Install dependencies
sudo yum install -y yum-utils

# Add Docker repository
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker
sudo yum install -y docker-ce docker-ce-cli containerd.io

# Enable and start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

### Verify Docker installation

```bash
docker --version
docker run hello-world
```

---

## Install Kubernetes with kubeadm

### Step 1: Disable swap

```bash
# Check if swap is enabled
free -h

# Disable swap
sudo swapoff -a

# Make it permanent by editing /etc/fstab
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Step 2: Load required kernel modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### Step 3: Configure sysctl parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### Step 4: Install kubeadm, kubelet, and kubectl

```bash
# Update package index
sudo apt-get update

# Install dependencies
sudo apt-get install -y apt-transport-https ca-certificates curl

# Add Kubernetes GPG key
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

# Add Kubernetes repository
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install kubeadm, kubelet, and kubectl
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Hold versions to prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl

# Enable and start kubelet
sudo systemctl enable kubelet
sudo systemctl start kubelet
```

### Step 5: Initialize Kubernetes cluster (on master node)

```bash
# Initialize the cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Copy kubeconfig to user's home
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify cluster
kubectl cluster-info
```

### Step 6: Install CNI plugin (Flannel)

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# Verify all nodes are ready
kubectl get nodes
```

### Step 7: Untaint master node (for single-node cluster)

```bash
# If running single node, remove the taint to allow pod scheduling
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

## Install kubectl

kubectl is usually installed with kubeadm, but if needed:

```bash
# Ubuntu/Debian
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
```

---

## Install Jenkins

### Option 1: Install Jenkins directly on the host

```bash
# Add Jenkins repository key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# Add Jenkins repository
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update package index
sudo apt-get update

# Install Java
sudo apt-get install -y default-jre-headless

# Install Jenkins
sudo apt-get install -y jenkins

# Enable and start Jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins

# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Access Jenkins at http://localhost:8080
```

### Option 2: Deploy Jenkins on Kubernetes (recommended)

```bash
# Create Jenkins namespace
kubectl create namespace jenkins

# Create Jenkins deployment
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: jenkins-data
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: NodePort
  selector:
    app: jenkins
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
EOF

# Access Jenkins at http://localhost:30080
```

---

## Configure Jenkins for Kubernetes

### Step 1: Install required plugins

1. Go to **Manage Jenkins** → **Manage Plugins**
2. Search and install:
   - **Kubernetes plugin**
   - **Google Kubernetes Engine plugin**
   - **Pipeline plugin**
   - **Git plugin**
   - **Credentials plugin**
   - **CloudBees Kubernetes Credentials plugin** (optional)

### Step 2: Add Kubernetes cloud configuration

1. Go to **Manage Jenkins** → **Configure System**
2. Scroll to **Cloud** section
3. Click **New Cloud** → Select **Kubernetes**
4. Configure:
   - **Kubernetes URL**: `https://kubernetes.default.svc.cluster.local` (if Jenkins is in cluster)
   - **Kubernetes Namespace**: `default`
   - **Jenkins URL**: `http://jenkins.jenkins:8080` (if Jenkins is in cluster)
   - **Test Connection**

### Step 3: Create credentials for Jira license and DB password

1. Go to **Manage Jenkins** → **Manage Credentials**
2. Click **Jenkins** (global)
3. Click **Add Credentials** and add:

   **First credential:**
   - **Kind**: Secret text
   - **Secret**: Your Atlassian Jira license string
   - **ID**: `jira-license`
   - **Description**: Jira License

   **Second credential:**
   - **Kind**: Secret text
   - **Secret**: Strong database password (e.g., `JiraD3vP@ss123`)
   - **ID**: `jira-db-password`
   - **Description**: Jira DB Password

### Step 4: Configure kubectl in Jenkins

If Jenkins is NOT in Kubernetes:

```bash
# Create a service account for Jenkins
kubectl create serviceaccount jenkins -n jenkins
kubectl create clusterrolebinding jenkins --clusterrole=cluster-admin --serviceaccount=jenkins:jenkins

# Get the token and certificate
kubectl get secret $(kubectl get secret -n jenkins | grep jenkins-token | awk '{print $1}') -n jenkins -o jsonpath='{.data.token}' | base64 --decode

# Use these in Jenkins Kubernetes plugin configuration
```

---

## Add the Pipeline to Jenkins

### Step 1: Create a new pipeline job

1. Go to Jenkins dashboard
2. Click **New Item**
3. Enter job name (e.g., `jira-deployment`)
4. Select **Pipeline**
5. Click **OK**

### Step 2: Add the Jenkinsfile

In the **Pipeline** section:

- **Definition**: Pipeline script from SCM (if using Git) OR Pipeline script
- If using script directly, paste the `Jenkinsfile` content

**Example inline script:**

```groovy
pipeline {
  agent any

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  parameters {
    string(name: 'JIRA_NAMESPACE', defaultValue: 'jira', description: 'Kubernetes namespace for Jira')
    string(name: 'JIRA_LICENSE', defaultValue: '', description: 'Atlassian Jira license string')
    password(name: 'DB_PASSWORD', defaultValue: '', description: 'Postgres password for Jira')
    string(name: 'JIRA_IMAGE', defaultValue: 'atlassian/jira-software:9.0.0', description: 'Jira Docker image')
    string(name: 'POSTGRES_IMAGE', defaultValue: 'postgres:15', description: 'Postgres Docker image')
  }

  stages {
    stage('Validate parameters') {
      steps {
        script {
          if (!params.JIRA_LICENSE?.trim()) {
            error 'JIRA_LICENSE parameter is required.'
          }
          if (!params.DB_PASSWORD?.trim()) {
            error 'DB_PASSWORD parameter is required.'
          }
          if (!params.JIRA_NAMESPACE?.trim()) {
            error 'JIRA_NAMESPACE parameter is required.'
          }
        }
      }
    }

    stage('Create namespace') {
      steps {
        sh '''
          set -euo pipefail
          kubectl create namespace "${JIRA_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
        '''
      }
    }

    stage('Render secrets and manifests') {
      steps {
        sh '''
          set -euo pipefail

          cat > jira-secret.yaml <<'EOF'
          apiVersion: v1
          kind: Secret
          metadata:
            name: jira-secrets
            namespace: ${JIRA_NAMESPACE}
          type: Opaque
          stringData:
            jira-license: "${JIRA_LICENSE}"
            postgres-password: "${DB_PASSWORD}"
          EOF

          cat > jira-postgres-pvc.yaml <<'EOF'
          apiVersion: v1
          kind: PersistentVolumeClaim
          metadata:
            name: postgres-data
            namespace: ${JIRA_NAMESPACE}
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 20Gi
          EOF

          cat > jira-home-pvc.yaml <<'EOF'
          apiVersion: v1
          kind: PersistentVolumeClaim
          metadata:
            name: jira-home
            namespace: ${JIRA_NAMESPACE}
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 20Gi
          EOF

          cat > jira-postgres-deployment.yaml <<'EOF'
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: postgres
            namespace: ${JIRA_NAMESPACE}
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: postgres
            template:
              metadata:
                labels:
                  app: postgres
              spec:
                containers:
                  - name: postgres
                    image: ${POSTGRES_IMAGE}
                    env:
                      - name: POSTGRES_DB
                        value: jiradb
                      - name: POSTGRES_USER
                        value: jira
                      - name: POSTGRES_PASSWORD
                        valueFrom:
                          secretKeyRef:
                            name: jira-secrets
                            key: postgres-password
                    ports:
                      - containerPort: 5432
                    volumeMounts:
                      - name: postgres-data
                        mountPath: /var/lib/postgresql/data
                volumes:
                  - name: postgres-data
                    persistentVolumeClaim:
                      claimName: postgres-data
          EOF

          cat > jira-postgres-service.yaml <<'EOF'
          apiVersion: v1
          kind: Service
          metadata:
            name: postgres
            namespace: ${JIRA_NAMESPACE}
          spec:
            selector:
              app: postgres
            ports:
              - port: 5432
                targetPort: 5432
          EOF

          cat > jira-deployment.yaml <<'EOF'
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: jira
            namespace: ${JIRA_NAMESPACE}
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: jira
            template:
              metadata:
                labels:
                  app: jira
              spec:
                containers:
                  - name: jira
                    image: ${JIRA_IMAGE}
                    ports:
                      - containerPort: 8080
                    env:
                      - name: JIRA_HOME
                        value: /var/atlassian/application-data/jira
                      - name: DB_TYPE
                        value: postgresql
                      - name: DB_HOST
                        value: postgres
                      - name: DB_PORT
                        value: "5432"
                      - name: DB_NAME
                        value: jiradb
                      - name: DB_USER
                        value: jira
                      - name: DB_PASSWORD
                        valueFrom:
                          secretKeyRef:
                            name: jira-secrets
                            key: postgres-password
                      - name: JIRA_LICENSE
                        valueFrom:
                          secretKeyRef:
                            name: jira-secrets
                            key: jira-license
                    volumeMounts:
                      - mountPath: /var/atlassian/application-data/jira
                        name: jira-home
                volumes:
                  - name: jira-home
                    persistentVolumeClaim:
                      claimName: jira-home
          EOF

          cat > jira-service.yaml <<'EOF'
          apiVersion: v1
          kind: Service
          metadata:
            name: jira
            namespace: ${JIRA_NAMESPACE}
          spec:
            type: ClusterIP
            selector:
              app: jira
            ports:
              - port: 8080
                targetPort: 8080
          EOF
        '''
      }
    }

    stage('Apply Jira stack') {
      steps {
        sh '''
          set -euo pipefail
          kubectl apply -f jira-secret.yaml
          kubectl apply -f jira-postgres-pvc.yaml
          kubectl apply -f jira-home-pvc.yaml
          kubectl apply -f jira-postgres-deployment.yaml
          kubectl apply -f jira-postgres-service.yaml
          kubectl apply -f jira-deployment.yaml
          kubectl apply -f jira-service.yaml
        '''
      }
    }

    stage('Wait for readiness') {
      steps {
        sh '''
          set -euo pipefail
          kubectl -n "${JIRA_NAMESPACE}" rollout status deployment/postgres --timeout=10m
          kubectl -n "${JIRA_NAMESPACE}" rollout status deployment/jira --timeout=15m
          kubectl -n "${JIRA_NAMESPACE}" get pods
        '''
      }
    }
  }

  post {
    failure {
      sh '''
        echo "Deployment failed. Current resource state in ${JIRA_NAMESPACE}:"
        kubectl -n "${JIRA_NAMESPACE}" get all
      '''
    }
  }
}
```

### Step 3: Save and run

1. Click **Save**
2. Click **Build with Parameters** (if parameters are configured)

---

## Execute the Pipeline

### Step 1: Prepare credentials

1. Get your Atlassian Jira license from https://my.atlassian.com
2. In Jenkins, go to **Credentials** → add your license as Secret text with ID `jira-license`

### Step 2: Run the pipeline

1. Go to your pipeline job
2. Click **Build with Parameters**
3. Fill in:
   - `JIRA_NAMESPACE`: `jira` (or your custom namespace)
   - `JIRA_LICENSE`: Your Atlassian license string
   - `DB_PASSWORD`: Strong password for PostgreSQL
   - `JIRA_IMAGE`: `atlassian/jira-software:9.0.0` (or version of choice)
   - `POSTGRES_IMAGE`: `postgres:15`
4. Click **Build**

### Step 3: Monitor the build

- Watch logs in real-time
- Verify each stage completes successfully

---

## Access Jira

### Via port-forward (if service is ClusterIP)

```bash
kubectl -n jira port-forward svc/jira 8080:8080
# Access at http://localhost:8080
```

### Via NodePort (if service is NodePort)

```bash
# Get node IP
kubectl get nodes -o wide

# Get service port
kubectl -n jira get svc jira

# Access at http://<node-ip>:<nodeport>
```

### Initial setup

1. Open Jira in browser
2. Follow the setup wizard
3. Enter your license when prompted
4. Create an admin account
5. Configure Jira

---

## Troubleshooting

### Jenkins cannot connect to Kubernetes

```bash
# Check kubeconfig
cat ~/.kube/config

# Verify cluster connectivity
kubectl cluster-info
kubectl get nodes
```

### Jira pod fails to start

```bash
# Check pod logs
kubectl -n jira logs deployment/jira

# Check pod events
kubectl -n jira describe pod <jira-pod-name>

# Check resource usage
kubectl top nodes
kubectl top pods -n jira
```

### PostgreSQL connection issues

```bash
# Test database connection
kubectl -n jira exec -it deployment/postgres -- psql -U jira -d jiradb -c "SELECT 1;"
```

### Pipeline secrets not working

- Verify credentials are correctly named in Jenkins
- Check that Jenkins has permission to read secrets
- Verify secret YAML syntax is correct

---

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [Atlassian Jira on Kubernetes](https://www.atlassian.com/software/jira)
- [Docker Documentation](https://docs.docker.com/)

---

## Support

For issues or questions:
1. Check logs: `kubectl logs -n jira <pod-name>`
2. Verify all prerequisites are installed
3. Ensure sufficient system resources
4. Review Jenkins pipeline output
