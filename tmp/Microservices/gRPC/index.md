# RPC

-


# gRPCのメリット

- 権限の細分化
  - SQSなどのサービスへの読み取り権限を安全な場所に移動させられる。
    - 例えば、キュー駆動のバッチサーバーがSQSを監視していたものを、コントローラーサーバーが監視することにして、コントローラーがバッチサーバーに命令を投げる
- 異なる言語のマイクロサービス間の通信でも通信を確立できる（streamingの方式を用途に合わせることで、省コネクションでやり取りを行える）
- Bidirectional streamingは1回のコネクションでクライアントとサーバー間で複数回のリクエストとレスポンスを送ることができる。（リクエスト・レスポンスのたびにコネクションを確立しない）1コネクションでバッチサーバーの起動などのアクションを行うことができる

- 流れとしてまず初めにprotoファイルを更新するため、ドキュメント化が楽（そもそもこれがドキュメント）



- HTTP/2ベースのRPC
- 1本のTCP接続を引っ張りなし+パイプライン化
- バイナリプロトコル
- HPACKによるヘッダ圧縮
- サーバープッシュ/ストリーミング
- Protocol Buffersによる言語非依存の通信定義
- 通信種別を4種類に整理
  - 単方向リクエスト/レスポンス（Unary）
  - サーバー -> クライアントストリーミング（ServerStreaming）
  - クライアント -> サーバーストリーミング（ClientStreaming）
  - 双方向ストリーミング（DuplexStreaming）

- 実装に集中できる
- クライアントが作りやすい（テストや多言語の対応）

# 欠点

- REST APIではないので外部からAPIを叩けない
  - curlとかで叩けず、gRPCでしか通信できない
    - `grpc-gateway` でgRPCをラップしてJSON APIを生成する `protoc` のpluginを利用することで、gRPCをリバプロしてREST APIで叩くことが出来る
- AWSのNLBをを利用していると、350sのidleでconnectionが切られる
  - keepalive

# .protoの管理

https://blog.stormcat.io/post/entry/protogen/


# ロードバランシング

- https://hakobe932.hatenablog.com/entry/2018/04/11/123000
- https://grpc.io/blog/loadbalancing
