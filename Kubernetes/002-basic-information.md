# Basic information

Kubernetesを利用するにあたって必要となる基本的な概念を軽く解説する。情報量が多いので、初めてKubernetesにチャレンジする人は流し読みしてもらっても構わない。

## Kubernetesアーキテクチャ

### Kubernetesクラスター

1つ以上のマスターノード、1つ以上のワーカーノードの集まりをKubernetesクラスターと呼ぶ。

MinikubeでローカルにKubernetes環境を構築した場合も、マスターノードとワーカーノードは同一ノードだがギリギリ条件は満たされるのでKubernetesクラスターと呼べる。

### Kubernetesノード

コンテナのオーケストレーションを可能とするKubernetesだが、コンテナの実行環境の実体はKubernetesによって束ねられた物理サーバーやVMの集まりである。そしてKubernetesではこの物理サーバー、VMひとつひとつを **ノード** と呼ぶ。

Kubernetesを構成するノードには2種類あり、それぞれ役割が違う `マスターノード` と `ワーカーノード` が存在する。

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)

### マスターノード（Kubernetes Master）

Kubernetesクラスターの望ましい状態を維持するためのノードで、1つのクラスターには必ず1つ以上のマスターノードが存在する。いわばKubernetesクラスターの司令塔である。

マスターノードは1つでも問題ないが、複数並べることで可用性を確保できる。GKEやEKSはこのあたりを勝手にやって落ちないマスターノードを提供してくれるのでとても楽。（個人で使うには高い）

- [Creating Highly Available Clusters with kubeadm](https://kubernetes.io/docs/setup/independent/high-availability/)

クラスター内では3つのプロセス（ `kube-apiserver` 、 `kube-controller-manager` 、 `kube-scheduler` ）を実行している単一のノードをマスターノードと呼び、このマスターノードが全てのユーザーとの対話（kubectlなど）に応じている。（kubectlなどを用いたKubernetesとの対話は、実質マスターノードとの対話である）

#### kube-apiserver

a

#### kube-controller-manager

a

#### kube-scheduler

a

### ワーカーノード（Kubernetes Nodes）

Dockerによってアプリケーションコンテナが実行されるノードで、実態はマスターノードが管理している物理サーバー、またはVMなど。アプリケーションのコンテナ以外にもマスターノードに情報を送り続けるためのサービスなど、管理されるために必要なものも同時に実行されている。

#### docker daemon process

コンテナを動かすためのDockerプロセス。

#### kubelet

各ワーカーノードで動作する、ポッド、コンテナ、イメージ、ボリュームなどの主要なリソースを管理する役割を担うプロセス。さまざまな仕組みで（主に `kube-apiserver` を介して）PodSpecを取得し、それらのPodSpecに記述されているコンテナが正常であるかなどを確認する機能も担う。

#### kube-proxy

各ワーカーノードで動作する、ネットワークプロキシとロードバランサのプロセス。 `Service` が持つ仮想的なCluster IPを転送するなどの役割を持ち、内部的にiptablesを保有してIPアドレスの管理を行う。

#### flanneld

etcd経由で他のホストと強調し、自ホストに割り当てるネットワークを決定、設定する。


## ペーシックオブジェクト（Kubernetesオブジェクト）

Kubernetesに現れるオブジェクトの解説。Kubernetesはシステムやコンテナの状態を表現するためにいくつかの抽象概念を持っており、これは全て `オブジェクト` として呼ばれて表現される。

- Pod
- Service
- Volume
- Namespace

### ポッド（Pod）

1つ以上のコンテナをまとめたユニットの単位。

ポッド単体で作成することもできるが、レプリカセットでポッドを複数生成して対障害性・スケーリング性を実現する、などの使われ方が多い。ポッドはジョブやデーモンセットから作成されることもある。

多くのKubernetesの機能はポッドを生成することで仕事をするので、Kubernetesは実質ポッドの集合と言っても良い。（たぶん）

- [Pod Overview - Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)

### サービス（Service）

複数のポッドに対して単一のIPアドレスを割り当てる機能。

サービスのIPアドレスへのアクセスは背後のポッドに割り振られるため、実質的な挙動はよくあるロードバランサと同じ。

- [Services - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/service/)

### ボリューム（Volume）

- [Volumes - Kubernetes](https://kubernetes.io/docs/concepts/storage/volumes/)

### ネームスペース（Namespace）

クラスターのリソースを分割する機能。

1つのクラスター内で複数のチームやプロジェクトが動いている環境での使用を想定されている。テスト環境、商用環境などでネームスペースを分けていた事例もあった。

（数人から数十人のユーザーが活動するレベルのクラスターであれば、ネームスペースを使用する必要はない。この場合はラベルを使用してリソースを区別できる）

- [Namespaces - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)


## コントローラーオブジェクト（Kubernetesオブジェクト）

前述のベーシックオブジェクトよりも上位の抽象概念として存在するオブジェクト。コントローラーオブジェクトはベーシックオブジェクトによって構成されており、PodやServiceなどを管理するなど特殊な役割を持つものが多い。便利な機能はだいたいこれで実装されている。

- ReplicationController
- ReplicaSet
- Deployment
- StatefulSet
- DaemonSet
- Job
- CronJob

### レプリケーションコントローラー（ReplicationController）

ポッドを管理する機能。

指定されたレプリカ数（ポッドの必要個数）を維持するため、ポッドを監視し、必要に応じて生成・削除を行う。ポッドが指定数より少ない場合は新しいポッドを生成し、指定数より多い場合は削除する。

このレプリケーションコントローラーを利用することで容易に対障害性・スケーリング可能を実現できる。また、等価ベースのセレクターをサポートしている。

現在はレプリカセットの使用が推奨されており、レプリケーションコントローラーはあまり使うべきではないかもしれない。

- [ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)

### レプリカセット（ReplicaSet）

次世代のレプリケーションコントローラーだが、機能はほとんど変わらない。レプリカセットも同じように、ステートレス（値を保持しない）なポッドを生成して管理する。

レプリケーションコントローラーと同じように使用できるが、レプリカセットはデプロイメントのバックエンドとして使用することを想定されているため、利用者が直接作成することはほとんど無い。

デプロイメントのバックエンドで使用されることもあり、現在はレプリケーションコントローラーよりもレプリカセットのほうが主流である。レプリケーションコントローラーに比べてセレクターのサポートも充実している。（セットベースのセレクターをサポート）

- [ReplicaSet - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

### デプロイメント（Deployment）

レプリカセットを生成・管理する仕組み。

ローリングアップデートを代表とするデプロイ・アップデート方法をサポートしており、アプリケーションサービスを公開する時に利用する。アップデート後も前バージョンのレプリカセットを保持しており、いつでもバージョンを巻き戻すことができるなどの特徴を持つ。

- [Deployments - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

### ステートフルセット（StatefulSet）

レプリカセットやデプロイメントがステートレスなのに対し、ステートフルセットはステートフル（値を保持する）なポッドを生成・管理するための機能。

ステートフルセットでは、データストアなどに使えるポッドをYAMLテンプレートなどから直接生成する。

- [StatefulSets - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

### デーモンセット（DaemonSet）

特定のノード、もしくは全てのノードでポッドを動作させる機能。すべてのノードで1つのポッドを動かし、ログ収集を行う際などに利用できる。

本来ならKubernetesスケジューラーによってノードを選択・ポッドが生成されるが、デーモンセットのポッドはデーモンセットコントローラーによって生成される。（起動するノードが予め決まっているため。）

そのため、デーモンセットのポッドはKubernetesスケジューラーが起動していなくても作成が可能。

- [DaemonSet - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

### ジョブ（Job）

1つ以上のポッドを作成し、指定の処理を実行する機能。

指定された数のポッドが処理を完遂することを保証するのでバッチ処理などに利用できる。レプリカセットなどと違って、このコンテナは処理が完了した時に終了するよう設定しておく。（例えば、単純な計算を出力するだけのスクリプトなど）

- [Jobs - Run to Completion - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)

### クーロンジョブ（CronJob）

コマンドのcrontabに似た機能で、設定したスケジュールに従って定期的にJobを実行する。

- [CronJob - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)



## その他機能

### ロール（RBAC）

RBACは `Role-based access control (Authorization)` の略で、これ自体はよく利用されている概念。役割ベースのアクセス制御を用いて、個々のユーザーの役割に基づいたリソースへのアクセス権限を付与する機能をKubernetesも実装している。

- [KubernetesのRBACについて](https://qiita.com/sheepland/items/67a5bb9b19d8686f389d)
- [Using RBAC Authorization - Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

### イングレス（Ingress）

クラスターのサービスHTTP(S)への外部アクセスを管理する機能。

サービスにはロードバランサーを設定するなどして外部インターネットからアクセスすることができるが、イングレスを使うことでより柔軟な設定が可能になる。簡単に言えばイングレスは外部アクセスをクラスターのサービスに接続させるための仕組み、ルールの集合。

- [Ingress - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress/)

### コンフィグマップ（ConfigMap）

設定情報を保持する機能。

コンフィグマップを使うことで、ステートレスなポッドが環境変数、コンテナの引数、Volume（シンボリックリンク）などの情報を利用することができる。

- [Configure a Pod to Use a ConfigMap - Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

### シークレット（Secret）

パスワード、トークンやキーなどを保存しておく機能。

ポッドから参照できる環境変数として公開することができるし、システムから利用するためのキーとして保存しておくことが可能。

- [Secrets - Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)