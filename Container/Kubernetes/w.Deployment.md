# Deployment

`Deployment` は `ReplicaSet` を生成・管理する仕組み。生成されたReplicaSetは `Pod` を生成・管理する。サービスの提供を目的としたポッド生成をこのDeploymentで行うことで、ReplicaSet（サービス）のローリングアップデートやロールバックを行える。

- [Deployment - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

ReplicaSet（ `ReplicationController` の後継）は `PodTemplate` と呼ばれるPodのテンプレートをもとにPodを作成し、Podを指定された数（レプリカ数）に調節するための仕組み。Podがレプリカ数より少ない場合はPodを追加し、多い場合はPodを削除する。この仕組みによって、ノードの障害やアプリのクラッシュでPodが落ちた際のセルフヒーリングが実現されている。

Deploymentはコンテナイメージのバージョンアップなどのアップデートがあった場合に新しいReplicaSetを作成し、新旧ReplicaSetの全体のPod数を調整しながらカナリアリリースを行う。Deploymentの機能によりロールバックも可能。

レプリカ数のみ変更された場合は新しいReplicaSetを作成せず、現在のReplicaSetのレプリカ数を変更する。（そして、実際のPod数が変更される）

テンプレート

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment # Deploymentの一意な名前
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template: # Pod Template
    metadata:
      labels: # Pod Label
        app: nginx # Name (require)
    spec:
      containers:
        - name: nginx
          image: nginx:1.7.9
          ports:
            - containerPort: 80
```

`spec.template.metadata.labels.app` の値はReplicaSetのラベルとセレクタの一部とPodのラベルに使われる。また、ReplicaSetごとに配下のPodを識別するための `pod-template-hash` というPodTemplateから生成したハッシュ値を持つ。

minikube想定だが、このコマンドでDeploymentの適用から公開、ブラウザでの閲覧まで行える。

```
kubectl create -f  https://k8s.io/examples/controllers/nginx-deployment.yaml
kubectl expose deployment nginx-deployment --type=LoadBalancer
minikube service nginx-deployment
```

削除する場合。

```
kubectl delete service nginx-deployment
kubectl delete deployment nginx-deployment
```

# ローリングアップデート

Deploymentではテンプレートの `spec.template` （コンテナイメージなどを含むPodTemplate）に変更があった場合、自動的にPodのローリングアップデートが行われる。

ローリングアップデートではまず、新しいPodTemplateをからPodを作成するための新たなReplicaSetを作成する。（この時点で、ReplicaSetは新旧合わせて2つ以上存在している。）その後、全体のPod数を調節しながら新しいPodに置き換えていき、最終的には全てのPodが新ReplicaSet以下で管理されることになる。コンテンツ配信はServiceを通して行っており、ServiceはどちらのReplicaSetも参照しているため、コンテンツの提供が途切れることはない。（カナリアリリース）

ローリングアップデートが終わった後も、アップデート前のReplicaSetは一定数保持されている。残しておくことで、新しいReplicaSetに不具合があった場合でもロールバックが可能になる。

ロールバックの例。

```
kubectl rollout history deployment nginx-deployment
kubectl rollout undo deployment nginx-deployment # 直前のバージョンへ
```