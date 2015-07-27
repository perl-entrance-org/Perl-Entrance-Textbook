# Serverspec

先程, プロビジョニングツールの1つ, Ansibleを使って, Vagrantで構築した仮想マシン上に簡単なWebアプリケーションが動作する環境を構築しました.

Ansibleを利用することで, 複数台のサーバに対するプロビジョニングを, 並列化しながら自動的に実行することが出来るようになりました.
次の課題は, プロビジョニングが期待通りに完了しているかどうかの確認です.
もし, プロビジョニング対象のサーバが1台であれば手動で確認することも可能ですが, 複数台のサーバが対象であれば, プロビジョニングと同じくプロビジョニング後の確認についても, 手動で行うのは困難になります.

Ansibleの場合, `unreachable`や`failed`が0であれば, Playbookの適用は成功しているはずです.
ですが, Playbook適用後の環境が本当に正しい環境になっているかどうかまでは, これだけで保証することは出来ません.
これを確認する為の手段の1つが, ｢サーバのテストツール｣, Serverspecです.

Serverspecは, 宮下剛輔さん([@gousukenator](https://twitter.com/gosukenator), 通称mizzyさん)が開発された, Ruby製のツールです.
Rubyのテストフレームワーク, RSpecを活用して実装されており, ｢リソース｣とそれに対応する｢マッチャー｣を組み合わせることで, サーバの状態を簡単にテストすることが出来ます.

ここでは, Serverpsecの導入及び使い方を紹介した後, Playbookに対応するテストを, Serverspecを使って構築していきます.

## Serverspecのインストール

Rubyがインストール済みで, `gem`コマンドが利用可能であれば, 以下のコマンドでインストールできます.

```
$ gem install serverspec
```

もし, Rubyのインストールが出来ていないのであれば, 次の項目を参考に, `rbenv`を利用してRubyの環境を構築しましょう.

### Rubyのインストール

[rbenv](https://github.com/sstephenson/rbenv)は, 複数の異なるバージョンのRubyをインストールし, 状況に応じて切り替えながら利用出来るようにするツールです.
Perlの[plenv](https://github.com/tokuhirom/plenv)の元になったツールでもあります.

```
$ git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
$ git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
```

まず, rbenv本体をGitHubから`clone`します.
同時に, Rubyをインストールする為の`ruby-build`というツールも`clone`しておきます.

```
$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
$ echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
```

続いて, `~/.bash_profile`にrbenvに必要な設定を書き込みます.
`~/.bash_profile`の部分は, 環境に応じて適宜変更して下さい(例えば, zshユーザであれば`.zshrc`に変更する, 等).

```
$ exec $SHELL -l
$ rbenv --version
rbenv 0.4.0-146-g7ad01b2
```

これで, `rbenv`コマンドが利用出来るようになっています.
それでは, `rbenv intall`コマンドで, 現時点での最新版であるRuby 2.1.4をインストールします.

```
$ rbenv install 2.1.4
Downloading ruby-2.1.4.tar.gz...
-> http://dqw8nmjcqpjn7.cloudfront.net/bf9952cdeb3a0c6a5a27745c9b4c0e5e264e92b669b2b08efb363f5156549204
Installing ruby-2.1.4...
Installed ruby-2.1.4 to /Users/username/.rbenv/versions/2.1.4
```

問題なくインストールが完了していれば, `which ruby`と`ruby -v`の出力が, 次のようになっているはずです.

```
$ which ruby
/Users/username/.rbenv/shims/ruby
$ ruby -v
ruby 2.1.4p265 (2014-10-27 revision 48166) [x86_64-darwin14.0]
```

Rubyのインストールと同時に, `gem`コマンドも利用出来るようになっているので, 後は冒頭の記述の通り, `gem install serverspec`でServerspecをインストールしましょう.

### コラム: anyenv

ここまで紹介してきた`rbenv`や, Perl用の`plenv`は, ある言語について複数のバージョン(の環境)をインストールし, 必要に応じて切り替えられるツールと説明してきました.
なぜこのようなツールが必要になるのでしょうか?
それは, 複数のプロダクトを開発している場合, そのプロダクトで適した(= 本番環境で利用している)言語のバージョンを利用しなければならないからです(もし, 異なるバージョンを使っていた場合, バージョン間の誤差によって, 正しいコードが動かなかったり, 正しくないコードが動いてしまったり, といった事が起こりえます).

そのため, Perl用の`plenv`やRuby用の`rbenv`の他に, Node用の`ndenv`, Python用の`pyenv`など, 様々な言語のバージョンを管理する`**env`が開発されています.
これらの`**env`をそれぞれインストールするのは手間なので, これらを統一的に扱う仕組みが開発されています.
それが, [`anyenv`](https://github.com/riywo/anyenv)です.

既に`plenv`や`rbenv`を導入済みであれば, 別途インストールしなければならない為二度手間となってしまいますが, 1つのツールで複数の言語の環境を管理出来る, というのはとても便利です.
機会があれば, 既に導入済みの`plenv`や`rbenv`を, `anyenv`で管理するように手を加えてみることをおすすめします.

## Serverspecの初期設定

Vagrantで立ち上げた仮想マシンは, 先程Ansibleで環境構築を行った仮想マシンをそのまま利用します.
`ssh vagrant`で仮想マシンに接続出来るか, 予め確認しておきましょう.

Serverspecの初期設定は, `serverspec-init`コマンドで行います.

```
$ serverspec-init
Select OS type:

  1) UN*X
  2) Windows

Select number: 1

Select a backend type:

  1) SSH
  2) Exec (local)

Select number: 1

Vagrant instance y/n: n
Input target host name: vagrant
 + spec/
 + spec/vagrant/
 + spec/vagrant/sample_spec.rb
 + spec/spec_helper.rb
 + Rakefile
```

`serverspec-init`コマンドでは, 以下のとおり設定します.

- OSの種類(`Select OS type: `)
    - Windows以外を利用しているのであれば`1`を入力し, リターンキーを押します.
- バックエンドの種類(`Select a backend type: `)
    - テスト対象となるホストにどのように接続するか選択します.
    - SSHで接続するのであれば`1`, ローカル環境に対してテストを実行するのであれば`2`を選択し, リターンキーを押します.
        - ここでは, Vagrantで立ち上げた仮想マシンを対象とするので, `1`を選択します.
- Vagrantのインスタンスか否か(`Vagrant instance y/n: `)
    - 本来であれば`y`ですが, ここは`n`を入力し, リターンキーを押します.
- テスト対象のホスト名(`Input target host name: `)
    - Vagrantで建てた仮想マシンには, `vagrant`で接続できるよう設定しているので, `vagrant`と入力し, リターンキーを押します.

これで, Serverspecの初期設定を実施することが出来ます.
この際, 入力した設定に応じて, Serverspecのためのファイルが生成されています.

## Serverspecのファイル構成

`serverspec-init`は, 以下のファイルを生成します.
これらは, Serverspecによるテストを実現する為の大切なファイルです.

### `Rakefile`

Serverspecを利用したテストのタスクが記述されているファイルです.
このファイルによって, 後述するように`rake spec`でテストを実行出来るようになります.

### `spec/spec_helper.rb`

各テストのヘルパークラスです.
SSHを利用して, テスト対象となるホストへ接続する為のコードが記載されています.

### `spec/vagrant/sample_spec.rb`

実際にテストを記述する部分です.
デフォルトで, 次のようなテストが記述されています.

```rb
require 'spec_helper'

describe package('httpd'), :if => os[:family] == 'redhat' do
  it { should be_installed }
end

describe package('apache2'), :if => os[:family] == 'ubuntu' do
  it { should be_installed }
end

describe service('httpd'), :if => os[:family] == 'redhat' do
  it { should be_enabled }
  it { should be_running }
end

describe service('apache2'), :if => os[:family] == 'ubuntu' do
  it { should be_enabled }
  it { should be_running }
end

describe service('org.apache.httpd'), :if => os[:family] == 'darwin' do
  it { should be_enabled }
  it { should be_running }
end

describe port(80) do
  it { should be_listening }
end
```

## テストの実装

Serverspecのテストは, Rubyのコードとして実装します.
といっても, Serverspecのテストを書く際にRubyの知識はほとんど求められません(但し, `Rakefile`や`spec_helper.rb`を書き換える場合は, その限りではありません).
基本的に, Serverspecの｢リソース｣を利用しながら, 下記のフォーマットに従ってサーバの仕様(Spec)を書いていけばOKです.

```rb
describe [リソースタイプ]([テスト対象]) do
  [テスト条件]
end
```

例として, テスト対象のホストに`nginx`のパッケージが導入されているかを確認するテストを書いてみます.
この場合, 各OSのパッケージマネージャについてテストすることができる[`package`](http://serverspec.org/resource_types.html#package)というリソースを利用すると良いでしょう.

```rb
describe package('nginx') do
  [テスト条件]
end
```

`package`リソースは, 例えば`package('nginx')`のように書くと, 対象ホストの`nginx`パッケージについてテストをすることが出来ます.

一方｢テスト条件｣については, [packageリソースのドキュメント](http://serverspec.org/resource_types.html#package)を見ると, インストール済みかどうかを検証する`be_installed`と呼ばれるマッチャー(matcher)が利用できる, と書かれています.

```rb
require 'spec_helper'

describe package('nginx') do
  it { should be_installed }
end
```

このように, `package`というリソースと, それに対応する`be_installed`というマッチャーを組み合わせ, `spec/vagrant/sample_spec.rb`をこのように書いておけば, ｢対象ホストに, nginxというパッケージがインストール済み(installed)か?｣をテストすることができます.

### コラム: ServerespecとSpecinfra

先程紹介した`package`というリソースは, テスト対象となるホストのOSを問わず実行することができます.
これはつまり, `package`というリソースであるパッケージについてテストする際, テスト対象のホストのOSがCentOSであれば`yum`を, Ubuntuであれば`apt`を利用し, テストを実行するということです(但し, 例えばCentOSとUbuntuでパッケージ名が異なる場合はその限りではありません).

これは, Serverspecが利用している[Specinfra](https://github.com/serverspec/specinfra)が, OS間のコマンドの差異を吸収し, 適切なコマンドを発行しているためです.
元々SpecinfraはServerspecの一部でしたが, 新しいインフラのテストツールやプロビジョニングツールの基盤として利用されることを狙って, Serverspecから切りだされました.
実際に, Cookpadが｢軽いChef｣を目指して開発した[Itamae](https://github.com/itamae-kitchen/itamae)というプロビジョニングツールも, その実装にSpecinfraを利用し, OS間の差異を吸収できるようにしています.

## テストの実行

それでは, このテストをVagrantで建てた仮想マシンに対して実行してみましょう.
既にAnsibleのPlaybookを適用済みであれば, `nginx`パッケージは導入済みのはずなので, テストは全て通るはずです.

```
$ rake spec
/Users/username/.rbenv/versions/2.1.4/bin/ruby -I/Users/username/.rbenv/versions/2.1.4/lib/ruby/gems/2.1.0/gems/rspec-support-3.2.2/lib:/Users/username/.rbenv/versions/2.1.4/lib/ruby/gems/2.1.0/gems/rspec-core-3.2.1/lib /Users/username/.rbenv/versions/2.1.4/lib/ruby/gems/2.1.0/gems/rspec-core-3.2.1/exe/rspec --pattern spec/vagrant/\*_spec.rb

Package "nginx"
  should be installed

Finished in 0.45598 seconds (files took 0.62706 seconds to load)
1 example, 0 failures
```

Serverspecのテストは, `Rakefile`が設置されているディレクトリで`rake spec`コマンドを実行することで実施できます.
実行結果としては最後に`1 example, 0 failures`が出力されていて, 1つのテストを実行し0の失敗, つまり全て成功という結果になりました.

では次に, 失敗時のパターンを確認しておきます.
`spec/vagrant/sample_spec.rb`を, 次のように書き換えましょう.

```rb
require 'spec_helper'

describe package('nginx') do
  it { should be_installed }
end

describe package('apache2') do
  it { should be_installed }
end
```

`apache2`は仮想マシン上で動いているUbuntu Serverにインストールしていないはずなので,

```
$ rake spec
/Users/username/.rbenv/versions/2.1.4/bin/ruby -I/Users/username/.rbenv/versions/2.1.4/lib/ruby/gems/2.1.0/gems/rspec-support-3.2.2/lib:/Users/username/.rbenv/versions/2.1.4/lib/ruby/gems/2.1.0/gems/rspec-core-3.2.1/lib /Users/username/.rbenv/versions/2.1.4/lib/ruby/gems/2.1.0/gems/rspec-core-3.2.1/exe/rspec --pattern spec/vagrant/\*_spec.rb

Package "nginx"
  should be installed

Package "apache2"
  should be installed (FAILED - 1)

Failures:

  1) Package "apache2" should be installed
     On host `vagrant'
     Failure/Error: it { should be_installed }
       expected Package "apache2" to be installed
       sudo -p 'Password: ' /bin/sh -c dpkg-query\ -f\ \'\$\{Status\}\'\ -W\ apache2\ \|\ grep\ -E\ \'\^\(install\|hold\)\ ok\ installed\$\'

     # ./spec/vagrant/sample_spec.rb:8:in `block (2 levels) in <top (required)>'

Finished in 0.58318 seconds (files took 0.58938 seconds to load)
2 examples, 1 failure

Failed examples:

rspec ./spec/vagrant/sample_spec.rb:8 # Package "apache2" should be installed

/Users/username/.rbenv/versions/2.1.4/bin/ruby -I/Users/username/.rbenv/versions/2.1.4/lib/ruby/gems/2.1.0/gems/rspec-support-3.2.2/lib:/Users/username/.rbenv/versions/2.1.4/lib/ruby/gems/2.1.0/gems/rspec-core-3.2.1/lib /Users/username/.rbenv/versions/2.1.4/lib/ruby/gems/2.1.0/gems/rspec-core-3.2.1/exe/rspec --pattern spec/vagrant/\*_spec.rb failed
```

失敗した場合, このような出力が得られます.

```
Package "apache2"
  should be installed (FAILED - 1)
```

この部分を見れば, `apache2`に関して, `should be installed`が`FAILED`になっていることがわかります.

## 代表的なリソース

ここでは, Serverspecで利用できる代表的なリソースと, そこで利用できるマッチャーについて紹介します.
Serverspecで利用できるリソースの一覧は, Serverspecの公式ウェブサイトの｢[Resource Types](http://serverspec.org/resource_types.html)｣というページで確認することができます.

### [`package`](http://serverspec.org/resource_types.html#package)リソース

任意のパッケージについてテストすることができます.
マッチャーとしては, 指定したパッケージがインストール済みかどうかをテストする, `be_installed`が用意されています.

```rb
describe package('nginx') do
  it { should be_installed }
end
```

### [`service`](http://serverspec.org/resource_types.html#service)リソース

任意のサービスについてテストすることができます.
マッチャーとしては, OS起動時に自動的に起動するよう設定されているかを確認する`be_enabled`, サービスが稼働しているかを確認する`be_running`などを利用することができます.

```rb
describe service('nginx') do
  it { should be_enabled }
  it { should be_running }
end
```

### [`port`](http://serverspec.org/resource_types.html#port)リソース

任意のポートについてテストすることが出来ます.
マッチャーとしては, そのポートがlisten(待ち受け)しているかを確認する`be_listening`を利用できます.

```rb
describe port(80) do
  it { should be_listening }
end
```

`be_listening`マッチャーについては, `with`を利用することで, どのプロトコルでlistenしているかをテストすることもできます(`tcp`, `udp`, `tcp6`, `udp6`を指定可能).

```rb
describe port(80) do
  it { should be_listening.with('tcp') }
end
```

この場合, 80番ポートが`tcp`でlistenされているかを確認することができます.

### [`process`](http://serverspec.org/resource_types.html#process)リソース

任意のプロセスについてテストする事ができます.
マッチャーとしては, そのプロセスが動いているかどうかを確認する`be_running`を利用することができます.

```rb
describe process('nginx') do
  it { should be_running }
end
```

また, `ps`コマンドで確認できる, `user`や`group`, `args`といったプロセスのパラメータについても, 次のような記述でテストすることができます.

```rb
describe process('memcached') do
  its(:user) { should eq 'memcached' }
  its(:args) { should match /-c 32000\b/ }
end
```

`its(:user) { should eq 'memcached' }`で, このプロセスが`memcached`ユーザによって動作していることを, `its(:args) { should match /-c 32000\b/ }`で, このプロセスが`-c 32000`というオプション付きで起動されていることをテストしています.

### [`file`](http://serverspec.org/resource_types.html#file)リソース

任意のファイルやディレクトリについてテストすることができます.
マッチャーとしては, まずファイルの種類を確認するマッチャーとして, ファイルであることを確認する`be_file`, ディレクトリであることを確認する`be_directory`, ソケットであることを確認する`be_socket`, symlinkであることを確認する`be_symlink`が用意されています.

```rb
describe file('/etc/passwd') do
  it { should be_file }
end
```

この場合, `/etc/passwd`というファイルがあることをテストしています.

また, モードを確認する`be_mode`, オーナーを確認する`be_owned_by`, グループの情報を確認する`be_grouped_info`, リンク先を確認する`be_linked_to`, 読み込み可能かを確認する`be_readable`, 書き込み可能か確認する`be_writable`, 実行可能か確認する`be_executable`, ディレクトリがマウントされているかを確認する`be_mounted`というマッチャーも用意されています.

また, `its(:content)`を使うことで, ファイルに任意の文字列が含まれているかを確認することができます.

```rb
describe file('/etc/httpd/conf/httpd.conf') do
  its(:content) { should match /ServerName www.example.jp/ }
end
```

この場合, `/etc/httpd/conf/httpd.conf`というファイルに, `ServerName www.example.jp`という記述があるかどうかを確認しています.

更に, `its(:md5sum)`や`its(:sha256sum)`を使えば, ファイルのチェックサムが同じかどうかをテストすることもできます.

```rb
describe file('/etc/services') do
  its(:sha256sum) { should eq 'a861c49e9a76d64d0a756e1c9125ae3aa6b88df3f814a51cecffd3e89cce6210' }
end
```

### [`command`](http://serverspec.org/resource_types.html#command)リソース

指定したコマンドを実行した結果について, テストすることができます.

```rb
describe command('ls -al /') do
  its(:stdout) { should match /bin/ }
end
```

この場合, ホスト側で`ls -al /`というコマンドを実行してから, その標準出力に`bin`という文字列が含まれているかどうかを確認することができます.

また, 標準出力について確認する`its(:stdout)`の他に, 標準エラー出力について確認する`its(:stderr)`, 終了時のステータスについて確認する`its(:exit_status)`も利用することができます.

`its(:stdout)`及び`its(:stderr)`では, `contain`というマッチャーを利用することができます.
これによって, 標準出力及び標準エラー出力について, より詳細に確認をすることができます.

```rb
describe command('apachectl -M') do
  its(:stdout) { should contain('proxy_module') }
end
```

このように記述した場合, `apachectl -M`を実行した際の標準出力に, `proxy_module`という文字列が含まれていることを確認できます.

```rb
describe command('apachectl -V') do
  # test 'Prefork' exists between "Server MPM" and "Server compiled".
  its(:stdout) { should contain('Prefork').from(/^Server MPM/).to(/^Server compiled/) }

  # test 'conf/httpd.conf' exists after "SERVER_CONFIG_FILE".
  its(:stdout) { should contain('conf/httpd.conf').after('SERVER_CONFIG_FILE') }

  # test 'Apache/2.2.29' exists before "Server built".
  its(:stdout) { should contain(' Apache/2.2.29').before('Server built') }
end
```

また, このように記述した場合, `apachectl -V`を実行した際の標準出力について,

- `^Server MPM`という正規表現に一致する文字列から`^Server compiled`という正規表現に一致する文字列の間に, `Prefork`という文字列が含まれていること
- `SERVER_CONFIG_FILE`という文字列の後に, `conf/httpd.conf`という文字列が含まれていること
- `Server build`という文字列の前に, ` Apache/2.2.29`という文字列が含まれていること

について, 確認することができます.

## 練習問題

Serverspecの[Resource Types](http://serverspec.org/resource_types.html)から適切なリソースを探しながら, 次の要素を満たすように`sample_spec.rb`を書き換えていきましょう.

### `SampleApp`のテスト

Ansible Playbookで自動的に構築出来るようにした`SampleApp`について, Serverspecを利用したテストを書いてみよう.

### MySQL

(実際に`SampleApp`では利用していませんが)Ansible Playbookを変更して, 仮想マシン内にMySQLをインストールしましょう.
また, ServerspecでMySQLについてテストするようにしましょう.
