# ReplicaSet

## 概要

レプリカセットを簡単に説明するなら、次世代のレプリケーションコントローラーと言える。機能としては殆ど変わらないが、新しいセットベースのセレクターがサポートされた。

また、レプリケーションコントローラーと比べると、ポッドのアップデートに関する機能 `rolling-update` が削除されている。これはレプリカセットが、 `デプロイメント` のバックエンドとして使用されることを前提とした機能だからだ。

- [replicaSet - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

## 機能

レプリカセットは予め設定しておいた `レプリカ` の数だけポッドを生成・維持するという、レプリケーションコントローラーとほぼ同じ機能とコマンドを実装している。

唯一違う点が、 `kubectl rolling-update` が削除されたことだ。`デプロイメント` のバックエンドとして設計され、実際にアップデート周りの機能が `デプロイメント` として分離されたので、レプリカセットからローリングアップデートのコマンドは廃止された。（ローリングアップデート機能を使用する場合は、代わりにデプロイメントを使用する）

レプリカセット単体でも使用できるが、特別な状況でない限りデプロイメントのバックエンドとして使用することが推奨されている。

### セレクタ

レプリケーションコントローラーでは等価ベースのセレクタをサポートしていた。これは、セレクタで指定した文字列と一致するラベルを持つポッドを管理するというものだ。

（ `.spec.metadata.labels` は設定しない時、 `.metadata.labels` と同じ値が設定される。違う値を自分で設定してもOK）

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp
spec:
  replicas: 3
  # Selector
  selector:
    app: my-nginx
  template:
    metadata:
      # Labels
      labels:
        app: my-nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

レプリカセットでは、セットベースのセレクタがサポートされた。これによって出来ることは細かいセレクタが定義できるようになったくらいなのだが、有用ではある。

セットベースのセレクターでは、次のようなキーがサポートされている。

- **matchLabels** : `{Key: Value}` のマップであり、同じラベルを持つポッドを管理する。やっていることは等価ベースのセレクタと同じ。
- **matchExpressions** : `{key: Key, operator: Op, values: [Value]}` の形でラベルをチェックする。keyはラベルのキー（ `name` など）であり、operatorは `In` `NotIn` `Exists` `DoesNotExits` のどれかを指定する。valuesはチェックに利用するラベルの値の配列であり、operatorで指定した演算子によってチェックされる

matchLabelsとmatchExpressionsの両方を設定した場合は、両方の条件のANDで扱われる。

```yaml
apiVersion: v1
kind: ReplicaSet
metadata:
  name: myapp-2
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  # Selector
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      # Labels
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
```

- [Labels and Selectors - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

## Kubernetes YAML

```yaml
apiVersion: v1
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
