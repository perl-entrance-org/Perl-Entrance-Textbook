# Perlのテスト

OOP入門で作成した`PerlEntrance::OOPTutorial`というモジュールを題材に, Perlというプログラミング言語におけるテストの実装方法と実施方法について学んでいきます.

# テストの実行

まず, テストを実行する方法について学びます.
`minil new`でモジュールのひな形を作成した場合, 最初から`t`ディレクトリに`00_compile.t`というテストが設置されています.

```perl:t/00_compile.t
use strict;
use Test::More 0.98;

use_ok $_ for qw(
    PerlEntrance::OOPTutorial
);

done_testing;
```

詳しい内容の解説は後にして, とりあえずこのテストを実行してみます.
このテストもPerlのスクリプトですので, `perl`コマンドで実行することもできますが, テストを実行する場合は`prove`コマンドを利用するのが簡単です.

```
$ ls
Build.PL   Changes    LICENSE    META.json  README.md  cpanfile   eg         lib        minil.toml t

$ prove -l t
[17:45:29] t/00_compile.t .. ok       27 ms
[17:45:29]
All tests successful.
Files=1, Tests=1,  0 wallclock secs ( 0.03 usr  0.01 sys +  0.02 cusr  0.01 csys =  0.07 CPU)
Result: PASS
```

`perl`コマンドで実行する場合と比べて, `prove`コマンドでテストを実行すると以下のメリットがあります.

- `-l`で`-Ilib`と同等のモジュールのディレクトリ指定ができる
- .provercに記述することで自動的にオプションを追加してくれて毎回指定する必要がない
- ディレクトリをテスト対象とすれば, 再帰的にテストを検索し, 実行してくれる

このため, 基本的には`prove -l t`でテストを実行すれば問題ないようになっています.

```
$ prove -l t/00_compile.t
[17:47:29] t/00_compile.t .. ok       20 ms
[17:47:29]
All tests successful.
Files=1, Tests=1,  0 wallclock secs ( 0.02 usr  0.01 sys +  0.02 cusr  0.00 csys =  0.05 CPU)
```

なお, このように, 任意のテストスクリプトを個別に実行することも可能です.

# Test::More

Perlでは, `Test::More`というモジュールを利用してテストを書くことが一般的です.
Test::MoreはPerlのコアモジュール(デフォルトでインストールされており, `cpanm`コマンド等で別途インストールする必要がない)であり, 多くのユーザが利用しているテスト用モジュールです.
`minil new`した際に生成されるテストスクリプトも, この`Test::More`を利用して実装されています.

Test::Moreの詳細については[POD](https://metacpan.org/pod/Test::More)が参考になりますが, ドキュメントの日本語訳が[perldoc.jp](http://perldoc.jp/)で[提供](http://perldoc.jp/docs/modules/Test-Simple-0.99/lib/Test/More.pod)されているので, こちらを参考にするのも良いでしょう(日本語訳は少し古いバージョンの日本語訳となっているので, 注意しましょう).

`Test::More`を利用したテストは, 次のように記述します.

```perl:t/00_compile.t
use strict;
use Test::More 0.98;

... テスト ...

done_testing;
```

`Test::More`を利用したテストでは, スクリプトの最後に`done_testing;`を記述するようにしましょう.
これは, テストが終了したことを`Test::More`に伝えるコードであり, `Test::More`を利用したテストにおける｢お約束｣と思っておいて下さい.

それでは, `Test::More`が提供するテスト用サブルーチンを紹介していきます.
これらのサブルーチンは, `Test::More`を`use`するだけで利用できるようになります.

## `use_ok`

まず, `minil new`でモジュールのひな形を生成したときに生成される, `t/00_compile.t`に最初から記載されている, `use_ok`について解説します.

```perl:t/00_compile.t
use strict;
use Test::More 0.98;

use_ok $_ for qw(
    PerlEntrance::OOPTutorial
);

done_testing;
```

`use_ok`は, 引数として与えたモジュールが正しく`use`できるかをテストするメソッドです.
ここでは, `PerlEntrance::OOPTutorial`のテストのみを行っているので, `PerlEntrance::OOPTutorial::Engineer`についてもテストするように書き換えてみましょう.

```perl:t/00_compile.t
use strict;
use Test::More 0.98;

use_ok $_ for qw(
    PerlEntrance::OOPTutorial
    PerlEntrance::OOPTutorial::Engineer
);

done_testing;
```

テストを実行してみます.

```
$ prove -l t
[17:53:01] t/00_compile.t .. ok       22 ms
[17:53:01]
All tests successful.
Files=1, Tests=2,  0 wallclock secs ( 0.03 usr  0.01 sys +  0.03 cusr  0.01 csys =  0.08 CPU)
Result: PASS
```

もし, テストに失敗した場合は次のような出力が得られます(`PerlEntrance::OOPTutorial`で, 最後に`1`ではなく`0`を返すように書き換えました).

```
[17:54:04] t/00_compile.t .. 1/?
not ok 1 - use PerlEntrance::OOPTutorial;
[17:54:04] t/00_compile.t .. Dubious, test returned 1 (wstat 256, 0x100)
Failed 1/2 subtests
[17:54:04]

Test Summary Report
-------------------
t/00_compile.t (Wstat: 256 Tests: 2 Failed: 1)
  Failed test:  1
  Non-zero exit status: 1
Files=1, Tests=2,  0 wallclock secs ( 0.03 usr  0.00 sys +  0.02 cusr  0.01 csys =  0.06 CPU)
Result: FAIL
```

## `is`

`is`は, 単純に値を比較するテストです.
`is`を利用したテストを, `PerlEntrance::OOPTutorial::Engineer`についてテストを行う, `01_engineer.t`を`t`ディレクトリで試してみます.

```perl:t/01_engineer.t
use strict;
use Test::More 0.98;

use PerlEntrance::OOPTutorial::Engineer;

my $papix = PerlEntrance::OOPTutorial::Engineer->new( name => 'papix' );
my $hoto  = PerlEntrance::OOPTutorial::Engineer->new( name => 'hoto' );

is $papix->name, 'papix';
is $hoto->name,  'hoto';

done_testing;
```

`is`は, 第1引数と第2引数が等しければテストが通り, 異なっていればテストが失敗します.
基本的に, テストしたい値を第1引数に, 期待値を第2引数に与えることが多いです.

## `is_deeply`

`is_deeply`は, リファレンスを比較する際に利用するサブルーチンです.

```perl:test_more.t
use strict;
use warnings;
use Test::More;

my $hashref = { test => 'more' };

is $hashref, { test => 'more' };

done_testing;
```

例えば次のようなテストを書いた場合, テストを実行すると失敗します.

```
$ perl test_more.t
not ok 1
#   Failed test at test_more.pl line 7.
#          got: 'HASH(0x7fec4c0032b8)'
#     expected: 'HASH(0x7fec4d007ff0)'
1..1
# Looks like you failed 1 test of 1.
```

これは, リファレンスに格納されたデータとしては等しいですが, `$hashref`と`{ test => 'more' }`は異なるリファレンスを指しているためです.

`is_deeply`は, `is`とは異なり, リファレンスに格納されたデータを全て比較し, 等しいか否かを判定します. そのため,

```perl:test_more.t
use strict;
use warnings;
use Test::More;

my $hashref = { test => 'more' };

is_deeply $hashref, { test => 'more' };

done_testing;
```

このようなテストにすれば, `$hashref`に格納されたデータと`{ test => 'more' }`が表すデータは等しいので,

```
ok 1
1..1
```

テストが通ります.

## `like`

`like`は, 第1引数の中に第2引数の正規表現が示す文字列が含まれるかをテストするサブルーチンです.

```perl:test_more.t
use strict;
use warnings;
use Test::More;

like 'Engineer', qr/ee/;

done_testing;
```

例えば, このように記述すると, 第1引数の`Engineer`という文字列が, `qr/ee/`という正規表現にマッチするか, つまり`ee`という文字列を含むかどうかをテストすることができます.

## `can_ok`

`can_ok`は, あるオブジェクトが第1引数で指定したメソッドを持っているかをテストするサブルーチンです.
例えば, `PerlEntrance::OOPTutorial::Engineer`から生成したオブジェクトが, `name`メソッドを実行できることを確かめるには,

```perl:t/01_engineer.t
use strict;
use Test::More 0.98;

use PerlEntrance::OOPTutorial::Engineer;

my $papix = PerlEntrance::OOPTutorial::Engineer->new( name => 'papix' );

can_ok $papix, 'name';

done_testing;
```

のようなテストを書くことで確認できます.

## `isa_ok`

`isa_ok`は, あるオブジェクトが第1引数で指定した名前空間に紐付いているかをテストするサブルーチンです.
例えば, `PerlEntrance::OOPTutorial::Engineer`から生成したオブジェクトは, `PerlEntrance::OOPTutorial::Engineer`という名前空間に紐付いているはずです.
これを確かめるには,

```perl:t/01_engineer.t
use strict;
use Test::More 0.98;

use PerlEntrance::OOPTutorial::Engineer;

my $papix = PerlEntrance::OOPTutorial::Engineer->new( name => 'papix' );

isa_ok $papix, 'PerlEntrance::OOPTutorial::Engineer';

done_testing;
```

のようなテストを書くことで確認できます.

## `subtest`

`subtest`を利用することで, テストスクリプト内のテストをまとめることができます.

```perl:t/01_engineer.t
use strict;
use Test::More 0.98;

use PerlEntrance::OOPTutorial::Engineer;

my $papix = PerlEntrance::OOPTutorial::Engineer->new( name => 'papix' );
my $hoto  = PerlEntrance::OOPTutorial::Engineer->new( name => 'hoto' );

can_ok $papix, 'name';

isa_ok $papix, 'PerlEntrance::OOPTutorial::Engineer';

is $papix->name, 'papix';
is $hoto->name,  'hoto';

done_testing;
```

このテストは, `subtest`を使えば, 次のようにまとめることができます.

```perl:t/01_engineer.t
use strict;
use Test::More 0.98;

use PerlEntrance::OOPTutorial::Engineer;

my $papix = PerlEntrance::OOPTutorial::Engineer->new( name => 'papix' );
my $hoto  = PerlEntrance::OOPTutorial::Engineer->new( name => 'hoto' );

subtest q{PerlEntrance::OOPTutorial::Engineerが正しく実装されていること} => sub {
    isa_ok $papix, '';
    can_ok $papix, 'name';
};

subtest q{nameメソッドが正しく実装されていること} => sub {
    is $papix->name, 'papix';
    is $hoto->name,  'hoto';
};

done_testing;
```

`prove`コマンドで実行する際, `-v`オプションを指定すると, `subtest`ごとのテスト結果を出力してくれます.

```
$ prove -l -v t/01_engineer.t
[18:21:18] t/01_engineer.t ..
    # Subtest: PerlEntrance::OOPTutorial::Engineerが正しく実装されていること
    ok 1 - An object of class 'PerlEntrance::OOPTutorial::Engineer' isa 'PerlEntrance::OOPTutorial::Engineer'
    ok 2 - PerlEntrance::OOPTutorial::Engineer->can('name')
    1..2
ok 1 - PerlEntrance::OOPTutorial::Engineerが正しく実装されていること
    # Subtest: nameメソッドが正しく実装されていること
    ok 1
    ok 2
    1..2
ok 2 - nameメソッドが正しく実装されていること
1..2
ok       24 ms
[18:21:18]
All tests successful.
Files=1, Tests=2,  0 wallclock secs ( 0.03 usr  0.01 sys +  0.03 cusr  0.01 csys =  0.08 CPU)
Result: PASS
```

また, `subtest`は, 次のようにネストすることも可能です.

```perl:t/01_engineer.t
use strict;
use Test::More 0.98;

use PerlEntrance::OOPTutorial::Engineer;

my $papix = PerlEntrance::OOPTutorial::Engineer->new( name => 'papix' );
my $hoto  = PerlEntrance::OOPTutorial::Engineer->new( name => 'hoto' );

subtest q{PerlEntrance::OOPTutorial::Engineerが正しく実装されていること} => sub {
    subtest q{オブジェクトがPerlEntrance::OOPTutorial::Engineerに紐付いていること} => sub {
        isa_ok $papix, 'PerlEntrance::OOPTutorial::Engineer';
    };
    subtest q{オブジェクトがnameメソッドを利用可能であること} => sub {
        can_ok $papix, 'name';
    };
};

subtest q{nameメソッドが正しく実装されていること} => sub {
    is $papix->name, 'papix';
    is $hoto->name,  'hoto';
};

done_testing;
```

実行結果は次のようになります.

```
$ prove -l -v t/01_engineer.t
[18:22:28] t/01_engineer.t ..
    # Subtest: PerlEntrance::OOPTutorial::Engineerが正しく実装されていること
        # Subtest: オブジェクトがPerlEntrance::OOPTutorial::Engineerに紐付いていること
        ok 1 - An object of class 'PerlEntrance::OOPTutorial::Engineer' isa 'PerlEntrance::OOPTutorial::Engineer'
        1..1
    ok 1 - オブジェクトがPerlEntrance::OOPTutorial::Engineerに紐付いていること
        # Subtest: オブジェクトがnameメソッドを利用可能であること
        ok 1 - PerlEntrance::OOPTutorial::Engineer->can('name')
        1..1
    ok 2 - オブジェクトがnameメソッドを利用可能であること
    1..2
ok 1 - PerlEntrance::OOPTutorial::Engineerが正しく実装されていること
    # Subtest: nameメソッドが正しく実装されていること
    ok 1
    ok 2
    1..2
ok 2 - nameメソッドが正しく実装されていること
1..2
ok       29 ms
[18:22:28]
All tests successful.
Files=1, Tests=2,  0 wallclock secs ( 0.03 usr  0.01 sys +  0.03 cusr  0.00 csys =  0.07 CPU)
Result: PASS
```

適切な粒度でテストスクリプトを分割するのも重要ですが, 1つのテストスクリプト内でも, `subtest`を利用してテストをグルーピングし, 何のテストをしているのか分かりやすくするのも重要です.

# テストを支援するモジュール

`Test::More`はPerlのテストを実装する為のコアを担うモジュールですが, それ以外にもテストの実装をサポートするモジュールが提供されています.
その中でもよく利用されるものを紹介します.

## `Test::MockTime`

時間に関連するテストを書く際に便利なモジュールです.
例えば, Perlでは現在のepoch秒を`time()`で取得することができます.
例として, 現在のepoch秒から1分後のepoch秒を取得するサブルーチン, `plus1min`と, それに対するテストを書いてみるとします.

```perl:test.t
use strict;
use warnings;
use Test::More;

is plus1min(), time + 60;

sub plus1min {
    time + 60;
}

done_testing;
```

これを, `Test::MockTime`を利用すると, 次のように書くことができます.

```perl:test.t
use strict;
use warnings;
use Test::More;
use Test::MockTime qw/ set_fixed_time /;

set_fixed_time(0);

is plus1min(), 60;

sub plus1min {
    time + 60;
}

done_testing;
```

`Test::MockTime`が提供する`set_fixed_time`を利用すると, このテストスクリプト中でのepoch秒を任意の値に固定(上記のコードの場合, `0`)に固定することができます.

`set_fixed_time`を利用した場合, それ以降はいくら時間が経過しようと`set_fixed_time`で指定した時間のまま固定されます.
そのため, 次のテストは失敗せず, 正しく終了します.

```perl:test.t
use strict;
use warnings;
use Test::More;
use Test::MockTime qw/ set_fixed_time /;

set_fixed_time(0);

is plus1min(), 60;
sleep 10; # 10秒経過
is plus1min(), 60; # 10秒経過しても, `set_fixed_time`で現在のepoch秒が0に固定されているので, テストが通る

sub plus1min {
    time + 60;
}

done_testing;
```

一方, `Test::MockTime`が提供する`set_absolute_time`は, 現在のepoch秒を`set_absolute_time`の引数に変更します.
それ以降は1秒ごとにepoch秒がインクリメントするので,

```perl:test.t
use strict;
use warnings;
use Test::More;
use Test::MockTime qw/ set_absolute_time /;

set_absolute_time(0);

is plus1min(), 60;
sleep 10; # 10秒経過
is plus1min(), 60; # 10秒経過したので, timeは0ではなく10を返すので, plus1minは70になる

sub plus1min {
    time + 60;
}

done_testing;
```

このテストは失敗します.

```
$ perl test.t
ok 1
not ok 2
#   Failed test at test.t line 10.
#          got: '70'
#     expected: '60'
1..2
# Looks like you failed 1 test of 2.
```

`set_fixed_time`や`set_absolute_time`で変更したepoch秒は, `restore_time`で復元することができます.

```perl:test.t
use strict;
use warnings;
use Test::More;
use Test::MockTime qw/ set_fixed_time restore_time /;

set_fixed_time(0);

is plus1min(), 60;
restore_time();
is plus1min(), 60; # restore_time()したので, timeは0ではなく現在のepoch秒を返す

sub plus1min {
    time + 60;
}

done_testing;
```

実行結果は次の通りとなります.

```
$ perl test.t
ok 1
not ok 2
#   Failed test at test.t line 10.
#          got: '1430826672'
#     expected: '60'
1..2
# Looks like you failed 1 test of 2.
```

## `Test::Mock::Guard`

任意の名前空間の任意のサブルーチンを上書きすることができます.

```perl:test.t
package Some::Class;
use strict;
use warnings;

sub hoge { 'foofoo' }
sub fuga { 'barbar' }

package main;
use strict;
use warnings;
use Test::More;
use Test::Mock::Guard qw/ mock_guard /;

my $guard = mock_guard(
    'Some::Class' => {
        hoge => sub { 'foo' },
        fuga => sub { 'bar' },
    },
);

is Some::Class::hoge(), 'foo';
is Some::Class::fuga(), 'bar';

done_testing;
```

`Some::Class`という名前空間で, `foofoo`を返す`hoge`というサブルーチンと, `barbar`を返す`fuga`というサブルーチンを定義しています.
一方`main`名前空間では, `Test::Mock::Guard`を利用して, これらのサブルーチンを上書きしています.

```perl:test.t(抜粋)
my $guard = mock_guard(
    'Some::Class' => {
        hoge => sub { 'foo' },
        fuga => sub { 'bar' },
    },
);
```

ここでは, `Some::Class`という名前空間に存在するサブルーチンについて, `hoge`というサブルーチンは`sub { 'foo' }`で, `fuga`というサブルーチンは`sub { 'bar' }`で上書きしています.

そのため,

```perl:test.t(抜粋)
is Some::Class::hoge(), 'foo';
is Some::Class::fuga(), 'bar';
```

というテストコードにおいて, `Some::Class::hoge()`と`Some::Class::fuga()`はそれぞれ`foo`と`bar`を返すので, このテストは問題なくパスします.

```
$ perl test.t
ok 1
ok 2
1..2
```

なお, `Test::Mock::Guard`を利用したサブルーチンの上書きは, `mock_guard`を利用したスコープ内のみとなります.

```perl:test.t
package Some::Class;
use strict;
use warnings;

sub hoge { 'foofoo' }
sub fuga { 'barbar' }

package main;
use strict;
use warnings;
use Test::More;
use Test::Mock::Guard qw/ mock_guard /;

subtest 'mock' => sub {
    my $guard = mock_guard(
        'Some::Class' => {
            hoge => sub { 'foo' },
            fuga => sub { 'bar' },
        },
    );
    is Some::Class::hoge(), 'foo'; #
    is Some::Class::fuga(), 'bar'; # mock_guardと同じスコープなので, Test::Mock::Guardで上書きしたサブルーチンが呼び出される
};

is Some::Class::hoge(), 'foo'; #
is Some::Class::fuga(), 'bar'; # mock_guardと同じスコープではないので, Test::Mock::Guardで上書きする前のサブルーチンが呼ばれる

done_testing;
```

そのため, このテストの実行結果は次のようになります.

```
$ perl test.t
    # Subtest: mock
    ok 1
    ok 2
    1..2
ok 1 - mock
not ok 2
#   Failed test at test.t line 26.
#          got: 'foofoo'
#     expected: 'foo'
not ok 3
#   Failed test at test.t line 27.
#          got: 'barbar'
#     expected: 'bar'
1..3
# Looks like you failed 2 tests of 3.
```

テストスクリプトから外部のリソース(例えばAPIや, MySQLやRedis, Dockerなど様々なミドルウェアなど)をPerlから利用するとき, 毎回直接リソースを操作していると時間がかかりますし, そもそもテストを実行する度に, 外部リソースを適切な状態に初期化する必要があります.
また, 外部のAPIを利用する場合はAPIに対して負荷がかかってしまいます(また, 万が一APIが落ちている場合はテストが出来なくなってしまいます).

このような場合に, Test::Mock::Guardを利用して, ｢本来外部リソースが返すべき値｣を返すように外部リソースへアクセスするサブルーチンを上書きして, 実行時間の短縮等を狙うことが可能です.

なお, APIなどのURLアクセスをモックする場合, [Test::Mock::LWP](https://metacpan.org/pod/Test::Mock::LWP)や[Test::Mock::Furl](https://metacpan.org/pod/Test::Mock::Furl)などを利用することも出来ます.

## `Capture::Tiny`

任意のコードについて, そのコード内で出力される標準出力や標準エラー出力の内容を取得することができます.
例えば, 次のコードのサブルーチン`p`は, 標準出力に`hello!`という文字列を出力するサブルーチンです.

このサブルーチンが, 正しく`hello!`を出力しているかを確かめる為には, `Capture::Tiny`を利用して, 次のように書くことができます.

```
use strict;
use warnings;
use Test::More;
use Capture::Tiny qw/ tee /;

my ($stdout, $stderr) = tee {
    p();
};

is $stdout, 'hello!';

sub p {
    print "hello!";
}

done_testing;
```

## `Test::Pretty` / `App::Prove::Plugin::Pretty`

テストの出力を整形してくれるモジュールです.
`prove`コマンドからは, `App::Prove::Plugin::Pretty`を利用すると便利です.

`PerlEntrance::OOPTutorial::Engineer`のために書いた`t/01_engineer.t`を例にして, `App::Prove::Plugin::Pretty`を利用した時の出力を紹介します.

```perl:t/01_engineer.t
use strict;
use utf8;
use Test::More 0.98;

use PerlEntrance::OOPTutorial::Engineer;

my $papix = PerlEntrance::OOPTutorial::Engineer->new( name => 'papix' );
my $hoto  = PerlEntrance::OOPTutorial::Engineer->new( name => 'hoto' );

subtest q{PerlEntrance::OOPTutorial::Engineerが正しく実装されていること} => sub {
    subtest q{オブジェクトがPerlEntrance::OOPTutorial::Engineerに紐付いていること} => sub {
        isa_ok $papix, 'PerlEntrance::OOPTutorial::Engineer';
    };
    subtest q{オブジェクトがnameメソッドを利用可能であること} => sub {
        can_ok $papix, 'name';
    };
};

subtest q{nameメソッドが正しく実装されていること} => sub {
    is $papix->name, 'papix';
    is $hoto->name,  'hoto';
};

done_testing;
```

`prove`コマンドから`App::Prove::Plugin::Pretty`を利用する場合, オプションに`-PPretty`を与えます.

```
$ prove -l -v -PPretty t/01_engineer.t

  PerlEntrance::OOPTutorial::Engineerが正しく実装されていること
      オブジェクトがPerlEntrance::OOPTutorial::Engineerに紐付いていること
        ✓  An object of class 'PerlEntrance::OOPTutorial::Engineer' isa 'PerlEntrance::OOPTutorial::Engineer'
      オブジェクトがnameメソッドを利用可能であること
        ✓  PerlEntrance::OOPTutorial::Engineer->can('name')
  nameメソッドが正しく実装されていること
    ✓    L20: is $papix->name, 'papix';
    ✓    L21: is $hoto->name,  'hoto';

ok
```

`-PPretty`を利用しない場合と比べ, 非常に見やすい出力になっています.
なお, 毎回`-PPretty`を指定するのが面倒な場合, `prove`コマンドへのデフォルトのオプションを`~/.proverc`で指定することができるので,

```:.proverc
-PPretty
```

のように記述しておけば, 毎回`-PPretty`オプションを付けた上で`prove`コマンドを実行することができます(同様に, `-l`を書いておけば, 毎回`-l`オプションを付けた上で`prove`コマンドを実行することができます).

なお, `-P`以外に`prove`コマンドで指定できるオプションについては, Perl Advent Calendar 2011の[prove についてのおさらい](http://perl-users.jp/articles/advent-calendar/2011/test/21)という記事が参考になります.

# 練習問題

｢OOP入門｣でそれぞれ作成した`PerlEntrance::OOPTutorial`モジュールについて, この記事で得た知識を使って, テストを書いてみよう.
