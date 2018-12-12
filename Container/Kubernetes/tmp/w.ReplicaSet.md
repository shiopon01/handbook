# ReplicaSet

# WIP

ReplicaSetは次世代のReplication Controllerである。この2つの唯一の違いは、セレクタサポート。ReplicaSetは新しいセットベースのセレクターをサポートするが、レプリケーションコントローラーは等価ベースのセレクター要件のみをサポートする。

- [replicaSet - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

その他の違いとして、 `rolling-update` の未サポートがあるが、これはReplicaSetがDeploymentのバックエンドとして使用されることを前提にしているからである。ローリングアップデート機能を使用する場合は、代わりにDeploymentを使用する。


```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend # require
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template: # Pod Template
    metadata:
      labels: # Pod Label
        app: guestbook
        tier: frontend
    spec:
      containers:
        - name: php-redis
          image: gcr.io/google_samples/gb-frontend:v3
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
          env:
            - name: GET_HOSTS_FROM
              value: dns
          ports:
            - containerPort: 80
```

`.metadata.labels`


`.spec.metadata.labels` は設定しない場合、 `.metadata.labels` と同じ値が設定される。違う値を自分で設定してもOK。