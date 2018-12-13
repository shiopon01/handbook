# Config template

Kubernetesのオブジェクトは `yaml` のテンプレートファイルから作成できる。このテンプレートファイルが初心者にとっての大きな壁となっていることは間違いない。ここはそんなYAMLの書き方を学び、多少なりともKubernetes YAMLに対する抵抗を無くすことを目標とする。

## Kubernetes YAML

ポッドを生成するテンプレートは次のようなものだ。基本的には何を生成するにしても変わるのは `kind` と `spec` の中身くらい。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

### API Version

APIグループを指定する行。（必須）

```yaml
apiVersion: v1
```

これは、このテンプレートファイルをどのように解釈するかというものを定めたもの。 `v1` はAPIで言うと `/api/v1` のようなもの。簡単にフィールドや表現を変更できるよう、つまりはKubernetes APIの拡張を容易にするために存在している。

安定したバージョンは `xX` となっているもので、大文字の `X` は数字になるが、alphaやbetaのバージョンも存在する。

- [API versioning](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning)
- [Supporting multiple API groups](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/api-group.md)

### Types (Kinds)

テンプレートが何を生成するかを指定する行。（必須）

```yaml
kind: Pod
```

このテンプレートで作成するオブジェクトの種類を指定する。ここに指定するのは `Pod` `Service` `Deployment` など、APIで予め決められている文字列である。

ここに設定する文字列によって、後の `spec` フィールドに設定する値が変わる。specはkindで指定したオブジェクトの詳細を設定する場所で、例えばkindが `ReplicaSet` だったなら、`.spec.replicas` や `.spec.template` の項目への記述が必要になる。

- [types-kinds](https://github.com/kubernetes/community/blob/master/contributors/devel/api-conventions.md#types-kinds)

### Metadata

Metadataでは生成するオブジェクトの識別情報を設定する。（必須）

```yaml
metadata:
  name: myapp-pod
  labels:
    app: myapp
```

全てのオブジェクトは `metadata` フィールドにメタデータ設定しなければならない。ここで言われるメタデータはいわゆる識別子で、オブジェクトの名前やラベル、名前空間の設定などを行う。

設定できるメタデータはいくつかあるが、 `name` 以外はデフォルト値が設定されているのでユーザーが入力する必要はない。（ただし、 `name` の設定は必須）

#### name

現在の名前空間（namespace）でこのオブジェクトを一意に識別するための文字列。テンプレートでの設定が必須。

- [Names - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/)

#### namespace

オブジェクトを細分化するためのDNS互換ラベル、またはグループのようなもの。デフォルトでは `default` が設定されているが、テンプレートで設定することで変更できる。

- [Namespaces - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

#### labels

オブジェクトの整理と分類に使用できる文字列キーと値のmap。

- [Labels and Selectors - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

#### uid

削除されて再生成された際などに生まれる、同じ名前の同じオブジェクトを区別するために使用される。

uidは名前（name）と名前空間（namespace）の中で一意な値となり、デフォルトでは乱数（RFC 4122）が設定されるが自分で設定することもできる。

#### resourceVersion

オブジェクトがいつ変更されたかを判断するための、オブジェクト内部バージョンを識別するための文字列。

#### generation

特定の世代を表すシーケンス番号？

#### creationTimestamp

オブジェクトが生成された日時の文字列（RFC 3339）。

#### deletionTimestamp

オブジェクトが削除される日時の文字列（RFC 3339）。テンプレートで設定するための項目で、deletionTimestampを設定されたオブジェクトは指定した時間に削除される。

#### annotations

外部ツールで使用できる文字列キーと値のmap？


### Spec

kindで指定されたオブジェクトの詳細を設定するフィールド。前述のテンプレートのように、Podのspecはこのように指定される。

```yaml
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

それぞれのオブジェクトに対するspecのパラメーターは[Kubernetes API Reference Docs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#-strong-workloads-strong-)で確認できる。（YAMLテンプレートはこれを見ながらがんばって作る）


## テンプレートの適用

テンプレートを用意できてなら、あとはkubectlでKubernetesに適用するだけだ。

オブジェクトの作成には `kubectl create` を使用する。場合は既に同名のオブジェクトが存在している場合はAlreadyExistsで怒られる。

```
$ kubectl create -f pod.yml
pod/myapp-pod created

$ kubectl create -f pod.yml
Error from server (AlreadyExists): error when creating "pod.yml": pods "myapp-pod" already exists
```

オブジェクトを更新したいときは `kubectl patch` を使用する。 `patch` は既存のオブジェクトに差分のみ反映（PATCH）され、 `replace` は完全に上書き（PUT）となる。

削除は `kubectl delete` で行う。

```
$ kubectl delete -f pod.yml
pod "myapp-pod" deleted
```

### apply

kubectlからオブジェクトを操作ためのコマンドは前述した4つ（ `create` `replace` `patch` `delete` ）以外にも、もうひとつ `apply` というものがある。

`apply` はこのうち `replace` 以外の3つのコマンドを合体させたようなもので、状況に合わせて自動的に使い分けてくれる。例えば、存在しない名前のオブジェクトのテンプレートをapplyした時は `create` 、存在する名前のオブジェクトのテンプレートをapplyした時は `patch` のように、使い分ける。

```
$ kubectl apply -f deployment.yml
deployment/myapp created
```

```
$ kubectl apply -f deployment.yml
deployment/myapp configured
```

## 参考

- [API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/api-conventions.md)
