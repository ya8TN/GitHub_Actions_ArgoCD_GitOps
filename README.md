# GitOps CI/CD with GitHub Actions and ArgoCD
In this video, we take DevOps to the next level by building a complete GitOps-driven CI/CD pipeline using GitHub Actions and ArgoCD, deployed on Kubernetes! 💥


YouTube Link: https://youtu.be/TZuNSMTWAcY?si=ZP5Sc7RbtQ0bFsgE


## Requirements
1. GitHub Actions 
2. ArgoCD: 
    1. Expose ArgoCD via Public Tunnel (For Dev ENV) e.g., ngrok, inlets
    2. Deploy ArgoCD on Public Cloud (For Prod ENV w/ TLS Certs) e.g., EC2, EKS, GKE, etc
3. Cloud Linux Instance (since GitHub Actions runs in the cloud) - AWS EC2 Ubuntu - t3.medium
3. Docker
4. Kubernetes cluster (Minikube)
5. Kubectl


## Create EC2 Instance & Set up Environment

### Make the keypair executable:
```sh
chmod 600 keypair.pem
```

### Log in to Ubuntu EC2 Instance:
```sh
ssh -i /home/paacyber/Downloads/<keypair.pem> ubuntu@PublicIP
```

### Update & Upgrade System:
```sh
sudo apt update && sudo apt upgrade -y
```

### Install Docker
```sh
sudo apt  install docker.io -y
```

```sh
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

```sh
docker version
```

```sh
systemctl status docker
```


### Install Kubectl
```sh
sudo snap install kubectl --classic
```
```sh
kubectl version --client
```


### Install & Start Minikube
```sh
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
```
```sh
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```
```sh
minikube version
```
```sh
minikube start --driver=docker
```
```sh
kubectl get nodes
```

### Enable Ingress on Minikube (Optional but useful)
```sh
minikube addons enable ingress
```


## Checkout Code into GitHub Actions
1. Click on the User Account
2. Click on Settings
3. Developer settings, and select Personal access tokens and Click Tokens (classic)
4. Generate new token, and select Generate new token (classic)
5. Note: actions-argocd-gitops00, Expiration: 30 days, and 
6. Scopes (select the following): 
    1. repo
    2. workflow (For GitHub Actions)
    3. admin:repo_hook (For Webhooks)
7. Generate token & save it somewhere safe 


### Clone the GitHub Repository (if you want to work locally & push changes to remote repo)
```sh
git clone https://github.com/iQuantC/GitHubActions-ArgoCD-GitOps.git
```

### Create GitHub Actions Workflow Files & Directories
```sh
mkdir .github
cd .github
```

```sh
mkdir workflows
cd workflow
touch argocd-actions.yml
```

## Create DockerHub Repository
1. Sign in to your DockerHub Account
2. Click "Create a repository"
3. Repository Name: gitHubActions-ArgoCD-00, Visibility: Public
4. Click Create.
5. Click on the User Account, Click on "Account settings", Click on "Personal access tokens" 
6. Click "Generate new token", Expiration: 30 days, Access permissions: RWD. 
7. Click Generate & save it somewhere safe 


## Create Environment Variables in GitHub
1. Click on the GitOps Repository and click on "Settings" 
2. Click on "Secrets and variables" and select "Actions"
3. Under Repository secrets, click on "New repository secret"
   
        Name: DOCKERHUB_USERNAME
        Secret: <dockerhub username>
        Add secret

        Name: DOCKERHUB_TOKEN
        Secret: <dockerhub token here>
        Add secret


## Set up ArgoCD

### Install ArgoCD in Minikube Namespace argocd 
```sh
kubectl create namespace argocd
```
```sh
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
```sh
kubectl get pods -n argocd
```
```sh
kubectl get svc -n argocd
```


### Expose ArgoCD Server
First, add port 8080 to Inbound Rules for the EC2 Instance 
```sh
kubectl port-forward --address 0.0.0.0 svc/argocd-server 8080:443 -n argocd
```


### Get ArgoCD Initial Admin Password
```sh
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
```

On your browser: 
```sh
PublicIP:8080
```
```sh
ARGOCD_USERNAME: admin
ARGOCD_PASSWORD: <argocd init password>
```


### Install ArgoCD CLI (In GitHub Actions Pipeline)
```sh
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/argocd
```

```sh
argocd version
```


### Expose ArgoCD Service via NodePort
```sh
kubectl get svc -n argocd
```

### Replace Service Type ClusterIP with a NodePort (or LoadBalancer) - (default: 30000-32767)
```sh
kubectl edit svc argocd-server -n argocd
```


### Add the NodePort (30007 and 30008 - Anywhere IPv4) to Inbound Rules for the EC2 Instance
and
### Rerun Port-forward command with the NodePort (30007) in a dedicated terminal
```sh
kubectl port-forward --address 0.0.0.0 svc/argocd-server 30007:80 -n argocd
```

## OR
### Rerun Port-forward command with the NodePort (30008) in a dedicated terminal
```sh
kubectl port-forward --address 0.0.0.0 svc/argocd-server 30008:443 -n argocd
```

## Login to ArgoCD (In GitHub Actions Pipeline)
First, 
1. Click Settings
2. Secrets and variables
3. Actions
4. New repository secret, to create new secrets for 
```sh
    Name: ARGOCD_SERVER
    Value: PublicIP:30008
    Add secret
```
```sh
    Name: ARGOCD_USERNAME
    Value: admin
    Add secret
```
```sh
    Name: ARGOCD_PASSWORD
    Value: <argocd init password>
    Add secret
```

## Configure Repository on ArgoCD UI (or CLI)
1. Go to Settings
2. Click on Repositories, and Connect Repo
3. Connection Method: Via HTTPS
4. Type: git
5. Project: default
6. Repository URL: 
7. Username (optional): 
8. Password (optional): 
9. TLS Client Certificate (optional): 
10. The remaining stuff optional. Leave as default and click CONNECT.


## Create a New Application on ArgoCD UI (or with CLI below)
1. Click on Applications
2. Click New App
3. Application Name: argocd-github-actions
4. Project Name: default
5. Sync Policy: Automatic
6. Check Prune Resources & Self Heal
7. Repo URL: Click and select the Repo you attached earlier
8. Revision: main (this is the branch from which app is deployed)
9. Path: manifest
10. Cluster URL: Click and select the kubernetes.default.svc
11. Namespace: argocd
12. Leave the rest as default or Set them up if you want to. 


## (Alternatively) Using ArgoCD CLI to Create New Application  
```sh
argocd app create my-app \
  --repo https://github.com/your-username/your-repo.git \
  --path manifest \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd
```

### Update Deployment file
1. Replace the Image with the Latest image built
2. ArgoCD UI will automatically Sync it & Deem it healthy


### Inspect Deployment & Service on Terminal
```sh
kubectl get deploy -n argocd
```
```sh
kubectl get svc -n argocd
```

### Automate ArgoCD Sync with GitHub Actions Workflow Pipeline
1. Add "argocd app sync argocd-github-actions" block to the pipeline
2. Commit changes and verify sync in the ArgoCD UI with Deploy, Svc, Pods, etc. 
3. Inspect a Pod to see the port it listens. On EC2 Inbound rules, allow the port  3000 - AnywhereIPv4 - node app
4. Install NPM Modules on terminal & run the app

```sh
sudo apt install npm -y
sudo npm install -y
```

The App has a page on /hello: 
```sh
cat app.js
```

### Run App Locally
```sh
node app.js
```
On your browser: 
```sh
PublicIP:3000/hello
```


### Run App Using Running Containers on ArgoCD
```sh
kubectl get svc -n argocd
```
```sh
kubectl port-forward --address 0.0.0.0 svc/myapp-service 8080:80 -n argocd
```
On your browser:
```sh
PublicIP:8080/hello
```


## Clean Up 

```sh
kubectl delete ns argocd
```
```sh
minikube stop
```
```sh
minikube delete --all
```
Terminate the EC2 Instance on AWS

```sh
Thanks for Watching
Please Like, Comment, and Subscribe to iQuant on YouTube
```
