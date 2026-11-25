# Building an End-to-End CI/CD Pipeline from Scratch: Jenkins, Harbor, Kubernetes & GitLab

Most DevOps tutorials stop at a "Hello World" pipeline or assume a pre-configured cloud environment. In real-world on-premises or private-cloud setups, however, building a complete delivery pipeline means wiring together multiple self-hosted components—and solving the networking, permissions, and runtime mismatches that happen along the way.

In this step-by-step guide, we will build a complete, automated CI/CD workflow across five progressive projects:

1. **Project 1:** Deploy Jenkins on Ubuntu & Build a Python CI Pipeline
2. **Project 2:** Containerize the Application with Docker in Jenkins
3. **Project 3:** Deploy Harbor Private Registry & Push Versioned Images
4. **Project 4:** Provision a 3-Node Kubernetes Cluster (`kubeadm`) & Integrate Harbor
5. **Project 5:** Automate Kubernetes Deployments & GitLab SCM Triggers

Every phase includes **architectural theory**, **step-by-step commands**, **real troubleshooting fixes**, and a **verification checkpoint** before moving to the next milestone.

---

## Infrastructure Topology & Lab Trade-Offs

To keep build workloads isolated from application runtime workloads, we separate the **CI/Registry plane** from the **Kubernetes runtime plane**:

| Node Role | Hostname Placeholder | IP Placeholder | OS & Specs | Responsibilities |
| :--- | :--- | :--- | :--- | :--- |
| **CI & Registry VM** | `ci-registry-vm` | `<JENKINS_HARBOR_IP>` | Ubuntu 22.04 (2 vCPU, 4 GB RAM) | Jenkins Controller (`:8080`), Docker Engine, Harbor Registry (`:80`), `kubectl` |
| **K8s Control Plane** | `k8s-master` | `<K8S_MASTER_IP>` | Ubuntu 22.04 (2 vCPU, 2–4 GB RAM) | API Server (`:6443`), `etcd`, Scheduler, Controller Manager, Calico CNI |
| **K8s Worker 1** | `k8s-worker1` | `<K8S_WORKER1_IP>` | Ubuntu 22.04 (2 vCPU, 2–4 GB RAM) | `kubelet`, `containerd` v2.x, Application Pods, NodePort Traffic |
| **K8s Worker 2** | `k8s-worker2` | `<K8S_WORKER2_IP>` | Ubuntu 22.04 (2 vCPU, 2–4 GB RAM) | `kubelet`, `containerd` v2.x, Application Pods, NodePort Traffic |

> **Lab vs. Production Note:** In this lab setup, Harbor runs over HTTP on the same VM as Jenkins to conserve memory, and Kubernetes pulls images using admin credentials. In production, always host Harbor on a dedicated node with TLS/HTTPS enabled and use scoped, pull-only Robot Accounts.

---

## Project 1: Deploy Jenkins & Run a Python CI Pipeline

### Architectural Theory
Jenkins operates on a **Controller–Agent** architecture powered by the Java Virtual Machine (JVM). The Controller manages job definitions, credentials, plugin states, and build scheduling, while execution steps run inside workspace directories on the host or dedicated agents. On Ubuntu/Debian hosts, `apt` verifies package authenticity using cryptographic GPG keys stored in `/etc/apt/keyrings/` to prevent tampered packages from entering your build server.

### Step 1.1: Install Java 21 and Jenkins
Run the following commands on `<JENKINS_HARBOR_IP>`:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y fontconfig openjdk-21-jre
java -version
```

Add the official Jenkins repository using the **2026 GPG signing key** and install Jenkins:

```bash
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```

> **Troubleshooting Note (`NO_PUBKEY`):** If you use an older `jenkins.io-2023.key`, `apt update` will fail with `GPG error: ... NO_PUBKEY 7198F4B714ABFC68` because Jenkins rotated its signing key in 2026. Replacing the file with `jenkins.io-2026.key` resolves the signature mismatch immediately.

Unlock Jenkins in your browser at `http://<JENKINS_HARBOR_IP>:8080` using the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Install the suggested plugins and create your admin user.

### Step 1.2: Create the Sample Python Flask Application
Create a new Git repository named `hello-jenkins` with the following structure:

```text
hello-jenkins/
├── app.py
├── requirements.txt
├── test_app.py
└── Jenkinsfile
```

**`app.py`**
```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/")
def home():
    return jsonify({
        "message": "Hello from Jenkins CI/CD!",
        "status": "success"
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**`requirements.txt`**
```text
flask
pytest
```

**`test_app.py`**
```python
from app import app

def test_home():
    client = app.test_client()
    response = client.get("/")
    assert response.status_code == 200
    assert response.json["status"] == "success"
```

### Step 1.3: Install `python3-venv` & Create Initial `Jenkinsfile`
Running Python dependencies inside a virtual environment (`venv`) isolates builds from system-wide OS packages. On minimal Ubuntu installs, `ensurepip` is omitted by default, so install the `venv` package on the Jenkins VM first:

```bash
sudo apt update && sudo apt install -y python3.10-venv
```

Create your initial **`Jenkinsfile`**:

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv venv
                    ./venv/bin/pip install -r requirements.txt
                '''
            }
        }
        stage('Run Tests') {
            steps {
                sh './venv/bin/pytest -v'
            }
        }
    }
}
```

### Verification Checkpoint 1
1. Push the files to your Git repository (`main` branch).
2. In Jenkins, click **New Item** -> enter `hello-jenkins` -> select **Pipeline** -> choose **Pipeline script from SCM** -> point to your Git repository URL and `Jenkinsfile`.
3. Click **Build Now** and confirm all three stages (`Checkout`, `Install Dependencies`, `Run Tests`) pass with `1 passed` in the console output.

---

## Project 2: Build Docker Images with Jenkins

### Architectural Theory
Jenkins executes pipeline shell commands under the unprivileged `jenkins` Linux service account. Because the Docker Engine listens on a root-owned Unix socket (`/var/run/docker.sock`), the `jenkins` user cannot build images until it is added to the `docker` system group.

### Step 2.1: Install Docker & Grant Jenkins Permissions
On `<JENKINS_HARBOR_IP>`:

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker

# Allow the jenkins user to communicate with the Docker daemon
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# Verify Docker access as the jenkins user
sudo -u jenkins docker version
```

### Step 2.2: Add a `Dockerfile` and Update `Jenkinsfile`
Add a **`Dockerfile`** to the root of your `hello-jenkins` repository:

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

Update your **`Jenkinsfile`** to build and verify the container image:

```groovy
pipeline {
    agent any
    environment {
        IMAGE_NAME = 'hello-jenkins'
        IMAGE_TAG  = '1.0'
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv venv
                    ./venv/bin/pip install -r requirements.txt
                '''
            }
        }
        stage('Run Tests') {
            steps {
                sh './venv/bin/pytest -v'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }
        stage('Verify Docker Image') {
            steps {
                sh 'docker images hello-jenkins'
            }
        }
    }
}
```

### Verification Checkpoint 2
1. Commit and push `Dockerfile` and `Jenkinsfile` to Git.
2. Trigger **Build Now** in Jenkins.
3. On the Jenkins VM, verify the image exists and test running it briefly:
   ```bash
   sudo docker images
   sudo docker run --rm -d --name test-flask -p 5000:5000 hello-jenkins:1.0
   curl http://localhost:5000
   sudo docker stop test-flask
   ```

---

## Project 3: Deploy Harbor Private Registry & Push Images

### Architectural Theory
Keeping container images locally on a single build server prevents multi-node clusters from pulling them. **Harbor** provides a private, CNCF-compliant OCI registry backed by Nginx, PostgreSQL, Redis, and role-based access control (RBAC). Instead of giving CI pipelines full administrator credentials, Harbor supports scoped **Robot Accounts** restricted solely to repository `push` and `pull` permissions.

### Step 3.1: Install Docker Compose & Configure Harbor
On `<JENKINS_HARBOR_IP>`:

```bash
sudo apt update && sudo apt install -y docker-compose-v2
docker compose version

cd /opt
sudo wget https://github.com/goharbor/harbor/releases/download/v2.14.0/harbor-online-installer-v2.14.0.tgz
sudo tar -xzf harbor-online-installer-v2.14.0.tgz
cd /opt/harbor

sudo cp harbor.yml.tmpl harbor.yml
sudo nano harbor.yml
```

Update the following fields in `harbor.yml` (comment out the `https:` block for this HTTP lab):

```yaml
hostname: <JENKINS_HARBOR_IP>
http:
  port: 80

# https:
#   port: 443
#   certificate: /your/certificate/path
#   private_key: /your/private/key/path

harbor_admin_password: <STRONG_HARBOR_PASSWORD>
```

Run the installer:

```bash
sudo ./install.sh
```

> **Troubleshooting Note (`connection refused` on Port 80):** On memory-constrained VMs (~4 GB RAM), Harbor's installer may exit early with only `harbor-log` running. Check `sudo docker compose ps` inside `/opt/harbor`. If core services (`nginx`, `core`, `portal`, `registry`) are missing, bring up the full stack manually with `sudo docker compose up -d`.

### Step 3.2: Configure Docker Insecure Registry & Harbor Project
Because Docker defaults to HTTPS for remote registries, configure the Docker daemon on `<JENKINS_HARBOR_IP>` to trust your HTTP registry:

```bash
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "insecure-registries": ["<JENKINS_HARBOR_IP>"]
}
EOF

sudo systemctl restart docker
sudo systemctl restart jenkins
```

Next, open `http://<JENKINS_HARBOR_IP>` in your browser, log in as `admin`, and:
1. Go to **Projects** -> **New Project** -> name it `hello-jenkins` (Access Level: **Public** or **Private**).
2. Open the `hello-jenkins` project -> **Robot Accounts** -> **New Robot Account** -> name it `jenkins-pusher` with **Push** and **Pull** permissions, and copy the generated secret.
3. In Jenkins, go to **Manage Jenkins** -> **Credentials** -> **System** -> **Global credentials** -> **Add Credentials** (**Username with password**), enter the robot account name and secret, and set the ID to `harbor-credentials`.

### Step 3.3: Update `Jenkinsfile` to Push Versioned Tags
Using Jenkins' built-in `${BUILD_NUMBER}` variable ensures every build produces an immutable, traceable image tag:

```groovy
pipeline {
    agent any
    environment {
        HARBOR_URL     = '<JENKINS_HARBOR_IP>'
        HARBOR_PROJECT = 'hello-jenkins'
        IMAGE_NAME     = 'hello-jenkins'
        IMAGE_TAG      = "${BUILD_NUMBER}"
        FULL_IMAGE     = "${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}"
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv venv
                    ./venv/bin/pip install -r requirements.txt
                '''
            }
        }
        stage('Run Tests') {
            steps {
                sh './venv/bin/pytest -v'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${FULL_IMAGE} .'
            }
        }
        stage('Push to Harbor') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'harbor-credentials',
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASSWORD'
                )]) {
                    sh '''
                        echo "$HARBOR_PASSWORD" | docker login ${HARBOR_URL} -u "$HARBOR_USER" --password-stdin
                        docker push ${FULL_IMAGE}
                        docker logout ${HARBOR_URL}
                    '''
                }
            }
        }
    }
}
```

### Verification Checkpoint 3
1. Push the updated `Jenkinsfile` and trigger **Build Now**.
2. Open Harbor (`http://<JENKINS_HARBOR_IP>`) -> **Projects** -> `hello-jenkins` -> **Repositories** -> `hello-jenkins` and confirm the artifact tag matching your Jenkins build number is present.

---

## Project 4: Provision a 3-Node Kubernetes Cluster & Integrate Harbor

### Architectural Theory
Kubernetes requires strict OS-level guarantees before initializing a cluster:
* **Swap Disabled:** The `kubelet` requires swap off (`swapoff -a`) so pod memory limits and Out-Of-Memory (OOM) eviction thresholds behave deterministically.
* **Kernel Modules & IP Forwarding:** `overlay` supports layered container filesystems, while `br_netfilter` and `net.ipv4.ip_forward=1` allow Linux `iptables` to inspect and route bridged pod traffic across the CNI overlay network.
* **Systemd Cgroup Driver:** Setting `SystemdCgroup = true` in `containerd` aligns the container runtime's cgroup driver with `kubelet`, preventing resource starvation under load.

### Step 4.1: Prepare All Three Ubuntu Nodes
Run these commands on **`k8s-master`**, **`k8s-worker1`**, and **`k8s-worker2`**:

```bash
# Set unique hostnames on each respective VM:
# sudo hostnamectl set-hostname k8s-master   (on master)
# sudo hostnamectl set-hostname k8s-worker1  (on worker 1)
# sudo hostnamectl set-hostname k8s-worker2  (on worker 2)

# Disable swap persistently
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl networking parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

### Step 4.2: Install `containerd` and Kubernetes Components (All 3 Nodes)
```bash
# Install and configure containerd
sudo apt update && sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd

# Add Kubernetes v1.37 apt repository and install kubelet, kubeadm, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -pm 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.37/deb/Release.key | \
  sudo gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.37/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

### Step 4.3: Initialize Control Plane, Calico CNI & Join Workers
On **`k8s-master` only**:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=<K8S_MASTER_IP> \
  --pod-network-cidr=192.168.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Calico CNI
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.33.1/manifests/calico.yaml
kubectl apply -f calico.yaml
```

Copy the `kubeadm join ...` command printed by `kubeadm init` (or regenerate it via `kubeadm token create --print-join-command`) and run it with `sudo` on **`k8s-worker1`** and **`k8s-worker2`**.

### Step 4.4: Configure `containerd` v2 for HTTP Harbor Registry (All 3 Nodes)
By default, `containerd` attempts to pull images over HTTPS (`:443`). Configure `certs.d` on all three Kubernetes nodes so `containerd` routes requests for `<JENKINS_HARBOR_IP>` over HTTP (`:80`):

```bash
sudo mkdir -p /etc/containerd/certs.d/<JENKINS_HARBOR_IP>
cat <<EOF | sudo tee /etc/containerd/certs.d/<JENKINS_HARBOR_IP>/hosts.toml
server = "http://<JENKINS_HARBOR_IP>"

[host."http://<JENKINS_HARBOR_IP>"]
  capabilities = ["pull", "resolve"]
EOF
```

> **Troubleshooting Note (`containerd` v2.x `config_path`):** On `containerd` v2.x, inspect `/etc/containerd/config.toml` under `[plugins.'io.containerd.cri.v1.images'.registry]`. If `config_path` contains multiple colon-separated directories (such as `'/etc/containerd/certs.d:/etc/docker/certs.d'`), `containerd` may ignore `hosts.toml` and fail with `dial tcp <JENKINS_HARBOR_IP>:443: connect: connection refused`. Ensure it points cleanly to `'/etc/containerd/certs.d'` and restart `containerd`:

```bash
sudo sed -i "s|config_path = '/etc/containerd/certs.d:/etc/docker/certs.d'|config_path = '/etc/containerd/certs.d'|" /etc/containerd/config.toml
sudo systemctl restart containerd
```

### Step 4.5: Create Pull Secret & Deploy Initial Workload
On **`k8s-master`**, create the `demo` namespace, configure registry credentials, and create the baseline deployment and NodePort service:

```bash
kubectl create namespace demo

kubectl create secret docker-registry harbor-secret \
  --docker-server=<JENKINS_HARBOR_IP> \
  --docker-username=admin \
  --docker-password='<HARBOR_ADMIN_PASSWORD>' \
  --docker-email=admin@example.com \
  --namespace=demo

kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "harbor-secret"}]}' \
  -n demo

kubectl create deployment hello-jenkins \
  --image=<JENKINS_HARBOR_IP>/hello-jenkins/hello-jenkins:<EXISTING_TAG> \
  --namespace=demo

kubectl expose deployment hello-jenkins \
  --type=NodePort \
  --port=5000 \
  --target-port=5000 \
  --name=hello-jenkins-service \
  -n demo
```

### Verification Checkpoint 4
1. Run `kubectl get nodes` and confirm all three nodes report `Ready`.
2. Run `kubectl get pods,svc -n demo` to confirm `hello-jenkins` is `1/1 Running` and note the assigned `<NODEPORT>`.
3. Test the endpoint from your terminal: `curl http://<K8S_WORKER1_IP>:<NODEPORT>` and confirm it returns `{"message":"Hello from Jenkins CI/CD!","status":"success"}`.

---

## Project 5: Automate Kubernetes Deployments & GitLab Triggers

### Architectural Theory
Instead of granting Jenkins unrestricted `cluster-admin` access, we apply the **principle of least privilege** using Kubernetes RBAC: a namespace-scoped `Role` bound to a dedicated `ServiceAccount` (`jenkins-deployer`) that can only update deployments and read pods/services in the `demo` namespace.

Furthermore, when integrating an internal GitLab instance with Jenkins, GitLab blocks outbound webhooks targeting private RFC 1918 IP addresses by default to prevent **Server-Side Request Forgery (SSRF)** attacks. When instance-level GitLab admin settings cannot be modified, **Jenkins SCM Polling** reverses the connection direction—Jenkins initiates outbound checks to GitLab, cleanly bypassing the webhook block.

### Step 5.1: Create Least-Privilege Kubernetes RBAC & Kubeconfig
Run on **`k8s-master`**:

```bash
kubectl create serviceaccount jenkins-deployer -n demo

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-deployer
  namespace: demo
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "patch", "update"]
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-deployer
  namespace: demo
subjects:
- kind: ServiceAccount
  name: jenkins-deployer
  namespace: demo
roleRef:
  kind: Role
  name: jenkins-deployer
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-deployer-token
  namespace: demo
  annotations:
    kubernetes.io/service-account.name: jenkins-deployer
type: kubernetes.io/service-account-token
EOF
```

Generate a portable `jenkins-kubeconfig` file:

```bash
TOKEN=$(kubectl get secret jenkins-deployer-token -n demo -o jsonpath='{.data.token}' | base64 -d)
SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
CA=$(sudo base64 -w0 /etc/kubernetes/pki/ca.crt)

cat > jenkins-kubeconfig <<EOF
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: ${CA}
    server: ${SERVER}
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    namespace: demo
    user: jenkins-deployer
  name: jenkins
current-context: jenkins
users:
- name: jenkins-deployer
  user:
    token: ${TOKEN}
EOF
```

### Step 5.2: Install `kubectl` on Jenkins & Upload Credentials
On **`<JENKINS_HARBOR_IP>`**, install the matching `kubectl` v1.37 binary so Jenkins can execute rollout commands:

```bash
sudo apt-get update && sudo apt-get install -y ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.37/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.37/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update && sudo apt-get install -y kubectl
kubectl version --client
```

In the Jenkins UI:
1. Go to **Manage Jenkins** -> **Credentials** -> **System** -> **Global credentials** -> **Add Credentials**.
2. Select **Secret file**, upload `jenkins-kubeconfig`, and set the ID to `kubernetes-kubeconfig`.

### Step 5.3: Final End-to-End `Jenkinsfile`
Update your repository's **`Jenkinsfile`** with the complete 6-stage CI/CD pipeline:

```groovy
pipeline {
    agent any
    environment {
        HARBOR_URL     = '<JENKINS_HARBOR_IP>'
        HARBOR_PROJECT = 'hello-jenkins'
        IMAGE_NAME     = 'hello-jenkins'
        IMAGE_TAG      = "${BUILD_NUMBER}"
        FULL_IMAGE     = "${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}"
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt
                '''
            }
        }
        stage('Run Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    pytest -v
                '''
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${FULL_IMAGE} .'
            }
        }
        stage('Push to Harbor') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'harbor-credentials',
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASSWORD'
                )]) {
                    sh '''
                        echo "$HARBOR_PASSWORD" | docker login ${HARBOR_URL} -u "$HARBOR_USER" --password-stdin
                        docker push ${FULL_IMAGE}
                        docker logout ${HARBOR_URL}
                    '''
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubernetes-kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    sh '''
                        kubectl set image deployment/hello-jenkins hello-jenkins=${FULL_IMAGE} -n demo
                        kubectl rollout status deployment/hello-jenkins -n demo --timeout=180s
                    '''
                }
            }
        }
    }
    post {
        success {
            echo "CI/CD successful! Deployed ${FULL_IMAGE}"
        }
        failure {
            echo "CI/CD pipeline failed. Check the stage logs."
        }
    }
}
```

### Step 5.4: Automate Pipeline Triggers with GitLab Poll SCM
If your internal GitLab blocks local-network webhooks with `Url is blocked: Requests to the local network are not allowed`:
1. Open your `hello-jenkins` job in Jenkins and click **Configure**.
2. Under **Pipeline**, confirm your GitLab repository URL, credentials, and branch (`*/main`) are configured.
3. Under **Build Triggers**, check **Poll SCM** and set the cron schedule to:
   ```text
   H/2 * * * *
   ```
4. Click **Save**. Jenkins will now poll GitLab roughly every two minutes for new commits.

### Final Verification Checkpoint 5
1. Edit the JSON message in `app.py` (for example, `"Hello from Automated CI/CD!"`), commit, and push to GitLab.
2. Wait ~2 minutes and watch Jenkins automatically trigger a new build, run `pytest`, push `<JENKINS_HARBOR_IP>/hello-jenkins/hello-jenkins:<BUILD_NUMBER>` to Harbor, and roll out the updated pod in Kubernetes.
3. Verify the live update in Kubernetes and via HTTP:
   ```bash
   kubectl get deployment hello-jenkins -n demo -o jsonpath='{.spec.template.spec.containers[0].image}'
   curl http://<K8S_WORKER1_IP>:<NODEPORT>
   ```

---

## Where to Go Next
With the core `GitLab -> Jenkins -> Harbor -> Kubernetes` loop running end to end, natural next upgrades to make this setup production-grade include:
* **Automated Rollbacks:** Wrap `kubectl rollout status` in a `post { failure { ... } }` block that executes `kubectl rollout undo deployment/hello-jenkins -n demo` if health checks fail.
* **Image Vulnerability Scanning:** Integrate **Trivy** into the pipeline between the Docker build and Harbor push stages to block images with critical CVEs.
* **TLS Everywhere:** Replace HTTP endpoints on Jenkins and Harbor with internal CA-signed TLS certificates.
