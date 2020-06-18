# rancher-rke-ha-azure-shell

## Azure Networking

### Resource Group
In Azure, all resouces need to belong to a resource group so we'll create this first. We're also going to set the default Region and Resource Group just to ensure that all of our resouses are created in the correct place.

```
az group create -l northeurope  -n mb-Rancher-RKE-ResourceGroup
az configure --defaults location=northeurope group=mb-Rancher-RKE-ResourceGroup
```
### Vnet, Public IPs and Network Security Group
Once these commands have completed, the networking elements are created inside the Resource Group. These include a vNet with a default subnet, two public IPs for the two Virtual Machines (VMs) that we will create later and a Network Secrity Group (NSG).

```
az network vnet create --resource-group mb-Rancher-RKE-ResourceGroup --name RancherRKEVnet --subnet-name RancherRKESubnet

az network public-ip create --resource-group mb-Rancher-RKE-ResourceGroup --name RancherRKEPublicIP1 --sku standard

az network public-ip create --resource-group mb-Rancher-RKE-ResourceGroup --name RancherRKEPublicIP2 --sku standard

az network public-ip create --resource-group mb-Rancher-RKE-ResourceGroup --name RancherRKEPublicIP3 --sku standard

az network nsg create --resource-group mb-Rancher-RKE-ResourceGroup --name RancherRKENSG1

az network nsg rule create -g mb-Rancher-RKE-ResourceGroup --nsg-name RancherRKENSG1 -n NsgRuleSSH --priority 100 \
    --source-address-prefixes '*' --source-port-ranges '*' \
    --destination-address-prefixes '*' --destination-port-ranges 22 --access Allow \
    --protocol Tcp --description "Allow SSH Access to all VMS."

```
### Azure Load Balancer


First, create a Public IP for the Load Balancer.
```
az network public-ip create --resource-group mb-Rancher-RKE-ResourceGroup --name RancherLBPublicIP --sku standard
```

Next, create the Load Balancer with a health probe.
```
az network lb create \
    --resource-group mb-Rancher-RKE-ResourceGroup \
    --name RKELoadBalancer \
    --sku standard \
    --public-ip-address RancherLBPublicIP \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool

az network lb probe create \
    --resource-group mb-Rancher-RKE-ResourceGroup \
    --lb-name RKELoadBalancer \
    --name myHealthProbe \
    --protocol tcp \
    --port 80   
```
Once the Load Balancer is created the NSG is updated. The ports added are **80 and 443** for access to Rancher Server and **6443** for access the the Kubernetes API of K3s.

```
az network nsg rule create \
    --resource-group mb-Rancher-RKE-ResourceGroup \
    --nsg-name RancherRKENSG1 \
    --name myNetworkSecurityGroupRuleHTTP \
    --protocol tcp \
    --direction inbound \
    --source-address-prefix '*' \
    --source-port-range '*' \
    --destination-address-prefix '*' \
    --destination-port-range 80 443 6443 \
    --access allow \
    --priority 200
```

Now add the Load Balancer configuration in the form of three rules. You need a rule for port 80 and a rule for port 443 to distrubute the load for Rancher server across our two VMs. The third rule is for port 6443, which gives access to the Kubernetes API running on each of the VMs.

```
az network lb rule create \
    --resource-group mb-Rancher-RKE-ResourceGroup \
    --lb-name RKELoadBalancer \
    --name myHTTPRule \
    --protocol tcp \
    --frontend-port 80 \
    --backend-port 80 \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool \
    --probe-name myHealthProbe

az network lb rule create \
    --resource-group mb-Rancher-RKE-ResourceGroup \
    --lb-name RKELoadBalancer \
    --name myHTTPSRule \
    --protocol tcp \
    --frontend-port 443 \
    --backend-port 443 \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool \
    --probe-name myHealthProbe

az network lb rule create \
    --resource-group mb-Rancher-RKE-ResourceGroup \
    --lb-name RKELoadBalancer \
    --name myHTTPS6443Rule \
    --protocol tcp \
    --frontend-port 6443 \
    --backend-port 6443 \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool \
    --probe-name myHealthProbe
```

## Azure Virtual Machines

Next, we'll create the THREE VMs and install DOCKER on ALL of them.

### Network Interfaces

With all of the network elements created, we can create the Network Interface Cards (NIC) for the VMs.
```
az network nic create --resource-group mb-Rancher-RKE-ResourceGroup --name nic1 --vnet-name RancherRKEVnet --subnet RancherRKESubnet --network-security-group RancherRKENSG1 --public-ip-address RancherRKEPublicIP1 --lb-name RKELoadBalancer --lb-address-pools myBackEndPool

az network nic create --resource-group mb-Rancher-RKE-ResourceGroup --name nic2 --vnet-name RancherRKEVnet --subnet RancherRKESubnet --network-security-group RancherRKENSG1 --public-ip-address RancherRKEPublicIP2 --lb-name RKELoadBalancer --lb-address-pools myBackEndPool

az network nic create --resource-group mb-Rancher-RKE-ResourceGroup --name nic3 --vnet-name RancherRKEVnet --subnet RancherRKESubnet --network-security-group RancherRKENSG1 --public-ip-address RancherRKEPublicIP3 --lb-name RKELoadBalancer --lb-address-pools myBackEndPool
```
### Create the Virtual Machines

To create two virtual machines, first create a text file with our cloud-init configuration. This will deploy Docker, add the ubuntu user to the docker group and install k3s.

```
cat << EOF > cloud-init.txt
#cloud-config
package_upgrade: true
packages:
  - curl
output: {all: '| tee -a /var/log/cloud-init-output.log'}
runcmd:
  - curl https://releases.rancher.com/install-docker/18.09.sh | sh
  - sudo usermod -aG docker ubuntu
EOF
```
Deploy the virtual machines.
```
az vm create \
  --resource-group mb-Rancher-RKE-ResourceGroup \
  --name rkeNode1 \
  --image UbuntuLTS \
  --nics nic1 \
  --admin-username ubuntu \
  --generate-ssh-keys \
  --custom-data cloud-init.txt


az vm create \
  --resource-group mb-Rancher-RKE-ResourceGroup \
  --name rkeNode2 \
  --image UbuntuLTS \
  --nics nic2 \
  --admin-username ubuntu \
  --generate-ssh-keys \
  --custom-data cloud-init.txt

az vm create \
  --resource-group mb-Rancher-RKE-ResourceGroup \
  --name rkeNode2 \
  --image UbuntuLTS \
  --nics nic3 \
  --admin-username ubuntu \
  --generate-ssh-keys \
  --custom-data cloud-init.txt

```

# Deploy Rancher Server with RKE and Helm 3

> Ensure you have completeed the setup tasks on the first VM.

## Install RKE, Kubctl, Helm and Rancher

### Test Docker Install is Present

```
docker version
```
### Install RKE CLI

Download the `RKE CLI`

```
sudo wget -O /usr/local/bin/rke \
https://github.com/rancher/rke/releases/download/v1.1.2/rke_linux-amd64
sudo chmod +x /usr/local/bin/rke
```

Next, let's validate that RKE is installed properly:

```
rke -v
```

You should have an output similar to:

`rke version v1.0.4`

### Install Kubectl

In order to interact with our Kubernetes cluster after we install it using `rke`, we need to install `kubectl`. 

The following command will add an apt repository and install `kubectl`.

```
sudo apt-get update && sudo apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg \
| sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" \
| sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

After the `apt-get install -y kubectl` command finishes, we can test `kubectl` and make sure it is properly installed.

```
kubectl version --client
```

### Install Helm 3 Client

Download and Install Helm 3
```
sudo curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
sudo chmod 700 get_helm.sh
sudo ./get_helm.sh
```

Check Heml 3 Install

```
helm version --client
```

### Run RKE To Build Our Cluster

Run RKE config to configure our cluster.yaml

```
rke config
```
Lets take a look at our `cluster.yaml`
```
cat cluster.yml
```

Install Kubernetes

```
rke up
```
Show the files created..
```
ls
```

We can soft symlink the `kube_config_cluster.yml` file to our `/home/ubuntu/.kube/config` file so that `kubectl` can interact with our cluster:

```
mkdir -p /home/ubuntu/.kube
ln -s /home/ubuntu/kube_config_cluster.yml /home/ubuntu/.kube/config
```

In order to test that we can properly interact with our cluster, we can execute two commands: 

```
kubectl get nodes
```

```
kubectl get pods --all-namespaces
```
>Kubernetes Is now up and Running....

# Install Rancher Server via Helm 3

### Install Cert-Manger

Instll Cert-Manager intot he cert-manager namespace.

```
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.12/deploy/manifests/00-crds.yaml
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.12.0
```

Check cert-manager pods
```
kubectl get pods --namespace cert-manager
```

### Install Rancher
```
kubectl create namespace cattle-system
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

Finally, we can install Rancher using our `helm install` command.

>Make sure to look up the server IP for the command below.

```
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.<AzurePublicIPHere>.xip.io
```
### YeeHaa!!!