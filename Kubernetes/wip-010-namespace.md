# Namespace

## 概要

**Namespace** は単一のクラスター内で複数の仮想クラスターを動作させることができる仕組み。これら仮想クラスターは名前空間と呼ばれる。

- [Namespace - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

最初の状態では、ネームスペースは3つ存在している。

```
$ kubectl get namespace
NAME          STATUS    AGE
default       Active    5d
kube-public   Active    5d
kube-system   Active    5d
```

#### default

デフォルトのネームスペース。

#### kube-public

全てのユーザーによって読み取られる、クラスター全体で共通のリソースを設置しておく場所。

#### kube-system

Kubernetesが作成するコンポーネントが入る。Kubenetes Materの `kube-apiserver` `kube-controller-manager` `kube-scheduler` 、Nodesの `kube-proxy` などがこのネームスペース内でポッドとして動作している。

## 機能

`-n` オプションで名前空間を指定することで、その名前空間の情報を取得できる。たとえば、何も設定していない状態で使用される名前空間は `default` だ。

```
$ kubectl -n default get pods
NAME                   READY     STATUS    RESTARTS   AGE
pod-double-container   2/2       Running   26         4d
```

同じように `kube-public` と `kube-system` の情報も取得してみる。（Minikube上で実行されているので、 `kube-system` にはMinikube用のコンポーネントが入っていた）

```
$ kubectl -n kube-public get pods
No resources found.
```

```
$ kubectl -n kube-system get pods
NAME                                    READY     STATUS    RESTARTS   AGE
etcd-minikube                           1/1       Running   0          6m
heapster-wpbbh                          1/1       Running   1          5d
influxdb-grafana-krrwn                  2/2       Running   2          5d
kube-addon-manager-minikube             1/1       Running   1          5d
kube-apiserver-minikube                 1/1       Running   0          6m
kube-controller-manager-minikube        1/1       Running   0          6m
kube-dns-86f4d74b45-8dq2v               3/3       Running   4          5d
kube-proxy-xt4xh                        1/1       Running   0          5m
kube-scheduler-minikube                 1/1       Running   0          6m
kubernetes-dashboard-5498ccf677-jv5ll   1/1       Running   3          5d
storage-provisioner                     1/1       Running   3          5d
```

### 名前空間の作成

```
$ kubectl create namespace test-namespace
namespace/test-namespace created
```

## その他

- デフォルトの名前空間を切り替える。

```
$ kubectl config set-context $(kubectl config current-context) --namespace=<my-namespace>
$ kubectl config view | grep namespace:
```

## 参考
