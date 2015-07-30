# はじめに

自動テストを用意しておくことで, プロダクトをリファクタリングしたり, 或いは新しい機能を追加した時に, その変更にバグがないか(その変更によって, 既存の機能に何らかの不整合が出ていないか)を簡単に確認することができます.

しかし, せっかく自動テストを書いたとしても, そのテストが実行されなければ何の意味もありません.
このような問題に対して, コードをリポジトリにプッシュしたタイミングで自動的にテストを実行するような仕組みを作り, 継続的にテストを実行するようにしていく, というアプローチがあります.
このような取り組みを, 継続的インテグレーション(Continuous Integration, CI)と呼びます.

この資料では, Bitbucketのリポジトリで管理しているPerlモジュールに対して, CIのSaaSであるWerckerを利用してテストを継続的に実行する環境を作る方法について解説します.

# CI概要

正確に言えば, CIは｢テストの継続的な実行｣のみを指すのではなく, そこから自動的にデプロイやリリースを行うことで, ｢デプロイやリリースの継続的な実行｣もCIに含まれます.
但し今回は簡単のため, 前述の通り｢テストの継続的な実行｣のみを扱うこととします.

さて, CIを実現する為には, そのためのツールを利用するのが手っ取り早いです.
かつてはJenkinsを使う例がありましたが, Jenkinsは利用者がサーバにデプロイしなければならない為, 環境構築が多少面倒です.
そこで近年は, CIを提供するSaaSが多く提供されるようになってきました.

CIを提供するSaaSとしては, Githubで公開されているライブラリからの利用例が多い[Travis CI](https://travis-ci.org/)の他, [CircleCI](https://circleci.com/), [drone.io](https://drone.io/), そして今回利用する[Wercker](http://wercker.com/)などが存在します.
今回紹介するWerckerは, Bitbucketに対応していること, そして現在Werckerそのものがベータ版の為, プライベートリポジトリであっても無料で出来ることが特徴です.

それでは早速, Werckerを利用した継続的なテスト環境の構築に取り組んでいきましょう.

# Werckerのサインアップ

まず始めに, Werckerのアカウントを作成しましょう.
こちらの[Sing up](https://app.wercker.com/users/new/)ページに必要な情報を入力し, ｢SIGN UP NOW｣ボタンをクリックします.

![](/ci/wercker/1.png)

入力したメールアドレスに認証用のメールアドレスが届きますので, メール
中の｢ACTIVATE YOUR ACCOUNT NOW｣をクリックします.

![](/ci/wercker/2.png)

次のような画面が表示されていれば, サインアップは完了です.

![](/ci/wercker/3.png)

# CI対象となるリポジトリの用意

WerckerでCI環境を構築する前に, まずは今回CIの対象となるリポジトリをBitbucketに用意します.
今回はテストが動作することを確認できれば良いので, `minil new SampleModule`で生成したモジュールの雛形を, 自分のBitbucketの`SampleModule`リポジトリに登録して利用しましょう.

まず, Bitbucketでリポジトリを作成します.
リポジトリは, Bitbucketの上メニューから｢Create｣ボタンを押し, ｢Create repository｣を選択します.

![](/ci/wercker/4.png)

｢name｣にリポジトリ名である｢SampleModule｣を入力し, ｢Create repository｣をクリックします.
このとき, もし, ｢Access Level｣の｢This is a private repository｣にチェックが入っていない場合, **チェックを入れておいて下さい**.

![](/ci/wercker/5.png)

リポジトリの作成が完了しました.

![](/ci/wercker/6.png)

既存のリポジトリをこのリポジトリに登録するための方法については, ｢I have an existing project｣をクリックすると確認することができます.

![](/ci/wercker/7.png)

それでは, `minil`コマンドを利用してモジュールを作成し, Bitbucketにプッシュしましょう.

```
$ minil new SampleModule
Writing lib/SampleModule.pm
Writing Changes
Writing t/00_compile.t
Writing .travis.yml
Writing .gitignore
Writing LICENSE
Writing cpanfile
Initializing git SampleModule
[SampleModule] $ git init
Initialized empty Git repository in /Users/username/SampleModule/.git/
Retrieving meta data from lib/SampleModule.pm.
Name: SampleModule
Abstract: It's new $module
Version: 0.01
fatal: bad default revision 'HEAD'
[SampleModule] $ git add .
Finished to create SampleModule

$ cd SampleModule
$ git commit -m "initial commit"
[master (root-commit) 484e95e] initial commit
 11 files changed, 569 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 .travis.yml
 create mode 100644 Build.PL
 create mode 100644 Changes
 create mode 100644 LICENSE
 create mode 100644 META.json
 create mode 100644 README.md
 create mode 100644 cpanfile
 create mode 100644 lib/SampleModule.pm
 create mode 100644 minil.toml
 create mode 100644 t/00_compile.t

$ git remote add origin git@bitbucket.org:papix/samplemodule.git
$ git push -u origin --all
Counting objects: 15, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (13/13), done.
Writing objects: 100% (15/15), 8.65 KiB | 0 bytes/s, done.
Total 15 (delta 1), reused 0 (delta 0)
To git@bitbucket.org:papix/samplemodule.git
 * [new branch]      master -> master
Branch master set up to track remote branch master from origin.
```

問題なくプッシュが終了すると, リポジトリをBitbucketした際の表示が次のようになります.

![](/ci/wercker/8.png)

これでリポジトリの準備は完了です.

# WerckerとBitbucketの連携

次に, WerckerからBitbucketのリポジトリを確認できるよう, WerckerとBitbucketの連携を行います.
メニュー右上の歯車アイコンをクリックし, [｢Profile Settings｣](https://app.wercker.com/#profile/personal)ページへ移動します.

![](/ci/wercker/9.png)

左のメニューから, ｢Git connections｣を選択します.

![](/ci/wercker/10.png)

Bitbucketのロゴの右にある, ｢Connect｣ボタンをクリックします.

![](/ci/wercker/11.png)

Bitbucketのセッションが残っている場合, Bitbucketのロゴの下の文字が｢Connected｣に変化します.
これでWerckerとBitbucketとの連携は完了です.

![](/ci/wercker/12.png)

# CI環境の構築

それではいよいよ, Werckerを使ったCI環境を構築していきます.
上部メニューから｢Create｣をクリックし, ｢Application｣を選択します.

![](/ci/wercker/13.png)

すると次のような画面になります.
まずは, CIが対象とするリポジトリを選択します.
先程, WerckerとBitbucketの連携を済ませたので, ｢Use Bitbucket｣というボタンが存在するはずです. これをクリックします.

![](/ci/wercker/14.png)

少し時間がかかった後, 次のような画面が表示されます.
テキストボックスにリポジトリ名を入力して検索し, 該当するリポジトリをクリックしてから, ｢Use selected repo｣をクリックします.

![](/ci/wercker/15.png)

次に, Werckerがリポジトリにアクセスする為の手段を設定します.
今回は, プライベートリポジトリを対象としているので, 一番下の｢Manually add the deploy key.｣を選択します.

｢key｣の部分に表示されたSSH公開鍵は, 一旦コピーして保存しておきましょう(後でBitbucketで登録します).
問題なければ, ｢Next step｣をクリックします.

![](/ci/wercker/16.png)

そのまま｢Next step｣をクリックします.

![](/ci/wercker/17.png)

これでWerckerの設定は完了です.
｢Make my app public｣は, 今回はプライベートリポジトリを対象としているのでチェックせず, そのまま｢Finish｣をクリックします.

![](/ci/wercker/18.png)

このような画面が表示されていれば, OKです.

![](/ci/wercker/19.png)

# SSH公開鍵の登録

最後に, Bitbucketで管理しているリポジトリに, Wercker用のSSH公開鍵を登録します.
まず, Bitbucketのリポジトリのページの左側にある, 歯車をクリックします.

![](/ci/wercker/20.png)

｢GENERAL｣の｢Deployment keys｣をクリックします.

![](/ci/wercker/21.png)

｢Add key｣ボタンを押し, SSH公開鍵の入力画面を開きます.

![](/ci/wercker/22.png)

｢Label｣に鍵の名前(今回の場合, ｢Wercker｣等で良いでしょう), ｢Key｣に先程Werckerで表示されていたSSH公開鍵を入力し, 右下の｢Add key｣ボタンをクリックします.

![](/ci/wercker/23.png)

このような表示になっていれば, SSH公開鍵の登録は完了です.

![](/ci/wercker/24.png)

# `wercker.yml`の用意

Werckerは, リポジトリのルートディレクトリに設置された`wercker.yml`というファイルに従ってCIを実施します.
今回は, 次のような設定を利用することとします.

```yml:wercker.yml
box: papix/perl5.18@0.0.1
build:
    steps:
        - script:
            name: Install modules
            code: cpanm -l local --installdeps .
        - script:
            name: Run test
            code: prove -l -Ilocal t
```

## `wercker.yml`の書き方

### `box`

```yaml:wercker.yml(抜粋)
box: papix/perl5.18@0.0.1
```

`wercker.yml`では, 必ずBoxを指定しなければなりません.
Boxは, Werckerがデフォルトで提供しているものを利用することができます.
ただ, WerckerでCIを行う際, 事前に依存となるライブラリ等のインストールを実施している場合, それらを既にインストールしたBoxを予め作成しておけば, インストールにかかる時間を短縮することができます.

ここでは, WerckerのBoxを作成する方法については省略します.
なお, Werckerがデフォルトで提供しているものを含め, Werckerから利用できるBoxについては, [こちらのページ](https://app.wercker.com/#explore/boxes)から検索することが可能です.

このファイルでは, Perl 5.18及びApp::cpanminusとCartonをインストール済みの[papix/perl5.18](https://app.wercker.com/#applications/542ec5cf4d7a367e23000257/tab/details)を利用しています.

### `services`

```yaml:wercker.yml(抜粋)
services:
    - wercker/mysql
    - wercker/redis
```

Werckerでは, MySQLやRedisは自由に起動出来ず, `service`という概念を利用して起動しなければなりません.
これらの指定を行うのが`services`です.

Werckerで利用できる`service`やその詳細については, [こちらのページ](http://old-devcenter.wercker.com/articles/services/)を確認して下さい.

### `build` / `deploy`

`build`はビルドの処理を, `deploy`はその後のデプロイ処理を書くフェイズです.
今回は`deploy`は実施しないので, テストの実行のみを`build`を利用して実施します.

`build`及び`deploy`は, ｢step｣と呼ばれるユーザが定義したひとまとまりの処理を利用することが可能です.
例えば, [moltar/carton-install](https://app.wercker.com/#applications/545e63cffa3ae0c709291ff7/tab/details)というstepを利用することで, Cartonを利用したライブラリのインストールを次のように書くことが可能です.

```yaml:wercker.yml(抜粋)
build:
  steps:
    - moltar/carton-install
```

なお, 提供されている`step`については, [こちらのページ](https://app.wercker.com/#explore/steps/search)から検索することが可能です.

また, `script`を利用して, 任意のスクリプトを実行することも可能です.

```yaml:wercker.yml(抜粋)
build:
  steps:
    - script:
      name: echo 1
      code: |-
        echo "hoge"
    - script:
      name: echo 2
      code: |-
        echo "fuga"
```

今回は, `script`を利用して, `cpanm`を利用したモジュールのインストールと, `prove`を利用したテストの実行を行っています.

なお, `build`や`deploy`は, その途中でstepの処理に失敗した場合, 或いは`script`で実行したコードが正しく終了しなかった場合, その時点で一連の処理を中断します.
そして, Wercker上ではその一連の処理は失敗したものとして扱われます.

## `after-steps`

`build`及び`deploy`では, `after-step`を利用するこが出来ます.
`after-step`に記述した処理は, `build`及び`deploy`の処理が途中で失敗した場合であっても, 最後に必ず実行されます.

```yaml:wercker.yml(抜粋)
build:
  steps:
    ...
    ...
    ...
  after-steps:
    - hipchat-notify:
      token: HIPCHAT_ROOM_TOKEN
      room-id: room_id
      from-name: wercker-bot
```

例えば, このように[hipchat-notify](https://app.wercker.com/#applications/51f26c380771b3526e000c1c/tab/details)のstepを利用すれば, `build`の`steps`に記載した処理の結果を通知してくれます.

※ここで求められる`token`は, HipChatのルーム単位で生成できるトークン(API v2用のトークン)ではなく, HipChatの管理者ユーザが生成できる全部屋共通のトークン(API v1用のトークン)です.

このように, 必ず最後に実行したい処理があるのであれば, 実行漏れがないように`after-steps`を利用して記述するようにしましょう.

この他, `wercker.yml`で利用できる機能の詳細については, [こちらのページ](http://old-devcenter.wercker.com/articles/werckeryml/)を確認しましょう.

# 実行結果の確認

それでは, この`wercker.yml`をコミットして, Bitbucketにプッシュしてみましょう.

```
$ git add .
$ git commit -m "add wercker.yml"
[master c003585] add wercker.yml
 Date: Fri May 8 00:21:01 2015 +0900
 1 file changed, 9 insertions(+)
 create mode 100644 wercker.yml

$ git push
Counting objects: 3, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 398 bytes | 0 bytes/s, done.
Total 3 (delta 1), reused 0 (delta 0)
To git@bitbucket.org:papix/samplemodule.git
 + 608ec62...c003585 master -> master
```

リポジトリをBitbucketにプッシュすると, Werckerはそれを自動的に検知し, `wercker.yml`に従って処理を実行します.

![](/ci/wercker/25.png)

テストの状況については, このようにWerckerに接続した際のトップページのフィードに表示されます.
また, フィード上のテスト結果をクリックすることで, 次のようにその詳細を確認することも可能です.

![](/ci/wercker/26.png)

万が一ビルドに失敗した場合は, 次のようにメールによる通知が行われます.

![](/ci/wercker/27.png)
