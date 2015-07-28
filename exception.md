# はじめに

アプリケーションを開発する中で, ｢ある処理｣を行った時に異常事態が生じうるような場合は, 多々想定できると思います.
例えば, アプリケーションからMySQLなどのミドルウェアを操作しようとした時に, ミドルウェアが起動されていない場合, アプリケーションが行おうとしていたミドルウェアの操作は当然ながら実施することが出来ません.
また, アプリケーションからミドルウェアを操作できたとしても, 操作した結果ミドルウェアでエラーが発生するといったパターンもあるでしょう.

このような異常事態が発生した場合, アプリケーションは期待する挙動を実現出来なくなってしまいます.
そこで大抵のプログラミング言語には, このような異常を表現する｢例外｣という仕組みが提供されており, 例外をきっかけに現在の処理を中止して, 別の処理(例外処理)を行える仕組みが提供されています.

本稿では, まずPerlにおいて｢例外｣を実現する方法と, それを処理するための方法について解説します.
その後に, 例外をどのように使うべきか, 例外を使う際に気をつけるべき点を, 例外理論として概説します.

# 例外実装

## 例外の発生

まずはPerlで例外を発生させる方法について解説します.
ここでは例外を処理する方法については解説していない為, 基本的には例外が発生したタイミングでプログラムの処理は終了するようになっています.
発生した例外を受け取り, 任意の処理を実現する為の方法は, 次の｢例外の処理｣にて解説します.

### `die`

Perlで例外を発生させる一番基本的な方法は, `die`の利用です.

```perl:die.pl
use strict;
use warnings;

print "before\n";

die "die!";

print "after\n";
```

このスクリプトを実行すると, 実行結果は次のようになります.

```
$ perl die.pl
before
die! at die.pl line 6.
```

コードで`die`が実行されたタイミング例外が発生し, `die`の第1引数をエラーメッセージを出力して処理が中断していることがわかります.

### `Carp`

Carpモジュールは例外を発生させるモジュールの1つであり, 例外発生時に`die`よりも多くの情報を得ることができます.
例えば, 次のコードを実行してみましょう.

```perl:carp.pl
use strict;
use warnings;
use Carp;

main();

sub main {
    print "before\n";

    Carp::croak "die!";

    print "after\n";
}
```

結果は次の通りです.

```
before
die! at die.pl line 10.
        main::main() called at die.pl line 5
```

`die`で例外を発生させた時と同様, ｢10行目で例外が発生した｣という情報の他に, ｢この処理は5行目にある`main()`の呼び出しから発生した｣という情報も得ることができました.
Carpモジュールを利用すると, このように例外が発生した処理がどのような経緯で呼びだされたかを知ることが出来ます(このような情報を｢スタックトレース｣と呼びます).

このような特徴に加え, CarpモジュールはPerlのコアモジュールでもあるため, CPANモジュールを実装する際の例外発生手段として使われることが多いです.

### `Exception::Tiny`

Exception::Tinyを利用することで, 例外をクラスの形で定義出来るようになります.

```perl:exception.pl
use strict;
use warnings;

print "before\n";

Exception::Hoge->throw;

print "after\n";

package Exception::Hoge;
use parent 'Exception::Tiny';
```

ここでは, とある異常な処理が起きた事を表す`Exception::Hoge`というクラス(パッケージ)を用意し, このクラスの`throw`メソッドを利用して例外を発生させています.
今回は適当な名前がない為に`Exception::Hoge`としていますが, 実際は`MyApp::Exception::ResourceNotFound`(外部リソースを利用しようとして見つからなかった場合)であったり, `MyApp::DB::Exception::DuplicateException`(DBでUNIQUE制約のカラムが重複してしまった場合)などのように名前を付ける事になるでしょう.

さて, このコードを実行した場合, 結果は次のようになります.

```
$ perl exception.pl
before
Exception::Hoge at die.pl line 6.
```

`Exception::Hoge`というクラスで用意された例外が発生した, という出力が行われた後に処理が中断します.

### 使い分け

`*.pl`のようなスクリプトを実装する場合,, 大抵は`die`で事足りるでしょう.
小規模〜中規模なモジュールを実装する場合であれば, `die`ではなくCarpモジュールを利用する方が利用者に多くの情報を提供できるでしょう.
大規模なモジュールや, Webアプリケーションを開発する場面では様々な種類の例外が想定されるので, Exception::Tinyを利用し, クラスとして細かく例外を表現すると便利です.

## 例外の処理

例外が発生した際に行われる処理を｢例外処理｣と呼びます.
ここでは, 例外処理を記述する方法について解説します.

### `Try::Tiny`

`try`と`catch`のような構文を提供するモジュールです.

```perl
use Try::Tiny;

try {
    ...
} catch {
    ...
};
```

`try`の後の`{ ... }`と`catch`の後の`{ ... }`は, それぞれコードリファレンス(サブルーチンリファレンス)です.
1つ目の, `try`の後のサブルーチンメソッドを実行した際, その中で例外が発生した場合のみ, `catch`の次のサブルーチンリファレンスが実行されます.

Try::Tinyを利用した例外処理は, DBIやORMなどを利用してデータベースを操作する際に, トランザクションとそのロールバックを実装する為に利用することが多いです.

```perl
my $teng = MyApp::DB->new( ... );

my $txn = $teng->txn_scope;

try {

    ... DB操作 ...

    $txn->commit;
} catch {
    $txn->rollback;
};
```

この場合, `try`の次のコードリファレンスにある｢DB操作｣の部分のコードで例外(`$teng`を利用したデータベース操作の失敗)が発生した場合, `catch`の次のコードリファレンスにある`$txn->rollback`が実行されることによって, ｢DB操作｣で行われたデータベース操作を, 操作する前の段階の内容に巻き戻すことが可能です.

なお, 発生した例外の情報は, `catch`の次のサブルーチンリファレンスに第1引数として渡されます.

```perl:exception.pl
use strict;
use warnings;
use Try::Tiny;
use Data::Dumper;

try {
    MyException::A->throw;
} catch {
    my ($e) = @_;
    print Dumper $e;
};

package MyException::A;
use parent 'Exception::Tiny';
```

実行結果は以下の通りです.

```
$ perl exception.pl
$VAR1 = bless( {
                 'subroutine' => 'main::try {...} ',
                 'package' => 'main',
                 'message' => 'MyException::A',
                 'line' => 7,
                 'file' => 'exception.pl'
               }, 'MyException::A' );
```

このように, `Exception::Tiny`を利用して定義した, `MyExceotion::A`という例外のオブジェクトが渡ってきています.

### `Try::Lite`

さて, Try::Tinyは簡単に例外処理を実装することが可能ですが, 少し複雑な例外処理を実装する為には向いていません.
例えば, 例外の種類によって例外処理の内容を変更したい場合は, 少し煩雑なコードを書かねばなりません.
このような場合は, Try::Liteを利用することを推奨します.

例えば, `Exception::Tiny`を利用して, `MyException::A`と`MyException::B`という2つの例外を用意したとします.
この際, `MyException::A`と`MyException::B`で異なる例外処理を行いたい場合, Try::Liteを利用して次のように実装することが出来ます.

```perl
use strict;
use warnings;
use Try::Lite;

try {
    MyException::A->throw;
} (
    'MyException::A' => sub {
        print "MyException::A\n";
    },
    'MyException::B' => sub {
        print "MyException::B\n";
    },
);

package MyException::A;
use parent 'Exception::Tiny';

package MyException::B;
use parent 'Exception::Tiny';
```

Try::Liteでは, `try`の次のサブルーチンリファレンスを実行した際, 例外が発生した場合はその次の`( ... )`で指定された例外処理を実行します.
`( ... )`は例外の名前(パッケージ名)とサブルーチンリファレンスのハッシュ形式になっており, 発生した例外がハッシュのキーとなっている例外と一致した場合, これに対応するサブルーチンリファレンスが実行されます.

ここでは, `try`の次のサブルーチンで`MyException::A`の例外が発生しているので, `MyException::A`に対応したサブルーチンリファレンスが実行され, ｢MyException::A｣が表示されます.
`MyException::A`ではなく`MyException::B`の例外が発生した場合は, 同様に`MyExceotion::B`に対応したサブルーチンリファレンスが実行され, ｢MyException::B｣が表示されます.

Try::Liteを利用すれば, 例えば｢同時にMySQLとRedisを操作する｣という場合に, ｢MySQLの操作で例外が発生した場合｣と｢Redisの操作で例外が発生した場合｣によって, 例外処理の内容を変更することが可能です.

なお, Try::Liteでは, 発生した例外を受け取る方法がTry::Liteとは異なります.
Try::Liteでは, 発生した例外は`$@`という変数を通じて受け取ります.

```perl
try {
    MyException::A->throw;
} (
    'MyException::A' => sub {
        print "MyException::A\n";
        $@->rethrow; # 例外処理をした後, 更に例外を投げる
    },
    'MyException::B' => sub {
        print "MyException::B\n";
    },
);
```

また, Try::Liteでは全ての例外に対する例外処理は, 次のように書く事が可能です.

```perl
try {
    ...
} (
    '*' => sub {
        ...
    },
);
```

但しこの場合であれば, Try::Liteを使わずともTry::Tinyで実装可能です.
Try::Liteが有用なのは, 次のように｢Exception::Hoge｣と｢それ以外｣の例外で, 処理を切り替えることが出来るという点です.

```perl
try {
    ...
} (
    'MyException::A' => sub {
        ... # MyException::Aの例外が起きた時のみ実行
    },
    '*' => sub {
        ... # MyException::A以外の例外が起きた時のみ実行
    },
);
```

### 例外の再送

Try::Tinyでは`catch`の次に記述する, 例外処理のためのサブルーチンリファレンスの第1引数として, Try::Liteでは`$@`を利用して, 発生した例外の詳細を取得することができました.

例外を`Exception::Tiny`を利用してクラスとして定義した場合, 発生した例外の詳細として, その例外のオブジェクトが渡ってきます.
このように取得した例外のオブジェクトからは, `rethrow`メソッドが利用できるようになっています.
このメソッドを利用することで, 例外を再度発生することが可能です(更に上に例外を渡す).

```perl:exception.pl
use strict;
use warnings;
use Try::Tiny;
use Data::Dumper;

try {
    MyException::A->throw;
} catch {
    my ($e) = @_; # `$e`には, 発生した`MyExceotion::A`のオブジェクトが渡ってくる
    $e->rethrow; # 再度例外(この場合は`MyExceotion::A`)が発生!
};

package MyException::A;
use parent 'Exception::Tiny';
```

実行結果は以下の通りです.

```
$ perl exception.pl
MyException::A at exception.pl line 7.
```

また, `$e->rethrow`の代わりに, `die $e`や`Carp::croak $e`と書いても, 同じような結果を得ることが可能です.
基本的に, 例外を再度発生させる場合, `die`や`Carp::croak $e`を利用して再送すると良いでしょう.

例えば, 次のように`die`を利用して例外を発生させた場合, 例外の詳細は文字列として渡ってきます.

```perl:exception.pl
use strict;
use warnings;
use Try::Tiny;
use Data::Dumper;

try {
    die "Exception!";
} catch {
    my ($e) = @_;
    print Dumper $e;
};
```

実行結果は次の通りです.

```
$ perl exception.pl
$VAR1 = 'Exception! at exp.pl line 7.
';
```

このように, `$e`は例外のオブジェクトではなく文字列になるため, `$e->rethrow`で例外を再発生することは出来ません.
そのため, `die`や`Carp::croak`を利用すれば, `$e`が例外のオブジェクトであっても, 文字列であっても, 同様に例外を発生させることが可能です.

### モジュールにおける例外処理

ここまで, Try::TinyやTry::Liteを利用した例外処理の実装方法について解説しました.
実は, ここまで学んできたPerlのモジュールの中には, 標準で例外処理の仕組みを持ったものも存在します.
ここでは, それらの利用方法について解説を行います.

#### `Teng`

ORMとしてTengを利用する場合, 次のように`handle_error`メソッドを用意することで, データベースを操作する際などに例外が発生したときの例外処理を書くことが可能です.

例えば, TengでSQL文を直接入力して検索できる`search_by_sql`において,

```perl
$teng->search_by_sql('SELECT * FOR user WHERE id = ?', [ 1 ]);
```

のように誤ったSQL(`FROM`が`FOR`になっている)を実行した場合, Tengは次のようなエラーメッセージを出力します.

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@@@@@ Teng 's Exception @@@@@
Reason  : DBD::SQLite::db prepare failed: near "FOR": syntax error at ...

SQL     : SELECT * FOR user WHERE id = ?
BIND    : $VAR1 = [
          1
        ];

@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
 at ...
```

これは, Tengの`handle_error`メソッドで出力されているので, `MyApp::DB`に`handle_error`メソッドを用意することでオーバーライドすることが可能です.

```perl:lib/MyApp/DB.pm
package MyApp:DB;
use utf8;
use warnings;
use parent 'Teng';
use Carp;

...

sub handle_error {
    my ($self, $stmt, $bind, $reason) = @_;

    Carp::croak $reason;
}

1;
```

例えば, このように書き換えた場合は次のような出力になります.

```
DBD::SQLite::db prepare failed: near "FOR": syntax error at ...
 at ...
```

`handle_error`の利用例として, 例えば次のようなコードを用意すれば, UNIQUE制約を満たさなかった場合と, それ以外のエラーでTengから発生する例外を使い分けることが出来ます.

```perl:lib/MyApp/DB.pm
package MyApp:DB;
use utf8;
use warnings;
use parent 'Teng';
use Carp;

...

sub handle_error {
    my ($self, $stmt, $bind, $reason) = @_;

    if ($reason =~ /DBD::mysql::st execute failed: Duplicate entry/) {
        MyApp::DB::Exception::DuplicateException->throw;
    } else {
        MyApp::DB::Exception->throw;
    }
}

package MyApp::DB::Exception;
use parent 'Exception::Tiny';

package MyApp::DB::Exception::DuplicateException;
use parent -norequire 'MyApp::DB::Exception';

1;
```

#### `Amon2`

Amon2では, ディスパッチャ(Basic Flavorの場合, `MyApp::Web::Dispatcher`)に`handle_exception`メソッドを定義することで, ディスパッチャのレイヤーで例外操作を行うことが可能です.

```perl:lib/MyApp/Dispatcher.pm
package MyApp::Web::Dispatcher;
use strict;
use warnings;
use utf8;
use Amon2::Web::Dispatcher::RouterBoom;

any '/' => sub {
    ...
};

sub handle_exception {
    my ($class, $c, $e) = @_;

    ...
}

1;
```

例えば, `MyApp::Exception::ValidationError`が発生した場合は400を, それ以外の場合500を返したい場合であれば, 次のように実装することが可能です.

```perl:lib/MyApp/Dispatcher.pm
package MyApp::Web::Dispatcher;
use strict;
use warnings;
use utf8;
use Amon2::Web::Dispatcher::RouterBoom;

any '/' => sub {
    ...
};

sub handle_exception {
    my ($class, $c, $e) = @_;

    if (UNIVERSAL::isa($e, 'MyApp::Exception::ValidationError')) {
        return $c->create_simple_status_page(400, 'BAD REQUEST');
    } else {
        return $c->res_500;
    }
}

1;
```

第2引数はAmon2のコンテキスト, そして第3引数に発生した例外の詳細が渡ってくるので, これらを利用して任意の処理を記述することが出来ます.
上記の例にもあるように, バリデーションエラーは発生したタイミングですぐキャッチしてリカバリせず, この`handle_exception`を利用して処理するようにすれば, 例外処理のためのコードを`handle_exception`に集約することが可能です.

### 例外のテスト

｢例外が起きたかどうか｣をテストするためには, Test::Exceptionの利用が楽です.

#### `dies_ok`

`dies_ok`は, 任意のコードで例外が発生するかをテストすることが可能です.

```perl:exception.t
use Test::More;
use Test::Exception;

dies_ok { die } 'Exception';

done_testing;
```

`dies_ok`は, 1つのサブルーチンリファレンスを受け取り, その中の処理で例外が発生していればテストが通ります.
`dies_ok`では, 例外であればどのような例外が発生してもテストが通りますので, その詳細を確認したい場合, 後述の`throws_ok`を利用しましょう.

#### `throws_ok`

`throws_ok`は, `dies_ok`と同じく1つのサブルーチンリファレンスを受け取り, その中で例外が発生するかをテストすることが出来ます.
`dies_ok`と異なり, 第2引数にどのような例外が発生したかを指定することが可能です(第1引数であるサブルーチンリファレンスと第2引数の間に`,`を入れてはいけません).

正規表現リテラル`qr//`を与えた場合, 正規表現リテラルが例外のメッセージと一致するかどうかを確認することが出来ます.
また, 文字列で例外クラスのクラス名を与えることができ, 発生した例外がそのクラスのものかどうかを確認することが出来ます.

```perl:exception.t
use Test::More;
use Test::Exception;

throws_ok { die "Exception!" } qr/Exception/;
throws_ok { die "Exception!" } qr/xxxxxxxxx/;

throws_ok { MyException::A->throw } 'MyException::A';
throws_ok { MyException::A->throw } 'MyException::B';

done_testing;

package MyException::A;
use parent 'Exception::Tiny';
```

実行結果は次の通りです.

```
$ perl exception.t
ok 1 - threw Regexp ((?^:Exception))
not ok 2 - threw Regexp ((?^:xxxxxxxxx))
#   Failed test 'threw Regexp ((?^:xxxxxxxxx))'
#   at 2.t line 5.
# expecting: Regexp ((?^:xxxxxxxxx))
# found: Exception! at 2.t line 5.
ok 3 - threw MyException::A
not ok 4 - threw MyException::B
#   Failed test 'threw MyException::B'
#   at 2.t line 8.
# expecting: MyException::B
# found: MyException::A at 2.t line 8.
1..4
# Looks like you failed 2 tests of 4.
```

# 例外の考え方

最後に, 例外を利用する際に意識しておくと良いポイントを簡単にまとめておきます.

まず, ｢例外が発生した｣ということは, 処理中に何らかの異常が発生したということです.
これをそのまま無視するのはプロダクトの品質に影響を及ぼす可能性があります.
例えば, データベースを操作する際に例外が発生した場合, そのままにしておくと予期せぬデータがデータベース挿入されるなど, データの不整合が発生するかもしれません.
また, 場合によってはサービスそのものやミドルウェアの負荷向上であったり, セキュリティの問題にまで至る可能性もあるでしょう.
例外が発生した場合, それを無視せず, 極力必要なレイヤーで例外処理を行ってリカバリーするようにしましょう.

しかしながら, 状況によってはリカバリー出来なかったり, 或いはリカバリーに失敗してしまう場合もあるでしょう.
その場合, 例外を無理に隠蔽してプログラムを動かし続けてはいけません.
再度例外を投げるなどして, 必ず｢処理が中断｣する(そして, Webアプリケーションの場合は500を返したり, 異常事態が発生した事をユーザに伝える処理を入れる)ようにしましょう.

# 参考資料

- [例外設計における大罪](http://www.slideshare.net/t_wada/exception-design-by-contract)
- [最近のPerl例外厨事情](http://www.songmu.jp/riji/entry/2013-08-05-exception.html)
- Teng Advent Calendar 2011 [error handling](http://perl-users.jp/articles/advent-calendar/2011/teng/17)
