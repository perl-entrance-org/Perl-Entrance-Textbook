# はじめに 

複数人でプロダクトを開発する場合, 問題となるのは｢コードの管理｣です.

例えば, あるプロダクトのあるファイルを, 同時に2人の開発者が変更を加えたとします.
それぞれがファイルを書き換え, 機能を追加した後, その2つのコードを統合するにはどうすればいいでしょう?
変更するファイルが1つであればまだ手動で頑張ることもできるでしょう.
しかし統合するために書き換える必要があるファイルが10ファイル, 20ファイルと増えると, 人力で処理するのは現実的ではありません.

或いは, 過去のコードを残しておきたい, という場合もあると思います.
よく使われる(使われていた)手として, ｢2015-04-30｣, ｢2015-05-01｣... のように日付ごとにディレクトリを切って, その中にその日の段階のプロダクトの全てのコードをコピーしておく(!?)というものがありますが...
言う前でもなく, その日変更がなかったファイルも含めて全てまるっと複製するのでディスク容量の負担になりますし, 間違えて過去のディレクトリのファイルを書き換えたり, 或いは削除してしまった場合, それはもはや｢過去のコードを残す｣という目的を果たしていません.

このような問題を解決するために開発されたのが, バージョン管理システム(Version Control System)です.
バージョン管理システムは, 独自のアルゴリズムに基づいて, 分岐したコードを1つのコードに結合(マージ)する手助けをしてくれます.
また, コードに対する変更(コミット)の履歴を保持し, いつでも任意の状態に戻れるようにしてくれます.
その際, コミットごとに全てのファイルを保存するのではなく, 前後の差分を保存するようになっているので, ディスク容量もさほど消費しません.

現代, 高速なプロダクト開発が求められている中で, ｢複数人が同時にコードに変更を加える｣という場面はもはや日常茶飯事となっています.
そのため, プロダクトのコードを効率よく管理する｢バージョン管理システム｣に関する知識は, もはやエンジニアとしての必須の知識, 一般常識であると言うことができるでしょう.

今回は, 代表的なバージョン管理システムの1つ, Gitの使い方について学びます.

# Git概要

Gitは, Linuxカーネルの開発者Linus Torvaldsが, Linuxカーネルで利用するために開発したバージョン管理システムです.
もともと, Linuxカーネルの開発にはBitKeeperという商用ツール(本来は有料のところ, BitKeeperを開発していたBitMover社の好意でLinuxカーネルの開発については無料で使えるようになっていた)が使われていました.
これに対し, `rsync`などの開発者でもあるAndrew TridgellがBitKeeperのプロトコルをリバースエンジニアリング(といっても, 内容としてはBitKeeperのサーバの適切なportに対して, `help`とタイプするだけの簡単なものだったそうです)したところ, この行為をBitMover社はライセンス違反として扱い, Linuxカーネルに対するBitKeeperの利用を停止することになってしまいました.
当時, BitKeeper以外にも様々なバージョン管理システムがありましたが, Linuxカーネルという巨大なプロダクトに(特に速度面で)耐えうるバージョン管理システムは存在しなかった為, Linus氏はそれを自らの手で作ることにしました.
これが, 現在Linuxカーネルだけでなく, Githubなどのホスティングサービスを軸に様々な企業で使われるようになったバージョン管理システム, ｢Git｣です.

バージョン管理システムとしては, Gitの他にMercurialやBasaarなどが存在しますが, ここ最近はGitがバージョン管理システムの世界におけるデファクトスタンダードとなりつつあります(一部, Subversionが使われている所もあるでしょう).
OSSの多くはGit(GitHub)でコードを管理していますし, 個人プロダクトの為にGitを使う場合もあれば, GitHubやBitBucketのOrganizationの機能を利用して, 企業のプロダクト開発のためにも, Gitが使われるようになっています.

# リポジトリの作成と取得

バージョン管理システムでは, ファイルの各バージョンの情報(= コミットの情報)を独自の仕組みで保持/管理しており, このデータを｢リポジトリ｣と呼びます.
まずは, バージョン管理の軸となる, リポジトリを用意する方法について解説します.

既存のコードを新たにGitでバージョン管理したい場合, `git init`コマンドでリポジトリを作成することができます.

```
$ git init
Initialized empty Git repository in /Users/username/git-tutorial/.git/
```

`git init`コマンドを実行すると, カレントディレクトリに`.git`というディレクトリが作成され, ここにリポジトリの各情報が保存されるようになります(その為, `.git`ディレクトリは削除したり, 書き換えたりしないようにしなければいけません).
これで, Gitのリポジトリの準備は完了です.

一方, 既存のリポジトリから新たに自分用のリポジトリを作成する場合, `git clone`というコマンドを利用します.
例えば, Githubにある[perl-entrance-org/Perl-Entrance-Textbook](https://github.com/perl-entrance-org/Perl-Entrance-Textbook)を取得する場合, 次のような手順が必要になります.

## リポジトリのURLの確認
GitHubやBitBucketの場合, それぞれリポジトリのページにURLが掲載されています

### Githubの場合
![](/git/1.png)
### Bitbucketの場合
![](/git/2.png)

矢印の部分がそうです. 今回は, `git@github.com:perl-entrance-org/Perl-Entrance-Textbook.git`のようなURLが記載されています.

なお,

- Githubの場合, URLの下側の`HTTPS`や`SSH`などをクリック
- Bitbucketの場合, URLの左側の`SSH`をクリック


すると, リポジトリを取得する際に利用するプロトコルを変更することができます.
基本的には`SSH`を利用するようにしましょう.

## リポジトリの取得

リポジトリのURLがわかれば, 後はリポジトリを取得するだけです.

```
$ git clone git@github.com:perl-entrance-org/Perl-Entrance-Textbook.git
Cloning into 'Perl-Entrance-Textbook'...
remote: Counting objects: 1459, done.
remote: Compressing objects: 100% (1092/1092), done.
remote: Total 1459 (delta 100), reused 1340 (delta 91)
Receiving objects: 100% (1459/1459), 5.34 MiB | 2.23 MiB/s, done.
Resolving deltas: 100% (100/100), done.
Checking connectivity... done.

$ ls
2015rookies
```

なお, 任意のリポジトリ名で`clone`したい場合, リポジトリのURLの後に希望するディレクトリ名を指定することができます.

```
$ git clone git@github.com:perl-entrance-org/Perl-Entrance-Textbook.git textbook
Cloning into 'Perl-Entrance-Textbook'...
remote: Counting objects: 1459, done.
remote: Compressing objects: 100% (1092/1092), done.
remote: Total 1459 (delta 100), reused 1340 (delta 91)
Receiving objects: 100% (1459/1459), 5.34 MiB | 2.23 MiB/s, done.
Resolving deltas: 100% (100/100), done.
Checking connectivity... done.

$ ls
textbook
```

`git clone`した場合も`git init`した場合と同じく, `.git`ディレクトリでリポジトリを管理していますので, 書き換えや削除をしないように気をつけましょう.

## コラム: 分散型と集中型

Gitは, バージョン管理システムの中でも｢分散型｣と呼ばれる仕組みに基づいています(対になるのが｢集中型｣, Subversionなどが該当します).
｢分散型｣と｢集中型｣はこのリポジトリの扱いが異なっていて, ｢分散型｣ではそれぞれのユーザ(開発者)がリポジトリを持っているのに対し, ｢集中型｣では1つのリポジトリを複数人で利用する, という仕組みになっています.

｢集中型｣の場合, コードに対する変更の履歴(コミット)は全て単一のリポジトリに保存されます.
そのため, 過去のコードの履歴などを取得する場合, 必ずリポジトリから取得しなければなりません.
また, リポジトリが設置されているサーバにネットワーク経由でアクセスできない場合, コミットを含む諸々の操作が出来なくなってしまいます.
更に, もしリポジトリが破損してしまった場合, 復元することは難しいので, 定期的にバックアップを取るなどの工夫も必要になってくるでしょう.

一方, ｢分散型｣の場合, リポジトリはそれぞれのユーザが保持しています.
分散型では, それぞれのユーザがリポジトリを持っているので, 集中型と異なり, ネットワークに繋がっていない状態でも(手元のリポジトリに対しては)作業を行うことができます.
また, ｢手元のリポジトリにはあるが, 他人のリポジトリやコアとなるリポジトリにはデータがない｣状態になるので, 心理的に試行錯誤もやりやすいです(集中型の場合, コミットすれば常にコードがリポジトリに保存されてしまうので, 他の開発者がそのコードを容易に見ることが出来てしまいます).
更に, 分散型の場合, リポジトリがユーザの数だけ存在しているので, もしもコアとなるリポジトリが破損してしまったとしても, 開発者が持っているリポジトリから新しくコアとなるリポジトリを用意する, ということもできます.
難点としては, リポジトリが複数存在するため, ｢自分のリポジトリに対する操作｣と｢他のリポジトリに対する操作｣が別々に用意されており, その分コマンド体系が集中型に比べて複雑になってしまっている, という点が挙げられるでしょう.

# コミット

バージョン管理システムでは, コードの変更を｢コミット｣という単位で保存します.
まずは, Gitでコードをコミットする方法について学びます.

`git init`をしたディレクトリで, 適当なファイル(例えば, `README.md`)を作成してから, `git commit`でコミットしてみます.

```
$ git init
$ touch README.md
$ git commit
On branch master

Initial commit

Untracked files:
        README.md

nothing added to commit but untracked files present
```

Gitでは, コードをコミットする前に, 変更したファイルを｢ステージ｣という領域に追加する必要があります.

ステージを使いこなすことで, 例えば｢AとBというファイルを更新したとき, 先にAを(ステージに追加して)コミットしてから, Bを(ステージに追加して)コミットする｣といった操作が出来るようになります.
一度に大量の変更を行う｢大きなコミット｣は, 後で見返した時に変更点が多く, コードを理解しにくくなります.
ステージを利用して, 極力｢1コミット = 1作業(変更)｣にするよう, 心がけましょう.

さて, 変更したファイルをステージに追加するためのコマンドは, `git add`です.

```
$ git add README.md
$ git status
On branch master

Initial commit

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

        new file:   README.md
```

`git add`した後, `git status`でリポジトリの状態を確認しています.
`git status`コマンドを利用すれば, 現在どのファイルが変更されていて, そのうちどのファイルがステージに追加されているかを確認することができます.

さて, `git add`コマンドで, `README.md`をステージに追加することができました.
なお, 複数のファイルを同時にステージに追加する場合, `git add -A`でリポジトリ内部の変更が行われた全ファイルをステージに追加することもできます.

ファイルをステージに追加した後は, `git commit`でステージに追加したファイルのみをコミットすることができます.

```
$ git commit -m 'commit'
[master (root-commit) 348094e] commit
 1 file changed, 1 insertion(+)
 create mode 100644 README.md
```

ここでは, `-m`オプションでコミットログ`commit`を指定した上でコミットを実行しています.

なお, `-m`オプションを指定していない場合, 自動的にエディタが開き, エディタを利用してコミットログを入力することが出来ます.
コミットログを記入してからエディタを閉じると, 自動的に入力したテキストをコミットログにして, コミットを実行してくれます.

# ログ

コミットのログは, `git log`コマンドで確認することができます.

```
commit 348094e4b518b8f6513d454c9971541bab647750
Author: papix <mail@papix.net>
Date:   Thu Apr 30 22:54:42 2015 +0900

    commit
```

`commit:`の後にある, `348094e4b518b8f6513d454c9971541bab647750`という文字列は｢ハッシュ｣と呼ばれる文字列で, コミットを一意に特定するためのユニークな文字列です.
例えば, Gitには, 任意のコミットの詳細を確認する`git show`コマンドがあります.
このハッシュは, `git show`コマンドで閲覧したいコミットを指定する為に利用することができます.

```
$ git show 348094e4b518b8f6513d454c9971541bab647750
```

なお, ハッシュは40桁ありますが, 前方一致で重なるものがなければ, それ以降は省略することができます.
大抵の場合, あるリポジトリに格納されたコミットについて, ハッシュの文字列の前方が4桁以上重複することは珍しいそうです.
そのため,

```
$ git show 3480
```

のような入力でも, 概ね問題なく実行することができます.

また, `git log`コマンドでは, グラフ形式でログを出力することも可能です.

```
$ git log --graph --date=short --decorate=short --pretty=format:'%Cgreen%h %Creset%cd %Cblue%cn %Cred%d %Creset%s'
    ... 前略 ...
* d0ae127 2015-07-15 nqounet  setup時は--deploymentを付けたほうが一人の環境に合わせ
やすい(master)
* 59c11a1 2015-07-08 Takuya Tsuchida  riji publish
*   daab264 2015-07-08 Takuya Tsuchida  Merge pull request #11 from tomcha/master
|\
| * 9569293 2015-07-08 tomcha  in福岡 住所修正
| * 049c5ac 2015-07-08 tomcha  in福岡の告知を更新
| * 970ab43 2015-07-08 tomcha  Update index.md
|/
* 34e67e6 2015-07-04 papix  すごく指摘されたので修正した
* db29913 2015-07-04 papix  oooooooooooooooops
    ... 後略 ...
```

これはPerl入学式の公式サイトのコードを管理しているリポジトリ内での実行例ですが, 後述するブランチの関係がグラフで可視化されており, 非常に確認しやすくなっています.

# ブランチ

Gitを含むバージョン管理システムには, ｢ブランチ｣と呼ばれる仕組みが存在します.
ブランチを利用することで, リポジトリの中で任意のコミットを枝分かれさせ, 併存することが出来るようになります.

Gitでは, デフォルトのブランチは`master`になっています.
現在のブランチとブランチの一覧は, `git branch`で確認することができます.

```
$ git branch
* master
```

## ブランチの使い方

例えば, `a`, `b`というコミットをした後に, 新しい機能を追加した`c`というコミットを`master`ブランチに作ったとしましょう.

```
master a ----> b ----> c
```

この後, `b`のコミットにバグが存在し, 修正することになったとしましょう.
そのまま`master`ブランチでコミットしてしまうと, `b`の機能修正と`c`という機能追加のコミット(`d`)が混ざってしまいます.

```
                       | 新しい機能の追加
                       v
master a ----> b ----> c ----> d
                               ^
                               | `b`の修正コミット
```

`b`の修正が`d`というコミットだけで終わればいいですが, この後`master`ブランチで機能追加のコミットと`b`の修正コミットを交互に繰り返してしまうと, 後で｢新しく機能追加した部分だけ確認したい!｣或いは｢`b`の修正コミットだけ確かめたい!｣となった際, 関係ないコミットを除外するのが非常に面倒になります.

そこで役立つのが, ブランチです.

Gitでは, 新しいブランチは`git branch <branch name>`で作成することが出来ます.

```
$ git branch new-branch
$ git branch
* master
  new-branch
```

更に, ブランチを作成する際, `git branch <branch name> <point>`のように指定することで, 任意のブランチやコミット(`<point>`)からブランチを作成することもできます.
今回の場合, `b`というコミットから, `b`のバグを修正する`fix-b`ブランチを生成するようにしてみましょう.

実はGitでは, 現在の先頭(最新)のコミットを`HEAD`で表せるようになっています.
そして, その1つ親のブランチを`HEAD^`, 更に1つ親のブランチを`HEAD^^`のように指定することができます.
現在の`master`ブランチの最新のコミットは`c`ですので, `b`はその1つ親のコミットに相当します.
そのため, `b`というコミットを指定するためには, (`git log`で`b`コミットのハッシュ値を確認して, ハッシュ値で指定する他に)`HEAD^`で指定することもできます.

```
$ git branch fix-b HEAD^
$ git branch
  fix-b
* master
```

これで, `b`というコミットから`fix-b`というブランチを作ることができました.

```
               | fix-bブランチ
               v
master a ----> b ----> c
```

ブランチ間の移動については, `git checkout <branch>`で実現できます.

```
$ git checkout fix-b
$ git branch
* fix-b
  master
```

現在, `fix-b`ブランチの最新のコミットは`b`であり, `c`で行った機能追加に関するコードは含まれていません.
ここから, `fix-b`ブランチで`b`ブランチに存在したバグを修正し, コミットをすると, `fix-b`ブランチと`master`ブランチに関するコミットは次のような形になります.

```
fix-b          +-----> d
               |
master a ----> b ----> c
```

このように, トピック(例えば, ｢A機能の追加｣とか, ｢B機能のバグの修正｣とか...)単位でブランチを作る(｢ブランチを切る｣, とも言います)ことで,  同時に複数の作業を行っても, それらのコードが混ざることなく管理することが出来るようになります.

## ブランチに関連する操作

ブランチの操作を学ぶにあたって, 実際にGitのリポジトリを作成して操作を体験してみるのも良いですが, 今回は[Learn Git Branching](http://k.swd.cc/learnGitBranching-ja/)を利用して学ぶこととします.

このサービスは, Gitの次のコマンドについて, 実際に操作しながら学ぶことができます.

- commit
- branch
- checkout
- cherry-pick
- reset
- revert
- rebase
- merge

### 練習問題

Learn Git Branchingが提供する課題のうち, 少なくとも以下のカリキュラムについては挑戦しておくことを推奨します.

- ｢まずはここから｣の'1' 〜 '4'
- ｢次のレベルに進もう｣の'1' 〜 '4'
- ｢Rebaseをモノにする｣の'1' 〜 '2'

なお, ｢様々なTips｣及び｢応用編｣については, 余裕があればチャレンジしてみるとよいでしょう.

#### ヒント

- `levels`でレベル選択画面へ遷移することができます
- `reset`で作業を初期化することができます
- `clear`でコマンドのログを消去することができます(これまでの作業内容は消えません)

# 複数人でGitを使う

次に, 複数人でGitを使う際に必要となる, 次のコマンドについて解説を行います.

- fetch
- pull
- push

## `origin`なリポジトリ

`git clone`でリポジトリを取得した場合, その取得元であるリポジトリ(例えば, 先程`clone`で取得した｢2015rookies｣リポジトリの場合, BitBucketでホスティングされているGitリポジトリ)が`origin`として登録されます.

分散型バージョン管理システムであるGitでは, BitBucketがホスティングしているリポジトリだけでなく, 他の開発者が持っているリポジトリともやり取りすることが出来ます.
しかしながら, 基本的にはリポジトリ間の操作は, `origin`として指定されている"コアとなるリポジトリ"(= GitHubやBitBucketといったリポジトリホスティングサービスや, Gitサーバに設置されているリポジトリ)とのやり取りだけで事足ります.
例えば, 他の開発者が自分の開発したコミットを取得したい場合, 直接自分のリポジトリから取得するのではなく, まず自分が`origin`で指定されたリポジトリにコミットを適用し, 他の開発者は`origin`のリポジトリから自分のコミットを取得すれば良いからです.

そのため, Gitではコアとなるリポジトリを`origin`として登録することができ, `fetch`や`pull`, `push`といったリポジトリ間の操作を行うコマンドについて, ｢デフォルトの操作対象｣として使うことができます.
今回は, コマンドによる操作対象のリポジトリ(これを｢リモートリポジトリ｣と呼びます. 一方, 手元にあるリポジトリは｢ローカルリポジトリ｣と呼びます)は, 基本的には`origin`で指定されているリポジトリであるとして, 解説を進めていきます.

## `fetch`

リモートリポジトリの変更点を取得するコマンドです.
例えば, ローカルリポジトリのmasterブランチより1つ進んだコミットがリモートリポジトリにある場合, `git fetch`コマンドを実行すると, 次のような出力が得られます.

```
$ git fetch
remote: Counting objects: 3, done.
remote: Total 3 (delta 0), reused 0 (delta 0)
Unpacking objects: 100% (3/3), done.
From github.com:your-name/your-repository.git
   d15cddc..d840590  master     -> origin/master
```

これによって, リモートリポジトリの`master`ブランチの内容が, ローカルリポジトリの`origin/master`というブランチに適用されました.
このブランチの内容は,

```
$ git checkout origin/master
```

でブランチを切り替え, 確認することができます.

ここからもし, `origin/master`ブランチを, ローカルリポジトリの`master`ブランチにマージしたい場合,

```
$ git checkout master
$ git merge origin/master
```

で, マージすることができます.

## `pull`

一言で言えば, `fetch`と`merge`を同時に行うコマンドです.

例えば, ローカルリポジトリの`master`ブランチで`git pull`を実行した場合, リモートリポジトリの`master`ブランチと差分がない場合は,

```
$ git pull
Already up-to-date.
```

と表示されます. 一方, 差分がある場合は

```
remote: Counting objects: 3, done.
remote: Total 3 (delta 0), reused 0 (delta 0)
Unpacking objects: 100% (3/3), done.
From github.com:your-name/your-repository.git
   d840590..e14eed7  master     -> origin/master
Updating d840590..e14eed7
Fast-forward
 README.md | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

このように, 自動的にリモートリポジトリの`master`ブランチの内容を, ローカルリポジトリにマージしてくれます.

なお, この際`--rebase`オプションを指定すると, `merge`ではなく`rebase`を使ってマージすることもできます.
必要に応じて使い分けるようにしましょう.

### `pull`の際のコンフリクトとその解消

リモートリポジトリから`pull`をする場合, 場合によってはコンフリクト(衝突)が発生する場合があります.
例えばこのように, リモートリポジトリの`master`から1つ進んだコミットをローカルリポジトリで作ったとしましょう(矢印がコミットの親子関係を示していて, `a`や`b`といったアルファベットが1つのコミットだと思ってください. 更に, `m`がローカルの`master`ブランチの最新のコミット, `M`がリモートの`master`ブランチの最新のコミットであるとします).

```
                m
                v
        +-----> c
        |
a ----> b
        ^
        M
```

しかし, その作業をしている間, 別のユーザがリモートリポジトリの`master`から別のコミットを作って, リモートリポジトリに`push`していたとします(下図における`x`のコミット).

```
                m
                v
        +-----> c
        |
a ----> b ----> x
                ^
                M <- 別のユーザがcommitして, pushした!
```

この状態で`git pull`をした場合, Gitは可能な限り`c`と`x`のコミットをマージしようとします.
しかし, 自動的なマージに失敗した場合, 次のような出力を表示します.

```
$ git pull
Auto-merging README.md
CONFLICT (content): Merge conflict in README.md
Automatic merge failed; fix conflicts and then commit the result.
```

この場合, ユーザが手動でコンフリクトを解消しなければなりません.
出力には, `README.md`でコンフリクトが発生している, と表示されているので, 確認してみます.

```
<<<<<<< HEAD
# こんにちはGit!!!

こんにちはこんにちは!
=======
# こんにちはGit!!!!!!!!
>>>>>>> 5dd36fe276784f9d18287279d91dfc19000c01fd
```

`<<<<<<< HEAD`から`=======`で囲まれた部分がローカルリポジトリの内容で, `=======`から`>>>>>>> 5dd36fe276784f9d18287279d91dfc19000c01fd`で囲まれた部分がリモートリポジトリの内容を指しています.
Gitは, この部分についてどちらの内容を採択すればいいか判断できなかった, ということです.

コンフリクトは1つのファイルで複数生じる場合もありますし, もしリポジトリに複数のファイルが存在するのであれば, 同時に複数のファイルで生じることもあります.
ただ, コンフリクトが起きているファイルについては, `git pull`をした時に全て確認することが出来ますし, `git status`コマンドで確認することもできます.

```
$ git status
On branch master
Your branch and 'origin/master' have diverged,
and have 1 and 1 different commit each, respectively.
  (use "git pull" to merge the remote branch into yours)
You have unmerged paths.
  (fix conflicts and run "git commit")

Unmerged paths:
  (use "git add <file>..." to mark resolution)

        both modified:   README.md

no changes added to commit (use "git add" and/or "git commit -a")
```

それでは, `README.md`のコンフリクトを解消しましょう.

```
# こんにちはGit!!!!!!!!

こんにちはこんにちは!
```

`README.md`を正しい(期待する)内容に書き換え, `git add .`してから`git commit`します.

```
$ git add .
$ git commit
```

すると, 次のような文面が入力された状態でエディタが開くはずです.

```
Merge branch 'master' of github.com:your-name/your-repository.git

# Conflicts:
#	README.md
#
# It looks like you may be committing a merge.
# If this is not correct, please remove the file
#	.git/MERGE_HEAD
# and try again.


# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
# On branch master
# Your branch and 'origin/master' have diverged,
# and have 1 and 1 different commit each, respectively.
#   (use "git pull" to merge the remote branch into yours)
#
# All conflicts fixed but you are still merging.
#
# Changes to be committed:
#	modified:   README.md
#
```

基本的に, このままの文面でコミットして構いません.
このコミットは, 異なるリポジトリ間のコミットをマージしているので, ｢マージコミット｣と呼ぶこともあります.

```
$ git commit
[master ea69b8e] Merge branch 'master' of github.com:your-name/your-repository.git
```

このとき, ローカル/リモートの`master`ブランチの最新コミットは, 次のようになります(`d`が, ローカルリポジトリの`c`とリモートリポジトリの`x`の内容をマージした, マージコミットです).

```
        +-----> c ------|
        |               v
a ----> b ----> x ----> d
                ^       ^
                M       m
```

なお, ここで紹介したコンフリクトの解決方法は, ローカルリポジトリにあるブランチ同士の`merge`でコンフリクトが発生した場合と同じです(前述の通り, `git pull`は`git fetch`と`git merge`を同時に実行するコマンドだからです).
もし, ローカルリポジトリにあるブランチをマージしてコンフリクトした場合も, `git pull`でコンフリクトが生じた時と同じように, 慌てず対処していきましょう.

## `push`

`fetch`や`pull`はリモートリポジトリの内容を取得するコマンドでしたが, `push`は逆にリモートリポジトリへ更新を送り込むコマンドです.
例えばローカルリポジトリの`master`ブランチで`git push`を実行したとします.
もし, ローカルリポジトリの`master`ブランチとリモートリポジトリの`master`ブランチに差分がないのであれば,

```
$ git push
Everything up-to-date
```

このような出力になります.

一方, ローカルリポジトリの`master`ブランチがリモートリポジトリの`master`ブランチより進んでいる場合,

```
$ git push
Counting objects: 3, done.
Writing objects: 100% (3/3), 239 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To github.com:your-name/your-repository.git
   e14eed7..5dd36fe  master -> master
```

このように, ローカルリポジトリの`master`ブランチをリモートリポジトリの`master`ブランチに適用することができます.

### `push`の際のコンフリクトとその解消

`git pull`と同様, `git push`の際もコンフリクトが起こる場合があります.

```
                m
                v
        +-----> c
        |
a ----> b ----> x
                ^
                M <- 別のユーザがcommitして, pushした!
```

このような状態で, ローカルの`master`ブランチをリモートに`push`しようとした場合, コンフリクトが発生し, 次のようなエラーが表示されます.

```
$ git push
To github.com:your-name/your-repository.git
 ! [rejected]        master -> master (fetch first)
error: failed to push some refs to 'github.com:your-name/your-repository.git'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. This is usually caused by another repository pushing
hint: to the same ref. You may want to first integrate the remote changes
hint: (e.g., 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

解決するためには, エラー文にも書かれている通り, `git pull`でリモートリポジトリの更新を取り込んでから`push`してやればOKです.
ここから先の`git pull`の手順については, 前述の｢`pull`の際のコンフリクトとその解消｣と同様ですので省略します.

`git pull`でリモートリポジトリの内容をマージすることが出来れば, 次のように`git push`が無事成功します.

```
$ git push
Counting objects: 6, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (6/6), 541 bytes | 0 bytes/s, done.
Total 6 (delta 0), reused 0 (delta 0)
To github.com:your-name/your-repository.git
   5dd36fe..ea69b8e  master -> master
```

ローカル/リモートの`master`ブランチの最新コミットは, 次のようになっているはずです.

```
        +-----> c ------|
        |               v
a ----> b ----> x ----> d
                        ^
                       m/M
```

### コラム: `push -f`の恐怖

`push`する際, コンフリクトを無視して無理やり`push`を行う`-f`オプション(`force`の`f`)がありますが, 特にチームで開発を行う際には必ず使わないようにしましょう.
もし`push -f`で無理やり`push`してしまうと, ローカル/リモートリポジトリ間のコミットの関係が崩壊し, 解消が非常に面倒なことになります.

このため, 一部界隈では`push -f`は｢核爆弾｣と呼ばれ, 恐れられて(嫌われて)いたりします.

# 練習問題

BitBucketの個人アカウントで`git-training`というリポジトリを作成し, 手元のマシンへ`clone`して, ここまで学んできたGitの使い方を振り返ってみよう.

- コミット
- ブランチの作成
- ブランチのマージ
- コードの`push`
- コードの`pull`
- コンフリクトが生じる状態で`pull`をして, コンフリクトを解消してから`push`

なお, それぞれの課題を達成するために実行したコマンドの履歴については, 課題提出用リポジトリの指定されたディレクトリに`git-training.md`というファイルを用意して, そこに書くようにしよう.
また気がついた事や苦戦したこと, メモしておきたい事があれば, これもファイル内に記載しておくようにしよう.

## ヒント

```
$ git clone github.com:your-name/your-repository.git user1
$ git clone github.com:your-name/your-repository.git user2
```

のように, 別々のディレクトリでリポジトリを`clone`すれば, コンフリクト状態の再現が簡単にできます(`user1`ディレクトリでコミットし, リモートリポジトリに`push`した後, `user2`ディレクトリでコミットし, リモートリポジトリに`push`すれば, ｢`pull`の際のコンフリクトとその解消｣で紹介したような状況を作り出すことができます).
