# minikube

[公式のドキュメント](https://github.com/kubernetes/minikube/tree/master/docs)

## 概要

minikubeのチュートリアルなんてもう山ほど転がっていると思うが、ここではminikubeを初める人に向けてできるだけ分かりやすく説明するつもりだ。

インストールは公式ドキュメントもあるので、事前に行ってもらいたい。必要なものは `Minikube` と `kubectl` だ。

- [Install Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
- [Install and Set Up kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)



起動時にはminikube

https://kubernetes.io/docs/tutorials/hello-minikube/


## 起動と接続

minikubeをインストールしたならば、まずは起動してみよう。

minikubeはDocker公式のオーケストレーションツールであるDocker Machineを使ってVM上にKubernetes環境を作成するので、立ち上げるVM（ドライバ）を指定できる。VirtualBoxとVMWareのドライバはminikubeに組み込まれているし、だいたいみんな `VirtualBox` を利用しているので、ここでもVirtualBoxを利用することにする。

> ドライバーについての記載は公式の[この記事](https://github.com/kubernetes/minikube/blob/master/docs/drivers.md)

起動には `minikube start` を使う。 `--vm-driver` のオプションがドライバの指定だ。ついでに、インストールされている `minikube version` のバージョンも確認しよう。

```
$ minikube version
minikube version: v0.28.2

$ minikube --vm-driver=virtualbox start
Starting local Kubernetes v1.10.0 cluster...
Starting VM...
Getting VM IP address...
Moving files into cluster...
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.
Loading cached images from config file.
```

コマンドを入力したなら、こんな表示を出してminikubeが起動するはずだ。

minikubeが起動出来ているなら `minikube status` でIPアドレス等を確認できる。 `kubectl cluster-info` を使うことでkubectlから情報を見ることも可能。

```
$ minikube status
minikube: Running
cluster: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.100

$ kubectl cluster-info
Kubernetes master is running at https://192.168.99.100:8443
KubeDNS is running at https://192.168.99.100:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

もしどこかのタイミングで次のようなエラーが出たなら、  `minikube delete` で解決するかもしれない。deleteしたあとは `minikube start` し直す。

```
Error restarting cluster:  restarting kube-proxy: waiting for kube-proxy to be up for configmap update: timed out waiting for the condition

$ minikube delete
```

さて、クラスターを起動すると、クラスターを操作する `kubectl` コマンドの向き先は自動的にminikubeのほうを向くはずだ。これによって特に設定しなくてもkubectlからminikubeのクラスターを操作することが出来る。

だた、もし自動的に設定されなかった場合は自分でコンテキスト（操作するクラスター）を切り替えることで対応しよう。コンテキストの一覧は `kubectl config get-contexts` で確認できる。（minikubeのコンテキストは `minikube` という名前）

```
$ kubectl config get-contexts
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         minikube   minikube   minikube

$ kubectl config use-context minikube
Switched to context "minikube".
```

これらの情報は `~/.kube/config` に格納されているのだが、kubectlの範囲なのでここではこれ以上触れない。


## ダッシュボード

minikubeを起動すると、 [Kubernetes Dashboard](https://github.com/kubernetes/dashboard)にアクセスできるようになる。このダッシュボードではクラスタのCPUやメモリーの状態、起動しているコンテナなどリソースの情報を見ることが出来る。（クラスタといっても、VMが1つあるだけだが）

`minikube dashboard` でブラウザが立ち上がってダッシュボードのページが開くが、もし開かなかった場合など `--url` オプションでURL表示だけすることができるので手動でアクセスしよう。

```
$ minikube dashboard --url
http://192.168.99.100:30000
```

> いろいろ

さて、ここで気になるのが `クラスター > ノード > minikube` のタブで見ることができるポッド（コンテナの集合体）がどのようにして動かされているのかだ。ホストで `docker ps` を実行してもkubernetesが動いているのはVM上のため実際にコンテナを確認できない。

こんなに時は、minikubeが動いているVMのDockerデーモン（コマンド）を利用できるコマンド `eval $(minikube docker-env)` が役に立つ。既存のDockerコマンドの設定を書き換え、VM内のDockerの出力結果を得られる、というものだ。（元に戻すときは `-u` オプション）

```
$ eval $(minikube docker-env)
$ eval $(minikube docker-env -u)
```

`docker ps` の結果では複数のコンテナが起動していて、 `kube-apiserver` 、 `kube-controller-manager` 、 `kube-scheduler` が動いているのでこのノードがマスターノードであることが確認できる。（見難いけど、 `docker ps` のIMAGEかNAMESの名前）






### deployment



```
docker build -t hello-node:v1 .
```

ビルドしたdocker imageを使ってデプロイメントを作成する。デプロイメントは、イメージからポッドを作ることができる。

```
kubectl run hello-node --image=hello-node:v1 --port=8080 --image-pull-policy=Never
```

```
$ kubectl get deployments
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-node   1         1         1            1           15s

$ kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
hello-node-57c6b66f9c-fb2wf   1/1       Running   0          2m
```

クラスタイベントやkubectl設定

```
kubectl get events
kubectl config view
```

---

デフォルトでは、PodはKubernetesクラスタ内部のIPアドレスからしかアクセスできない。サービスを利用してKubernetes仮想ネットワークに外部からアクセスする。（ポッドの公開）

```
kubectl expose deployment hello-node --type=LoadBalancer
```

```
$ kubectl get services
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
hello-node   LoadBalancer   10.104.61.79   <pending>     8080:31380/TCP   20s
kubernetes   ClusterIP      10.96.0.1      <none>        443/TCP          1h
```

`--type=LoadBalancer` はサービスをクラスタ外に公開することを示している。ロードバランサをサポートするクラウドプロバイダでは、アクセスするために外部IPアドレスがプロビジョニングできる！（AWSのELBとか）

minikubeでは `minikube service` コマンドを介してサービスにアクセスできる。（ブラウザが立ち上がる）

```
minikube service hello-node
```

ポッドへのアクセスログを見ることも可能

```
kubectl logs hello-node-57c6b66f9c-fb2wf(ポッドNAME)
```


```
docker build -t hello-node:v2 .
```

```
$ kubectl set image deployment/hello-node hello-node=hello-node:v2
deployment.extensions/hello-node image updated
```

```
$ minikube service hello-node
Opening kubernetes service default/hello-node in default browser...
```

```
minikube stop
eval $(minikube docker-env -u)
minikube delete
```





