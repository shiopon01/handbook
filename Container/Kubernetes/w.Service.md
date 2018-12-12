# Service

# WIP

- ロングランニングアプリケーションを希望する状態に保つ
  - Task Definition
  - Taskの数
- ALBとの動的ポートマッピング（Optional）
  - コンテナはランダムなホストのポートを使って登録
- Application Auto Scaling（Optional）
  - AlarmとPolicyを使って、希望するTask数を自動的に変更

各Clusterに対し、できるだけ効率の良いコンテナ配置を行う

サービスはタスクとクラスターを紐付ける

内部的には、各ワーカーノード上で動く `kube-proxy` というプロセスがiptablesにエントリを追加することで、サービスのIPアドレスを作っている

kubectl: Kubernetesの管理用コマンドラインツール
etcd: 分散KVSで、Kubernetesのマスタープロセスが扱うデータストア（SkyDNSもflanneIdもそれぞれetcdをデータストアとして利用）
etcdctl: etcdの管理用コマンドラインツール
