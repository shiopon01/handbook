# Deployment

## 概要

デプロイメントはコントローラーオブジェクトのひとつで、レプリカセットを作成し、管理する仕組み。作成されたレプリカセットはポッドテンプレートに従ってポッドを作成する。

サービスの提供を目的としたポッドの作成をデプロイメントから行うことで、サービスのアップデートやロールバックを容易に行うことが出来る。

- [Deployment - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

> アプリケーションを提供する最善の仕組み

## 仕組みと役割

デプロイメントの主な機能は、テンプレートからレプリカセットを作成すること。デプロイメントに作成されたレプリカセットはポッドテンプレートからポッドを作成し、サービスを提供する。サービスのアップデートはデプロイメントがレプリカセットを切り替えることで行われる。

レプリケーションコントローラーがポッドの管理（レプリカセット）とロールアウトの管理（デプロイメント）に分離したところのロールアウトの部分なだけあって、レプリカセットのロールアウトとバージョン管理周りの機能が実装されている。

セレクタはレプリカセットと同じく、セットベースのセレクタが提供されている。

### ロールバック（巻き戻し）

デプロイメントは全てのロールアウト履歴をシステムで保持しているので、必要に応じていつでもロールバックを行うことができる。（具体的には、ロールアウトがトリガーされた時に `.spec.template` の内容がデプロイメントによって保持されている）


## Kubernetes template yaml

次のテンプレートはポッドを3つ生成するレプリカセットを作成するデプロイメントの `yaml` 。（デプロイメントは `apiVersion: apps/v1` の機能なので `apps/v1` を指定している）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-deployment-rs-pod
  template:
    metadata:
      labels:
        app: my-deployment-rs-pod
    spec:
      containers:
      - name: my-pod-container
        image: busybox
        command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

このテンプレートはデプロイメントを作成し、他にもレプリカセットとポッドが3つ立ち上がる。レプリカセットでも同じだが、この時セレクタとポッドのラベルが噛み合っていない時はエラーが出る。

`.spec.template.metadata.labels` に設定される値はポッドのラベルのように見えるが、ポッドのラベルと、レプリカセットのラベルとセレクタの一部にも使用される共通の値だ。またデプロイメントから作成されたレプリカセットのポッドは、レプリカセットごとに配下のポッドを識別するための **pod-template-hash** という `PodTemplate` から生成したハッシュ値を持っている。

作成したデプロイメントは `kubectl get deployment` で確認できる。同時にレプリカセットとポッドも確認する。

```
$ kubectl apply -f deployment.yml
deployment.apps/my-deployment created

$ kubectl get deployment,rs,pods
NAME                                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/my-deployment   3         3         3            3           47s

NAME                                             DESIRED   CURRENT   READY     AGE
replicaset.extensions/my-deployment-6f578c74d9   3         3         3         47s

NAME                                 READY     STATUS    RESTARTS   AGE
pod/my-deployment-6f578c74d9-h6zgk   1/1       Running   0          47s
pod/my-deployment-6f578c74d9-srs86   1/1       Running   0          47s
pod/my-deployment-6f578c74d9-ss4tq   1/1       Running   0          47s
```

もっと詳細な情報を見たい場合は `kubectl describe` を使える。

```
$ kubectl describe deployment my-deployment
Name:                   my-deployment
Namespace:              default
CreationTimestamp:      Thu, 20 Dec 2018 11:49:11 +0900
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision=1
                        kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"my-deployment","namespace":"default"},"spec":{"replicas":3,"selector":...
Selector:               app=my-deployment-rs-pod
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=my-deployment-rs-pod
  Containers:
   my-pod-container:
    Image:      busybox
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      echo Hello Kubernetes! && sleep 3600
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   my-deployment-6f578c74d9 (3/3 replicas created)
Events:          <none>
```

### ロールアウト（公開）

デプロイメントはimageのバージョンアップなどのアップデートがあった場合、具体的にはテンプレートの `.spec.template` （コンテナイメージなどを含むPodTemplate）に変更があった時、自動的にポッドのローリングアップデートを開始する。

更新が実行された時、デプロイメントは新しいレプリカセットを作成し、新しいレプリカセット（+1）と古いレプリカセット（-1）でポッドを1ずつスケールし、ポッドを全て新しいレプリカセットに移動させる。

レプリケーションコントローラーと違い、レプリカセットはアップデートが終わった後もアップデート前の古いレプリカセットは保持されている。異常があった場合に自動で、または手動でロールバックを行った時、古いレプリカセットもまたポッドを保有することになるからだ。

試しにテンプレートのポッドの `image` を変更し、アップデートを行ってみる。

```yaml
spec:
  containers:
  - name: my-pod-container
    image: nginx
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

変更を適用すると、前述したようにデプロイメントは新しいレプリカセットを作成し、ポッド新しいレプリカセット（+1）と古いレプリカセット（-1）でポッドを1ずつスケールしながらポッドのアップデートを実施する。

適用後にレプリカセットとポッドを確認すると、新しいレプリカセットにポッドが移動し、古いレプリカセットに属していたポッドは削除されていることがわかる。その時、新しいポッドもレプリカぶんだけ作成されている。

```
$ kubectl apply -f deployment.yml
deployment.apps/my-deployment configured

$ kubectl get rs,pods
NAME                                             DESIRED   CURRENT   READY     AGE
replicaset.extensions/my-deployment-67666b8575   3         3         3         3m
replicaset.extensions/my-deployment-6f578c74d9   0         0         0         1h

NAME                             READY     STATUS        RESTARTS   AGE
my-deployment-67666b8575-4vmrm   1/1       Running       0          16s
my-deployment-67666b8575-r7twh   1/1       Running       0          11s
my-deployment-67666b8575-tn4sf   1/1       Running       0          21s
my-deployment-6f578c74d9-h6zgk   1/1       Terminating   1          1h
my-deployment-6f578c74d9-srs86   1/1       Terminating   1          1h
my-deployment-6f578c74d9-ss4tq   1/1       Terminating   1          1h
```

古いレプリカセットが削除されていないことも確認できる。

### ロールバック（巻き戻し）

デプロイメントに保存されているバージョンは `kubectl rollout history` で確認することができ、ロールアウトの項目でロールアウトも行ったので現在は2つのバージョンが存在している。コマンドの出力にある `REVERSION 1` はimageがbusyboxのバージョンで、 `REVERSION 2` はimageがnginxのバージョンだ。

```
$ kubectl rollout history deployment my-deployment
deployments "my-deployment"
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

次の例は、1つ前のバージョンに手動でロールバックを行う例。ロールバックは `kubectl rollout undo` で行うことが出来る。この時デプロイメントのバージョンは `1` に戻るとかではなく、新しく `3` が作成されることに注意。

```
$ kubectl rollout undo deployment my-deployment
deployment.extensions/my-deployment

$ kubectl rollout history deployment my-deployment
deployments "my-deployment"
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

ロールバックを行うと、ロールアウトと同じようにポッドのスケールが行われる。レプリカセットはロールアウトした時の物が残っているはずなので、それが利用される。（ `AGE` を見ると、ポッドを保有しているレプリカセットがたった今作成されたものではないことが分かる）

```
$ kubectl get rs,pods
NAME                                             DESIRED   CURRENT   READY     AGE
replicaset.extensions/my-deployment-67666b8575   0         0         0         11m
replicaset.extensions/my-deployment-6f578c74d9   3         3         3         2h

NAME                                 READY     STATUS    RESTARTS   AGE
pod/my-deployment-6f578c74d9-57mhj   1/1       Running   0          46s
pod/my-deployment-6f578c74d9-9bjq6   1/1       Running   0          52s
pod/my-deployment-6f578c74d9-w4k5m   1/1       Running   0          40s
```

## コマンドからDeploymentを作成する

このコマンドでデプロイメントの作成から公開、ブラウザでの閲覧まで行える。公開はMinikubeで行う。

```
$ kubectl create -f  https://k8s.io/examples/controllers/nginx-deployment.yaml
$ kubectl expose deployment nginx-deployment --type=LoadBalancer
$ minikube service nginx-deployment --url
```

`kubectl expose deployment` はデプロイメントを公開するサービスを作成するコマンド。ここでサービスタイプに `LoadBalancer` を指定しているが、ロードバランサーを配置するクラウドが設定されていないためExternal-IPは `pending` のままになる。（つまり、ただのNodePortが作成されただけだ）

`minikube service` は作成されたサービスに接続するためのコマンド。これで公開されたサービスにアクセスできる。

削除する場合は次のコマンドを実行する。

```
$ kubectl delete service nginx-deployment
$ kubectl delete deployment nginx-deployment
```

### ビルドしたDocker Imageを使ってデプロイ

自分でビルドしたDocker Imageからコマンドでデプロイメントを作成する例。

```
$ docker build -t hello-node:v1 .
```

```
$ kubectl run hello-node --image=hello-node:v1 --port=8080 --image-pull-policy=Never
```

```
$ kubectl get services
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
hello-node   LoadBalancer   10.104.61.79   <pending>     8080:31380/TCP   20s
kubernetes   ClusterIP      10.96.0.1      <none>        443/TCP          1h

$ kubectl get deployments
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-node   1         1         1            1           15s

$ kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
hello-node-57c6b66f9c-fb2wf   1/1       Running   0          2m
```

Imageのアップデートもコマンドから行える。

```
$ kubectl set image deployment/hello-node hello-node=hello-node:v2
deployment.extensions/hello-node image updated
```
