# WIP


# training Rails Directory structure

Ruby on Railsのディレクトリ構造を記載する。以下の表は、Railsの設定が終了した後、Welcome controllerが生成された時点のものを使用している。
（数が多いので、不要なものは削除している）

開発ではほとんど`/app/`以下のファイル群と`/config/routes.rb`を変更・追記することになる。
他には、データベースを定義する時は`/db/`以下のマイグレーションファイルの編集、テストは`/spec/`以下のファイルに記述する。

```
.
├── Gemfile      # railsプロジェクトで利用するgem（ライブラリ）一覧
├── Gemfile.lock # gemのバージョン固定用 `bundle update`で作り直される
├── Guardfile    # gem`guard`の設定ファイル 監視ファイルやアクションの記述
├── app/ # 設定を除く、アプリ本体のほとんどのソース
│   ├── assets/      # image, javascript, styleseetsなどの静的ファイル
│   │   ├── images/
│   │   ├── javascripts/
│   │   └── stylesheets/
│   ├── channels/    # ActionCable（railsでWebSocket）用ディレクトリ
│   ├── helpers/     # コントローラで利用するメソッドを作れる
│   │   ├── application_helper.rb
│   │   └── welcome_helper.rb
│   ├── models/      # MVCの M
│   │   └── application_record.rb
│   ├── views/       # MVCの V
│   │   ├── layouts/
│   │   │   └── application.html.erb
│   │   └── welcome/
│   │       └── index.html.erb
│   └── controllers/ # MVCの C
│       ├── application_controller.rb
│       └── welcome_controller.rb
├── bin/    # rails用スクリプト格納ディレクトリ
├── config/ # 設定ファイル格納ディレクトリ
│   ├── database.yml # データベース接続先の設定
│   └── routes.rb    # ルーティングの設定
├── db/     # データベース設定用ディレクトリ テーブル定義ファイルなどが入る
├── public/ # 静的ファイルの補完
├── spec/   # テストファイル格納ディレクトリ
└── tmp/    # キャッシュファイルなどが入る
```