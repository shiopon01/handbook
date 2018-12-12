# WIP

Rubyのインストールにかぎらず、言語をインストールする時は、言語のバージョン管理ツールが良く利用される。最新版が出ても面倒な入れ直しをしなくても良く、新しいバージョンをインストールして切り替えるだけで良いからだ。Rubyでは、`rbenv`がよく利用されている。

# rbenvのインストール方法

Ubuntuならこれでインストールから、起動時の設定まで完了する。（bash以外を利用している場合は、適切な設定ファイルに追記すること）

```
sudo apt install -y rbenv libssl-dev libreadline-dev zlib1g-dev
git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
source ~/.bash_profile
```

公式GitHubを見る限り、Macなら`brew`からインストールできるらしい。

https://github.com/rbenv/rbenv#installation


# rubyのインストール方法

まず、rbenvの機能を利用してrubyをインストールしなければならない。
どのバージョンがインストール可能か、rbenvの機能を使って確かめてみる。

```
rbenv install --list
```

今回は、比較的新しめな`2.4.2`をインストールすることにしよう。
`install`にバージョン番号を渡すことでインストールできる。

```
rbenv install 2.4.2
```

現在インストールしているバージョンは`versions`で確認できる。今回は`2.4.2`をインストールしたので、それが表示されているはずだ。
この時、自分が何のバージョンを利用しているかも表示される。（アスタリスクが付いているものが、今使っているバージョン）

```
$ rbenv versions
* system
  2.4.2 (set by /home/you/.rbenv/version)
```

`system`は普通にインストールされているもののことだ。これをさっきインストールした`2.4.2`に切り替えるには、`global`を利用する。
この際、rubyのライブラリ管理によく利用される`bundler`のインストールと、それを利用するためにrbenvの`rehash`も行っておこう。

```
$ rbenv global 2.4.2
$ rbenv versions
  system
* 2.4.2 (set by /home/you/.rbenv/version)
```

```
gem install bundler
rbenv rehash
```

# Gem

`Gem`はrubyに標準で付属されているライブラリを管理ツール。インストールされたライブラリは、ソースコード内の`require`で指定することで利用できる。標準でもいろいろインストールされており、代表的なものではオブジェクトを見やすく出力してくれる`pp`、rubyのコードを対話モードで簡単に実行できる`irb`などがある。

```
gem install [インストールしたいライブラリ]
```

```ruby
require 'pp'
```

https://www.ruby-lang.org/ja/libraries/

# bundler

`bundler`はgemのインストールを簡略化するための仕組み。ほとんどはプロジェクトで利用するgemを合わせるために利用される。
ふつう、gemのインストールは`gem install`コマンドで行われるが、`Gemfile`と`bundler`を利用することで必要なgemを一括でインストールすることができる。

## Gemfile

`Gemfile`自体は、プロジェクトに必要なgemが記述されたただのテキストファイルだ。Gemfileにはこのように、必要なgemを記述していく。

```ruby
# Gemfile
gem "brakeman"
gem "bundler-audit"
gem "rack-mini-profiler"
gem "better_errors"
```

この`Gemfile`が存在するディレクトリで`bundle install`をタイプすることで、ファイル内の全てのgemをインストールすることが可能だ。

```
$ bundle install
```

また、この時に`Gemfile.lock`というものが生成される。これはチームメンバ間でのgemのバージョンを合わせるためのもので、gemと、インストールされたバージョン、そのgemの依存関係などが記載されている。

```
# Gemfile.lock
brakeman (4.0.1)
bundler-audit (0.6.0)
  bundler (~> 1.2)
  thor (~> 0.18)
rack-mini-profiler (0.10.6)
  rack (>= 1.2.0)
better_errors (2.4.0)
  coderay (>= 1.0.0)
  erubi (>= 1.0.0)
  rack (>= 0.9.0)
```

`Gemfile.lock`が存在している場所でに`bundle install`をタイプすると、bundlerはまず`Gemfile.lock`を参照して、同じバージョンのgemをインストールしてくれる。その後`Gemfile`も参照してくれて、その時にもし追加のgemを見つけた場合は、インストールした後に`Gemfile.lock`への追記もしてくれる。
現在の`Gemfile`で`Gemfile.lock`を新しく作り直したい場合は`bundle update`で可能だ。

```
# Gemfile.lockを参照した後、Gemfileも見てくれる
$ bundle install

# 現在のGemfileからGemfile.lockを作り直す
$ bundle update
```



Rubyはインタプリタ方式の、全てがオブジェクトの言語だ。
ソースコードは、だいたいこんな感じ。

```ruby
10.times do |n|
  puts n
end
```

ユーザが楽しく記述することに重点を置いた言語であるため、不要なもの（宣言など）は排除されており、構文の自由度は高いため、雰囲気で記述してもほとんどの場合はうまく動作してくれる。

Rubyは、PHPと同じような`インタプリタ方式`なので、開発時の実行などが容易である。コンパイラ方式のようにコンパイル、リンクして実行ファイルを作り出す必要ないので、書き換えてから実行するまでの確認は素早くできる。（インタプリタ方式の特徴として、実行ファイルと比べると速度は桁違いに遅いという短所はある）

枠組みではJavaと同じような`オブジェクト指向`が採用されており、オブジェクト同士の相互作用やそれ自体の振る舞いによってシステムを形成する。そのため、他の言語に比べると記述量は少なくて済む。
また、Ruby言語の方針として`動的型付け`が採用されており、変数の宣言が必要ないので記述量は更に少なくて済む。

最近では当たり前の機能である`例外処理機構`、`ガベージコレクション`も採用されており、他には`マルチスレッド`もサポートされている。

https://www.ruby-lang.org/ja/about/


# 実行する

Rubyを試すために、ソースコードを書いたファイルを用意する必要はない。Rubyには、対話的なセッションでコードを実行できる`irb`が標準で搭載されている。

実際にRubyを試してみるには、Ruby公式のチュートリアル[「20分ではじめるRuby」](https://www.ruby-lang.org/ja/documentation/quickstart/)が参考になると思う。