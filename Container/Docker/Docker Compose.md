# WIP

# Docker Compose

**Docker Compose** は、Dockerfileを指定してイメージの生成（build）から、起動時オプションの設定やコンテナの起動（run）までこなしてくれる、Dockerコンテナの管理支援ツールだ。
複数コンテナを利用する場合、コンテナ同士の依存関係の設定まで全て`docker-compose`がカバーしてくれる。起動する順番が決まっている場合も、順番を設定して適切に起動できる。

使い方はさくらインターネットがほとんど説明してくれていたので、ここでは自分が利用しているパターンを記載する。

[複数のDockerコンテナを自動で立ち上げる構成管理ツール「Docker Compose」（Dockerの最新機能を使ってみよう：第7回）](https://knowledge.sakura.ad.jp/5736/)


# 自分の利用パターン

自分はDockerを使う全ての環境で、docker-composeも合わせて利用している。Dockerとdocker-composeさえ入っていれば、どの環境でも設定なしで`docker-compose up`とタイプするだけで全く同じ環境が立ち上がるからだ。
`docker-compose down`で関連コンテナを全て停止することができ、`docker-compose build`とすればイメージが作り直されるし、`docker-compose rm`とすればコンテナが削除されるなど、大変使い勝手が良い。

## Dockerfileが1つの場合

Dockerファイルが1つのプロジェクトで利用する`docker-compose.yml`はこのようになる。自分はどのプロジェクトでもこれを使いまわしている。例で挙げているのは、Railsのプロジェクトで利用しているやつ。
`valumes`オプションでカレントディレクトリのrailsプロジェクトをコンテナの`/var/www/rails`として共有、`expose`オプションでポートを開いている。`ports`オプションでポートのパイプを設定することで、ホストマシンから`0.0.0.0:3000`でアクセスできるようになる。

```yaml
version: '2'
services:

  app:
    build:
      context: ./
    container_name: 'container-name'
    volumes:
      - ./:/var/www/rails
    ports:
      - 3000:3000
    expose:
      - 3000
    command: bash -c 'bundle install && puma -C config/puma.rb'
```

## Dockerfileがいっぱいある場合

これは、rails+mysql+nginxのテンプレートであり、[このリポジトリ](https://github.com/shiopon01/docker-template-for-rails)で公開しているものだ。

`link`オプションでそれぞれのコンテナを繋げることができる。仕組み的には、繋げる先のアドレス等が自動的に環境変数に設される、とかだったと思う。それを利用することでコンテナの連携を実現する。
`depends_on`オプションを利用すると、指定した名前のコンテナの起動を待つことができる。これで、順番の問題を解決する。（今回なら、`mysql`→`rails`→`nginx`の順番で起動する）

```yaml
version: '2'
services:

  nginx:
    build: './nginx'
    container_name: 'example-nginx'
    ports:
      - 80:80
    depends_on:
      - rails
    links:
      - rails:rails

  rails:
    build:
      context: ./rails
      args:
        app_name: example_app
    container_name: 'example-rails'
    volumes:
      - ./rails/example_app:/app/example_app # source code
    depends_on:
      - mysql
    links:
      - mysql:mysqld
    extends:
      file: ./mysql/mysql_env.yml
      service: env
    expose:
      - 3000
    command: bash -c 'bundle install && puma -C config/puma.rb'

  mysql:
    build: './mysql'
    container_name: 'example-mysql'
    volumes:
      - ./mysql/volumes:/var/lib/mysql # datastore
    extends:
      file: ./mysql/mysql_env.yml
      service: env
    expose:
      - 3306
```