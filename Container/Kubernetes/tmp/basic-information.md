# Kubernetes Basic information

# 用語解説

## Control Plane

Kubernetesの基本的な構成の解説。

### マスターノード（Kubernetes Master）

Kubernetesクラスターの望ましい状態を維持するためのノードで、1つのクラスターには必ず1つのマスターノードが存在する。

クラスタ内では `kube-apiserver` `kube-controller-manager` `kube-scheduler` プロセスを実行している単一のノードがマスターノードとされ、このマスターノードがユーザーとの対話（kubectlなど）に応じている。（Kubernetesとの対話は、マスターノードとの対話である。）

- **kube-apiserver** :
- **kube-controller-manager** :
- **kube-scheduler** :

### ワーカーノード（Kubernetes Nodes）

Dockerによってアプリケーションコンテナが実行されるノードで、実態はマスターノードが管理している物理サーバー、またはVMなど。アプリケーションのコンテナ以外にもマスターノードに情報を送り続けるためのサービスなど、管理されるために必要なものも同時に実行されている。

- **docker daemon process** : コンテナを動かすためのDockerプロセス。
- **kubelet** : 各ワーカーノードで動作する、ポッド、コンテナ、イメージ、ボリュームなどの主要なリソースを管理する役割を担うプロセス。さまざまな仕組みで（主に `kube-apiserver` を介して）PodSpecを取得し、それらのPodSpecに記述されているコンテナが正常であるかなどを確認する機能も担う。
- **kube-proxy** : 各ワーカーノードで動作する、ネットワークプロキシとロードバランサのプロセス。 `Service` が持つ仮想的なCluster IPを転送するなどの役割を持ち、内部的にiptablesを保有してIPアドレスの管理を行う。
- **flanneld** : etcd経由で他のホストと強調し、自ホストに割り当てるネットワークを決定、設定する。


## Basic Objects

Kubernetesに現れる基本的なオブジェクトの解説。Kubernetesはシステムやコンテナの状態を表すためにいくつかの抽象概念を持っており、これらの抽象概念は全て `オブジェクト` として表現されている。

- Pod
- Service
- Volume
- Namespace

### ポッド（Pod）

1つ以上のコンテナをまとめたユニットの単位。レプリケーションコントローラーかレプリカセットの管理下で複数のポッドを水平に並べることで対障害性・スケーリングが実現される。ジョブやデーモンセットによって実行されることもある。

- [Podの解説](/docs/Container/Kubernetes/w.Pod.html)

### サービス（Service）

複数のポッドに対して単一IPアドレスを割り当てる機能。サービスのIPアドレスに向けられたアクセスは背後のポッドへ割り振られるため、実質ロードバランサのように振る舞う。

- [Serviceの解説](/docs/Container/Kubernetes/w.Service.html)

### ボリューム（Volume）

**調査中**

- [Volumeの解説](/docs/Container/Kubernetes/w.Volume.html)

### ネームスペース（Namespace）

クラスタのリソースを分割する機能。1つのクラスタで複数のチームやプロジェクトが動いている環境での使用を想定されている。数人から数十人のユーザーが活動するレベルのクラスタであればネームスペースを考える必要はない。（ネームスペースを使用するほどでない環境では、ラベルを使用してリソースを区別する。）

- [Namespaceの解説](/docs/Container/Kubernetes/w.Namespace.html)


## Controller Objects

基本的なオブジェクトを操作するための高レベルのオブジェクト、コントローラーの解説。コントローラーは基本オブジェクトによって構成されており、このオブジェクトによってさらに便利な機能が提供される。

- ReplicationController
- ReplicaSet
- Deployment
- StatefulSet
- DaemonSet
- Job
- CronJob

### レプリケーションコントローラー（ReplicationController）

ポッドを管理する機能。レプリカ数（ポッドの必要個数）を維持するため、ポッドを監視し、必要に応じて作成や削除を行う。このレプリケーションコントローラーを利用することで容易に対障害性・スケーリング可能を実現できる。（等価ベースのセレクターをサポート）

- [ReplicationControllerの解説](/docs/Container/Kubernetes/w.ReplicationController.html)

### レプリカセット（ReplicaSet）

次世代のレプリケーションコントローラー。機能はほとんど変わらないため同じように使用できるが、こちらはデプロイメントのバックエンドとして使用することを想定されている。レプリケーションコントローラーに比べセレクターのサポートも充実した。（セットベースのセレクターをサポート）

デプロイメントのバックエンドで使用されることもあり、現在はレプリケーションコントローラーよりレプリカセットのほうが主流。

- [ReplicaSetの解説](/docs/Container/Kubernetes/w.ReplicaSet.html)

### デプロイメント（Deployment）

`レプリカセット` を生成して管理する仕組み。ローリングアップデート等をサポートしており、アプリケーションサービスの公開等でよく利用される。レプリカセットやデプロイメントはステートレス（値を保持する）なポッドを作成・管理するための機能。

- [Deploymentの解説](/docs/Container/Kubernetes/w.Deployment.html)

### ステートフルセット（StatefulSet）

レプリカセットやデプロイメントがステートレスなのに対し、 `ステートフルセット` はステートフル（値を保持する）なポッドを作成・管理するための機能。ステートフルセットはテンプレートから直接ポッドを生成し、これは例えばデータベースなどに使用できる。

- [StatefulSetの解説](/docs/Container/Kubernetes/w.StatefulSet.html)

### デーモンセット（DaemonSet）

特定のノード、もしくは全てのノードでポッドを動作させる機能。すべてのノードでログ収集を行う際などに利用できる。

通常、ポッドが動作するノードはKubernetesスケジューラーによって選択・作成されるが、デーモンセットから作成されたポッドはデーモンセットコントローラーによって作成される。（起動するノードが決まっているため。）そのため、デーモンセットのポッドはKubernetesスケジューラーが起動していなくても作成が可能。

- [DaemonSetの解説](/docs/Container/Kubernetes/w.DaemonSet.html)

### ジョブ（Job）

1つ以上のポッドを作成し、指定された数のポッドが処理を完遂することを保証する機能。バッチ処理などに利用でき、レプリカセットなどと違ってこのコンテナには処理を完了、終了できる命令を設定しておく。（単純な計算スクリプトなど）

- [Jobの解説](/docs/Container/Kubernetes/w.Job.html)

### クーロンジョブ（CronJob）

crontabに似た機能で、設定したスケジュールに従って定期的にJobを実行する。

**調査中**

- [CronJobの解説](/docs/Container/Kubernetes/w.CronJob.html)

## その他機能

### ロール（RBAC）

RBACは `Role-based access control (Authorization)` の略。役割ベースのアクセス制御を用いて、個々のユーザーの役割に基づいたリソースへのアクセス権限を付与する機能。

- [RBACの解説](/docs/Container/Kubernetes/w.RBAC.html)

### イングレス（Ingress）

クラスターのサービス（HTTP）への外部アクセスを管理する機能。

サービスとポッドにはクラスター内のネットワークからのみルーティング可能なIPが割り振られているが、外部からはアクセスできない。イングレスは外部アクセスをクラスターのサービスに接続させるための仕組み、ルールの集合。

- [Ingressの解説](/docs/Container/Kubernetes/w.Ingress.html)

### コンフィグマップ（ConfigMap）

設定情報を扱うための機能。ポッドに対して環境変数、コンテナの引数、Volume（シンボリックリンク）で情報を渡すことが出来る。

**調査中**

- [ConfigMapの解説](/docs/Container/Kubernetes/w.ConfigMap.html)

### シークレット（Secret）

**調査中**

- [Secretの解説](/docs/Container/Kubernetes/w.Secret.html)
