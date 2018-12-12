# WIP

# 概要

Dockerは、仮想化技術「コンテナ」を利用したOS仮想化ツールだ。
VMWareなどのホスト型（OSに土台となるソフトウェアをインストールしてその上で仮想OSを立ち上げる）とは違い、OSの一部のみを仮想化して利用するため、非常に軽量に扱える。

各コンテナはアクセスできるリソースや権限を制限/分離したホストOSの「プロセス」として扱われるため、コンテナを管理するコストはプロセス管理とほとんど変わらない。
コンテナはホストOSのLinuxカーネルを共有するので、カーネルとコンテナの関係はJVMとJarに近い。（Docker for Windowsでは、LinuxカーネルをHyper-Vで動かしているっぽく、少し手間。Linux系での利用を推奨）

# Dockerのインストール方法

おおまかなインストール手順は以下の通り。ふつう、Docker Community Edition (CE)を利用する。

https://docs.docker.com/engine/installation/#server

## Windows

Docker for Windowsをインストールする。
基本的にはインストール手順に従っていけば良いが、WindowsではHyper-Vを有効にしなければならない。

https://docs.docker.com/docker-for-windows/install/

## Mac

Docker for Macをインストールする。
ほんとうにそれだけで動いたと思う。

https://docs.docker.com/docker-for-mac/install/

## Linux

これも公式のドキュメントにあるやつでだいたいOK。
しかし、そのままだとdockerコマンドの利用に`sudo`が必要になるので、ユーザーをdockerグループに入れてあげる。

Ubuntuだと、こんなかんじで。

```bash
sudo apt -y install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt -y install docker-ce
sudo groupadd docker
sudo usermod -aG docker $USER
```

https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/#install-docker-ce


# 利用手順

とりあえず利用してみるのが一番早い。

Dockerでコンテナを作成するためには、まずコンテナを作成するための型であるイメージが必要だ。イメージがたい焼き器なら、コンテナがたい焼きという認識で問題ない。

イメージはDockerHubという公式のイメージ共有サービスからダウンロードすることができる。まず、目的のイメージを検索する。今回は練習で、ubuntuのイメージをダウンロードして使ってみる。検索する時は`docker search`が使える。

（イメージはブラウザのDockerHubで探したほうが楽）
https://hub.docker.com/

```
docker search ubuntu
```

今回はOFFICIALの、公式が出しているubuntuイメージを利用する。
DockerHubのイメージをローカルに落としてきたい場合は`docker pull [イメージ名]`が使える。この時、`ubuntu:17.04`のようにタグを指定すれば、指定のタグのバージョンを手に入れる事ができる。何も付けなければ、`latest (最新)`のイメージが手に入る。

```
docker pull ubuntu
```

今までにダウンロードしたイメージは`docker images`で確認できる。この時に表示される`REPOSITORY`か`IMAGE ID`を指定して、イメージの削除も可能。

```
docker images
docker rmi [指定イメージ]
```

コンテナの生成と起動は`docker run [イメージ名]`で行う。しかし、役目を終えたdockerのコンテナすぐに消えてしまうので、このままでは何も起こらない。イメージ名の後に、実行したいコマンドが必要になる。例えば、さっきダウンロードした`ubuntu:latest`で`echo 1`を実行する場合。

```
docker run ubuntu echo 1
```

コマンドを指定するとubuntuは`1`を出力して、消えてしまう。消えると言っても実際は消滅しているわけではなく、`STATUS: Exited`になって寝ているだけなのだ。その様子は、`docker ps -a`で確認できる。この時に表示される`CONTAINER ID`を指定して、コンテナの削除が可能。

Exitedのコンテナがたまっていくのは嫌なので、コンテナの終了時にコンテナは削除したい。その時は、`run`コマンドの`--rm`オプションが利用できる。

- --rm コンテナの終了時にコンテナをクリーンアップし、ファイルシステムを削除する

```
docker run --rm ubuntu echo 1
```

また、起動したコンテナの中に入って作業をしたいという場合もある。この場合、`-it`オプションが利用できる。

- -t 疑似ターミナルでコンテナを起動

```
docker run -it --rm ubuntu echo 1
```


この時に表示される`1`は、dockerの疑似ターミナル内で表示される`1`になる。しかし、役目を終えたコンテナはすぐに消滅してしまう。ほんとうに中で作業をしたいなら、`bash`などのshellを起動しよう。
`exit`などでshellを抜けることで終了できる。

- -i 対話モードでコンテナを起動

```
docker run -it --rm ubuntu bash
```

## docker runのオプション

- -d デタッチドモード（デーモン）でコンテナを起動
- -i 対話モードでコンテナを起動
- -t 疑似ターミナルでコンテナを起動
- -v Data Volumeの指定（所謂、フォルダ共有）
- -w 開始ディレクトリの指定（デフォルトはルート）
- --name [名前] : コンテナに名前をつける:
- --rm コンテナの終了時にコンテナをクリーンアップし、ファイルシステムを削除する
- --reset [ポリシー] : 再起動ポリシーを設定（no, always, など）
- -p ポートのバインド（例: 80:8080ならホスト80番でコンテナ8080番）
- -e 環境変数設定（RAILS_ENV=development）

普通に`docker`コマンドで利用するくらいなら、`-it --rm`くらいしか使わない（たぶん）。
その他はほとんど、docker-compose（後述）の設定ファイルで設定する。

## よく使うdockerコマンド

先述した、疑似ターミナルの対話モードでbashを起動など。
停止しない状態で元のターミナルに戻りたい（コンテナからデタッチ）なら Ctrl + P + Qを押せば良いらしい。

```
docker run -it --rm [image] bash
```

コンテナが増えた場合、このコマンドで全コンテナを削除できる。

```
docker rm $(docker ps -aq)
```

# Dockerfile

VMを使ってチームメンバー全員が同じ開発環境を整えようとすると、とにかく手間がかかる。Vagrantで生成した400MB以上あるboxファイルを全員に配布しても良いが、利用していく中で好き勝手にカスタマイズされ、気付けば全く別の環境で開発していたという状態もあり得ない話ではない。

そこで、Dockerfileの出番になる。  Dockerfileは、全ての環境で同じDockerのイメージを作成するための設定ファイルだ。実態は、ただのテキストファイルである。

このテキストファイル1つで、DockerHubから指定のイメージの取得とコンテナの作成、記載されたコマンドを実行、環境変数を変更し、全く同一の環境を整えてくれる。そこで整えたものを、`新しい一つのイメージ`として提供してくれるのがDockerfileの役割だ。

以下の例では、`ruby:2.4.2`のイメージを元に、nodejsの9系をインストールできる。
最後は環境変数`ROOT`に設定されたディレクトリに移動しているので、このDockerfileから生成されたイメージで作られたコンテナは、`/var/www`からスタートする。

```
FROM ruby:2.4.2

RUN curl -sL https://deb.nodesource.com/setup_9.x | bash
RUN apt install -y nodejs

ENV ROOT /var/www
WORKDIR $ROOT
```

大まかな使い方は公式リファレンスが丁寧に書いてくれているので割愛。
ここには、ややこしい`CMDとENTRYPOINT`、`ADDとCOPY`の違いを記載する。

[Dockerfileリファレンス](http://docs.docker.jp/engine/reference/builder.html)


## 起動時コマンド、CMDとENTRYPOINT

Dockerfileにこれを記述することで、そのイメージのコンテナを実行する際に実行するコマンドを設定できる。ここで`bash`を設定しておけば、`docker run`する時にわざわざ指定しなくても良いということになる。
例えば、こんな感じで設定する。

```
CMD ["ping","127.0.0.1","-c","100"]
```

```
ENTRYPOINT ["ping","127.0.0.1","-c","100"]
```

1つのDockerfile内では、CMDとENTRYPOINTはそれぞれ1回ずつしか使えない。併用は可能になっている。（複数記載されている場合、最後のコマンドが実行される）

CMDで指定したコマンドは、Docker run時に指定したコマンド指定で上書きされてしまう。

```
$ docker run --rm ubuntu cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=16.04
DISTRIB_CODENAME=xenial
DISTRIB_DESCRIPTION="Ubuntu 16.04.3 LTS"
```

ENTRYPOINTで指定したコマンドは、Docker run時に指定したコマンド指定で上書きされない、というのが大きな違いだ。
ただし、ENTRYPOINTでも`docker run --entrypoint=""`を使用されると上書されてしまうので注意。

```
$ docker run --rm ubuntu cat /etc/debian_version
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.028 ms
```

##  ["command"]とcommand

起動時のコマンドでも2種類あったが、さらにそれぞれ2種類の書き方がある。
括弧で囲む書き方と、括弧で囲まない書き方だ。

括弧で囲んだコマンドは、シェルを介さずにコマンドが実行される。`["command"]`
一般的にはこちら使われる。

```
CMD ["ping","127.0.0.1","-c","100"]
ENTRYPOINT ["ping","127.0.0.1","-c","100"]

#=> ping 127.0.0.1 -c 100
```

括弧で囲まないコマンドは、シェルを介してコマンドが実行される。

```
CMD ping 127.0.0.1 -c 100
ENTRYPOINT ping 127.0.0.1 -c 100

#=> /bin/sh -c ping 127.0.0.1 -c 100
```


## 特徴を活かして併用する

`CMD`と`ENTRYPOINT`を両方書いた場合、連結されて1行のコマンドとして扱われる。
コマンドのみ固定にしておいて、引数を変更するなどの使い方ができる。

```
ENTRYPOINT ["ping"]
CMD ["127.0.0.1", "-c", "50"]
```

こう書くことで、dcoker runの際にコマンドは書き換え不可、引数を書き換え可能にできる。
引数が無い場合は127.0.0.1へpingが送られ、引数がある場合はそのIPにpingを送るコマンドだ。

```
$ docker run ubuntu localhost
PING localhost (127.0.0.1) 56(84) bytes of data.
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=0.027 ms
```

（DockerfileにENTRYPOINTを書かなかった場合、docker-compose.ymlに下記のように追記することで、サービスの全ての設定が終わった後に実行するコマンドを設定することが出来る。）

```
app-name:
  command: node app.js
```

## ファイルの転送の違い（ADD、COPY）

`ADD [ソース] [送信先]`で、ホストに存在するファイルやディレクトリをコンテナにコピーできる。このソースファイルが`tar`アーカイブであった場合、自動的に解凍・展開される。

```
ADD default.conf /etc/apache2/sites-available/default.conf
```

`COPY [ソース] [送信先]`で、ホストに存在するファイルやディレクトリをコンテナにコピーできる。こちらは`tar`を展開しない。

```
ADD default.conf /etc/apache2/sites-available/default.conf
```

ほとんど同じなので、`ADD`と`COPY`を明確に使い分ける必要はない。（ADDだけでよさそう）