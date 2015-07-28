# はじめに

Webアプリケーションに対するテストを実施する際には, 更に幾つかの知識と工夫が必要となります.
例えば, RDBMSやKVSを利用したテストを実施したいのであれば, テストを実行する前にこれらを初期化し, 一定の状態にしておかなければなりません.
もし, テストを実行する時点でRDBMSやKVSに一意でないデータが格納されていると, これが原因でテストの結果が変わってしまうかもしれないからです.
Perlでは, このような状況で役に立つWebアプリケーションに特化したテスト支援モジュールがいくつか用意されています.

この資料では, まず最初にテスト実行時にテスト用のMySQLとRedisを準備してくれるTest::mysqldとTest::RedisServerを紹介します.
また, これら以外のテスト支援モジュールについても, 概要を簡単に説明します.
最後に, Test::WWW::Mechanize::PSGIを利用したWebアプリケーションのテストと, Test::JsonAPI::Autodocを利用したAPIサーバのテストの実施方法について解説を行います.

どちらかといえば, この資料は予め読んでおくべき資料というよりは, アプリケーション開発を実施する際に｢このテストを楽に行う為には?｣と感じたら読むべき資料なのかもしれません.

なお, この資料ではWAFはAmon2を利用するものとして記述することとします.

# テスト支援モジュール

## [Test::mysqld](https://metacpan.org/pod/Test::mysqld)

Test::mysqldは, テストスクリプトごとにテスト専用のMySQLサーバ(`mysqld`)を用意してくれるモジュールです.
このモジュールを使うことによって, テスト実行時にデータベース内に格納されているデータが常に同じであることを保証することが可能です.

Test::mysqldは, newメソッドを利用してインスタンスを用意すると, その段階で新しいMySQLサーバ(`mysqld`)を立ち上げます.
更に, このインスタンスから`dsn`メソッドを利用することで, このMySQLサーバへ接続するためのDSN(Data Source Name)を取得することが出来ます.

```perl
use strict;
use warnings;

use Test::mysqld;

my $mysqld = Test::mysqld->new(); # この時点で, 新しいMySQLサーバが立ち上がる
my $dsn = $mysqld->dsn; # 立ち上げた`mysqld`へのDSNが取得出来る
```

更に, Test::mysqldはスクリプトが終了した段階(正確に言えば, 上記コードにおける`$mysqld`のデストラクタが動作した段階)で, 立ち上げたMySQLサーバが自動的に終了するようになっています.

これによって, 開発者はテスト用MySQLサーバの起動と停止を意識することなくテストを記述, 実行することができます.

### Amon2での利用例

Amon2では, `t/Util.pm`にテストの為のユーティリティ的なコードを書く事が出来ます.
Test::mysqldと, 次に説明するTest::RedisServerを利用する場合, 必要なコードは`t/Util.pm`に記述することが多いです.

```perl:t/Util.pm(抜粋)
use Test::mysqld;

my $MYSQLD;
unless (defined $ENV{TEST_DSN}) {
    $MYSQLD = Test::mysqld->new(
        my_cnf => {
            "skip-networking" => "", # TCPソケットを利用しない
        }
    );
    $ENV{TEST_DSN} = $MYSQLD->dsn;
}
```

ここでは, 環境変数`TEST_DSN`に, Test::mysqldが生成したMySQLサーバに接続するためのDSNを格納しています.
後は, テスト用の設定ファイル`config/test.pl`で次のように設定すれば, アプリケーションはTest::mysqldによって生成したMySQLサーバを利用するようになります(Tengを利用してRDBMSに接続する際, `connect_info`として `$c->config->{DBI}`を与えている場合).

```perl:config/test.pl
+{
    'DBI' => [ $ENV{TEST_DSN}, '', ''],
};
```

なお, Test::mysqldによって生成したMySQLサーバとその中のテスト用データベースには, `sql/mysql.sql`などに記述しているデータベースのスキーマ等は一切適用されていません.
そのため, 例えば次のようなコードで, 予めスキーマの流し込みを行う必要があるでしょう.

```perl:t/Util.pm(抜粋)
use DBI;
use Path::Tiny;
use Test::mysqld;

unless (defined $ENV{TEST_DSN}) {
    $MYSQLD = Test::mysqld->new(
        my_cnf => {
            "skip-networking" => ""
        }
    );
    $ENV{TEST_DSN} = $MYSQLD->dsn;

    # スキーマの流し込み
    my $dbh = DBI->connect($MYSQLD->dsn);
    my $source = path('sql/mysql.sql')->slurp;
    for my $stmt (split /;/, $source) {
        next unless $stmt =~ /\S/;
        $dbh->do($stmt) or die $dbh->errstr;
    }
}
```

## [Test::RedisServer](https://metacpan.org/pod/Test::RedisServer)

Test::RedisServerは, Test::mysqldのRedis版と言えるモジュールです.
テストスクリプトごとにテスト専用のRedisを利用することが可能です.

```perl
use strict;
use warnings;

use Test::RedisServer;
use Redis;

my $redis_server = Test::RedisServer->new(); # この時点で新しいRedisサーバが立ち上がる

my $redis = Redis->new( $redis_server->connect_info );
```

### Amon2での利用例

Test::RedisServerに関する設定も, Amon2であれば`t/Util.pm`に記載することが多いです.

```perl:t/Util.pm(抜粋)
use Test::RedisServer;

my $REDIS;
unless (defined $ENV{TEST_REDIS}) {
    $REDIS = Test::RedisServer->new;
    $ENV{TEST_REDIS} = $REDIS->connect_info;
}
```

これに対応するように, 設定ファイルでは次のように記述すると良いでしょう.

```perl:config/test.pl
+{
    'DBI' => [ $ENV{TEST_DSN}, '', ''],
    redis => $ENV{TEST_REDIS},
};
```

これで, Amon2のコンテキストを利用して`$c->config->{redis}`の形で, Test::RedisServerが起動したRedisに接続する為の情報が利用できるようになりますので,

```
my $redis = Redis->new( $c->config->{redis} );
```

このような形で, Redisを利用できるようになります.

## その他

### [Harriet](https://metacpan.org/pod/Harriet)
Test::mysqldやTest::RedisServerは, テストスクリプト単位でMySQLサーバやRedisの起動を行います.
ただ, テストスクリプト単位でMySQLサーバやRedisを起動してしまうと, 起動や終了のコストが重く, テストスクリプトの数に比例してテストにかかる時間が長くなってしまいます.

Harrietを使うことで, テスト実行前にMySQLサーバやRedisなどを起動して, テスト中はそれを使い回せるようになります.
これによって, MySQLサーバやRedisの起動を1回に抑えることが可能です.

但し, MySQLサーバやRedisを全てのテストスクリプトで使い回すので, 何もしないとテストを実行する度にデータベース内にデータが蓄積され続けてしまいます.
そのため, `t/Util.pm`などを利用して, テストスクリプトごと等の単位でMySQLサーバやRedisに格納されたデータを削除するような施策が別途必要になります.

### [App::Prove::Plugin::MySQLPool](https://metacpan.org/pod/App::Prove::Plugin::MySQLPool)

Harrietのように, テスト前に自動的にMySQLサーバを立ち上げ, それを全てのテストスクリプトで使い回すことが出来るようになります.
Harrietとの違いは, `prove`コマンドの`-j N`オプション(並列実行, 例えば`-j 4`でテストを4並列実行する)を与えた時に, `N`の数だけMySQLサーバを立ち上げてくれる点です.

これまでのTest::mysqldやHarrietの場合, 基本的には1つのMySQLサーバしか立ち上げることができないので, 並列実行してしまうと異なるテストスクリプトが1つのMySQLサーバを操作することになり, データベース操作の整合性が取れずにテストが失敗してしまう例が多いです.

App::Prove::Plugin::MySQLPoolを利用することによって, MySQLサーバを利用したテストについて, `prove`コマンドの`-j`オプションを利用して並列実行し, テスト実行時間を削減することが可能です.

# WebアプリケーションにおけるE2Eテスト

次に, WebアプリケーションにおけるE2Eテスト(リクエストを投げ, それに対応するレスポンスが返ってくるかどうか)で有用なモジュールを紹介します.

## [Test::WWW::Mechanize::PSGI](https://metacpan.org/pod/Test::WWW::Mechanize::PSGI)

Test::WWW::Mechanize::PSGIは, WWW::Mechanizeというブラウザ操作の自動化モジュールを利用したテストモジュールです.
これを利用することで, ｢'/'にアクセスしたら, 'Hello!'という文字列が表示される｣であったり, ｢フォームに'Bob'と入力してフォームを送信すると, 'Hello, Bob!'と表示される｣といったテストを実施することが出来ます.

```perl
use strict;
use warnings;
use utf8;
use t::Util;
use Plack::Test;
use Plack::Util;
use Test::More;
use Test::Requires 'Test::WWW::Mechanize::PSGI';

my $app = Plack::Util::load_psgi 'script/myapp-server';

my $mech = Test::WWW::Mechanize::PSGI->new(app => $app);
$mech->get_ok('/');

done_testing;
```

このスクリプトは, Test::WWW::Mechanize::PSGIを使った単純なテストの1つです.
`Plack::Util::load_psgi`でWebアプリケーションの起動用スクリプト(今回の場合, `script/myapp-server`)をロードし, Test::WWW::Mechanize::PSGIのコンストラクタに対して渡すことで, Test::WWW::Mechanize::PSGIのインスタンスはこのWebアプリケーションを操作出来るようになります.

Test::WWW::Mechanize::PSGIのインスタンスから実行している`get_ok`は, 第1引数のパスにGETメソッドで出来ればテスト成功, 出来なければテスト失敗と扱うメソッドです.
これによって, Webアプリケーションの'/'にGETでアクセスできるかどうかというテストを実現することが可能です.

### Test::WWW::Mechanize::PSGIで利用できるメソッド

詳細は, Test::WWW::Mechanize::PSGIのドキュメントの[HTTP VERBS](https://metacpan.org/pod/Test::WWW::Mechanize::PSGI#METHODS:-HTTP-VERBS)や[CONTENT CHECKING](https://metacpan.org/pod/Test::WWW::Mechanize::PSGI#METHODS:-CONTENT-CHECKING)などを参考にして下さい.

例えば, 次のようなテストを用意した場合, `title_is`メソッドでページのタイトルが`MyApp`であるかどうかを, `content_contains`メソッドでページの中に`Amon2`という文字列があるかどうかをテストすることが可能です.

```perl
use strict;
use warnings;
use utf8;
use t::Util;
use Plack::Test;
use Plack::Util;
use Test::More;
use Test::Requires 'Test::WWW::Mechanize::PSGI';

my $app = Plack::Util::load_psgi 'script/myapp-server';

my $mech = Test::WWW::Mechanize::PSGI->new(app => $app);
$mech->get_ok('/');
$mech->title_is('MyApp');
$mech->content_contains('Amon2');

done_testing;
```

## [Test::JsonAPI::Autodoc](https://metacpan.org/pod/Test::JsonAPI::Autodoc)

Test::JsonAPI::Autodocを利用することで, APIのテストからAPIのドキュメントを作成することが可能になります.
元々は, Ruby製のautodocというライブラリがあり, これをPerlに移植したものがTest::JsonAPI::Autodocです.

例えば, GETメソッドで`/success`にアクセスすると, ステータスコードが200で`{ 'status': 200 }`のようなJSONが返ってくるWebアプリケーションがあったとします.

このテストは, 次のように記述することが可能です.

```perl:sample.t
use strict;
use warnings;
use utf8;
use Test::More;
use Test::JSON;
use Test::JsonAPI::Autodoc;
use Plack::Util;
use Plack::Test;
use HTTP::Request::Common;
use JSON;

my $app = Plack::Util::load_psgi 'script/myapp-server';

test_psgi $app, sub {
    my $cb = shift;

    describe 'GET /success' => sub {
        my $req = GET '/success';
        plack_ok($cb, $req, 200, 'success!');
    };

    subtest 'レスポンスのJSONが正しいこと' => sub {
        my $req = GET '/success';
        my $res = $cb->($req);

        is_json $res->content, encode_json({
            status => 200,
        });
    };
};

done_testing;
```

ここでは, Plack::Testを利用してテストを実装しています.

```perl:sample.t(抜粋)
my $app = Plack::Util::load_psgi 'script/myapp-server';

test_psgi $app, sub {
    my $cb = shift;

    ...

    my $res = $cb->($req); # (1)

    ...
};
```

`Plack::Util::load_psgi`でWebアプリケーションの実行スクリプトを読み込み, `test_psgi`の第1引数に与えます.
これによって, `script/myapp-server`を起動することで動作するWebアプリケーションのテストを実行することが可能です.

テストコードは, `test_psgi`の第2引数に渡されるサブルーチンリファレンスに渡します.
このサブルーチンリファレンスの第1引数はサブルーチンリファレンス(コード内の`$cb`)になっており, これに対してHTTP::Requestなどで作成したリクエストを送ると, それに対応したレスポンスを得ることが出来ます(コード内の(1)の部分).
なお, このテストスクリプトでは, HTTP::Request::Commonを利用してHTTPのリクエストを生成しています.

テストコードは2つあります.
最初の`describe`以下がTest::JsonAPI::Autodocを利用したAPIのテスト, そして次の`subtest`の以下が実際にAPIが返すJSONを確認するコードです.

```perl:sample.t(抜粋)
    describe 'GET /success' => sub {
        my $req = GET '/success';
        plack_ok($cb, $req, 200, 'success!');
    };
```

ここでは, `plack_ok`を利用して, GETメソッドで`/success`にアクセスした際, ステータスコードが200であることを確認するテストを書いています.
Test::JsonAPI::Autodocは, `describe`以下の`plack_ok`もしくは`http_ok`を利用してAPIのドキュメントを作成します.

テストを実行する際に, 環境変数として`TEST_JSONAPI_AUTODOC`を`1`にした場合, Test::JsonAPI::Autodocはドキュメントの自動生成を行います.

```
$ TEST_JSONAPI_AUTODOC=1 prove t/sample.t
[10:52:34] t/sample.t ..
  GET /health_check
    ✓    L20: plack_ok($cb, $req, 200, 'success!');
  レスポンスのJSONが正しいこと
    ✓    L27: is_json $res->content, encode_json({

ok
ok     3087 ms
[10:52:37]
All tests successful.
Files=1, Tests=1,  3 wallclock secs ( 0.03 usr  0.01 sys +  0.59 cusr  0.13 csys =  0.76 CPU)
Result: PASS
```

ドキュメントは, `docs`ディレクトリ内に生成されます.

```markdown:docs/sample.t
generated at: 2015-05-20 10:52:37

## GET /success

success!

### Target Server

http://localhost

(Plack application)

### Parameters


### Request

GET /success

### Response

- Status:       200
- Content-Type: application/json

```json
{
   "status" : 200
}

```

このようなAPIドキュメントを自動的に取得することが出来ます.
利用者向けに提供するには少し不親切ですが, エンジニア(例えばAPIサーバを利用するフロントエンドのエンジニア)間の情報共有には十分有用に使えると思います.

# 参考資料

- [Test::mysqldのcopy_data_fromでテストが更に捗る話](http://www.songmu.jp/riji/archives/2013/06/testmysqldcopy.html)
- [Harriet ー テストのときつかうにデーモンの取扱を簡単にするためのフレームワーク](http://blog.64p.org/entry/2013/05/16/201916)
- [Test::JsonAPI::Autodocをリリースしました](http://moznion.hatenadiary.com/entry/2013/11/02/232144)
