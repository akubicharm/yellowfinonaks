## 環境変数の準備

```
export AKS_NAME=myaks
export RG_NAME=mygroup
```

## アプリケーションのデプロイ

### AKSのクレデンシャルの取得

```
az aks get-credentials --admin --name $AKS_NAME --resource-group $RG_NAME
```

### namespace の作成

namespace を作成し、kubectl コマンド実行時に毎回 `-n` オプションでnamespaceを指定しなくても済むようにコンテキストを設定。

```
export YF_NS=yf
kubectl create ns $YF_NS
kubectl config set-context --current --namespace $YF_NS
```

### Podのデプロイ

```
kubectl apply -f deployment.yaml
kubectl get pods
```

### Serviceのデプロイ (Load Balancerタイプ)

LoadBalancerタイプのServiceをデプロイ

```
kubectl apply -f serviceLB.yaml
kubectl get svc
```

外部公開用のIPアドレスが付与されるまでは、EXTERNAL-IPがpendingの状態。
```
NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
yellowfin-multi-instance-dev   LoadBalancer   10.0.161.195   <pending>     80:30170/TCP   3s
```

ロードバランサーにIPが登録されると、EXTERNAL-IPに外部公開ようのIPアドレスが表示される
```
NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
yellowfin-multi-instance-dev   LoadBalancer   10.0.161.195   20.210.56.216   80:30170/TCP   63s
```

### Azure マネージドリソースグループを確認

Azure マネージドリソースグループの名前を確認

```
az aks show --resource-group $RG_NAME --name $AKS_NAME --query nodeResourceGroup
```

LoadBalancerタイプのServiceをデプロイすると、AzureマネージドリソースグループにPublicIPが作成され、そのPublic IPがLoadBalancerのフロントエンドIPとして登録されるので、AzureポータルでAzureマネージドリソースグループを開き、Public IPとロード版ランサーのFrontend IPを確認する。


### ブラウザからアクセス

AKSをデプロイしたリソースグループにNetwork Security Groupができている場合は、Inboundに My IP Address を許可する設定を追加する。
※Azure LBは、Source IPにClientIPを指定したままアクセスするので

aksのサービスのExternal IPを指定してブラウザからアクセス
