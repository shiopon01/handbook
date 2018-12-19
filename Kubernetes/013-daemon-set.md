# DaemonSet

## 概要

DaemonSetはすべて（または一部）のノードにポッドを作成できる機能。

新しいノードがクラスターに追加された際も、そのノード上にポッドが作成される。ノードが削除された際は、ポッドも同時に削除される。

- [DaemonSet - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

## ユースケース

- すべてのノードでログを収集収集する（ `logstash` など）
- すべてのノードでシステム情報を収集する（ `collectd` など）

とにかく、すべて（または一部）のノードで指定のプログラムを動作させたい時に利用できる。

## 機能

### デーモンポッドのスケジュール

Kubernetesのバージョン1.12より前は、デーモンポッドを作成するために `DaemonSet Controller` を使用していた。

通常ポッドが配置されるノードはKubernetesスケジューラーによって選択されるが、デーモンセットはデーモンセットコントローラーによって選択されていた。これはスケジューラが起動していなくてもデーモンポッドが作成できる利点ではあったが、一貫性のないポッドの動作としてユーザーにとって紛らわしいものでもあった。

そのため、バージョン1.12からは、標準でKubernetesスケジューラーを用いてデーモンポッドを作成するよう変更されている。

## Kubernetes YAML

```yaml
apiVersion: v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

## 参考
