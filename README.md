# StreamingApp — DevOps / Kubernetes Deployment

A production-style deployment of the **StreamingApp MERN microservices application** using Docker, Amazon ECR, Jenkins, Amazon EKS, Helm and the AWS Load Balancer Controller.

> **Current project status:** GitHub → Jenkins → ECR CI is working successfully.  
> Kubernetes/Helm deployment is already prepared and the application has been deployed to EKS.  
> Jenkins → EKS automatic CD is the next enhancement.

---

## 1. Architecture

```text
                           GitHub
                    Ansarisjd/StreamingApp
                              |
                         Git push
                              |
                         GitHub Webhook
                              |
                              v
                         +---------+
                         | Jenkins |
                         +----+----+
                              |
                 +------------+------------+
                 |                         |
            Docker Build              ECR Login
                 |                         |
                 +------------+------------+
                              |
                              v
                    Amazon ECR (ap-south-1)
                 +------+------+------+------+------+
                 |      |      |      |      |
                Auth  Stream  Admin  Chat  Frontend
                              |
                              | Helm deployment
                              v
                     Amazon EKS (us-east-1)
                              |
                     AWS Load Balancer
                         Controller
                              |
                         Internet ALB
                              |
                     +--------+--------+
                     |                 |
                 Frontend           APIs
                     |                 |
             /api/auth             Auth 3001
             /api/streaming        Stream 3002
             /api/admin            Admin 3003
             /api/chat             Chat 3004
                                      |
                                   MongoDB
                                  Atlas/S3
```

---

## 2. Application Components

| Component | Port | ECR Repository |
|---|---:|---|
| Auth Service | 3001 | `streamingapp-auth` |
| Streaming Service | 3002 | `streamingapp-streaming` |
| Admin Service | 3003 | `streamingapp-admin` |
| Chat Service | 3004 | `streamingapp-chat` |
| Frontend | 80 | `streamingapp-frontend` |

The application uses:

- React frontend
- Node.js/Express backend microservices
- MongoDB
- AWS S3 for object storage
- Socket.IO for chat/WebSocket communication
- Kubernetes/EKS for orchestration
- Helm for deployment
- Jenkins for CI

---

# 3. Prerequisites

Install/configure:

```bash
aws --version
docker --version
kubectl version --client
helm version
eksctl version
git --version
```

Configure AWS CLI:

```bash
aws configure
```

Verify the AWS identity:

```bash
aws sts get-caller-identity
```

> Never commit AWS access keys, MongoDB passwords, JWT secrets or Kubernetes secrets to GitHub.

---

# 4. Clone the GitHub Repository

Clone the fork:

```bash
git clone https://github.com/Ansarisjd/StreamingApp.git
cd StreamingApp
```

Verify:

```bash
git remote -v
git branch
git status
```

Expected remote:

```text
https://github.com/Ansarisjd/StreamingApp.git
```

---

# 5. Project Structure

The important project structure is:

```text
StreamingApp/
├── backend/
│   ├── authService/
│   │   └── Dockerfile
│   ├── streamingService/
│   │   └── Dockerfile
│   ├── adminService/
│   │   └── Dockerfile
│   └── chatService/
│       └── Dockerfile
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   └── src/
├── k8s/
│   └── streamingapp/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── charts/
│       └── templates/
├── Jenkinsfile
├── docker-compose.yml
└── README.md
```

---

# 6. Build Docker Images Manually

## 6.1 Auth

```bash
cd backend/authService

docker build -t streamingapp-auth:1.0.0 .

cd ../..
```

Run locally if required:

```bash
docker run -d \
  --name streamingapp-auth \
  -p 3001:3001 \
  streamingapp-auth:1.0.0
```

---

## 6.2 Streaming

The Streaming Dockerfile expects the `backend` directory as its build context.

```bash
docker build \
  -f backend/streamingService/Dockerfile \
  -t streamingapp-streaming:1.0.0 \
  ./backend
```

---

## 6.3 Admin

```bash
docker build \
  -f backend/adminService/Dockerfile \
  -t streamingapp-admin:1.0.0 \
  ./backend
```

---

## 6.4 Chat

```bash
docker build \
  -f backend/chatService/Dockerfile \
  -t streamingapp-chat:1.0.0 \
  ./backend
```

---

## 6.5 Frontend

```bash
docker build \
  --build-arg REACT_APP_AUTH_API_URL=/api/auth \
  --build-arg REACT_APP_STREAMING_API_URL=/api/streaming \
  --build-arg REACT_APP_STREAMING_PUBLIC_URL= \
  --build-arg REACT_APP_ADMIN_API_URL=/api/admin \
  --build-arg REACT_APP_CHAT_API_URL=/api/chat \
  --build-arg REACT_APP_CHAT_SOCKET_URL= \
  -t streamingapp-frontend:1.0.0 \
  ./frontend
```

Verify all images:

```bash
docker images | grep streamingapp
```

---

# 7. Amazon ECR

## 7.1 Region and Account

The ECR repositories are in:

```text
AWS Region: ap-south-1
```

Account ID is represented below by a placeholder:

```bash
AWS_ACCOUNT_ID=<YOUR_AWS_ACCOUNT_ID>
AWS_REGION=ap-south-1
ECR_REGISTRY=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

Verify:

```bash
aws sts get-caller-identity
```

---

## 7.2 Create ECR Repositories

If they do not already exist:

```bash
aws ecr create-repository \
  --repository-name streamingapp-auth \
  --region ap-south-1

aws ecr create-repository \
  --repository-name streamingapp-streaming \
  --region ap-south-1

aws ecr create-repository \
  --repository-name streamingapp-admin \
  --region ap-south-1

aws ecr create-repository \
  --repository-name streamingapp-chat \
  --region ap-south-1

aws ecr create-repository \
  --repository-name streamingapp-frontend \
  --region ap-south-1
```

List repositories:

```bash
aws ecr describe-repositories --region ap-south-1
```

---

# 8. Authenticate Docker to ECR

```bash
aws ecr get-login-password --region ap-south-1 |
docker login \
  --username AWS \
  --password-stdin \
  ${ECR_REGISTRY}
```

Successful result:

```text
Login Succeeded
```

---

# 9. Tag Docker Images for ECR

```bash
docker tag streamingapp-auth:1.0.0 \
  ${ECR_REGISTRY}/streamingapp-auth:1.0.0

docker tag streamingapp-streaming:1.0.0 \
  ${ECR_REGISTRY}/streamingapp-streaming:1.0.0

docker tag streamingapp-admin:1.0.0 \
  ${ECR_REGISTRY}/streamingapp-admin:1.0.0

docker tag streamingapp-chat:1.0.0 \
  ${ECR_REGISTRY}/streamingapp-chat:1.0.0

docker tag streamingapp-frontend:1.0.0 \
  ${ECR_REGISTRY}/streamingapp-frontend:1.0.0
```

---

# 10. Push Images to ECR

```bash
docker push ${ECR_REGISTRY}/streamingapp-auth:1.0.0

docker push ${ECR_REGISTRY}/streamingapp-streaming:1.0.0

docker push ${ECR_REGISTRY}/streamingapp-admin:1.0.0

docker push ${ECR_REGISTRY}/streamingapp-chat:1.0.0

docker push ${ECR_REGISTRY}/streamingapp-frontend:1.0.0
```

Verify:

```bash
aws ecr describe-images \
  --repository-name streamingapp-auth \
  --region ap-south-1
```

Repeat for the other repositories if required.

---

# 11. Create EKS Cluster

The EKS cluster is deployed in:

```text
us-east-1
```

Create the cluster:

```bash
eksctl create cluster \
  --name streamingapp-cluster \
  --region us-east-1 \
  --nodegroup-name streamingapp-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 2
```

Verify:

```bash
kubectl get nodes
```

Expected:

```text
NAME                          STATUS   ROLES
ip-...                        Ready    <none>
ip-...                        Ready    <none>
```

Check cluster:

```bash
eksctl get cluster --region us-east-1
```

---

# 12. Configure kubectl for EKS

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name streamingapp-cluster
```

Verify:

```bash
kubectl cluster-info
```

```bash
kubectl get nodes
```

---

# 13. Enable EKS OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster streamingapp-cluster \
  --region us-east-1 \
  --approve
```

Verify:

```bash
eksctl get iamserviceaccount \
  --cluster streamingapp-cluster \
  --region us-east-1
```

---

# 14. AWS Load Balancer Controller

The AWS Load Balancer Controller creates the internet-facing Application Load Balancer used by the Kubernetes Ingress.

## Add Helm repository

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

## Create IAM service account

The IAM policy used by the controller must exist first.

```bash
eksctl create iamserviceaccount \
  --cluster streamingapp-cluster \
  --region us-east-1 \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

Verify:

```bash
kubectl get serviceaccount \
  aws-load-balancer-controller \
  -n kube-system \
  -o yaml
```

## Install controller

```bash
helm upgrade --install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=streamingapp-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<YOUR_EKS_VPC_ID>
```

Verify:

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller
```

Expected:

```text
Running
```

---

# 15. MongoDB and Application Secrets

The Kubernetes application requires:

```text
MONGO_URI
JWT_SECRET
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

A Kubernetes secret named:

```text
streamingapp-secret
```

is used by the backend deployments.

Create it without committing the values to Git:

```bash
kubectl create namespace streamingapp
```

Then create the secret interactively:

```bash
kubectl create secret generic streamingapp-secret \
  -n streamingapp \
  --from-literal=MONGO_URI='<YOUR_MONGO_URI>' \
  --from-literal=JWT_SECRET='<YOUR_JWT_SECRET>' \
  --from-literal=AWS_ACCESS_KEY_ID='<YOUR_AWS_ACCESS_KEY_ID>' \
  --from-literal=AWS_SECRET_ACCESS_KEY='<YOUR_AWS_SECRET_ACCESS_KEY>'
```

Verify only the keys, not the secret values:

```bash
kubectl describe secret streamingapp-secret \
  -n streamingapp
```

> Do not paste secret values into GitHub, README files, screenshots or Jenkins console output.

---

# 16. Helm Chart

The Helm chart is located at:

```text
k8s/streamingapp/
```

Validate it:

```bash
helm lint k8s/streamingapp
```

Expected:

```text
1 chart(s) linted, 0 chart(s) failed
```

Inspect rendered Kubernetes manifests:

```bash
helm template streamingapp \
  k8s/streamingapp \
  -n streamingapp
```

---

# 17. Helm Values

The chart contains configurable values for:

- image repositories
- image tags
- replicas
- services
- ingress
- application configuration

Example image configuration:

```yaml
image:
  repository: <ECR_REPOSITORY>
  tag: <IMAGE_TAG>
```

The ALB hostname is supplied to backend `CLIENT_URLS` configuration after the ALB is available.

---

# 18. Deploy Application to EKS

Install/upgrade the application:

```bash
helm upgrade --install streamingapp \
  k8s/streamingapp \
  -n streamingapp \
  --create-namespace
```

Check Helm:

```bash
helm list -n streamingapp
```

Check deployments:

```bash
kubectl get deployments -n streamingapp
```

Check services:

```bash
kubectl get services -n streamingapp
```

Check pods:

```bash
kubectl get pods -n streamingapp
```

---

# 19. Check Kubernetes Pods

```bash
kubectl get pods -n streamingapp -o wide
```

All application pods should eventually show:

```text
1/1 Running
```

Describe a pod if there is a problem:

```bash
kubectl describe pod <POD_NAME> -n streamingapp
```

View logs:

```bash
kubectl logs <POD_NAME> -n streamingapp
```

---

# 20. Kubernetes Health Checks

Backend health endpoints:

### Auth

```text
/api/auth/health
```

### Streaming

```text
/api/health
```

### Admin

```text
/api/health
```

### Chat

```text
/api/health
```

The external ALB routing is:

```text
/                  -> frontend
/api/auth          -> auth service
/api/streaming     -> streaming service
/api/admin         -> admin service
/api/chat          -> chat service
```

---

# 21. Get the ALB Address

```bash
kubectl get ingress -n streamingapp
```

Or:

```bash
kubectl get ingress \
  -n streamingapp \
  -o wide
```

Wait for the `ADDRESS` field to contain the AWS ALB DNS name.

Example format:

```text
k8s-....us-east-1.elb.amazonaws.com
```

---

# 22. Test the Application

Set:

```bash
export ALB_URL="http://<YOUR_ALB_DNS_NAME>"
```

Test frontend:

```bash
curl -I ${ALB_URL}/
```

Test Auth:

```bash
curl -i ${ALB_URL}/api/auth/health
```

For internal service health checks:

```bash
kubectl run health-check \
  -n streamingapp \
  --rm -it \
  --restart=Never \
  --image=curlimages/curl \
  -- \
  curl -s http://streamingapp-streaming:3002/api/health
```

Repeat for Admin and Chat using their Kubernetes service names and ports.

---

# 23. Jenkins CI/CD

## Jenkins Pipeline

The repository contains:

```text
Jenkinsfile
```

The CI pipeline performs:

```text
Checkout
   ↓
Login to ECR
   ↓
Build Images
   ├── Auth
   ├── Streaming
   ├── Admin
   ├── Chat
   └── Frontend
   ↓
Push Images
   ├── Auth
   ├── Streaming
   ├── Admin
   ├── Chat
   └── Frontend
```

Jenkins uses the AWS credential:

```text
aws-jenkins
```

The AWS IAM identity used by Jenkins must have ECR permissions.

---

# 24. Jenkins Job Configuration

Create a Pipeline job:

```text
Streaming-App-Sajid
```

Pipeline definition:

```text
Pipeline script from SCM
```

SCM:

```text
Git
```

Repository:

```text
https://github.com/Ansarisjd/StreamingApp.git
```

Branch:

```text
*/main
```

Script path:

```text
Jenkinsfile
```

For a public repository, Git credentials are not required for checkout.

---

# 25. Jenkins Automatic GitHub Trigger

Enable:

```text
GitHub hook trigger for GITScm polling
```

in:

```text
Jenkins
  → Streaming-App-Sajid
  → Configure
  → Build Triggers
```

---

# 26. GitHub Webhook

GitHub:

```text
StreamingApp
  → Settings
  → Webhooks
  → Add webhook
```

Payload URL:

```text
https://jenkinsacademics.herovired.com/github-webhook/
```

Content type:

```text
application/json
```

Events:

```text
Just the push event
```

Active:

```text
Enabled
```

This makes:

```text
git push
   ↓
GitHub Webhook
   ↓
Jenkins
   ↓
Streaming-App-Sajid
```

---

# 27. Jenkins CI Image Tags

The Jenkinsfile tags images using the Jenkins build number.

For example, build `2` produces:

```text
streamingapp-auth:2
streamingapp-streaming:2
streamingapp-admin:2
streamingapp-chat:2
streamingapp-frontend:2
```

and also:

```text
:latest
```

The ECR registry format is:

```text
<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/<REPOSITORY>:<TAG>
```

---

# 28. ECR → EKS Image Flow

Pushing an image to ECR does **not by itself update a running Kubernetes deployment**.

The complete deployment flow is:

```text
GitHub
   |
   | push
   v
Jenkins
   |
   | docker build
   v
Docker Images
   |
   | docker push
   v
Amazon ECR
   |
   | Helm upgrade
   v
Amazon EKS
   |
   | Pod image pull
   v
New Kubernetes Pod
```

The EKS node/container runtime pulls the requested image from ECR when a new Pod is created.

---

# 29. Current CI/CD Status

### Completed

- GitHub fork and source repository
- Dockerfiles for all application components
- ECR repositories
- Docker image builds
- ECR image pushes
- Jenkins Pipeline
- AWS Jenkins credentials
- GitHub webhook
- EKS cluster
- EKS node group
- OIDC provider
- AWS Load Balancer Controller
- Kubernetes secrets
- Helm chart
- Kubernetes application deployment
- ALB/Ingress
- Frontend and backend routing

### Remaining

- Add Jenkins → EKS automatic Helm deployment (CD)
- Validate Kubernetes horizontal scaling
- Validate rolling updates
- Validate Kubernetes self-healing
- Configure CloudWatch metrics
- Configure CloudWatch alarms
- Configure centralized CloudWatch logging
- Complete end-to-end application testing
- Add architecture/deployment documentation screenshots
- Final GitHub push
- Submit repository link through Vlearn

---

# 30. Kubernetes Scaling Validation

Scale a deployment:

```bash
kubectl scale deployment streamingapp-auth \
  -n streamingapp \
  --replicas=3
```

Verify:

```bash
kubectl get deployment streamingapp-auth -n streamingapp
```

```bash
kubectl get pods -n streamingapp -l app.kubernetes.io/component=auth
```

Scale back:

```bash
kubectl scale deployment streamingapp-auth \
  -n streamingapp \
  --replicas=1
```

For the final project, repeat the validation for the services that are required to scale.

---

# 31. Self-Healing Validation

Find a running pod:

```bash
kubectl get pods -n streamingapp
```

Delete one:

```bash
kubectl delete pod <POD_NAME> -n streamingapp
```

Immediately watch:

```bash
kubectl get pods -n streamingapp -w
```

Kubernetes should create a replacement Pod because the Deployment maintains the desired replica count.

---

# 32. Rolling Update Validation

Check the current Deployment:

```bash
kubectl rollout status \
  deployment/streamingapp-auth \
  -n streamingapp
```

Check rollout history:

```bash
kubectl rollout history \
  deployment/streamingapp-auth \
  -n streamingapp
```

After deploying a new image tag:

```bash
kubectl rollout status \
  deployment/streamingapp-auth \
  -n streamingapp
```

Check ReplicaSets:

```bash
kubectl get rs -n streamingapp
```

Rollback if required:

```bash
kubectl rollout undo \
  deployment/streamingapp-auth \
  -n streamingapp
```

---

# 33. Troubleshooting Commands

## Pods

```bash
kubectl get pods -n streamingapp
```

## Deployments

```bash
kubectl get deployments -n streamingapp
```

## Services

```bash
kubectl get svc -n streamingapp
```

## Ingress

```bash
kubectl get ingress -n streamingapp
```

## Events

```bash
kubectl get events -n streamingapp \
  --sort-by=.lastTimestamp
```

## Pod logs

```bash
kubectl logs <POD_NAME> -n streamingapp
```

## Previous crashed container logs

```bash
kubectl logs <POD_NAME> \
  -n streamingapp \
  --previous
```

## Describe deployment

```bash
kubectl describe deployment <DEPLOYMENT_NAME> \
  -n streamingapp
```

## Helm

```bash
helm list -n streamingapp
```

```bash
helm status streamingapp -n streamingapp
```

---

# 34. Git Workflow

Check changes:

```bash
git status
```

Stage:

```bash
git add .
```

Commit:

```bash
git commit -m "Update deployment configuration"
```

Push:

```bash
git push origin main
```

After the webhook is configured, a push to `main` should trigger Jenkins automatically.

---

# 35. Security Notes

Never commit:

```text
.env
AWS access keys
AWS secret keys
MongoDB passwords
JWT secrets
Kubernetes Secret manifests containing real credentials
```

Use Kubernetes Secrets for runtime credentials.

If a secret has been exposed in terminal screenshots, chat messages or public Git history, rotate it before final submission.

---

# 36. Evidence / Screenshots

## GitHub Webhook

The GitHub repository webhook was configured to send push events to Jenkins.

![GitHub Webhook](docs/screenshots/github-webhook.png)

---

## Jenkins Job

The Jenkins job is configured as the project pipeline and is connected to the GitHub repository.

![Jenkins Job](docs/screenshots/jenkins-job.png)

---

# 37. Final Validation Checklist

Before submitting the project:

```text
[ ] GitHub repository is public/accessible
[ ] Dockerfiles exist for frontend and all backend services
[ ] Five ECR repositories exist
[ ] All five images can be built
[ ] Jenkins can authenticate to ECR
[ ] Jenkins can build all five images
[ ] Jenkins can push all five images
[ ] GitHub webhook triggers Jenkins
[ ] EKS cluster is healthy
[ ] All EKS nodes are Ready
[ ] AWS Load Balancer Controller is Running
[ ] Kubernetes Secret exists
[ ] Helm lint succeeds
[ ] Helm release is deployed
[ ] All Pods are Running/Ready
[ ] ALB is reachable
[ ] Frontend is reachable
[ ] Auth API is reachable
[ ] Streaming API is reachable
[ ] Admin API is reachable
[ ] Chat API/WebSocket is reachable
[ ] MongoDB connectivity works
[ ] Scaling is tested
[ ] Rolling update is tested
[ ] Self-healing is tested
[ ] CloudWatch metrics configured
[ ] CloudWatch alarms configured
[ ] Centralized logging configured
[ ] Architecture diagram documented
[ ] Deployment commands documented
[ ] Final changes pushed to GitHub
[ ] Repository link submitted to Vlearn
```

---

# 38. Recommended Final CI/CD Pipeline

The final target architecture is:

```text
Developer
    |
    | git push
    v
GitHub
    |
    | webhook
    v
Jenkins
    |
    +--> Checkout
    |
    +--> Build 5 Docker Images
    |
    +--> Push 5 Images to ECR
    |
    +--> Helm Upgrade
             |
             v
          EKS Cluster
             |
             +--> Auth
             +--> Streaming
             +--> Admin
             +--> Chat
             +--> Frontend
             |
             v
          AWS ALB
             |
             v
           Users

EKS
 |
 +--> CloudWatch Metrics
 |
 +--> CloudWatch Alarms
 |
 +--> Centralized Logs
```

The Jenkins → EKS Helm deployment is intentionally listed as **pending** until it is configured and tested; this README documents the project accurately as of the current stage.
