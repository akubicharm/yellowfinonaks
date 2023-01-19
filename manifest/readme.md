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

namespace を作成し、kubectl コマンド実行時に毎回 `-n` or `--namespace` オプションでnamespaceを指定しなくても済むようにコンテキストを設定。

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


## Podの色々な操作

### Podのログ(stdout, stderr)の参照

Pod名の確認
```
kubectl get pods

上記コマンドの出力結果
NAME                                            READY   STATUS    RESTARTS   AGE
yellowfin-multi-instance-dev-7d64c8b74c-vcv2p   1/1     Running   0          122m
```

stdout, stderrの参照
```
kubectl logs -f yellowfin-multi-instance-dev-7d64c8b74c-vcv2p
```

### Podにアクセス

```
 kubectl exec -it yellowfin-multi-instance-dev-7d64c8b74c-vcv2p  -- /bin/bash
```


### Podとローカルでのファイルのコピー

```
cd [ファイルを保存するディレクトリ]
kubectl cp yf:yellowfin-multi-instance-dev-7d64c8b74c-vcv2p:/opt/yellowfin/appserver/logs .
```
※Dockerイメージ作成時に指定したWORKDIR以外を指定した場合には、「tar: Removing leading `//' from member names」というメッセージが出る
