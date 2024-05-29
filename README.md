# devops-pipeline
A secure DevOps pipeline setup with monitoring on AWS.
___
# Phase 1
## Seeting up EC2 instances
- Created a new EC2 security group `devops-pipeline-security-group` and added following inbloud rules.
  
  ![image](https://github.com/i-umairkhan/devops-pipeline/assets/81556052/c868b0d7-a57d-410a-b95e-2ec960c31214)

- Created 3 EC2 `t2.medium` instances in K8s cluster in default `-` VPC, `devops-pipeline-security-group` security group, 25 GB storage and `Ubuntu` image.

  ![image](https://github.com/i-umairkhan/devops-pipeline/assets/81556052/a6c20af7-552d-49bf-8bb7-794c92bc2e8d)

## Initializing K8S cluster with Kubeadm
- SSH into all three nodes using key.
- On Master Node run following commands
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
- Created 2 EC2 `t2.medium` instances in default `-` VPC, `devops-pipeline-security-group` security group, 20 GB storage and `Ubuntu` image.
- Created an EC2 `t2.large` instances in default `-` VPC, `devops-pipeline-security-group` security group, 30 GB storage and `Ubuntu` image.

  ![image](https://github.com/i-umairkhan/devops-pipeline/assets/81556052/45ef403a-dd2d-4946-9b9b-d36321c48a27)

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
- On SonarQube instance run sonarqube docker container
` docker run -d --name sonar -p 9000:9000 sonarqube:lts-community`
- On Nexus instance run nexus container ` docker run -d --name nexus -p 8081:8081 sonatype/nexus3`
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
