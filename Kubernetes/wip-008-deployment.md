# Deployment

## 概要

デプロイメントはレプリカセットを作成・管理する仕組み。サービスの提供を目的としたポッド生成をデプロイメントから行うことで、レプリカセットのカナリアリリースやロールバックを行うことが出来る。

- [Deployment - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

## 機能

デプロイメントはテンプレートからレプリカセットを作成する仕組み。レプリカセットは記述されたポッドの情報（ `PodTemplate` ）をもとにポッドを作成し、指定された数レプリカ数に調節する。

次のテンプレートはデプロイメントからNginxポッドを3つ作成する。

```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.4
        ports:
        - containerPort: 80
```

`.spec.template.metadata.labels.app` の値はポッドのラベルだけでなく、レプリカセットのラベルとセレクタの一部にも使用される。またレプリカセットやポッドは、レプリカセットごとに配下のポッドを識別するための **pod-template-hash** という `PodTemplate` から生成したハッシュ値も持つ。

### リリース

デプロイメントはコンテナイメージのバージョンアップなどのアップデートがあった場合、具体的にはテンプレートの `.spec.template` （コンテナイメージなどを含むPodTemplate）に変更があった時、自動的にポッドのローリングアップデートを開始する。

ローリングアップデートではまず、更新されたPodTemplateから新しいレプリカセットが作成される。その後、新旧両方のレプリカセットのポッドの合計を調整しながらポッドを新しいものに置き換えていき、最終的には全てのポッドが新しいレプリカセット配下のポッドとなり、ポッドを持たない古いレプリカセットは削除される。

ただし、ローリングアップデートが終わった後も、アップデート前の古いレプリカセットはデプロイメントに保持されている。異常があった場合は自動で、または手動でロールバックを行うことが出来るためだ。

次のコマンドは、デプロイメントに保存されているバージョンを確認し、手動でロールバックを行う例。

```
$ kubectl rollout history deployment nginx-deployment
$ kubectl rollout undo deployment nginx-deployment # 直前のバージョンにロールバック
```


## 実際に使ってみる

Minikube上での実行になるが、このコマンドでデプロイメントの作成から公開、ブラウザでの閲覧まで行える。

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
