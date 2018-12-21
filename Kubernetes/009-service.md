# Service

## 概要

各ポッドには専用のIPアドレスが割り振られているが、それは永続的なものではない。例えばポッドが更新された時、またはデプロイメントによってレプリカセットが切り替えられた時など、ポッドのIPアドレスが変更されるタイミングはいくつか存在する。そのため、アプリを公開する時にポッドのIPアドレスをそのまま使うという選択は現実的ではない。

ここで **Service** を利用できる。サービスは不変のIPアドレスを持ち、負荷分散を行いながらリクエストをポッドに転送するロードバランサのような役割を持っている。このおかげで複数存在するポッドを意識すること無くポッドにアクセスすることができるようになっている。

- [Service - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/service/)

> ポッドへの橋渡し

## 仕組みと役割

サービスの機能は届いたリクエストをポッドに転送することだ。サービスは不変の仮想IPアドレス（ifconfigで見れないやつ）を持ち、このアドレスに送信されたパケットをポッドに転送する役割を持つ。この際、ポッドが複数あった場合は負荷分散が行われる。

この役割をもつサービスだが、実際はServiceプロセスなどがこの役割を担っている訳ではないというところに注意してもらいたい。サービスはKubernetesの抽象概念であり、実体は `kube-proxy` 、 `kube-dns` 、 `Endpoints` やそれを支えるいくつかの機能の集合だ。（普通に利用するだけなら、抽象概念としてのサービスを知っているだけで問題はない）



あるノードに属するポッドが同クラスタ内のサービスにアクセスしようとした時、アクセスには相手のサービス名を利用できる（ `http://frontend:8080` ）。このサービス名をサービスの仮想IPアドレスに変換するのが `kube-dns` の役割で、パケットの宛先を変換したIPアドレスに書き換える。（この情報に限らず、多くのネットワークの情報はKVS `etcd` に保存されている。）

次に、サービスの仮想IPアドレスを実際に存在するポッドIPアドレスに変換する。この作業は同ノードの `iptables` ルールによって宛先の変換が行われ、パケットの宛先がポッドIPアドレスに書き換えられる。ポッドが複数存在する場合はここでルールに従って負荷分散が行われ、パケットはポッドに送信される。

さて、この `iptables` ルールは常に `kube-proxy` によって変更され続けているものだ。kube-proxyは `Endpoints records` の情報を `kube-apiserver` から定期的に取得し、同ノードのiptablesを更新している。このおかげでサービスへのリクエストはポッドにたどり着くことができる。

このエンドポイントレコードは基本的に *セレクタを持つサービスを作成した時に自動生成される* もので、マップされたサービスが持つポッドのIPアドレスを全て保有するものだ。自分で作成し、サービスにマップすることも可能。

このように、サービスは多くの機能から成り立つ抽象概念であることが分かる。

より詳しいKubernetesのネットワークについては次の記事。

- [Kubernetes network]

### Endpoints

対応するサービス名とポッドのIPアドレスを保存したものを `Endpoints records` と呼ぶ。エンドポイントレコードは `kubectl get endpoints` で取得でき、例えば次のようなものだ。

```
$ kubectl get endpoints
NAME     ENDPOINTS                                   AGE
my-svc   172.17.0.4:80,172.17.0.7:80,172.17.0.8:80   8m
```

ここで表示されているレコードは `my-svc` という名前のサービスとひも付いており、 `172.17.0.4:80` などを含む3つのエンドポイント（ポッドのIPアドレス）を保有している。このレコードが作成されたタイミングは、3つのレプリカを持つレプリカセットのサービスを作成したタイミングだ。（レプリカセットがスケールするなどでポッドが増減、IPアドレスが変化した時、エンドポイントレコードは自動で更新される。）

エンドポイントの情報はマスターノードのKVS `etcd` に保存され、この情報は `kube-proxy` が `kube-apiserver` を使って定期的な取得を行う。

### セレクタなしのサービス

セレクタなしのサービスを作成すると、エンドポイントレコードは作成されない。（セレクタでポッドを識別するため、エンドポイントレコードは作成できない）

これは主に、サービスの宛先をクラスタ外部（または別のネームスペース）のサービスやIPアドレスに向ける必要がある場合に使用される。以下、公式ドキュメントより引用。

- 本番環境では外部データベースクラスタを使用したいが、テストでは独自のデータベースを使用する
- サービスをNamespace別のクラスタまたは別のクラスタのサービスに向ける 必要があります。
- 作業負荷をKubernetesに移行し、バックエンドの一部をKubernetesの外部で実行しています。

セレクタが存在しないサービスには、独自のエンドポイントレコードをマップすることができる。（ `.metadata.name` が同じ名前のサービスにマップされる）

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-svc
subsets:
- addresses:
  - ip: 1.2.3.4
  ports:
  - port: 9376
```

## ServiceTypes

Kubernetesのサービスには4つの種類があり、作成時にどのタイプのサービスを作成するか設定できる。

デフォルトのタイプは `ClusterIP` で、他にもClusterIPを拡張したタイプの `NodePort` 、NodePortを拡張したタイプの `LoadBalancer` が存在する。

### ClusterIP

ポッドをクラスタ内部に公開するためのサービスタイプ。

`仕組みと役割` 項目の内容がこれで、仮想IPアドレス（ClusterIP）を持つサービスを作成できる。 `Endpoints` や `セレクタなしのサービス` での内容のように、指定するセレクタの有無によってClusterIPをどこに接続させるかを変えられる。

ClusterIPを使ったクラスタ内部でのサービス同士の通信はマイクロサービスのような仕組みに活用することができる。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-svc
spec:
  selector:
    app: my-app
  ports:
  - port: 80
```

サービスの確認には `kubectl get svc` を使う。テンプレートの適用後、設定ファイルでは指定していないが `TYPE: ClusterIP` となっていることが分かる。（明示的に指定したい場合はテンプレートで `type: ClusterIP` を指定する）

```
$ kubectl apply -f svc.md
service/my-svc created

$ kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
my-svc       ClusterIP   10.96.50.75   <none>        80/TCP    5m
```

この時、 `.spec.selector` で指定した `my-app` で指定できるポッドが3つ存在していた場合、エンドポイントレコードには3つのポッドのIPアドレスが保存されている。レプリカ数の変更などでポッドの数やIPアドレスが変化しても自動で更新される。

もしポッドが存在しないセレクタを指定してもサービス・エンドポイントレコードの作成は失敗せず、レコードのエンドポイントが `<none>` と表示される。

```
$ kubectl get endpoints
NAME         ENDPOINTS                                   AGE
my-svc       172.17.0.4:80,172.17.0.7:80,172.17.0.8:80   5m
my-svc2      <none>                                      5m
```

ポッド内部からこのサービスへは `http://my-svc` でアクセスが可能。（なはず）

### NodePort

サービスをクラスター外部に公開するためのサービスタイプ。

NodePortはClusterIPを拡張する形でサービスを外部に公開する。iptablesにはClusterIPの設定に加え、外部からアクセスできるNodePortに来たリクエストをローカルアドレスに転送するルールが追加される。

公式ドキュメントのこの記述は、NodePortをよく知らない時は正直分からないとおもう。

> 各ノードのIP上のサービスを静的ポート（the NodePort）に公開します。ClusterIP先のサービス、NodePortサービスはルート、自動的に作成されます。NodePort要求することで、クラスタの外からサービスに連絡することができます<NodeIP>:<NodePort>。

NodePortが適用されると、Kubenetes Masterはサービス用に3000-32767からポートを割り当て、全ノードで同じポートを開放させる。リクエストを受けたノードはkube-proxyを経由して指定されたセレクタのポッドにリクエストを割り振る。

HTTPロードバランサである `ingress` はここで公開されるポートを使って実装されている。

### LoadBalancer

サービスをクラウドプロバイダのロードバランサと紐付けるためのサービスタイプ。

LoadBalancerはNodePortを拡張する形でサービスをロードバランサと紐付ける。ロードバランサとの紐付け時に設定される `EXTERNAL-IP` 宛のリクエストは、NodePortと同じくローカルアドレスに転送される。（結果、ポッドに転送される）

Cloud Providerの設定が正しければ、Kubernetesの設定を行った時にクラウドプロバイダのロードバランサも自動的に設定される。（GKE、EKSとか使ってる時は不要だと思う…）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: 0.0.0.0/0
spec:
  type: LoadBalancer
  selector:
    app: express
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
  loadBalancerIP: 10.132.160.51
```


## 参考

- https://cloud.google.com/kubernetes-engine/docs/concepts/network-overview
- https://qiita.com/apstndb/items/9d13230c666db80e74d0#nodeport-service
- https://kubernetes.io/docs/concepts/cluster-administration/networking/