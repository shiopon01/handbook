# Service

## 概要

各ポッドには専用のIPアドレスが割り振られているが、それは永続的なものではない。例えばポッドが更新された時、またはリスタートされた時など、ポッドのIPアドレスが変更されるタイミングはいくつか存在する。

そのため、ポッドのIPアドレスをそのまま使って何かを行うというのは現実的でないことが分かる。そこで、リクエストをポッドに転送する窓口を用意しよう。これをサービスと呼ぶ。サービスを使用することで、複数存在するポッドを意識すること無くポッドにアクセスすることができるようになる。

## 機能

> サービスは不変のIPアドレスとポートを持ち、サービス作成時にラベルセレクタで定義したすべてのラベルと一致するラベルを持つ一連のポッド間で負荷分散を行います。

サービスは負荷分散を行うロードバランサと言っても良い。ここでポッドのIPアドレスは使用されない。

Kubernetes は、各ノードで動作する `kube-proxy` を使用して、ポッドとサービス間の接続を管理している。

`kube-proxy` はノードに対し、 `iptables` ルールを追加および削除することで、このポートの再マッピングを管理している。ServiceのIPレイヤは、kube-proxyが動的に生成したiptablesのルールを使ってLinuxカーネルが処理を行う。

### ServiceTypes

サービスには4つのタイプがあり、作成時にそれを決定する。デフォルトは **ClusterIP** で、クラスター内にサービスを公開する設定になっている。

#### ClusterIP

サービスをクラスター内部だけに公開するためのサービスタイプ。

サービスに割り当てられたIPのことをClusterIPと呼ぶ。デフォルトのタイプであり、これがクラスター内にサービスを公開するサービスタイプだ。同一クラスター内からのリクエストを受け付けることができるようになる。

次のコマンドを入力すると、Kubernetesには指定ポッド（ラベル・セレクタ）への `Endpoints` と、そのエンドポイントにリクエストを転送するiptablesのルールが作成される。

```
$ kubectl expose deployment hostnames --port=80 --target-port=9376
```

ポッドが2つ以上存在する場合、ClusterIPはポッドの負荷分散を行う。これはiptablesのstatistic moduleのrandomモードで行われているらしい。（2つの負荷分散の場合： `-m statistic --mode random --probability 0.50000000000` ）

#### NodePort

サービスをクラスター外部に公開するためのサービスタイプ。

NodePortはClusterIPを拡張する形でサービスを外部に公開する。iptablesにはClusterIPの設定に加え、外部からアクセスできるNodePortに来たリクエストをローカルアドレスに転送するルールが追加される。

公式ドキュメントのこの記述は、NodePortをよく知らない時は正直分からないとおもう。

> 各ノードのIP上のサービスを静的ポート（the NodePort）に公開します。ClusterIP先のサービス、NodePortサービスはルート、自動的に作成されます。NodePort要求することで、クラスタの外からサービスに連絡することができます<NodeIP>:<NodePort>。

NodePortが適用されると、Kubenetes Masterはサービス用に3000-32767からポートを割り当て、全ノードで同じポートを開放させる。リクエストを受けたノードはkube-proxyを経由して指定されたセレクタのポッドにリクエストを割り振る。

HTTPロードバランサである `ingress` はここで公開されるポートを使って実装されている。

#### LoadBalancer

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

#### ExternalName

> その値を持つレコードを返すことによって、サービスをexternalNameフィールドの内容（例えばfoo.bar.example.com）にマッピング CNAMEします。どのような種類のプロキシも設定されていません。これにはバージョン1.7以上が必要ですkube-dns。

## Kubernetes YAML



## 参考

- https://cloud.google.com/kubernetes-engine/docs/concepts/network-overview
- https://qiita.com/apstndb/items/9d13230c666db80e74d0#nodeport-service
- https://kubernetes.io/docs/concepts/cluster-administration/networking/