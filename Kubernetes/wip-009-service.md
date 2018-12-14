# Service

## 概要

各ポッドには専用のIPアドレスが割り振られているが、それは永続的なものではない。例えばポッドが更新された時、リスタートされた時など、IPアドレスが変更されるタイミングはいくつか存在する。

そのため、ポッドのIPアドレスをそのまま利用して何かを行う、ということは現実的ではない。そこでリクエストをポッドに伝送する、ポッドの窓口とも呼べるサービスが使用される。サービスを使用することで、複数存在するポッドを意識すること無くポッドにアクセスすることができるようになる。

## 機能

サービスはリクエストを受けるとラベル・セレクタでポッドを判定し、指定のポートにリクエストを伝送する。

### ServiceTypes

サービスには4つのタイプがあり、作成時にそれを決定する。デフォルトは **ClusterIP** で、クラスター内にサービスを公開する設定になっている。

#### ClusterIP

クラスター内にサービスを公開する。このタイプのサービスは、同一クラスター内からのリクエストしか受け付けない。

#### NodePort

> 各ノードのIP上のサービスを静的ポート（the NodePort）に公開します。ClusterIP先のサービス、NodePortサービスはルート、自動的に作成されます。NodePort要求することで、クラスタの外からサービスに連絡することができます<NodeIP>:<NodePort>。

？

サービスを適用するとKubenetes Masterは3000-32767からポートを割り当て、全ノードで同じポートが設定・開放される。リクエストを受けたノードはKube-proxy・iptablesとかから、さらにサービスに指定されたセレクタのポッドにリクエストを割り振る。（この割り振りはランダムらしい）

NodePortサービスを作成するとClusterIPサービスも自動的に作成されるため、クラスタ外部からもアクセスできるようになる。

#### LoadBalancer

クラウドプロバイダのロードバランサを使用して、サービスを外部に公開する。設定が正しければ、内部の設定だけでなく、外部のロードバランサも自動的に設定される。（Cloud Providerの設定が必要？）

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

仕組みとしてはKubernetesが内部でタイプ `NodePort` のサービスを作成し、ロードバランサがそこにアクセスしにいく形で実現されている。自動的な内部の設定ではNodePortの作成などを行う。

#### ExternalName

> その値を持つレコードを返すことによって、サービスをexternalNameフィールドの内容（例えばfoo.bar.example.com）にマッピング CNAMEします。どのような種類のプロキシも設定されていません。これにはバージョン1.7以上が必要ですkube-dns。


## Kubernetes YAML
