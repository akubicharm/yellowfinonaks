# ボリュームの利用

## アクセスモード

* ReadWriteOnce
  * ボリュームは単一のNodeで読み取り/書き込みとしてマウントできます

* ReadOnlyMany
  * ボリュームは多数のNodeで読み取り専用としてマウントできます

* ReadWriteMany
  * ボリュームは多数のNodeで読み取り/書き込みとしてマウントできます

* ReadWriteOncePod
  * ボリュームは、単一のPodで読み取り/書き込みとしてマウントできます。クラスタ全体で1つのPodのみがそのPVCの読み取りまたは書き込みを行えるようにする場合は、ReadWriteOncePodアクセスモードを使用します。これは、CSIボリュームとKubernetesバージョン1.22以降でのみサポートされます。


|ボリュームプラグイン|ReadWriteOnce|ReadOnlyMany|ReadWriteMany|
| ---- | ---- | ---- | ---- |
|AzureFile CSI|○|○|○|
|AzureDisk CSI|○|-|-|
|hostPath|○|-|-|
|Azure Blog CSI (NFS)|○|○|○|


## 環境変数の準備

リソースグループ、AKSのクラスタ名、Yellowfinアプリをデプロイするk8sのネームスペースを環境変数に設定します。

```
export RG_NAME=mygroup
export AKS_NAME=myaks
export YF_NS=yellowfin
```

## 永続ボリューム(Azure管理リソースグループのストレージクラスを利用する場合)

Persistent Volume Claim を使って永続ボリュームを要求し、Podから利用できるようにする。

![Persistent Volume](https://learn.microsoft.com/ja-jp/azure/aks/media/concepts-storage/aks-storage-options.png)
https://learn.microsoft.com/ja-jp/azure/aks/concepts-storage

AKSデプロイ時に作成されたkubernetesのストレージクラスを利用して、Azure管理リソースグループに生成されるストレージアカウントを永続ボリュームとして利用する場合は、PVC作成、アプリケーションのデプロイを行う。    

### PVCの作成

[サンプル manifest/azurefile-pvc.yaml](./manifest/azurefile-pvc.yaml)を利用して、Persistent Volume Claim を作成

### アプリケーションのデプロイ

[サンプル manifest/deployment-pvc.yaml](./manifest/deployment-pvc.yaml)を利用して、アプリケーションをデプロイ


## 永続ボリューム（自作のストレージクラスを利用する場合）

Storage　AccountをAzure管理のリソースグループに作って利用する場合は、同一リソースグループ内でのアクセスなので権限の付与は入りませんが、異なるリソースグループの場合には、IAMでの権限の不要が必要です。

## 権限の付与

#### 1. ストレージアカウントを作成
#### 2. ストレージアカウントの管理画面の左Paneで「アクセス制御 (IAM)」をクリック
#### 3. 右Pane上部の「+追加」をクリックしプロダウンメニューで「ロール割り当ての追加」を選択
#### 4. 一覧から「ストレージアカウント共同作成者」を選択
#### 5.　アクセス割り当て先「マネージドID」をチェックし、マネージドIDの選択ダイアログで該当するkubernetesサービスを選択
![Managed IDの割り当て](images/volume_managedid.png)


[参考：Azureドキュメント Azureポータルを使用してAzureロールを割り当てる](https://learn.microsoft.com/azure/role-based-access-control/role-assignments-portal?tabs=current)

### ストレージクラスの作成

[サンプル manifest/azurefile-storageclass.yaml](./manifest/azurefile-storageclass.yaml)を利用して、ストレージクラスを作成

### PVCの作成
[サンプル manifest/deployment-pvc.yaml](./manifest/deployment-pvc.yaml)を編集し、Persistent Volume Claimを作成。
storageClassNameを前の手順で作成したストレージクラス名に、storageの容量は必要な値に変更し、AKSに適用

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
  namespace: yellowfin-prod
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: [ここを前の手順で作成したストレージクラスの名前に変更]
  resources:
    requests:
      storage: 5Gi <--- 容量は適宜変更
```

### アプリケーションのデプロイ
[サンプル manifest/deployment-pvc.yaml](./manifest/deployment-pvc.yaml)を利用して、アプリケーションをデプロイ。



## HostPath

hostPathボリュームは、ファイルまたはディレクトリをワーカーノードのファイルシステムからPodにマウントする。
yellowfinのログ出力ディレクトリ　/opt/yellowfin/appserver/logs としてワーカーノードの/var/tmp/tomcatlogs をマウントする場合は、次のように定義する。

```
          volumeMounts:
          - mountPath: "/opt/yellowfin/appserver/logs"
            name: volume
      volumes:
        - name: volume
          hostPath:
            path: /var/tmp/tomcatlogs
            type: DirectoryOrCreate
```

### アプリケーションのデプロイ

[サンプル manifest/deployment_hostpath.yaml](./manifest/deployment_hostpath.yaml)を利用して、Yellowfinアプリケーションをデプロイ。

```
kubectl apply -f deployment_hostpath.yaml -n $YF_NS
```

### ログファイルの確認

#### (1) Podがデプロイされているワーカーノードの確認

```
% kubectl get pods -n $YF_NS -o wide
NAME                                            READY   STATUS    RESTARTS   AGE     IP            NODE                                NOMINATED NODE   READINESS GATES
yellowfin-multi-instance-dev-5c88d8759b-lrsct   1/1     Running   0          2m54s   10.244.2.21   aks-nodepool1-31142085-vmss000006   <none>           <none>
```

#### (2) ノードにアクセスしてログファイルの確認 (kubectl debug)

`kubectl debug` で、ノードにSSHをしてログファイルを確認する。このコマンドでsshした場合には、ワーカーノードのディレクトリツリーは `/host` から辿る。

```
kubectl debug node/aks-nodepool1-31142085-vmss000006 -it --image=mcr.microsoft.com/dotnet/runtime-deps:6.0

cd /host/var/tmp/tomcatlogs
```

node-shell の場合は、サーバにSSHをするのと同じなので`cd /var/tmp/tomcatlogs`でログファイルを確認することが可能。



## 参照
* https://learn.microsoft.com/ja-jp/azure/aks/node-access
* https://github.com/kvaps/kubectl-node-shell
* https://kubernetes.io/ja/docs/concepts/storage/persistent-volumes/