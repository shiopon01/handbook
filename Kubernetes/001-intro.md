# Kubernetes intro

Kubernetesは自動デプロイ、スケーリング等を効率的に行うためのオープンソースのコンテナオーケストレーションツール。高い可用性や拡張性を提供してくれる。

簡単に言うと、絶対にサービスを落とさないためのツール。

（読み方はたぶん `クーべネティス` ）

# 目次

ファイル先頭の番号順に読み進めていくことを推奨するが、必ずその順番で読むことは強制しない。いちおう、理解しやすいであろう順番で記述しているつもりである。

- [Intro](001-intro.md)
- [Basic information](002-basic-information.md)
- [Minikube setup](003-minikube-setup.md)
- [Config template](004-config-template.md)

## ペーシックオブジェクト

- [Pod](005-pod.md)
- [Service](009-service.md)
- [Namespace](010-namespace.md)
- [Volume](011-volume.md)

## コントローラーオブジェクト

- [ReplicationController](006-replication-controller.md)
- [ReplicaSet](007-replica-set.md)
- [Deployment](008-deployment.md)
- [StatefulSet](012-stateful-set.md)
- [DaemonSet](013-daemon-set.md)
- [Job](014-job.md)
- [CronJob](015-cron-job.md)

## アクセス制御

- RBAC

## その他

- Ingress
- ConfigMap
- Secret
