# Ansible

先程, Vagrantの練習問題で簡単なWebアプリケーションの環境構築をVagrant上に｢手動で｣行いました.
練習問題では, 構築する台数が1台だったので何とかなりましたが, もし同じ環境を10台作って欲しい, と言われたらどう思いますか?
手動で行えば, 10倍の時間がかかりますし, その間にコマンドの入力ミス(オペレーションミス, オペミス)など起こしてしまう可能性も十分あるでしょう.

こんな時に役立つのが, 環境構築を自動的に行ってくれる｢プロビジョニングツール｣と呼ばれるツール類です(｢構成管理ツール｣と呼ぶこともあります).
プロビジョニングツールを使えば, 構築手順(構築手順を記述する方法は, プロビジョニングツールによって異なっています)をコードとして書いてしまいさえすえば, それを複数のサーバに適用することが出来るようになります.
エンジニアは, プロビジョニングツールに対して環境構築を行うように命令するだけで済むので, 手動で構築する際に起こりえるオペレーションミスも防ぎやすいです.

近年, サービスのインフラストラクチャをコードで表現/実現する｢Infrastructure as Code｣が流行していますが, プロビジョニングツールはその中の｢環境構築｣という位置領域を担うツールと言うことができます.

後述しますが, ｢Infrastructure as Code｣の流行によって, 近年様々なプロビジョニングツールが産まれています.
そのため, 各社/各チームにとって適切なプロビジョニングツールを選択していく必要性があるでしょう.
今回は, Python製のプロビジョニングツールであるAnsible(アンシブル)を使って, プロビジョニングツールとInfrastructure as Codeを体験していきます.

## 代表的なプロビジョニングツール

|名前     |特徴                                                                                                               |
|:--------|:------------------------------------------------------------------------------------------------------------------|
| Ansible | 今回紹介するプロビジョニングツール. Python製. 構築手順はYAMLで記述する. 他のツールと比べると利用までの障壁は低い. |
| Chef    | Ruby製. 構築手順は｢レシピ｣と呼ばれ, Rubyで記述する. プロビジョニングツールとしては多機能.                         |
| Fabric  | Python製. 構築手順はPythonで記述する. Ansibleよりもプログラマブルに構築手順を記述できる. Chefよりはシンプル.      |
| Itamae  | Ruby製. Rubyで構築手順を記述でき, Chefよりもシンプルなプロビジョニングツール.                                     |

この他, プロビジョニングツールで構築した環境に, アプリケーションをデプロイするためのデプロイツールと呼ばれるツールもあります.
例えば, Capistrano(Ruby)やCinnamon(Perl)などが有名ですが, これらのデプロイツールとプロビジョニングツールの境界はかなり曖昧です.
実際に, プロビジョニングツールのAnsibleでプロビジョニングだけでなくデプロイも行う, という事例もあります.

# Ansibleのインストール

Homebrewからインストールすることができます.

```
$ brew install ansible
```

Python製のツールなので, `easy_install`や`pip`でもインストールできます.
ただ, Mac OS XであればHomebrewでインストールするのが一番手っ取り早いですし, 安心でしょう.

# Ansible入門

## 仮想マシンの準備

それでは, まずAnsibleで環境構築を行う対象となる仮想マシンを作っていきましょう.

```
$ vagrant init ubuntu/trusty64
$ vagrant up
```

これまで学んできたように, これで仮想マシンの準備は完了です.

Vagrantを利用するのであれば, `ssh`コマンド経由でVagrantに接続出来るようにしておくと楽になります.
`ssh-config`で, この仮想マシンに接続するために必要な設定を出力し, `.ssh/config`に追記しましょう.

```
$ vagrant ssh-config --host vagrant >> ~/.ssh/config
```

接続出来るか確認してみましょう.

```
$ ssh vagrant
Welcome to Ubuntu 14.04.2 LTS (GNU/Linux 3.13.0-48-generic x86_64)
    ... 中略 ...
vagrant@vagrant-ubuntu-trusty-64:~$
```

問題ないですね.

## `hosts`とPlaybookの準備

Ansibleで環境構築を行うためには, 予めいくつかの準備が必要となります.

まずは`hosts`ファイルです.
このファイルには, 環境構築を行う対象となるホストを, IPないしホスト名で記述していきます.
ファイルはini形式で記載することができ, ホストのロール(役割)に応じてホストの集合をグループとして定義することもできます.

例えば, アプリケーションを動かすサーバが3台(`app1`〜`app3`), データベース用のサーバが2台(`db1`〜`db2`)があるのであれば, 次のように書くと良いでしょう.

```
[app]
app1
app2
app3

[db]
db1
db2
```

今回は, Amon2::Liteで作成した簡単なアプリケーションを動かすサーバが1台だけなので, 次のように設定しておきます.

```
[app]
vagrant
```

### Playbook

Playbookは, Ansibleにおける｢構築手順書｣そのものです.
先程設定した`hosts`に記載したホストやグループに, Playbookを適用することで, Ansibleは環境構築を行います.

Ansibleでは, PlaybookはYAML形式で記述します.
まずは試しに, 仮想マシンの`/tmp`ディレクトリに, ｢Hello, Ansible!｣と書かれた｢ansible.txt｣というファイルを設置するようなPlaybookを書いてみることとします.

Playbookの書き方についての詳細は後述します.
まずはとりあえず, このPlaybookをVagrantで作った仮想マシンに適用するところまでをやってみましょう.

```yaml
- hosts: app
  sudo: yes
  tasks:
    - name: Hello, Ansible!
      shell: echo 'Hello, Ansible!' > /tmp/ansible.txt
```

これを, `sample-playbook.yml`というファイル名で保存しておきます.
`hosts`ファイルとPlaybookファイルの設置が終わっていれば, カレントディレクトリは次のようになっているでしょう.

```
$ tree
.
├── Vagrantfile
├── hosts
└── sample-playbook.yml

0 directories, 3 files
```

## Playbookの実行

設定した`hosts`ファイルを使って, Playbook(`sample-playbook.yml`)を仮想マシンに適用してみましょう.

```
$ ansible-playbook -i hosts sample-playbook.yml
```

Playbookの適用は, `ansible-playbook`コマンドを利用します.
`-i`で`hosts`ファイルを指定し, その後にPlaybookのファイル名を指定します.

```
$ ansible-playbook -i hosts sample-playbook.yml

PLAY [app] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [vagrant]

TASK: [Hello, Ansible!] *******************************************************
changed: [vagrant]

PLAY RECAP ********************************************************************
vagrant                    : ok=2    changed=1    unreachable=0    failed=0
```

最後に, 実行結果が出力されています.

```
PLAY RECAP ********************************************************************
vagrant                    : ok=2    changed=1    unreachable=0    failed=0
```

タスクを処理するごとに, Playbookの処理が成功したら`ok`, 変更を伴う処理が成功すれば`changed`, ホストに接続できなければ`unreachable`, そして失敗した場合は`failed`が, それぞれ1ずつ増えていきます.
そのため, `unreachable`と`failed`が0であれば, Playbookの適用は成功した, ということになります.

それでは, 本当に仮想マシン上にファイルが生成されたのか, 仮想マシンに接続して確認してみましょう.

```
$ ssh vagrant
Welcome to Ubuntu 14.04.2 LTS (GNU/Linux 3.13.0-48-generic x86_64)
    ... 中略 ...
vagrant@vagrant-ubuntu-trusty-64:~$ cat /tmp/ansible.txt
Hello, Ansible!
```

...確かに, 生成されていますね!

ここまで, 簡単なPlaybookを通して, Ansibleを使った環境構築の自動化を体験してきました.
今回は, ただファイルを設置する簡単なPlaybookだったので, そこまで大きいメリットを感じなかったかもしれません.
ですが, 実際にアプリケーションを動かすための環境を構築するためには, これまで体験してきたように様々なツールやミドルウェアをホストに導入し, その設定ファイルをホスト内に適切に設置していかなければなりません.

Ansibleのようなプロビジョニングツールが真価を発揮するのは, まさにこのような場面です.
そしてこのような場面は, 私達がエンジニアとして仕事をしている中では, もはや日常茶飯事と言うことが出来るでしょう.
そういった意味で, Ansibleのようなプロビジョニングツールやデプロイツールの知識は, 恐らくこれからのエンジニアにとって必須の知識になってくると思います.
なので, まずはこのカリキュラムで, しっかりAnsibleに慣れてしまいましょう.

...それでは次に, Ansibleの｢キモ｣となる, Playbookの書き方について, さらに細かく見ていきます.

## Playbookの書き方

先程述べたように, AnsibleのPlaybookは次のようなYAML形式で記述します.

```yaml
- hosts: app
  sudo: yes
  tasks:
    - name: Hello, Ansible!
      shell: echo 'Hello, Ansible!' > /tmp/ansible.txt
```

Playbookで設定出来る項目のうち, いくつかを紹介します.

### hosts

`ansible-playbook`コマンドに対し`-i`オプションで指定したホストファイルのうち, このPlaybookを適用する(実行する)ホストやグループを指定します.
カンマ区切りやYAMLのリスト指定で, 同時に複数のホストやグループを指定できます.

### sudo

`sudo`が`yes`の場合, このPlaybookはホスト側では`sudo`を使って実行されます.
デフォルトでは`root`として実行しますが, `sudo_user`を指定すれば任意のユーザとして実行することもできます.

### gather_facts

`yes`の場合, あるいは無指定の場合, Playbookを実行するを実行する前に対象となるホストの情報を取得します(取得できる情報については, ｢Ansible Tutorial｣の[対象ホストの情報を取得(GATHERING FACTS)](http://yteraoka.github.io/ansible-tutorial/#gather-facts)などを参考にしてください).
取得した情報は, Playbook中で参照することも出来るので, 例えば｢OSや, そのバージョンに応じて, 実行する処理を変更する｣などといった事も可能です.

ホスト情報の取得には少し時間がかかりますので, ホスト情報を利用しない場合は`gather_facts`を`no`にすることで, Playbookを実行する時間を少し減らすことが出来ます.

### tasks

ホストで実行したい処理を記述することができます.

```yaml
- tasks:
  - name: Hello, Ansible!
    shell: echo 'Hello, Ansible!' > /tmp/ansible1.txt
```

タスクは, Ansibleが提供する｢モジュール｣を使って記述します(英語ですが, モジュールの一覧は[こちら](http://docs.ansible.com/list_of_all_modules.html)にあります).
ここでは, ホスト側で任意のコマンドを実行する[shell](http://docs.ansible.com/shell_module.html)というモジュールを実行しています.

タスクを指定する際は, 別途`name`でその解説を記入することが出来ます.
`name`を指定した場合, `ansible-playbook`でPlaybookを実行した際に, 次のように表示されます.

```
TASK: [Hello, Ansible!] *******************************************************
```

実行結果の可読性が向上するので, 是非指定するようにしておきましょう.

なお, Playbookで利用できるモジュールのうち, 代表的なものについては, この後でいくつか紹介します.

### vars_files

Playbook中で利用できる変数を指定できます.
例えば, `vars.yml`というファイルに,

```
word: 'Hello, Ansible!'
```

のように指定しておけば, Playbookの中から`Hello, Ansible!`という文字列を, `{{ word }}`として参照することができます.

```yaml
- hosts: app
  sudo: yes
  vars_files:
    - vars.yml
  tasks:
    - name: Hello, Ansible!
      shell: echo '{{ word }}' > /tmp/ansible.txt
```

例えばこのようなPlaybookを実行した場合, ホスト側の`/tmp/ansible.txt`には, ｢{{ word }}｣ではなく｢word｣を`vars.yml`で展開した｢Hello, Ansible!｣が書き込まれます.
変数は, ここで紹介している`vars_files`で指定したものの他に, デフォルトで利用できる[マジック変数](http://qiita.com/h2suzuki/items/15609e0de4a2402803e9)というものもあります.

### コラム: vars_filesの暗号化

Playbookで`vars_files`で指定する, 変数が書かれたファイルは暗号化することもできます.
例えば, `vars.yml`というファイルを暗号化したいのであれば, `ansible-vault encrypt vars.yml`というコマンドを実行します.

```
$ ansible-vault encrypt vars.yml
Vault password: [復号用パスワードを入力]
Confirm Vault password: [もう一度同じ復号用パスワードを入力]
Encryption successful
```

すると, 次のように`vars.yml`の中身が暗号化されます.

```
cat vars.yml
$ANSIBLE_VAULT;1.1;AES256
39333631313465373036653437346363633330396232313564303239656564653038346466353165
3161633864666163303131346438643734653861393265330a383635353131346538353366346435
66666536663034306336656433633833323336313466383061376264333463373562633333303337
3437643166663233340a653333336637316433613036646537623539646635646464393137633933
30396166376635386136336661383136333037373933643364383435326235366232
```

このファイルを利用してPlaybookを実行する場合, `ansible-playbook`コマンドに対して`--ask-vault-pass`オプションを与え, 復号用パスワードを入力しなければなりません.

```
$ ansible-playbook --ask-vault-pass -i hosts sample-playbook.yml
Vault password: [復号用パスワードを入力]

PLAY [app] ********************************************************************
    ... 後略 ...
```

もし, 復号用パスワードが異なっている場合, 次のようなエラーが出てPlaybookの実行は終了します.

```
$ ansible-playbook --ask-vault-pass -i hosts sample-playbook.yml
Vault password: [復号用パスワードを入力]
ERROR: Decryption failed
```

暗号化したファイルは, `ansible-vault decrypt`で復号することが出来ます.

```
$ ansible-vault decrypt vars.yml
Vault password: [復号用パスワードを入力]
Decryption successful
```

しかし, ファイルの中身を変更する度に`decrypt`と`encrypt`を繰り返すのは手間ですし, `encrypt`を忘れて平文のままcommit/pushしてしまうと, このファイルを暗号化する意味がなくなってしまいます.

そこで, 一度暗号化した後は, `ansible-vault edit`コマンドを使う事をおすすめします.
`ansible-vault edit vars.yml`のように実行すると, 復号用パスワードを入力でき, パスワードが正しい場合はエディタ(環境変数`$EDITOR`で指定しているエディタ)が自動的に開いて対象ファイル(今回の場合, `vars.yml`)を復号した状態で編集できるようになります.

エディタを閉じると, 自動的に同じパスワードで暗号化してくれるので, 編集後の暗号化を忘れる心配がなくなります.

## 代表的なモジュール

ここでは, Ansibleのモジュールのうち, Playbookを用意する際に頻繁に利用するモジュールを紹介します.
前述していますが, Ansibleで利用できるモジュールの一覧は[こちら](http://docs.ansible.com/list_of_all_modules.html)にあります.
ここで紹介しているモジュールの他にもたくさんのモジュールが用意されているので, 一度目を通してみることをおすすめします.

### [yum](http://docs.ansible.com/yum_module.html) / [apt](http://docs.ansible.com/apt_module.html)

```yaml
- apt: name=foo
- yum: name=foo
```

`yum`コマンドや`apt-get`コマンドを使って, `name`で指定したパッケージをインストールします.

### [shell](http://docs.ansible.com/shell_module.html)

```yaml
- shell: echo 'sample' >> log.txt
```

指定したコマンドを, ホスト側で実行します.

### [file](http://docs.ansible.com/file_module.html)

```yaml
- file: path=/tmp/sample state=directory
- file: path=/tmp/sample/log state=touch
```

ファイルやディレクトリの操作をします. `state`が｢directory｣の場合, `path`で指定したディレクトリを作成し, `state`が｢touch｣の場合は空のファイルを作成することができます.

### [service](http://docs.ansible.com/service_module.html)

```yaml
- service: name=nginx state=started
- service: name=nginx state=started enabled=yes
```

`name`で指定したパッケージを操作することができます.
`state`で, パッケージの動作状況を変更することができ, `started`で起動, `stopped`で停止, `restarted`で再起動することができます.

また, `enabled`でブート時に自動起動するかを設定することができます(`enabled=yes`でブート時に自動的に起動するようになる).

### [git](http://docs.ansible.com/git_module.html)

```yaml
- git: repo=https://github.com/gotandaGM/text-dev-env.git dest=/tmp/dev-env
```

`repo`で指定したリポジトリを, ホストの`dest`で指定したディレクトリにチェックアウトします.

### [copy](http://docs.ansible.com/copy_module.html)

```yaml
- copy: src=file.txt dest=/tmp/file.txt
```

ローカルマシンにある, `src`で指定したファイルを, ホストの`dest`で指定したパスにコピーすることができます.
ライブラリやアプリケーションの設定ファイルの設置などに利用することができます.

### [template](http://docs.ansible.com/template_module.html)

```yaml
- template: src=file.txt dest=/tmp/file.txt
```

ローカルマシンにある, `src`で指定したテンプレートを元にファイルを生成し, ホストの`dest`で指定したパスにコピーすることができます.
`copy`モジュールとの違いは, `src`で指定したファイルをテンプレートとして, 動的にファイルを生成できる点です.

例えば, `vars_files`で`type`という変数で`Ansible`を指定している場合, `file.txt`の中身が

```
Hello, {{ type }}!
```

このようになっているのであれば, `dest`で指定した`/tmp/file.txt`の中身は, 次のようになります.

```
Hello, Ansible!
```

templateモジュールはPythonのJinja2というテンプレートエンジン(日本語マニュアルは[こちら](http://ymotongpoo.appspot.com/jinja2_ja/index.html))を利用しているので, Ninja2の構文に従って条件分岐や繰り返しなども実現することができます.

## コラム: Playbookのシンタックスチェック

AnsibleのPlaybookのシンタックスチェックは, `ansible-playbook`に対して`--syntax-check`オプションを指定すればOKです.
シンタックスエラーがなければ, 次のような出力になります.

```
$ ansible-playbook -i hosts sample-playbook.yml --syntax-check

playbook: sample-playbook.yml
```

一方, シンタックスエラーが発生した場合はその詳細が出力されます.
例えば, Playbookの内容が次のようになっている場合...

```yaml
- hosts: app
  sudo: yes
  tasks:
    - name: Hello, Ansible!
    shell: echo 'Hello, Ansible!' > /tmp/ansible.txt
```

シンタックスチェックの結果は, 次の通りになります.

```
$ ansible-playbook -i hosts sample-playbook.yml --syntax-check
ERROR: Syntax Error while loading YAML script, sample-playbook.yml
Note: The error may actually appear before this position: line 5, column 5

    - name: Hello, Ansible!
    shell: echo 'Hello, Ansible!' > /tmp/ansible.txt
    ^
```

`name`と`shell`の行頭を揃えていないのでシンタックスエラーが生じていることがわかります.

## 練習問題

Vagrantの｢練習問題｣において手動で行った環境構築を, Ansibleを利用して自動化してみましょう(Amon2::Liteアプリケーションの準備や, Vagrantfileの用意と起動などは手動で行ってください).
`build.yml`というファイルを用意し, ここにPlaybookを書いていきましょう.

```yaml
- hosts: app
  sudo: yes
  gather_facts: no
  tasks:
    - name: Install git
      apt: name=git
    - name: Clone xbuild
      git: repo=https://github.com/tagomoris/xbuild.git dest=/tmp/xbuild
    - name: Install Perl (and App::cpanminus / Carton)
      shell: /tmp/xbuild/perl-install 5.20.1 /opt/perl-5.20
```

ヒントとして, xbuildを利用してPerl 5.20.1をインストールするまでのPlaybookを用意しておきました.
このPlaybookをベースに, `carton install`の実行やNginx, Supervisorのインストールと初期設定などをAnsibleで自動的に実行できるようにしてみましょう.

## 複雑なPlaybook

先程の練習問題では, 環境構築に必要なタスクを全て`build.yml`に記述しました.
しかし, 1つのファイルに全てのタスクを書いてしまうと, 1ファイルにおける記述量が大量になってしまいます.
そのため, Ansibleではタスクを｢ロール｣という単位で分割することが出来るようになっています.

ここでは, 先程の練習問題で書いたPlaybookのタスクのうち, Perlのインストールに関連する部分を, `perl`というロールに切り分けてみます.
まずは, `build.yml`側の変更です.

```yaml
- hosts: app
  sudo: yes
  gather_facts: no
  roles:
    - perl
```

`tasks`の変わりに`roles`という設定が入り, これから作る`perl`というロールを指定しています.

一方, 具体的なタスクについては, `/roles/[role名]/tasks/main.yml`に記載することになります.
今回の場合, ロールの名前は`perl`なので, `roles/perl/tasks/main.yml`に転記しましょう.

```
vi roles/perl/tasks/main.yml
- name: Install git
  apt: name=git
- name: Clone xbuild
  git: repo=https://github.com/tagomoris/xbuild.git dest=/tmp/xbuild
- name: Install Perl (and App::cpanminus and Carton)
  shell: /tmp/xbuild/perl-install 5.20.1 /opt/perl-5.20
```

今回の場合, ロールへの切り出しはこれで完了です.
`tree`コマンドでディレクトリ構成を見てみると, 次のようになっているはずです.

```
.
├── Vagrantfile
├── hosts
├── roles
│   └── perl
│       └── tasks
│           └── main.yml
└── sample-playbook.yml
```

では, `ansible-playbook`コマンドでPlaybookを(再度)実行してみましょう.

```
$ ansible-playbook -i hosts build.yml

PLAY [app] ********************************************************************

TASK: [perl | Install git] ****************************************************
ok: [vagrant]

TASK: [perl | Clone xbuild] ***************************************************
ok: [vagrant]

TASK: [perl | Install Perl (and App::cpanminus and Carton)] *******************
changed: [vagrant]

PLAY RECAP ********************************************************************
vagrant                    : ok=3    changed=1    unreachable=0    failed=0
```

ロールに切りだす前と同じ処理が, 問題なく実行することができましたね.

### ロールのディレクトリ構成

先程, ロールではタスクを`/roles/[role名]/tasks/main.yml`に配置する, と説明しました.
Ansibleのロールでは, このようなディレクトリやファイル構成が重要で, `tasks`以外にも`vars`や`meta`, `templates`, `files`などを指定することが出来ます.
これらの設定によって, 複雑な環境構築手順もシンプルに記述することが出来るようになります.

#### `/roles/[role名]/tasks/main.yml`

前述のように, ロールとして実行するタスクを記載するファイルです.

#### `/roles/[role名]/vars/main.yml`

`/roles/[role名]/tasks/main.yml`で利用する変数を定義出来るファイルです.
変数の定義方法については, `vars_files`で指定する場合と同様です.

#### `/roles/[role名]/meta/main.yml`

ロールの依存関係を定義出来ます.
例えば, 以下のような記述をした場合, このロールを実行する前に依存関係のある`foobar`というロールを実行してから, 現在のロールを実行します.

Nginxの設定ファイルを配置するロールにおいて, Nginxをインストールするロールを依存として指定しておく, といった場合に利用できます.

```yaml
dependencies:
  - { role: foobar }
```

#### `/roles/[role名]/files/main.yml`

`/roles/[role名]/tasks/main.yml`内で`copy`モジュールを利用する際に使えるファイルを配置するディレクトリです.
例えば, このディレクトリ内に`foobar.txt`というファイルがあったとすると, そのファイルは`copy`モジュールで`src=foobar.txt`と指定することで利用することができます.

#### `/roles/[role名]/templates/main.yml`

`/roles/[role名]/tasks/main.yml`内で`template`モジュールを利用する際に使えるテンプレートを配置するディレクトリです.
例えば, このディレクトリ内に`template.txt`というファイルがあったとすると, そのテンプレートは`template`モジュールで`src=template.txt`と指定することで利用することができます.

## コラム: 階層的なロール名

`/roles/[role名]/tasks/main.yml`について, `[ロール名]`の部分は階層的にすることが出来ます.
例えば, `carton install`をするロールを, `/roles/perl/carton/tasks/main.yml`などの形で設置することも出来ます.

但し, これまで見てきたように, ロールを構成するディレクトリ名として`vars`や`meta`, `templates`, `files`などは既に利用されているので, これらの名前は避けなければなりません.

## 練習問題

前の練習問題で作ったPlaybook中のタスクを, ロールを使って整理してみよう.
時間があれば, 仮想マシンを初期化した上で, もう一度最初からPlaybookを適用してみましょう.

## コラム: Vagrantとの連携

Vagrantと連携し, 仮想マシンを起動する際に自動的にPlaybookを適用することも可能です.
`Vagrantfile`と同じ階層に, Playbookを記載した`playbook.yml`というファイルがあるのであれば, `Vagrantfile`に次のように記載することで, 仮想マシン起動後にPlaybookを適用することができます.

```rb
Vagrant.configure("2") do |config|
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
  end
end
```
