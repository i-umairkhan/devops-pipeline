# devops-pipeline
A secure DevOps pipeline setup with monitoring on AWS.
___
# Phase 1
## Seeting up EC2 instances
- Created a new EC2 security group `devops-pipeline-security-group` and added following inbloud rules.
  
  ![image](https://github.com/i-umairkhan/devops-pipeline/assets/81556052/c868b0d7-a57d-410a-b95e-2ec960c31214)

- Created 3 EC2 `t2.medium` instances for K8s cluster in default `-` VPC, `devops-pipeline-security-group` security group and `Ubuntu` image.

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
On Worker nodes run following commands
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
## For security scan of K8s cluster with kubeaudit run following commands on Master
```
wget https://github.com/Shopify/kubeaudit/releases/download/v0.22.1/kubeaudit_0.22.1_linux_386.tar.gz
tar -xzf kubeaudit_0.22.1_linux_386.tar.gz
sudo mv kubeaudit /usr/local/bin/
kubeaudit all
```
