# Ulimate Wanderlust DevSecOps Project🌍✈️

WanderLust is a simple MERN travel blog website ✈ This project is aimed to help people to contribute in open source, upskill in react and also master git.

![Preview Image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/ulimate-devsecops-project.png)
#

# Wanderlust Mega Project End to End Implementation

### In this demo, we will see how to deploy an end to end three tier MERN stack application on EKS cluster.
#
### <mark>Project Deployment Flow:</mark>
<img src="https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/Jenkins%20CI%20Job.gif" />

#

## Tech stack used in this project:
- GitHub (Code)
- Docker (Containerization)
- Jenkins (CI)
- OWASP (Dependency check)
- SonarQube (Quality)
- Trivy (Filesystem Scan)
- ArgoCD (CD)
- Redis (Caching)
- AWS EKS (Kubernetes)
- Helm (Monitoring using grafana and prometheus)

### How pipeline will look after deployment:
- <b>CI pipeline to build and push</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/Screenshot%202025-01-18%20133419.png)

- <b>CD pipeline to update application version</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/Screenshot%202025-01-18%20133807.png)

- <b>ArgoCD application for deployment on EKS</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/argocd.gif)

#
> [!Important]
> Below table helps you to navigate to the particular tool installation section fast.

| Tech stack    | Installation |
| -------- | ------- |
| Jenkins Master | <a href="#Jenkins">Install and configure Jenkins</a>     |
| eksctl | <a href="#EKS">Install eksctl</a>     |
| Argocd | <a href="#Argo">Install and configure ArgoCD</a>     |
| Jenkins-Worker Setup | <a href="#Jenkins-worker">Install and configure Jenkins Worker Node</a>     |
| OWASP setup | <a href="#Owasp">Install and configure OWASP</a>     |
| SonarQube | <a href="#Sonar">Install and configure SonarQube</a>     |
| Email Notification Setup | <a href="#Mail">Email notification setup</a>     |
| Monitoring | <a href="#Monitor">Prometheus and grafana setup using helm charts</a>
| Clean Up | <a href="#Clean">Clean up</a>     |
#

### Pre-requisites to implement this project:
#

> [!Note]
> This project will be implemented on Mumbai region (ap-south-1).

- <b>Create 1 Master machine on AWS with 2CPU, 8GB of RAM (t2.large) and 29 GB of storage and install Docker on it.</b>
#
- <b>Open the below ports in security group of master machine and also attach same security group to Jenkins worker node (We will create worker node shortly)</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/Screenshot%202025-01-18%20135759.png)

> [!Note]
> We are creating this master machine because we will configure Jenkins master, eksctl, EKS cluster creation from here.

Install & Configure Docker by using below command, "NewGrp docker" will refresh the group config hence no need to restart the EC2 machine.

```bash
sudo apt-get update
```
```bash
sudo apt-get install docker.io -y
sudo usermod -aG docker ubuntu && newgrp docker
```
#
- <b id="Jenkins">Install and configure Jenkins (Master machine)</b>
```bash
sudo apt update -y
sudo apt install fontconfig openjdk-17-jre -y

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
  
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
  
sudo apt-get update -y
sudo apt-get install jenkins -y
```
- <b>Now, access Jenkins Master on the browser on port 8080 and configure it</b>.
#
- <b id="EKS">Create EKS Cluster on AWS (Master machine)</b>
  - IAM user with **access keys and secret access keys**
  - AWSCLI should be configured (<a href="https://github.com/harshitsahu2311/DevOps-Tools-Installation/blob/main/AWSCLI/AWSCLI.sh">Setup AWSCLI</a>)
  ```bash
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  sudo apt install unzip
  unzip awscliv2.zip
  sudo ./aws/install
  aws configure
  ```

  - Install **kubectl** (Master machine)(<a href="https://github.com/harshitsahu2311/DevOps-Tools-Installation/blob/main/Kubectl/Kubectl.sh">Setup kubectl </a>)
  ```bash
  curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
  chmod +x ./kubectl
  sudo mv ./kubectl /usr/local/bin
  kubectl version --short --client
  ```

  - Install **eksctl** (Master machine) (<a href="https://github.com/harshitsahu2311/DevOps-Tools-Installation/blob/main/eksctl%20/eksctl.sh">Setup eksctl</a>)
  ```bash
  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
  sudo mv /tmp/eksctl /usr/local/bin
  eksctl version
  ```
  
  - <b>Create EKS Cluster (Master machine)</b>
  ```bash
  eksctl create cluster --name=wanderlust \
  --region=ap-south-1 \
  --version=1.30 \
  --without-nodegroup
  ```
  - <b>Associate IAM OIDC Provider (Master machine)</b>
  ```bash
  eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster wanderlust \
  --approve
  ```
  - <b>Create Nodegroup (Master machine)</b>
  ```bash
  eksctl create nodegroup --cluster=wanderlust \
  --region=ap-south-1 \
  --name=wanderlust \
  --node-type=t2.large \
  --nodes=2 \
  --nodes-min=2 \
  --nodes-max=2 \
  --node-volume-size=29 \
  --ssh-access \
  --ssh-public-key=mumbai-key 
  ```
> [!Note]
>  Make sure the ssh-public-key "mumbai-key is available in your aws account"
#
- <b id="Jenkins-worker">Setting up jenkins worker node</b>
  - Create a new EC2 instance (Jenkins Worker) with 2CPU, 8GB of RAM (t2.large) and 29 GB of storage and install java on it
  ```bash
  sudo apt update -y
  sudo apt install fontconfig openjdk-17-jre -y
  ```
  - Create an IAM role with <mark>administrator access</mark> attach it to the jenkins worker node <mark>Select Jenkins worker node EC2 instance --> Actions --> Security --> Modify IAM role</mark>
  ![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/iamrole.png)

  - Configure AWSCLI (<a href="https://github.com/harshitsahu2311/DevOps-Tools-Installation/blob/main/AWSCLI/AWSCLI.sh">Setup AWSCLI</a>)
  ```bash
  sudo su
  ```
  ```bash
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  sudo apt install unzip
  unzip awscliv2.zip
  sudo ./aws/install
  aws configure
  ```
#
  - <b>generate ssh keys (Master machine) to setup jenkins master-slave</b>
  ```bash
  ssh-keygen
  ```
  ![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/ssh.png)
#
  - <b>Now move to directory where your ssh keys are generated and copy the content of public key and paste to authorized_keys file of the Jenkins worker node.</b>
#
  - <b>Now, go to the jenkins master and navigate to <mark>Manage jenkins --> Nodes</mark>, and click on Add node </b>
    - <b>Name:</b> Node
    - <b>type:</b> permanent agent
    - <b>Number of executors:</b> 2
    - <b>Remote root directory:</b> /home/ubuntu
    - <b>Labels:</b> Node
    - <b>Usage:</b> Only build jobs with label expressions matching this node
    - <b>Launch method:</b> Via ssh
    - <b>Host:</b> \<public-ip-worker-jenkins\>
    - <b>Credentials:</b> <mark>Add --> Kind: ssh username with private key --> ID: Worker --> Description: Worker --> Username: ubuntu --> Private key: Enter directly --> Add Private key</mark>
    - <b>Host Key Verification Strategy:</b> Non verifying Verification Strategy
    - <b>Availability:</b> Keep this agent online as much as possible
#
  - And your jenkins worker node is added
  ![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/node.png)

# 
- <b id="docker">Install docker (Jenkins Worker)</b>

```bash
sudo apt install docker.io -y
sudo usermod -aG docker ubuntu && newgrp docker
```
#
- <b id="Sonar">Install and configure SonarQube (Master machine)</b>
```bash
docker run -itd --name SonarQube-Server -p 9000:9000 sonarqube:lts-community
```
#
- <b id="Trivy">Install Trivy (Jenkins Worker)</b>
```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update -y
sudo apt-get install trivy -y
```
#
- <b id="Argo">Install and Configure ArgoCD (Master Machine)</b>
  - <b>Create argocd namespace</b>
  ```bash
  kubectl create namespace argocd
  ```
  - <b>Apply argocd manifest</b>
  ```bash
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  ```
  - <b>Make sure all pods are running in argocd namespace</b>
  ```bash
  watch kubectl get pods -n argocd
  ```
  - <b>Install argocd CLI</b>
  ```bash
  sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
  ```
  - <b>Provide executable permission</b>
  ```bash
  sudo chmod +x /usr/local/bin/argocd
  ```
  - <b>Check argocd services</b>
  ```bash
  kubectl get svc -n argocd
  ```
  - <b>Change argocd server's service from ClusterIP to NodePort</b>
  ```bash
  kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
  ```
  - <b>Confirm service is patched or not</b>
  ```bash
  kubectl get svc -n argocd
  ```
  - <b> Check the port where ArgoCD server is running and expose it on security groups of a worker node</b>
  ![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/argocdport.png)
  - <b>Access it on browser, click on advance and proceed with</b>
  ```bash
  <public-ip-worker>:<port>
  ```
  ![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/errorargo.png)

   - <b>If you find error like this in the above image, then run the below the command:</b>
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 31797:80 --address 0.0.0.0 &
   ```
  
  ![image](https://github.com/user-attachments/assets/1ffa85c3-9055-49b4-aab0-0947b95f0dd2)
  - <b>Fetch the initial password of argocd server</b>
  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
  ```
  - <b>Username: admin</b>
  - <b> Now, go to <mark>User Info</mark> and update your argocd password
#
## Steps to add email notification
- <b id="Mail">Go to your Jenkins Master EC2 instance and allow 465 port number for SMTPS</b>
#
- <b>Now, we need to generate an application password from our gmail account to authenticate with jenkins</b>
  - <b>Open gmail and go to <mark>Manage your Google Account --> Security</mark></b>
> [!Important]
> Make sure 2 step verification must be on

  ![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/2-step.png)

  - <b>Search for <mark>App password</mark> and create a app password for jenkins</b>
  ![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/app-pas.png)
  ![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/app-pas2.png)
  
#
- <b> Once, app password is create and go back to jenkins <mark>Manage Jenkins --> Credentials</mark> to add username and password for email notification</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/gmailpas.png)

# 
- <b> Go back to <mark>Manage Jenkins --> System</mark> and search for <mark>Extended E-mail Notification</mark></b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/smtp%20tool.png)
#
- <b>Scroll down and search for <mark>E-mail Notification</mark> and setup email notification</b>
> [!Important]
> Enter your gmail password which we copied recently in password field <mark>E-mail Notification --> Advance</mark>

![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/email_notify.png)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/test_conf.png)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/testemail.png)

#
## Steps to implement the project:
- <b>Go to Jenkins Master and click on <mark> Manage Jenkins --> Plugins --> Available plugins</mark> install the below plugins:</b>
  - OWASP
  - SonarQube Scanner
  - Docker
  - Pipeline: Stage View
#
- <b id="Owasp">Configure OWASP, move to <mark>Manage Jenkins --> Plugins --> Available plugins</mark> (Jenkins Worker)</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/owasp_plugin.png)

- <b id="Sonar">After OWASP plugin is installed, Now move to <mark>Manage jenkins --> Tools</mark> (Jenkins Worker)</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/owasp_tool.png)
#
- <b>Login to SonarQube server and create the credentials for jenkins to integrate with SonarQube</b>
  - Navigate to <mark>Administration --> Security --> Users --> Token</mark>
  ![image](https://github.com/user-attachments/assets/86ad8284-5da6-4048-91fe-ac20c8e4514a)
  ![image](https://github.com/user-attachments/assets/6bc671a5-c122-45c0-b1f0-f29999bbf751)
  ![image](https://github.com/user-attachments/assets/e748643a-e037-4d4c-a9be-944995979c60)

#
- <b>Now, go to <mark> Manage Jenkins --> credentials</mark> and add Sonarqube credentials:</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/sonar-cred.png)
#
- <b>Go to <mark> Manage Jenkins --> Tools</mark> and search for SonarQube Scanner installations:</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/sonar-tool.png)
#
- <b> Go to <mark> Manage Jenkins --> credentials</mark> and add Github credentials to push updated code from the pipeline:</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/github-cred.png)
> [!Note]
> While adding github credentials add Personal Access Token in the password field.
#
- <b>Go to <mark> Manage Jenkins --> System</mark> and search for SonarQube installations:</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/sonar-system.png)
#
- <b>Now again, Go to <mark> Manage Jenkins --> System</mark> and search for Global Trusted Pipeline Libraries:</b
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/trusted_pipe.png)
#
- <b>Login to SonarQube server, go to <mark>Administration --> Webhook</mark> and click on create </b>
![image](https://github.com/user-attachments/assets/16527e72-6691-4fdf-a8d2-83dd27a085cb)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/sonar-webhook.png)
#
- <b>Now, go to github repository and under <mark>Automations</mark> directory update the <mark>instance-id</mark> field on both the <mark>updatefrontendnew.sh updatebackendnew.sh</mark> with the k8s worker's instance id</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/instance-id.png)
#
- <b>Navigate to <mark> Manage Jenkins --> credentials</mark> and add credentials for docker login to push docker image:</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/dockerhuub.png)
#
- <b>Create a <mark>Wanderlust-CI</mark> pipeline</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/ci-job.png)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/ci-pipeline.png)

#
- <b>Create one more pipeline <mark>Wanderlust-CD</mark></b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/cd-job.png)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/cd-pipeline.png)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/all-job.png)
#
- <b>Provide permission to docker socket so that docker build and push command do not fail (Jenkins Worker)</b>
```bash
chmod 777 /var/run/docker.sock
```
![image](https://github.com/user-attachments/assets/e231c62a-7adb-4335-b67e-480758713dbf)
#
- <b> Go to Master Machine and add our own eks cluster to argocd for application deployment using cli</b>
  - <b>Login to argoCD from CLI</b>
  ```bash
   argocd login 65.2.186.181:31797 --username admin
  ```
> [!Tip]
> 65.2.186.181:31797 --> This should be your argocd url

  ![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/argo-login.png)

  - <b>Check how many clusters are available in argocd </b>
  ```bash
  argocd cluster list
  ```
  ![image](https://github.com/user-attachments/assets/76fe7a45-e05c-422d-9652-bdaee02d630f)
  - <b>Get your cluster name</b>
  ```bash
  kubectl config get-contexts
  ```
  ![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/get-context.png)
  - <b>Add your cluster to argocd</b>
  ```bash
  argocd cluster add iam-root-account@wanderlust.ap-south-1.eksctl.io --name wanderlust-eks-cluster
  ```
  > [!Tip]
  > iam-root-account@wanderlust.ap-south-1.eksctl.io --> This should be your EKS Cluster Name.

  ![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/iam-cluster.png)
  - <b> Once your cluster is added to argocd, go to argocd console <mark>Settings --> Clusters</mark> and verify it</b>
  ![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/argo-setting.png)
#
- <b>Go to <mark>Settings --> Repositories</mark> and click on <mark>Connect repo</mark> </b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/argo-repo.png)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/argo-repo-add.png)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/argo-repo-check.png)
> [!Note]
> Connection should be successful

- <b>Now, go to <mark>Applications</mark> and click on <mark>New App</mark></b>

![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/argo-general.png)

> [!Important]
> Make sure to click on the <mark>Auto-Create Namespace</mark> option while creating argocd application

![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/argo-source.png)

- <b>Congratulations, your application is deployed on AWS EKS Cluster</b>
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/argo-ash.png)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/argo-work.png)
- <b>Open port 31000 and 31100 on worker node and Access it on browser</b>
```bash
<worker-public-ip>:31000
```
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/aws.png)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/wander.png)
![image](https://github.com/user-attachments/assets/64394f90-8610-44c0-9f63-c3a21eb78f55)

#
## How to monitor EKS cluster, kubernetes components and workloads using prometheus and grafana via HELM (On Master machine)
- <p id="Monitor">Install Helm Chart</p>
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
```
```bash
chmod 700 get_helm.sh
```
```bash
./get_helm.sh
```

#
-  Add Helm Stable Charts for Your Local Client
```bash
helm repo add stable https://charts.helm.sh/stable
```

#
- Add Prometheus Helm Repository
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

#
- Create Prometheus Namespace
```bash
kubectl create namespace prometheus
```
```bash
kubectl get ns
```

#
- Install Prometheus using Helm
```bash
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
```

#
- Verify prometheus installation
```bash
kubectl get pods -n prometheus
```

#
- Check the services file (svc) of the Prometheus
```bash
kubectl get svc -n prometheus
```

#
- Expose Prometheus and Grafana to the external world through Node Port
> [!Important]
> change it from Cluster IP to NodePort after changing make sure you save the file and open the assigned nodeport to the service.

```bash
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
```
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/prom-node.png)
![image](https://github.com/user-attachments/assets/ed94f40f-c1f9-4f50-a340-a68594856cc7)

#
- Verify service
```bash
kubectl get svc -n prometheus
```

#
- Now,let’s change the SVC file of the Grafana and expose it to the outer world
```bash
kubectl edit svc stable-grafana -n prometheus
```
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/grafana-node.png)

#
- Check grafana service
```bash
kubectl get svc -n prometheus
```

#
- Get a password for grafana
```bash
kubectl get secret --namespace prometheus stable-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
> [!Note]
> Username: admin

#
- Now, view the Dashboard in Grafana
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/dash1.png)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/dash2.png)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/dash3.png)
![image](https://github.com/harshitsahu2311/Wanderlust-GitOps-Project/blob/main/Assets/Images/dash4.png)

#
## Clean Up
- <b id="Clean">Delete eks cluster</b>
```bash
eksctl delete cluster --name=wanderlust --region=us-west-1
```

#
