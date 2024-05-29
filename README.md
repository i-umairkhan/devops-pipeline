# devops-pipeline
A secure DevOps pipeline setup on Jenkins with monitoring on AWS.
___
# Phase 1
## Seeting up EC2 instances
- Created a new EC2 security group `devops-pipeline-security-group` and added following inbloud rules.

![image](https://github.com/i-umairkhan/devops-pipeline/assets/81556052/09888a75-c884-4bff-9dc5-53d23f5d88c2)

- Created 3 EC2 `t2.medium` instances in K8s cluster in default `-` VPC, `devops-pipeline-security-group` security group, 25 GB storage and Ubuntu image.

![image](https://github.com/i-umairkhan/devops-pipeline/assets/81556052/918b0831-31c0-4525-ae7f-c6447d4da296)

## Initializing K8S cluster with Kubeadm
- SSH into all three nodes using key.
- On Master Node run following commands.
```
sudo apt update

sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock

sudo apt install -y apt-transport-https ca-certificates curl
sudo mkdir -p -m 755 /otc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1

sudo kubeadm init --pod-network13~-cidr=10.244.0.0/16

# Save join command for furthur use to join worker nodes.

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
```
- On Worker nodes run following commands
```
sudo apt update

sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock

sudo apt install -y apt-transport-https ca-certificates curl
sudo mkdir -p -m 755 /otc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1

sudo kubeadm join 172.31.49.24:6443 --token jstyvd.ofqbdvue9jql4rp8 --discovery-token-ca-cert-hash sha256:686fc231f2995847cfefc13c42528af5eb4c0069c260dfe9d5ba9c5b28ba8ad2
```
## Security scan of K8s cluster with kubeaudit
- Run following commands on Master
```
wget https://github.com/Shopify/kubeaudit/releases/download/v0.22.1/kubeaudit_0.22.1_linux_386.tar.gz
tar -xzf kubeaudit_0.22.1_linux_386.tar.gz
sudo mv kubeaudit /usr/local/bin/
kubeaudit all
```
## Instance setup for SonarQube and Nexus
- Created 2 EC2 `t2.medium` instances in default `-` VPC, `devops-pipeline-security-group` security group, 20 GB storage and Ubuntu image.

- Created an EC2 `t2.large` instances in default `-` VPC, `devops-pipeline-security-group` security group, 30 GB storage and Ubuntu image.

![image](https://github.com/i-umairkhan/devops-pipeline/assets/81556052/b9eb4db3-1f93-4ae8-b6eb-dfb0a9de7ea0)


- Run following commands on SonarQube and Nexus instance to install docker.
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# Give all users permissions to runs docker
sudo chmod 666 /var/run/docker.sock
```
- On SonarQube instance run sonarqube docker container  `docker run -d --name sonar -p 9000:9000 sonarqube:lts-community`

- On Nexus instance run nexus container  `docker run -d --name nexus -p 8081:8081 sonatype/nexus3`

- On Jenkins instance install Java, Jenkins and Docker using commands below.
```
sudo apt update
sudo apt install openjdk-17-jre-headless

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins

  sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# Give all users permissions to runs docker
sudo chmod 666 /var/run/docker.sock
```
- Login to SonarQube, Nexus and Jenkins and change default passwords.
  - Jenkins Port: 8080
  - SonarQube Port: 9000
  - Nexus Port: 8081
# Phase 2
## Jenkins Plugins Installation 
- Install all suggested Jenkins plugins.
- Install following pliugins:
  - Eclipse Temurin installer
  - Config File Provider
  - Pipeline Maven Integration
  - SonarQube Scanner
  - Docker
  - Docker Pipeline
  - docker-build-step
  - Kubernetes
  - Kubernetes Client API
  - Kubernetes Credentials
  - Kubernetes CLI
## Jenkins Plugins Configuration
- Do following plugins/tools configuration

![image](https://github.com/i-umairkhan/devops-pipeline/assets/81556052/e7e39a5a-943c-40cc-8ecf-985a5e721a3c)

![image](https://github.com/i-umairkhan/devops-pipeline/assets/81556052/db16225f-16e4-482e-aaf0-322d00d65b8f)

![image](https://github.com/i-umairkhan/devops-pipeline/assets/81556052/5b78a445-7e72-4f3e-9707-f41558779d25)

![image](https://github.com/i-umairkhan/devops-pipeline/assets/81556052/543b14b2-fe63-48f7-a3dd-4725c5391898)

## Pipeline configuration
- Create a credential `git-cred` in Jenkins cradentials for clonning git repo.
- Create a token in sonarqube `sonar-token` and add it in Jenkins credentials with name `sonar-token`.
- In manage Jenkins system add SonarQube server `sonar` with public IP of sonarqube instance and give it `sonar-token` as a credential.
- Create a webhook in SonarQube for Jenkins.

  ![image](https://github.com/i-umairkhan/devops-pipeline/assets/81556052/2b52eddc-46b8-4727-8f05-47ca3acc593d)

- Copy `maven-releases` and `maven-snapshots` URL form nexus and add in `pom.xml`.
```
<distributionManagement>
   <repository>
      <id>maven-releases</id>
        <url>http://52.91.117.197:8081/repository/maven-releases/</url>
   </repository>
	<snapshotRepository>
      <id>maven-snapshots</id>
        <url>http://52.91.117.197:8081/repository/maven-snapshots//</url>
   </snapshotRepository>
</distributionManagement>
```
- Create a Global maven setting in Mnaged files section and add following:
```
    <server>
      <id>maven-releases</id>
      <username>admin</username>
      <password>umair123</password>
    </server>
    
     <server>
      <id>maven-snapshots</id>
      <username>admin</username>
      <password>umair123</password>
    </server>
```
- Add dockerhub credentials `docker-cred` in Jenkins.
- 
- Install triry on Jenkins server.
```
sudo apt-get install wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```
- Create a Pipeline `devops-pipeline` and add max number of builds to 2.
- 

