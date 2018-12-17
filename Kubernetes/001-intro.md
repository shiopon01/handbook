# Kubernetes intro

KubernetesはDockerコンテナのオーケストレーションツールの1つだ。コンテナオーケストレーションツールとは、多数のDockerコンテナの複雑な設定や管理を簡略化し、自動化してくれるツールのことを指す。基本的には、絶対にサービスを落とさないためのツールとして採用される。

# 目次

ファイル先頭の番号順に読み進めていくことを推奨するが、必ずその順番で読むことは強制しない。いちおう、理解しやすいであろう順番で記述しているつもりである。

- [intro](001-intro.md)
- [basic information](002-basic-information.md)
- [Minikube setup](003-minikube-setup.md)
- [config template](004-config-template.md)

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

## その他

- RBAC
- Ingress
- ConfigMap
- Secret

