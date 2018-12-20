# Service

## 概要

各ポッドには専用のIPアドレスが割り振られているが、それは永続的なものではない。例えばポッドが更新された時、またはデプロイメントによってレプリカセットが切り替えられた時など、ポッドのIPアドレスが変更されるタイミングはいくつか存在する。そのため、アプリを公開する時にポッドのIPアドレスをそのまま使うという選択は現実的ではない。

ここで **Service** を利用できる。サービスは不変のIPアドレスを持ち、負荷分散を行いながらリクエストをポッドに転送するロードバランサーのような役割を持っている。このおかげで複数存在するポッドを意識すること無くポッドにアクセスすることができるようになっている。

- [Service - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/service/)

> ポッドへの橋渡し

## 仕組みと役割

サービスの機能は自身に届いたリクエストをポッドに転送することだ。不変なIPアドレスを持つサービスはリクエストを受けた時にセットベースのセレクタを使って転送先のポッドを識別し、負荷分散を行ってポッドにリクエストをポッドに転送する。

サービスは負荷分散を行うロードバランサと言っても良い。ここでポッドのIPアドレスは使用されない。



Kubernetes は、各ノードで動作する `kube-proxy` を使用して、ポッドとサービス間の接続を管理している。

`kube-proxy` はノードに対し、 `iptables` ルールを追加および削除することで、このポートの再マッピングを管理している。ServiceのIPレイヤは、kube-proxyが動的に生成したiptablesのルールを使ってLinuxカーネルが処理を行う。


しかし実際のところ、サービスがポッドにアクセスしているわけではない。サービスは自分に対応した `エンドポイントレコード` を使ってポッドへアクセスしている。

### Endpoints

セレクタを使用したサービスが適用されると同時に `エンドポイントコントローラー` を使って作成される **エンドポイントレコード** は、ポッドに直接アクセスするためのIPアドレスを保持する役割を持っている。

単純に、対応したサービスが参照するためのポッドのIPアドレスを保持するだけの役割を持ち、それ以外の役割は存在しない。（故にレコードと呼ばれる。）レプリカセットがスケールするなどでポッドが増減した際も、エンドポイントは自動で更新される。

エンドポイントの確認は `kubectl get endpoints` で可能。

次の出力結果は、

```
$ kubectl get endpoints
NAME     ENDPOINTS                                   AGE
my-svc   172.17.0.4:80,172.17.0.7:80,172.17.0.8:80   8m
```


## ServiceTypes

サービスには4つの種類があり、テンプレートでどのサービスを作成するかを設定する。（デフォルトは `ClusterIP` ）

### ClusterIP

クラスター内部に公開するためのサービスタイプ。

サービスに割り当てられたIPのことを `ClusterIP` と呼び、ClusterIPが設定されたサービスには同一のクラスター内部からのみアクセスできる。クラスター内で頻繁に通信を行うマイクロサービスのような仕組みに活用できる。

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

サービスの確認には `kubectl get svc` を使う。適用後、テンプレートでは設定していないが `CLUSTER-IP` が割り振られていることが分かる。



また、セレクタ―

```
$ kubectl apply -f svc.md
service/my-svc created

$ kubectl get svc,endpoints
NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/my-svc       ClusterIP   10.96.50.75   <none>        80/TCP    5m
```





次の結果は、設定したセレクタで識別できるポッドが3つ存在する時の例。（レプリカセットによってポッドが3つ作成されている状況）k



```
$ kubectl get endpoints
NAME         ENDPOINTS                                   AGE
my-svc       172.17.0.4:80,172.17.0.7:80,172.17.0.8:80   5m
```

逆に、次の結果は設定したセレクタで識別できるポッドが存在しない時。

エンドポイント `my-svc` にリクエストを投げても、転送する先のポッドが存在しないことが分かる。

```
$ kubectl get endpoints
NAME         ENDPOINTS             AGE
my-svc       <none>                5m
```



次のコマンドを入力すると、Kubernetesには指定ポッド（ラベル・セレクタ）への `Endpoints` と、そのエンドポイントにリクエストを転送するiptablesのルールが作成される。

```
$ kubectl expose deployment hostnames --port=80
```

ポッドが2つ以上存在する場合、ClusterIPはポッドの負荷分散を行う。これはiptablesのstatistic moduleのrandomモードで行われているらしい。（2つの負荷分散の場合： `-m statistic --mode random --probability 0.50000000000` ）

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

### ExternalName

> その値を持つレコードを返すことによって、サービスをexternalNameフィールドの内容（例えばfoo.bar.example.com）にマッピング CNAMEします。どのような種類のプロキシも設定されていません。これにはバージョン1.7以上が必要ですkube-dns。

## 参考

- https://cloud.google.com/kubernetes-engine/docs/concepts/network-overview
- https://qiita.com/apstndb/items/9d13230c666db80e74d0#nodeport-service
- https://kubernetes.io/docs/concepts/cluster-administration/networking/