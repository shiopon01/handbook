# Basic information

Kubernetesを利用するにあたって必要となる基本的な概念・用語を大まかに紹介する。

覚える用語は多くあるため、初めての人は流し読みしても問題ない。ファイルを先頭の番号順に読み進めることでそれらの用語をなんとなくでも理解していけるはずだ。

## Kubernetes Cluster

1つ以上のマスターノード、1つ以上のワーカーノードの集まりをKubernetesクラスターと呼ぶ。クラスターはKubernetesが動作するために必要な構成であり、Kubernetesの実体とも言える。

MinikubeでローカルにKubernetes環境を構築した場合も、マスターノードとワーカーノードは同一ノードだが条件は満たされるのでKubernetesクラスターと呼べる。

## Kubernetes Node

コンテナのオーケストレーションを可能とするKubernetesだが、コンテナの実行環境の実体はKubernetesによって束ねられた物理サーバーやVMの集まりである。そしてKubernetesではこの物理サーバー、VMひとつひとつを **ノード** と呼ぶ。

Kubernetesを構成するノードには2種類あり、それぞれ役割が違う `マスターノード` と `ワーカーノード` が存在する。

- [Kubernetes Components - Kubernetes](https://kubernetes.io/docs/concepts/overview/components/)

## Kubernetes Master（マスターノード）

2種類あるKubernetes Nodeのうちのひとつ、マスターノードと呼ばれるノード。

Kubernetesクラスターの望ましい状態を維持するためのノードで、Kubernetesクラスターの司令塔の役割を持つ。司令塔なので、1つのクラスターには必ず1つ以上のマスターノードが存在している。

マスターノードと呼ばれるノードには3つのプロセス（ `kube-apiserver` 、 `kube-controller-manager` 、 `kube-scheduler` ）が実行されており、ユーザーとの対話（kubectlなど）などは全てこのノードが応じている。kubectlなどを用いたKubernetesとの対話は、実質マスターノードとの対話である。

余談だが、マスターノードがダウンすることは実質クラスターの死なので、マスターノードも複数並べることで可用性を確保することができる。GKEやEKSは絶対にダウンしないマスターノードを提供してくれるのでとても楽だが、個人で使うには高い。

- [Creating Highly Available Clusters with kubeadm](https://kubernetes.io/docs/setup/independent/high-availability/)

### kube-apiserver

Kubernetesのリソースを管理するためのAPIを提供するプロセス。

全てのプロセスはこのAPIサーバーへのアクセスを通して操作を行う。そのため、認証・認可の仕組みも `kube-apiserver` に備えられている。

具体的には、kubectlはkube-apiserverと通信してKubernetesのリソースを操作するなど。このAPIはkubectlが呼び出す以外にも、ポッドがルーティングの情報を取得するために呼び出すなどワーカーノードやその中で立ち上がったオブジェクトから呼び出される場合も多い。

- [kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)

### kube-controller-manager

コントローラーオブジェクトを起動するマネージャー。コントローラーオブジェクトには `ReplicaSet` や `DaemonSet` などが含まれる。

- [kube-controller-manager - Kubernetes](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)

### kube-scheduler

新しいポッドをどのノードに作成するかの割り当てを行うスケジューラー。

`kube-apiserver` のAPIを使ってリソースの状態監視を行い、ポッドとノードを紐付ける役割を持つ。

- [kube-scheduler - Kubernetes](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)

## Kubernetes Nodes（ワーカーノード）

2種類あるKubernetes Nodeのうちのひとつ、ワーカーノードと呼ばれるノード。

マスターノードによってアプリケーションコンテナが実行されるノードで、クラスターが使用可能なリソースの拡張はノードの拡張（追加）でもある。クラスター内のワーカーノードはマスターノードによって完璧に管理されており、利用者はワーカーノードを意識することなくクラウドを扱うようにリソースを利用できる。

### docker daemon process

コンテナを動かすためのDockerプロセス。

### kubelet

ノードのメイン処理であるポッド、コンテナ、イメージ、ボリュームなどの主要なリソースを管理する役割を担うプロセス。

さまざまな仕組みで（主に `kube-apiserver` を介して）PodSpecを取得し、それらのPodSpecに記述されているコンテナが正常であるかなどを確認する機能も担う。

### kube-proxy

ネットワークプロキシとロードバランサのプロセス。

`Service` が持つ仮想的なCluster IPを転送するなどの役割を持ち、内部的にiptablesを保有してIPアドレスの管理を行う。

### flanneld

etcd経由で他のホストと強調し、自ホストに割り当てるネットワークを決定、設定する。


## ペーシックオブジェクト

Kubernetesに現れるオブジェクトの解説。Kubernetesはシステムやコンテナの状態を表現するためにいくつかの抽象概念を持っており、これは全て `オブジェクト` として呼ばれて表現される。

- Pod
- Service
- Volume
- Namespace

### [Pod](005-pod.md)

1つ以上のコンテナをまとめたユニットの単位。

ポッド単体で作成することもできるが、レプリカセットでポッドを複数生成して対障害性・スケーリング性を実現する、などの使われ方が多い。ポッドはジョブやデーモンセットから作成されることもある。

多くのKubernetesの機能はポッドを生成することで仕事をするので、Kubernetesは実質ポッドの集合と言っても良い。（たぶん）

### [Service](009-service.md)

複数のポッドに対して単一のIPアドレスを割り当てる機能。

サービスのIPアドレスへのアクセスは背後のポッドに割り振られるため、実質的な挙動はよくあるロードバランサと同じ。

### [Volumes](011-volume.md)


### [Namespaces](010-namespace.md)

クラスターのリソースを分割する機能。

1つのクラスター内で複数のチームやプロジェクトが動いている環境での使用を想定されている。テスト環境、商用環境などでネームスペースを分けていた事例もあった。

（数人から数十人のユーザーが活動するレベルのクラスターであれば、ネームスペースを使用する必要はない。この場合はラベルを使用してリソースを区別できる）


## コントローラーオブジェクト

前述のベーシックオブジェクトよりも上位の抽象概念として存在するオブジェクト。コントローラーオブジェクトはベーシックオブジェクトによって構成されており、PodやServiceなどを管理するなど特殊な役割を持つものが多い。便利な機能はだいたいこれで実装されている。

### [ReplicationController](006-replication-controller.md)

ポッドを管理する機能。

指定されたレプリカ数（ポッドの必要個数）を維持するため、ポッドを監視し、必要に応じて生成・削除を行う。ポッドが指定数より少ない場合は新しいポッドを生成し、指定数より多い場合は削除する。

このレプリケーションコントローラーを利用することで容易に対障害性・スケーリング可能を実現できる。また、等価ベースのセレクターをサポートしている。

現在はレプリカセットの使用が推奨されており、レプリケーションコントローラーはあまり使うべきではないかもしれない。


### [ReplicaSet](007-replica-set.md)

次世代のレプリケーションコントローラーだが、機能はほとんど変わらない。レプリカセットも同じように、ステートレス（値を保持しない）なポッドを生成して管理する。

レプリケーションコントローラーと同じように使用できるが、レプリカセットはデプロイメントのバックエンドとして使用することを想定されているため、利用者が直接作成することはほとんど無い。

デプロイメントのバックエンドで使用されることもあり、現在はレプリケーションコントローラーよりもレプリカセットのほうが主流である。レプリケーションコントローラーに比べてセレクターのサポートも充実している。（セットベースのセレクターをサポート）

### [Deployment](008-deployment.md)

レプリカセットを生成・管理する仕組み。

ローリングアップデートを代表とするデプロイ・アップデート方法をサポートしており、アプリケーションサービスを公開する時に利用する。アップデート後も前バージョンのレプリカセットを保持しており、いつでもバージョンを巻き戻すことができるなどの特徴を持つ。

### [StatefulSets](012-stateful-set.md)

レプリカセットやデプロイメントがステートレスなのに対し、ステートフルセットはステートフル（値を保持する）なポッドを生成・管理するための機能。

ステートフルセットでは、データストアなどに使えるポッドをYAMLテンプレートなどから直接生成する。

### [DaemonSet](013-daemon-set.md)

特定のノード、もしくは全てのノードでポッドを動作させる機能。すべてのノードで1つのポッドを動かし、ログ収集を行う際などに利用できる。

本来ならKubernetesスケジューラーによってノードを選択・ポッドが生成されるが、デーモンセットのポッドはデーモンセットコントローラーによって生成される。（起動するノードが予め決まっているため。）

そのため、デーモンセットのポッドはKubernetesスケジューラーが起動していなくても作成が可能。

### [Job](014-job.md)

1つ以上のポッドを作成し、指定の処理を実行する機能。

指定された数のポッドが処理を完遂することを保証するのでバッチ処理などに利用できる。レプリカセットなどと違って、このコンテナは処理が完了した時に終了するよう設定しておく。（例えば、単純な計算を出力するだけのスクリプトなど）

### [CronJob](015-cron-job.md)

コマンドのcrontabに似た機能で、設定したスケジュールに従って定期的にJobを実行する。

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
