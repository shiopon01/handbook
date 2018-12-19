
# ReplicationController

## 概要

レプリケーションコントローラーはコントローラーオブジェクトのひとつで、指定された数のポッドが実行されていることを確実にするためのオブジェクト。主に水平分散を行う際に使用され、ローリングアップデートも実装されているのでそのままアプリケーションのアップデートも容易に行うことが出来る。

現在は次世代の機能としてレプリカセットが提供されているので、レプリケーションコントローラーを使うことはあまり推奨されていない。

- [ReplicationController - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)

> ReplicaSetとDeploymentを使ってください

## 仕組みと役割

レプリケーションコントローラーの主な機能は、予め指定しておいた `レプリカ` の数だけポッドを起動することを保証することだ。

あるレプリケーションコントローラーに紐付くポッドの数が指定されたレプリカ数より多くなった時、レプリケーションコントローラーは増えすぎたポッドを削除してレプリカ数とポッドの数を合わせる。逆にポッドの数が少なかった時はポッドを追加して数を合わせる。（ポッドをどのノードに作成するかは `kube-scheduler` 決定する）

このセルフヒーリングを実現するため、レプリケーションコントローラーはポッドを常に監視して生存確認を行っている。単純なケースでは、1つのレプリケーションコントローラーを作成して1つのポッドを作成することで、無期限に確実にポッドを動作させることができる。

### スケーリング

レプリケーションコントローラーを使用すると、レプリカの数（ `` フィールド）を更新するだけで簡単にレプリカの数を増減できる。これは手動か、自動的なスケーリングのプログラムなどによって行うことが出来る。

レプリケーションコントローラーが複数のノードに作成したポッドは `Service` によって内部、または外部に公開される。複数のノードにポッドを作成しても問題なく通信を行うことができるのはこの `Service` のおかげだ。（これは自分で別途作成する必要がある）

### ローリングアップデート

レプリケーションコントローラーはポッドを1つずつ置きかえていくことでローリングアップデートを実現している。（ `kubectl rolling-update` ）

更新が実行された時、Kubernetesは全く新しいレプリケーションコントローラーを作成し、新しいコントローラー（+1）と古いコントローラー（-1）でポッドを1ずつスケールし、古いコントローラーのポッドが0に達した時点で古いコントローラーを削除する。

アップデートに使用される2つのレプリケーションコントローラーは、イメージのタグなどで少なくとも1つ以上の差別化できるラベルを持つ必要がある。


## Kubernetes template yaml

次のテンプレートは何もしないポッドを3つ維持するレプリケーションコントローラーのを作成する `yaml` 。

- [ドキュメント](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#replicationcontroller-v1-core)

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-rc
spec:
  replicas: 3 # レプリカ数の指定
  selector:
    app: my-rc-pod # 管理するポッドのラベル
  template:
    metadata:
      labels:
        app: my-rc-pod # .spec.selector.app と同じもの
    spec:
      containers:
      - name: my-pod-container
        image: busybox
        command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

このテンプレートでポッドを3つ作成し、維持するレプリケーションコントローラーが立ち上がる。

`.spec.selector` フィールドはこのレプリケーションコントローラーが管理するポッドを特定するためのもので、ポッドのラベル **.spec.template.metadata.labels** と同じものにする。このセレクターとラベルを使うことで、レプリケーションコントローラーはポッドのIPアドレスを知らなくてもポッドを管理することができる。（このラベルは異なる役割のポッドインスタンスで重複させるべきではない）

レプリケーションコントローラーの確認には `kubectl get rc` を使う。同時に `kubectl get pods` でレプリケーションコントローラーが作成したポッドも確認できるはずだ。

```
$ kubectl apply -f rc.yml
replicationcontroller/my-rc created

$ kubectl get rc
NAME      DESIRED   CURRENT   READY     AGE
my-rc     3         3         3         24s

$ kubectl get pods
NAME          READY     STATUS    RESTARTS   AGE
my-rc-hj8bh   1/1       Running   0          38s
my-rc-ngwhm   1/1       Running   0          38s
my-rc-tl4qr   1/1       Running   0          38s
```

### レプリカの維持

レプリカ数は3に設定されているのでポッドは3つの状態で維持されるはずだ。試しに、ポッドの削除を試してみる。

```
$ kubectl delete pods my-rc-ngwhm my-rc-tl4qr
pod "my-rc-ngwhm" deleted
pod "my-rc-tl4qr" deleted

$ kubectl get pods
NAME          READY     STATUS        RESTARTS   AGE
my-rc-49frn   1/1       Running       0          12s
my-rc-hj8bh   1/1       Running       0          5m
my-rc-ngwhm   1/1       Terminating   0          5m
my-rc-sfdl9   1/1       Running       0          12s
my-rc-tl4qr   1/1       Terminating   0          5m
```

ポッドが削除された時は即座に新しいポッドが立ち上がり、3つの状態が維持されていることが確認できる。ステータスが `Terminating` のポッドは後に完全に削除される。

### レプリカ数の更新

テンプレートのレプリカ数を変更して更新すると、維持されるポッドの数も変更されたことが分かる。 `.spec.replicas` の値を2にして更新してみる。

```yaml
spec:
  replicas: 2 # レプリカ数の指定
```

```
$ kubectl apply -f rc.yml
replicationcontroller/my-rc configured

$ kubectl get pods
NAME          READY     STATUS        RESTARTS   AGE
my-rc-49frn   1/1       Running       0          4m
my-rc-hj8bh   1/1       Running       0          9m
my-rc-sfdl9   1/1       Terminating   0          4m
```
