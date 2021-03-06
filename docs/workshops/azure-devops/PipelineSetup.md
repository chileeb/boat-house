# 使用Azure Pipeline & Azure Kubernetes Service (AKS) 搭建BoatHouse流水线

## 为BoatHouse准备AKS集群和ACR

```shell
## 使用Azure CLI登录
az login

## 设置默认订阅
az account set -s <SubscriptionId>

## 获取当前区域最新版的aks版本号
version=$(az aks get-versions -l <Location> --query 'orchestrators[-1].orchestratorVersion' -o tsv)

## 创建资源组
az group create --name <ResourceGroupName> --location <Location>

## 创建aks集群
az aks create --resource-group <ResourceGroupName> --name <AKSClusterName> --enable-addons monitoring --kubernetes-version $version --generate-ssh-keys --location <Location>

## 创建ACR容器镜像仓库
az acr create --resource-group <ResourceGroupName> --name <ACRName> --sku Standard --location <Location>
az acr login --name <ACRName>

## 获取aks访问密钥
az aks get-credentials --resource-group <ResourceGroupName> --name <AKSClusterName>

# 获取aks集群的clientID
CLIENT_ID=$(az aks show --resource-group <ResourceGroupName> --name <AKSClusterName> --query "identityProfile.kubeletidentity.clientId" --output tsv)

 # 获取acr的资源id
ACR_ID=$(az acr show --name <ACRName> --resource-group <ResourceGroupName> --query "id" --output tsv)

# 授权aks直接访问acr
az role assignment create --assignee $CLIENT_ID --role acrpull --scope $ACR_ID

## 创建namespaces
kubectl create namespace boathouse-test
kubectl create namespace boathouse-prod
kubectl get namespaces

```

## 使用azure voting app演示AKS node pool阔缩容

```shell
## 获取代码
git clone https://github.com/Azure-Samples/azure-voting-app-redis.git
cd azure-voting-app-redis
## build & run
docker-compose up -d
## 发布镜像到ACR
docker tag mcr.microsoft.com/azuredocs/azure-vote-front:v1 <acrname>.azurecr.io/azure-vote-front:v1
az acr login --name <acrname>
docker push <acrname>.azurecr.io/azure-vote-front:v1
## 在AKS上启动应用
kubectl create namespace voting
kubectl apply -f azure-vote-all-in-one-redis.yaml -n voting
kubectl get pods -n voting
kubectl get svc -n vogint
## 更新node pool自动扩容配置
## 更新yaml文件中的replica数量并从新部署
```

## 使用Virutal Node进行动态扩容

```shell
## 注册ACI Provider
az provider register --namespace Microsoft.ContainerInstance
az provider list --query "[?contains(namespace,'Microsoft.ContainerInstance')]" -o table

## 创建资源组
az group create --name <ResourceGroupName> --location <Location>
# 创建vnet
az network vnet create \
    --resource-group <ResourceGroupName> \
    --name myVnet \
    --address-prefixes 10.0.0.0/8 \
    --subnet-name myAKSSubnet \
    --subnet-prefix 10.240.0.0/16

az network vnet subnet create \
    --resource-group <ResourceGroupName> \
    --vnet-name myVnet \
    --name myVirtualNodeSubnet \
    --address-prefixes 10.241.0.0/16

## 创建 service principal 获取 <appId> <password>
az ad sp create-for-rbac --skip-assignment
## 记录你的service principal信息

## 获取VNetId
az network vnet show --resource-group <ResourceGroupName> --name myVnet --query id -o tsv

## 为vnet授权
az role assignment create --assignee <appId> --scope <VNetId>

## 获取<SubNetId>
az network vnet subnet show --resource-group <ResourceGroupName> --vnet-name myVnet --name myAKSSubnet --query id -o tsv

## 创建aks集群
az aks create \
    --resource-group <ResourceGroupName> \
    --name myAKSCluster01 \
    --node-count 1 \
    --network-plugin azure \
    --service-cidr 10.0.0.0/16 \
    --dns-service-ip 10.0.0.10 \
    --docker-bridge-address 172.17.0.1/16 \
    --vnet-subnet-id <SubNetId> \
    --service-principal <appId> \
    --client-secret <password>

## 获取集群密钥
az aks get-credentials --resource-group <ResourceGroupName> --name myAKSCluster01

## 激活虚拟节点
az aks enable-addons \
    --resource-group <ResourceGroupName> \
    --name myAKSCluster01 \
    --addons virtual-node \
    --subnet-name myVirtualNodeSubnet

## 获取节点列表
kubectl get nodes

## 部署示例
kubectl apply -f virtual-node.yaml
kubectl get pods -o wide

## 运行临时容器并测试ACI容器可访问
kubectl run -it --rm testvk --image=mcr.microsoft.com/aks/fundamental/base-ubuntu:v0.0.11
    apt-get update && apt-get install -y curl
    curl -L http://10.241.0.4

```




