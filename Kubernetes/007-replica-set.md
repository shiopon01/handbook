# ReplicaSet

## 概要

レプリカセットは次世代のレプリケーションコントローラーと呼ばれている。とはいっても機能はほとんどレプリケーションコントローラーと同じで、変わったところはセットベースのセレクターがサポートされた点と、ポッドのアップデートに関する機能が削除された点くらい。

ポッドのアップデート機能が削除されたのは、レプリカセットがコントローラーオブジェクト `Deployment` のバックエンドとして使用されることを前提とした機能として生まれたからだ。レプリカセット単体でも使用できるが、特別な状況でない場合はデプロイメントのバックエンドとして使用することが推奨されている。

- [ReplicaSet - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

> ReplicaSetはDeploymentのバックエンド

## 仕組みと役割

レプリカセットの主な機能は、予め指定しておいた `レプリカ` の数だけポッドを起動することを保証すること。レプリケーションコントローラーの後継だけあって、ほぼ同じ機能とコマンドが実装されている。

レプリケーションコントローラーとレプリカセットの違いは、レプリケーションコントローラーが単体で動作するよう設計されたことに対し、レプリカセットが `Deployment` のバックエンドとして設計された点である。実際、 `rolling-update` はコマンドから削除され、アップデートの機能は `Deployment` に分離された。

### セレクタ

レプリケーションコントローラーは等価ベースのセレクタをサポートしていた。これは、セレクタで指定した文字列と完全に一致するラベルを持つポッドを管理するというもの。

次のセレクタの例は、レプリケーションコントローラーがラベル `app: my-app` を持つポッドを管理する設定。

```yaml
spec:
  selector:
    app: my-app
```

一方レプリカセットに実装されているセットベースのセレクタはオプションが追加されていて、細かいセレクタが定義できるようになった。

セットベースのセレクターでは、次のキーがサポートされている。

- **matchLabels** : `{Key: Value}` のマップであり、一致するラベルを持つポッドを管理する。等価ベースのセレクタと同じ
- **matchExpressions** : `{key: Key, operator: Op, values: [Value]}` の形で定義し、 `operator` に指定された条件でラベルをチェックする。keyはラベルのキー（ `app` など）を指定し、operatorには `In` `NotIn` `Exists` `DoesNotExits` のうちひとつを指定する。valuesはチェックに利用するラベルの値の配列であり、operatorで指定した演算子によってチェックされる

`matchLabels` と `matchExpressions` の両方を設定した場合は条件のANDで扱われる。

例えば次のセレクタの例では、ラベル `app: my-app` を持っていてなおかつ `tier: frontend` または `tier: backend` のどちらかを持つポッドを管理する条件になる。

```yaml
selector:
  matchLabels:
    app: my-app
  matchExpressions:
    - {key: tier, operator: In, values: [frontend, backend]}
```

- [Labels and Selectors - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)


## Kubernetes template yaml

次のテンプレートは何もしないポッドを3つ維持するレプリカセットを作成する `yaml` 。（レプリカセットは `apiVersion: apps/v1` の機能なので `apps/v1` を指定している）

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-rs # レプリカ数の指定
spec:
  replicas: 3
  selector: # セレクタの指定
    matchLabels:
      app: my-rs-pod
    matchExpressions:
    - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      labels:
        app: my-rs-pod
        tier: frontend
    spec:
      containers:
      - name: my-pod-container
        image: busybox
        command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

このテンプレートでポッドを3つ作成し、維持するレプリカセットが立ち上がる。

この時、セレクタとポッドのラベルが噛み合っていない時はエラーが出るので、管理されていないポッドが生成されてしまうことはない。

レプリカセットは `kubectl get rs` で確認できる。同時にポッドも確認する。

```
$ kubectl apply -f rs.yml
replicaset.apps/my-rs created

$ kubectl get rs,pods
NAME                          DESIRED   CURRENT   READY     AGE
replicaset.extensions/my-rs   3         3         3         1m

NAME              READY     STATUS    RESTARTS   AGE
pod/my-rs-h8c4r   1/1       Running   0          1m
pod/my-rs-h8v2f   1/1       Running   0          1m
pod/my-rs-s6cfv   1/1       Running   0          1m
```
