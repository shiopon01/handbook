# shiopon01/handbook

昨今、様々なブログが乱立して検索結果の上位を占領している。この汚染されたインターネットでは入門記事を探そうとしても一苦労であるし、そもそも公式ドキュメントも初学者にとって内容が難しいものが多い。

この **handbook** の目的は、 `shiopon01` が学んだ知見や経験を文書化し、共有することにある。

連絡先: [@shiopon01](https://twitter.com/shiopon01)

## 目次

### [AWS](/AWS)

AWSを初めて使う時、一番困ったのはまあまあ分かりづらい公式ドキュメントの山だった。サービスのUIも統一感が無く、全体で見ると扱いやすいサービスとは言えない。

これを踏まえ、AWSの章では自分がAWSサービスを使ってきた中で得た知見と経験を合わせ、初学者にも順序立ててできるだけ分かりやすく説明することを目指す。結果として、そのサービスを理解してもらうことを目的としている。

- [CloudFormation](/AWS/CloudFormation)

### [Container](/Container)

Container

- [Kubernetes](/Container/Kubernetes)

## 構成

基本、章を表すディレクトリ名の下には主題となるディレクトリがあり、その中に副題となるマークダウンのファイルが配置される。

※ファイル名は **数字3桁-ファイル名.md** のものがこのリポジトリの正しいファイルだ。それ以外のものは記述途中や、残骸の場合が多い。

※本構成は予告なく変更される場合があります。

- AWS/
  - DynamoDB/
    - 001-article.md

- Ruby/
  - Features/
    - 001-article.md
  - Rails/
    - 001-article.md

**tmp** フォルダは記述途中のもの、またはGitBook管理していた頃の残骸。