## 環境変数の準備

```
export AKS_NAME=myaks
export RG_NAME=mygroup
export LOCATION=japaneast
export VNET_NAME=myvnet
export AKS_SUBNET_NAME=akssubnet
```


## AKSクラスタの作成

### リソースグループの作成

```
az group create -n $RG_NAME -l $LOCATION
```

### VNETの作成

```
az network vnet create -g $RG_NAME -n $VNET_NAME --address-prefix 10.1.0.0/16  --subnet-name $AKS_SUBNET_NAME --subnet-prefix 10.1.1.0/24
```

### AKSクラスタの作成

パラメータで指定するSubnetIDを取得。

```
AKS_SUBNET_ID=$(az network vnet subnet show --resource-group $RG_NAME --vnet-name $VNET_NAME --name $AKS_SUBNET_NAME --query id -o tsv)
echo $AKS_SUBNET_ID
```

AKSクラスタの作成

```
az aks create \
-n $AKS_NAME \
-g $RG_NAME \
--network-plugin kubenet \
--load-balancer-sku standard \
--vnet-subnet-id  $AKS_SUBNET_ID \
--location $LOCATION \
--node-count 1 \
--node-vm-size "Standard_DS2_v2" \
--no-ssh-key
```

--generate-ssh-key オプションがないというエラーが出た場合には、
. `--no-ssh-key` オプションを指定する
. `--generate-ssh-key` オプションを指定してSSH Keyを生成する
. `ssh-key-value [public key file]` オプションを指定する

各種オプションは[Azure CLI リファレンス (az aks create)](https://learn.microsoft.com/ja-jp/cli/azure/aks?view=azure-cli-latest#az-aks-create)参照。


### AKSクラスタの確認
AKSクレデンシャルの取得

```
az aks get-credentials --admin --name $AKS_NAME --resource-group $RG_NAME
```

ノードの確認
```
kubectl get nodes
```


### Azure管理のリソースグループの確認

AKSをデプロイすると、`MC_` で始まるリソースグループが作成されVMなどがデプロイされるので、そのリソースグループ名を確認する。 

```
az aks show --resource-group $RG_NAME --name $AKS_NAME --query nodeResourceGroup
```

### Internal Loadbalanerの利用

AKS上のアプリケーションへのアクセスをVNET内に限定するために、Internal Loadbalancerを利用するためのロールアサインをする。

```
export SUBS_ID=$(az account show --query id)
export SPID=$(az aks show -g $RG_NAME -n $AKS_NAME --query identity.principalId -o tsv)
export NODE_RG=$(az aks show -g $RG_NAME -n $AKS_NAME --query nodeResourceGroup -o tsv)
export 

az role assignment create \
--assignee $SPID \
--role "Network Contributor" \
--scope "/subscriptions/$SUBS_ID/resourceGroups/$NODE_RG"
```