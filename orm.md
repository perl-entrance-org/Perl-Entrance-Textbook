# はじめに

Webアプリケーションを作る際, そのWebアプリケーションが利用するデータを永続的に保存する為にデータストアを利用することが多いでしょう.
データストアには様々な種類がありますが, 一般的にはRDBMS, その中でも特にMySQLを使う機会が多いと思います.

Perlには, RDBMSのインターフェースとしてDBIというモジュールがあり, これと各種RDBMSと接続するDBD系モジュールを組み合わせることでRDBMSの操作を行います.
実際にRDBMSと接続するDBD系モジュールと, そのインターフェイスとなるDBIモジュールで分離することで, ユーザは利用しているRDBMSの違いをほぼ気にせずにデータベースを操作することが可能です.

一方, ORM(O/Rマッパ, Object-Relational Mapper)は, RDBMSから取得した単一行のデータを, 特定のクラスと紐付けてオブジェクト化してくれる仕組みです.
DBIを利用してRDBMSからデータを取得した場合, RDBMSから取得したデータは, ハッシュリファレンスや配列リファレンスといった形で取得することになります.
これに対してORMを利用した場合はRDBMSからオブジェクト(の集合)として取得することが出来るので, ハッシュリファレンスや配列リファレンスをそのまま利用する場合と比べて, データの取り回しが比較的楽になります.

ここでは, Perl製の軽量ORMであり, Amon2でデフォルトで利用されているTengというORMについて解説します.

# Tengのインストールと初期設定

`cpanm`コマンドからインストールすることができます.

```
$ cpanm Teng
--> Working on Teng
Fetching http://www.cpan.org/authors/id/S/SA/SATOH/Teng-0.28.tar.gz ... OK
Configuring Teng-0.28 ... OK
Building and testing Teng-0.28 ... OK
Successfully installed Teng-0.28 (upgraded from 0.25)
1 distribution installed
```

## スキーマ情報の設定

ORMを使う為には, 予め利用するデータベースのスキーマ情報を提供しなければなりません.
Tengの場合, Teng::Schema::DeclareかTeng::Schema::Loaderを利用することでスキーマ情報を提供することが可能です.

Teng::Schema::Declareは, このモジュールが提供するDSLを使ってスキーマ情報を定義することが出来ます.
後述する, `inflate`や`deflate`機能の設定を含む, 細かいスキーマ情報の指定が可能です.
Teng::Schema::Declareを利用したスキーマは, 手で書くことも出来ますが, Teng::Schema::Dumperを利用してデータベースからスキーマを取得して生成することも可能です.
また, データベースのスキーマをDBIx::Schema::DSLを利用して定義している場合, SQL::TranslatorとSQL::Translator::Producer::Tengを利用してTengのスキーマを用意することもできます.

一方, Teng::Schema::Loaderは, Tengはデータベースから直接スキーマ情報を取得して利用します.
Teng::Schema::Declareのように, 細かいスキーマ定義は出来ませんが, 非常に簡単にスキーマを用意することが出来るので, 簡単なアプリケーションの作成や, 軽くTengを試してみたい時に利用するとよいでしょう.

今回は, 手動でTeng::Schema::Declareを利用したスキーマを用意し, これを利用してTengを使うことにします.
また, RDBMSは簡単のためにSQLiteを利用することとしましょう.

```sql
CREATE TABLE IF NOT EXISTS user (
    id           INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    name         VARCHAR(255),
    created_at   INTEGER NOT NULL,
    updated_at   INTEGER NOT NULL
);
```

今回利用するテーブルのDDL(Data Definition Language)は上記の通りにします.
`user`というテーブルに, 主キーかつAuto IncrementなINT型の`id`, ユーザの名前を持つVARCHAR(255)の`name`, そしてデータ生成日時/更新日時を保持する`created_at`と`updated_at`を持たせます.
SQLiteには, MySQLにおける(時間を保持するための)DATETIME型のようなデータ型がないので, ここではepoch秒をINTEGER型で持たせることとしましょう.

さて, 今回の課題では, `teng.pl`というスクリプトからTengを利用し, スキーマファイルは`lib/MyApp/DB/Schema.pm`に配置することにします.
`tree`コマンドでディレクトリ構成を確認すると, 次のようになります.

```
.
├── lib
│   └── MyApp
│       └── DB
│           └── Schema.pm
└── teng.pl
```

早速, 先程のDDLにあわせて`Schema.pm`を用意します.

```perl:lib/MyApp/DB/Schema.pm
package MyApp::DB::Schema;
use strict;
use warnings;
use utf8;

use Teng::Schema::Declare;

base_row_class 'MyApp::DB::Row';

table {
    name 'user';
    pk 'id';
    columns qw(id name created_at updated_at);
};

1;
```

## その他の準備

更にTengを利用するにあたって, Tengを継承したクラスを用意しなければなりません.
今回は, `MyApp::DB`という名前空間で提供することにします.

```perl:lib/MyApp/DB.pm
package MyApp::DB;
use parent qw/Teng/;

1;
```

また, Tengがオブジェクトを作る時のベースクラスとなる, `MyApp::DB::Row`も作っておきます.

```perl:lib/MyApp/DB/Row.pm
package MyApp::DB::Row;
use strict;
use warnings;
use utf8;
use parent qw(Teng::Row);

1;
```

これでTengの準備は完了です.

# データベースへの接続

## SQLiteの準備

Tengを使ってデータベースに接続する前に, 接続先となるSQLiteの準備をしておきましょう.
`teng.pl`と同じ階層に`sqlite.sql`を設置し, 先程紹介したDDLを記載しておきます.

```sql:sqlite.sql
CREATE TABLE IF NOT EXISTS user (
    id           INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    name         VARCHAR(255),
    created_at   INTEGER NOT NULL,
    updated_at   INTEGER NOT NULL
);
```

SQLiteはファイルベースのデータベースなので, 今回は`teng.pl`と同じ階層に`sqlite.db`という名前で用意することにします.

```
$ sqlite3 sqlite.db < sqlite.sql
```

問題なく処理が終了していれば, カレントディレクトリに`sqlite.db`というファイルが生成されているはずです.
この段階で, `tree`コマンドの出力は次のようになります.

```
$ tree
.
├── lib
│   └── MyApp
│       ├── DB
│       │   ├── Row.pm
│       │   └── Schema.pm
│       └── DB.pm
├── sqlite.db
├── sqlite.sql
└── teng.pl

3 directories, 6 files
```

## Tengによるデータベースへの接続

それでは, `teng.pl`からTengを利用してSQLiteのデータベースに接続してみましょう.

```perl:teng.pl
use strict;
use warnings;
use utf8;

use lib 'lib';
use MyApp::DB;
use MyApp::DB::Schema;

my $teng = MyApp::DB->new(
    connect_info => ['dbi:SQLite:dbname=sqlite.db', '', '', +{ sqlite_unicode => 1 } ],
    schema_class => 'MyApp::DB::Schema',
);
```

Tengを継承した`MyApp::DB`のコンストラクタ(`new`)に, `connect_info`と`schema_class`を渡しています.

`connect_info`は, DBIでデータベースに接続するときの`connect`メソッドに渡すパラメータと同じです(詳しくは, DBIのドキュメントの`connect`の項を参照して下さい. 日本語ドキュメントは[こちら](http://perldoc.jp/docs/modules/DBI-1.612/DBI.pod#connect)).
第1引数はDSN(接続対象となるデータベースを指定する為の識別名), 第2引数はデータベースに接続するユーザ名, 第3引数はデータベースに接続するときのパスワード, そして第4引数はオプションを指定します.

`schema_class`は, 先程生成したTengのスキーマファイルを指定します. 今回は`MyApp::DB::Schema`という名前空間に用意したので, この文字列をそのまま渡します.

これで, `$teng`からTengを利用することができるようになりました.
これ以降, Tengで利用できる各種メソッドを紹介していきますが, このTengに接続するためのコードについては, 今後は省略していくことにします.

# Tengの機能

## `insert`

`insert`は, その名の通りデータベースにデータを挿入するメソッドです.
第1引数にテーブル名を, 第2引数に挿入するデータをハッシュリファレンス形式で指定することができます.

```perl:teng.pl(抜粋)
my $obj = $teng->insert(user => {
    name       => 'papix',
    created_at => scalar time(),
    updated_at => scalar time(),
});
```

なお, `insert`メソッドは, 返り値として挿入したデータのオブジェクトを返します(上記コードにおける`$obj`).

`teng.pl`を実行した後に, `sqlite3`コマンドで挿入できていることを確認してみましょう.

```
$ perl teng.pl
$ sqlite3 sqlite.db
SQLite version 3.7.13 2012-07-17 17:46:21
Enter ".help" for instructions
Enter SQL statements terminated with a ";"
sqlite> select * from user;
1|papix|1430897608|1430897608
sqlite>
```

SQLiteのデータベースに, `insert`メソッドで追加したデータが問題なく追加されていることがわかります.
`user`テーブルの中身が1行だけでは寂しいので, `insert`メソッドを使って, 更にいくつかデータを足しておくことにします.

```
$ sqlite3 sqlite.db
SQLite version 3.7.13 2012-07-17 17:46:21
Enter ".help" for instructions
Enter SQL statements terminated with a ";"
sqlite> select * from user;
1|papix|1430897608|1430897608
2|hoto|1430897796|1430897796
3|akms|1430897804|1430897804
sqlite>
```

以降は, このデータをベースに解説していきます.

## `single`

次に, データベースからデータを取得するためのメソッド, `single`を紹介します.
`single`は, その名の通りデータベースから1件のデータを取得する為のメソッドです.

```perl:teng.pl(抜粋)
my $obj = $teng->single('user' => {
    name => 'papix' # 'name = papix'という条件
});
```

`single`メソッドは, 第1引数に検索対象となるテーブル名を, 第2引数に検索条件(いわゆる'WHERE句')を指定できます.
該当するデータが存在すればそのオブジェクトを, 該当するデータが存在しなければ`undef`を返します.

なお, TengはクエリビルダとしてSQL::Makerを利用しているので, WHERE句にあたる部分の書き方については[SQL::Maker::Condition](https://metacpan.org/pod/SQL::Maker::Condition)のドキュメントが参考になります.

内部的には, `single`メソッドは内部で`LIMIT 1`を指定することによって, ｢1件のデータの取得｣を実現しています.
万が一, 条件に該当するデータが複数存在した場合の挙動については, 利用しているRDBMSで`LIMIT 1`を指定した時と同じものが取得されることになります.

### オブジェクトから利用できるメソッド 〜getter〜

`single`や, 次に説明する`search`メソッドなどで取得したオブジェクトからは, 初期状態でいくつかのメソッドが利用出来るようになっています.

例えば, オブジェクトはスキーマファイルで定義した｢カラム名｣のメソッドが利用できるようになっています.
このメソッドを利用することで, そのオブジェクトに格納された, 各カラムのデータを取得することができます.

```perl:teng.pl(抜粋)
my $obj = $teng->single('user' => {
    name => 'papix',
});
print $obj->name; # => 'papix'
print $obj->id;   # => 1
```

## `search`

`search`メソッドは, データを1件のみ取得する`single`メソッドとは異なり, 条件に当てはまるデータを全て取得するメソッドです.

```perl:teng.pl(抜粋)
my @users = $teng->search('user' => { like => '%a%' }); # ｢'name'に'a'を含む｣という条件
```

なお, 第2引数として与えることができる条件を省略し, 第1引数のテーブル名のみを与えた場合は, テーブルに含まれる全てのデータを取得することができます.

```perl:teng.pl(抜粋)
my @all_users = $teng->search('user');
```

### コンテキストによる`search`メソッドの挙動

さて, この`search`メソッドの返す値ですが, コンテキストによって挙動が異なります.

### スカラコンテキスト

```
my $itr = $teng->search('user');
```

このようにスカラコンテキストで受けた場合, `search`メソッドはTeng::Iteratorのオブジェクトを返します.
Teng::Iteratorからは`next`というメソッドが利用できるようになっており, このメソッドを利用して取得したデータを1件(1オブジェクト)ずつ取得することが可能です.

`next`メソッドは, 1回目の呼び出しで取得したデータの1番目, 2回目の呼び出しで取得したデータの2番目... といった形でオブジェクトを返し, 全てのオブジェクトを返した後は`undef`を返します.
そのため, Teng::Iteratorの`next`メソッドと`while`文を利用して, 検索した全てのデータを取得することができます.


```
while (my $user = $itr->next) {
    print $user->name; # => 'papix', 'hoto', 'akms'がそれぞれ順番に表示される
}
```

また, Teng::Iteratorには`all`メソッドが用意されており, 取得した全てのデータのオブジェクトをリスト形式で返してくれます.

```
my $itr = $teng->search('user');
my @all_users = $itr->all;

for my $user (@all_users) {
    print $user->name; # => 'papix', 'hoto', 'akms'がそれぞれ順番に表示される
}
```

#### リストコンテキスト

一方, リストコンテキストの場合, Tengは内部でTeng::Iteratorに対して`all`メソッドを実行し, 取得した全てのデータのオブジェクトをリスト形式で返してくれます.

```
my @all_users = $teng->search('user');

for my $user (@all_users) {
    print $user->name; # => 'papix', 'hoto', 'akms'がそれぞれ順番に表示される
}
```

## `update`

`update`メソッドは, データベースに格納されている任意のデータを更新するメソッドです.

```perl:teng.pl(抜粋)
my $count = $teng->update('user' => { name => 'super papix' }, { id => 1 });
```

第1引数に変更対象となるテーブル名を, 第2引数にハッシュリファレンス形式で変更するデータを, そして第3引数にハッシュリファレンスで更新したいデータの条件を指定します.
上記のコードでは, `user`テーブルに格納されたデータについて, `id`が1であれば, `name`カラムの値を`super papix`に変更する, という更新を行います.

なお, 万が一第3引数を省略した場合, 指定したテーブルに格納されている全てのデータが書き換えられてしまうので, 注意しましょう.

`update`メソッドは返り値として, 変更したデータの数を返しますので, 更新が無事成功したかを確認することが可能です.

```perl:teng.pl(抜粋)
my $count = $teng->update('user' => { name => 'super papix' }, { id => 1 });

if ($count) {
    print "成功!";
} else {
    print "失敗!";
}
```

## `delete`

`delete`メソッドは, データベースに格納されている任意のデータを削除するメソッドです.

```perl:teng.pl(抜粋)
my $count = $teng->delete('user' => { name => 'super papix' });
```

第1引数に変更対象となるテーブル名を, 第2引数にハッシュリファレンス形式で削除するデータの条件をハッシュリファレンスで指定します.
この場合, `user`テーブルに格納されたデータについて, `name`カラムが`super papix`であるデータを削除する, という処理になります.

`delete`メソッドも, `update`メソッドと同じく, 変更した(削除した)データの数を返します.

## オブジェクトから利用できるメソッド 〜update/delete〜

`update`及び`delete`メソッドは, `single`や`search`メソッドなどを利用して取得したオブジェクトからも利用することができます.

```perl:teng.pl(抜粋)
my $obj = $teng->single(user => { id => 1 });
$obj->update({ name => 'papix' });
```

このようにすれば, `$obj`に格納されたオブジェクトの`name`カラムの値を`papix`に書き換えることができます.

また,

```perl:teng.pl(抜粋)
my $obj = $teng->single(user => { id => 1 });
$obj->delete();
```

このようにすれば, `$obj`に格納されたオブジェクトに当てはまるデータを, データベース上から削除することができます.

## オブジェクトから利用できるメソッド 〜refetch〜

`refetch`は, オブジェクトに格納されているデータを再度データベースから引き直す処理です.

```perl:teng.pl(抜粋)
my $papix = $teng->single(user => { id => 1 });

$teng->update(user => { name => 'super papix' }, { id => 1 });

print $papix->name; # => 'papix' -> 同期していないので, 'name'は'papix'のまま
$papix->refetch;
print $papix->name; # => 'super papix' -> `refetch`で同期したので, 'super papix'になった
```

オブジェクトに格納されているデータと, その元になったデータベース上のデータは同期していません.
そのため, 上記のコードのように, オブジェクトを取得した後に, その元になったデータをデータベース上で書き換えたとしても, その変更は既に取得済みのオブジェクトには反映されません.
同期したい場合は`refetch`メソッドを利用して明示的に同期するようにしましょう.

## トランザクション(`txn_scope` / `commit` / `rollback`)

データベースを操作する際, ｢ある処理とある処理は, 同時に行われて欲しい｣という場合があると思います.
例えば, 何らかのパラメータの交換処理が発生して, ｢ユーザAのあるパラメータを100減らしてから, ユーザBのあるパラメータを100増やしたい｣といった場合です.

このとき, もしも｢ユーザAのあるパラメータを100減らす｣段階でエラーが発生してしまうと, ユーザAのあるパラメータが100減ったにも関わらず, ユーザBのあるパラメータが100増えない, という現象が発生してしまいます.

これを防ぐためには, トランザクションを利用しましょう.
トランザクションを利用すれば, 必ず同時に変更したい｢ある処理A｣と｢ある処理B｣があったとして,

- 全ての変更が成功 -> 変更を適用(commit)
- どちらかが失敗 / 途中でエラー -> 全ての変更を巻き戻し(変更前の状態に戻す = rollback)

という振る舞いを実現することができます.

### トランザクションの開始

トランザクションを開始するには, `txn_scope`メソッドを利用します.

```perl
my $txn = $teng->txn_scope(); # トランザクションの開始
```

これ以降の処理がトランザクションの対象となります.

### 操作の確定

処理した内容を確定したい場合は, `$txn`オブジェクトから`commit`メソッドを実行します.

```perl
my $txn = $teng->txn_scope();

    ... データベースの操作 ...

$txn->commit();
```

これで, `txn_scope`メソッドを実行してから以降に行われたデータベースの操作が, データベースに適用されます.

### 操作の巻き戻し

もし, 途中でエラーが発生した場合は, `$txn`オブジェクトから`rollback`メソッドを実行することで, データベースの状態をトランザクション開始前の状態に巻き戻す(rollback)ことが可能です.

```perl
my $txn = $teng->txn_scope();

    ... データベースの操作 ...

$txn->rollback();
```

### トランザクションの例

トランザクションは, 次のように[Try::Tiny](https://metacpan.org/pod/Try::Tiny)モジュールと組み合わせると便利です.

```perl
use Try::Tiny;

my $txn = $teng->txn_scope();

my $user1 = $teng->single( user => { id => 1 } );
my $user2 = $teng->single( user => { id => 2 } );

try {
    $user1->update({ name => 'super '.$user1->name });
    $user2->update({ name => 'super '.$user2->name });

    $txn->commit();
} catch {
    $txn->rollback(); # `try { ... }`の中で例外が発生した場合のみ実行される
};
```

Try::Tinyは, `try`の次にある1つ目のコードリファレンスで例外が発生した場合, `catch`の次にある, 2つ目のコードリファレンスの処理を行う, という機能を提供するモジュールです.

これによって, `try`の中で例外が発生することなく無事に成功した場合, 最後に`$txn->commit()`を実行して変更を確定することができます.
また, 万が一`try`の中で例外が発生した場合には, `catch`で`$txn->rollback()`を実行して, 変更した内容を変更前の状態に巻き戻しすることが出来ます.

### トランザクションとロック

また, RDBMSとしてMySQLを利用しているのであれば, 次のようにして更新対象のデータをロックしてしまうと良いでしょう.

```perl
use Try::Tiny;

my $txn = $teng->txn_scope();

my $user1 = $teng->single( user => { id => 1 } );
my $user2 = $teng->single( user => { id => 2 } );

try {
    $user1->refetch({ for_update => 1 });
    $user2->refetch({ for_update => 1 });

    $user1->update({ name => 'super '.$user1->name });
    $user2->update({ name => 'super '.$user2->name });

    $txn->commit();
} catch {
    $txn->rollback();
};
```

`refetch`の際に, `{ for_update => 1 }`を与えることで, そのオブジェクトの元になったデータベース上のデータに対して, ロック(悲観的ロック)をかけることができます.
これによって, ロックをかけてから解除するまで, `$user1`や`$user2`に該当するデータベース上のデータは, 他の操作によって書き換えられないようにすることができます.
なお, このロックは, `$txn->commit()`するか`$txn->rollback()`することで自動的に解除されます.

# `inflate`と`deflate`

Tengには, `inflate`と`deflate`という仕組みがあります.
`inflate`を利用することで, Tengがデータベースからデータを取り出してオブジェクトにする際に任意のカラムの値に対して任意の処理を挟むことが出来ます.
`deflate`はその逆で, Tengからデータベースにデータを書き込む際に任意のカラムの値に対して任意の処理を挟むことが出来ます.

これを利用することで, データベース上ではDATETIME型などで保存している`created_at`などのカラムに格納されたデータを, Tengでオブジェクトにする段階でTime::Pieceのオブジェクトに変更したり, 逆にこのTime::Pieceのオブジェクトをデータベースに書き込む前に, DATETIME型に適応するフォーマットに変換してからデータベースに書き込んだりといった事が実現出来るようになります.

今回は, `user`テーブルの`created_at`と`updated_at`について,

- `inflate`を利用して, SQLiteに格納されたepoch秒をTengでオブジェクト化する際にTime::Pieceのオブジェクトにする
- `deflate`を利用して, Time::PieceのオブジェクトをSQLiteに書き込む前にepoch秒にする

という処理を実現してみます.

## `inflate`/`deflate`の設定

`inflate`及び`deflate`の設定は, スキーマファイルで行うことができます.
これらの設定は, 基本的にはテーブル単位で行います(スキーマファイルの, `table`に与えるコードリファレンスの中に記載する).

まずは`inflate`です.

```
    inflate qr/^.+_at$/ => sub {
        my ($col_value) = @_;
        Time::Piece->strptime($col_value, '%s');
    };
```

第1引数にフィルタリングするカラム名を, 第2引数にその処理をコードリファレンスで与えます.
第1引数は文字列の他, このように正規表現も与えることができます.
この場合は, `created_at`など, 末尾が`_at`で終わる全てのカラムについて処理を行う事になります.

第2引数のコードリファレンスには, フィルタリングしたカラムのデータが渡ってきます.
そして, このサブルーチンリファレンスの返り値が, 生成されるオブジェクトに格納されます.

今回の場合, `created_at`や`updated_at`はepoch秒が格納されているので, Tine::Pieceの`strptime`を利用して, epoch秒からTime::Pieceのオブジェクトを作成しています.

`deflate`も同様です.

```
    deflate qr/^.+_at$/ => sub {
        my ($col_value) = @_;
        $col_value->epoch;
    };
```

データベースに書き込むデータのうち, 第1引数で指定した条件を満たすカラムのデータは, 第2引数で与えたコードリファレンスに渡り, その返り値がデータベースに書き込まれます.

`inflate`が正しく動作していれば, `created_at`や`updated_at`にはTime::Pieceのオブジェクトが格納されているので, この`epoch`メソッドでepoch秒を取り出し, これを返すことによって, データベースにはepoch秒が数値で格納されることになります.

```perl:lib/MyApp/DB/Schema.pm
package MyApp::DB::Schema;
use strict;
use warnings;
use utf8;

use Teng::Schema::Declare;
use Time::Piece;

base_row_class 'MyApp::DB::Row';

table {
    name 'user';
    pk 'id';
    columns qw(id name created_at updated_at);

    inflate qr/^.+_at$/ => sub {
        my ($col_value) = @_;
        Time::Piece->strptime($col_value, '%s');
    };
    deflate qr/^.+_at$/ => sub {
        my ($col_value) = @_;
        $col_value->epoch;
    };
};

1;
```

それでは, 実際に`inflate`や`deflate`を使ってみましょう.
まず, `inflate`からです. 次のコードを実行してみましょう.

```perl:teng.pl(抜粋)
use Data::Dumper;
my $user = $teng->single('user' => { id => 1 });
print Dumper $user->created_at;
```

結果は次のようになります.

```
$ perl -Ilib teng.pl
$VAR1 = bless( [
                 28,
                 33,
                 7,
                 6,
                 4,
                 115,
                 3,
                 125,
                 0,
                 undef,
                 0
               ], 'Time::Piece' );
```

`$user->created_at`で, データベースに格納されているepoch秒ではなく, Time::Pieceのオブジェクトを得ることができました.

```perl:teng.pl(抜粋)
use Time::Piece;

my $user = $teng->single('user' => { id => 1 });
$user->update({
    name       => 'super papix',
    updated_at => scalar localtime,
});

$user->refetch;
print $user->updated_at->datetime;
```

次は, あるオブジェクトを更新するときに, `deflate`が有効か確かめてみましょう.
Time::Pieceを`use`すると, スカラコンテキストでの`localtime`の呼び出しで, 現在時刻のTime::Pieceのオブジェクトを取得することができるので, これを`updated_at`の更新内容として渡しています.

実行結果は次の通りになります(実際の時間より9時間ずれていますが, これはタイムゾーンの違いによるものです).

```
$ perl -Ilib teng.pl
2015-05-07T09:25:48
```

実際に`sqlite3`コマンドでデータベースの中身を確認してみます.

```
SQLite version 3.8.5 2014-08-15 22:37:57
Enter ".help" for usage hints.
sqlite> .headers on
sqlite> select * from user where id = 1;
id|name|created_at|updated_at
1|super papix|1430897608|1430990748
```

`updated_at`の値が正しく書き換わっていることがわかります.

# Rowオブジェクトの拡張

Tengを利用してデータベースからデータを取得する際, TengはTeng::Row(今回の場合は, Teng::Rowを拡張したMyApp::DB::Row)をベースにオブジェクトを生成します.
このMyApp::DB::Rowを利用して, オブジェクトを拡張することが可能です.

```perl:lib/MyApp/DB/Row.pm
package MyApp::DB::Row;
use strict;
use warnings;
use utf8;
use parent qw(Teng::Row);

1;
```

この名前空間にサブルーチンを書くと, そのサブルーチンがオブジェクトから利用できるようになります.
ここでは例として, `name`カラムのデータのprefixに`super`を付ける, `super_name`というメソッドを利用できるようにしてみましょう.

```
package MyApp::DB::Row;
use strict;
use warnings;
use utf8;
use parent qw(Teng::Row);

sub super_name {
    my ($obj) = @_;

    return 'super ' . $obj->name;
}

1;
```

それではスクリプトから利用してみましょう.

```perl:teng.pl(抜粋)
my $user = $teng->single('user' => { id => 1 });
print $user->name."\n";
print $user->super_name."\n";
```

実行結果は次のようになります.

```
$ perl -Ilib teng.pl
papix
super papix
```

## 特定のテーブルのオブジェクトのみの拡張

今回は, `MyApp::DB::Row`にサブルーチンを書いてオブジェクトを拡張しました.
この名前空間で用意したサブルーチンは, `single`や`search`メソッドを利用して取得した全てのオブジェクトで利用することができます.
しかし, 特定のテーブル(に関連するオブジェクト)だけで利用できるメソッドを提供したい, という場面もあるでしょう.

例えば, 先程用意した`super_name`メソッドを, `user`テーブルから取得したオブジェクトだけでのみ利用できるようにするには, `MyApp::DB::Row::User`という名前空間を利用し, 次のように書けばOKです.

```perl:lib/MyApp/DB/Row/User.pm
package MyApp::DB::Row::User;
use strict;
use warnings;
use utf8;
use parent qw(MyApp::DB::Row);

sub super_name {
    my ($obj) = @_;

    return 'super ' . $obj->name;
}

1;
```

`MyApp::DB::Row`以下に, テーブル名をCamelCaseにしたもの(例えば, `team_user`なら`TeamUser`)を設置すれば, Tengはそのテーブルからオブジェクトを生成する際にそのファイルを利用してくれます.

実行結果は次のようになります.

```
$ perl -Ilib teng.pl
papix
super papix
```

このように, Tengのオブジェクトは自由に拡張することができます.
かといって自由に拡張しすぎると, 逆に使い勝手が悪くなったり, コードの保守が難しくなってしまう可能性もあるでしょう.
そのため, 実際にオブジェクトを拡張する際は, 用途を絞って(例えば, あるカラムのデータをフィルタリングしたり, データの内容に応じて真偽値を返したり)利用することをおすすめします.

## 参考資料

- [Perl Advent Calendar 2011 - Teng Track](http://perl-users.jp/articles/advent-calendar/2011/teng/)
- Perl Hackers Hub [｢第30回 データベースプログラミング入門 - 汎用インタフェースDBIと, O/RマッパTengの使い方(3)｣](http://gihyo.jp/dev/serial/01/perl-hackers-hub/003003)
