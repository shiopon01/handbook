# ReplicationController

# WIP

レプリケーションコントローラは、podを複数インスタンス化するKubeの概念。
実運用上、podを単一インスタンスで使ってもkubeの旨味が少ないので、必要なレプリカ数を指定したrcを設定するのが普通。

rcは指定したレプリカ数分のpodをインスタンス化する。（プロセスがどのホストで起動するかは実行時にkubeが決める）

rcは指定レプリカ数分のpodを維持する。つまり、どこかのpodが死ぬと、新しいpodが立ち上がる。

`ReplicationController` は、Podを指定の数実行しておくための仕組み。ReplicationControllerの定義ファイルをKubenetesに適用すると、指定されたPodが指定の数（レプリカ数）だけ起動する。また、Podを監視し続けて稼働していることを確認する役割も持つ。（ECSでは `Task Definition` ）

Podが多すぎる場合、ReplicationControllerは過剰ぶんのPodを終了させ、不足している場合は追加する。手動で作成したPodとは異なり、ReplicationControllerで作成・維持されているPodはノードの障害やアプリのクラッシュなどで終了した時、レプリカ数を維持するため自動的に置き換えられる。このレプリカ数を維持する仕組みによって、セルフヒーリングが実現されている。

また、ReplicationControllerはアップデートの際、Podを1つずつ置き換えることでサービスへのローリングアップデートが容易になるように設計されている。新しいController (+1) と古いController (-1) を1ずつスケールし、古いControllerはPodが0個のレプリカに達した時点で削除される。 `kubectl rolling-update`

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-master
spec:
  replicas: 3 # レプリカ数の指定 この数だけPod数を維持
  selector:
    app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis-master
          image: redis:2.8.23
          ports:
            - containerPort: 6379
```