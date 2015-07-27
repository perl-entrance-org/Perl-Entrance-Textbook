# 諸注意

HomebrewはMac OS X用のアプリケーションです. 予めご了承下さい.

# はじめに

プログラムを開発する場面では, 様々なツールやライブラリが必要になります.
プログラミング言語そのものもそうですし, 例えばデータベースを使うのであればMySQLやSQLite, KVSが必要であればRedisやMemcached, VMで開発環境を用意するのであればVagrantが必要ですし, デプロイやサーバの環境構築にはCapistranoやAnsibleなどが必要になるでしょう.

通常, このようなツールやライブラリを導入してMac上に開発環境を整える際は, ツールやライブラリをそれぞれ1つずつ, Applicationディレクトリに配置したり, インストーラを使ったりしてインストールしていくと思います.
もちろんそれでも良いのですが, 例えば依存となるツールやライブラリがある場合は別途導入する必要がありますし, ツールやライブラリそのものの更新も, 1つずつ対応しなければなりません.

このような問題を解決するのが, Mac OS X向けのパッケージ管理ツール, [Homebrew](http://brew.sh/index_ja.html)です.
Homebrewを利用すれば, CentOSの`yum`やUbuntuの`apt-get`のように, 例えばテキストエディタのvimであれば`brew install vim`でインストールすることができますし, その際依存となるライブラリやツールがあればそのインストールも自動的に実施してくれます.
また, Homebrewで導入済みのライブラリやツールについては, `brew upgrade`で全て(Homebrewが対応している)最新版に更新することができます.

まだHomebrewを利用していないのであれば, **(研修カリキュラムの一部はHomebrewの導入が前提となっているので)**この機会に是非Homebrewを基盤にした開発環境を構築しましょう.

# Homebrewの導入

Homebrewを導入する前に, XcodeのCommand Line Toolsを導入しておく必要がります.
まだ導入していない場合, 以下のコマンドを入力しておきましょう.

```
$ xcode-select --install
```

XcodeのCommand Line Toolsが導入できていれば, 後は[Homebrew](http://brew.sh/index_ja.html)の公式サイトに書かれている通り, 以下のコマンドで導入することができます.

```
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

## コラム: Homebrew Cask

Homebrewが対応しているのは, 主にプログラムの開発に利用するツールやライブラリが主で, 例えばHipChatのクライアントはChromeといったブラウザ(GUIアプリケーション)には対応していません.
このようなアプリケーションをHomebrewで管理するための拡張として, [Homebrew Cask](http://caskroom.io/)が開発されています.

別途, [Homebrew-file](https://github.com/rcmdnk/homebrew-file)などの仕組みを利用すれば, 新しいMacを用意した後, まずHomebrewを入れてから, Homebrew経由で各種アプリケーションやツール, ライブラリを導入すれば, 開発環境がほぼ完成する, といった環境を準備することができます(自分の場合, Homebrew-fileなどを利用せず, `brew install ...`を連ねて書いたシェルスクリプトを各PCで共有して管理しています).

なお, 本研修では, Homebrew CaskやHomebrew-fileの導入方法や使い方について詳しく解説しません.
興味がある方は, 次の記事を参考に挑戦してみて下さい.

- [homebrew cask](http://qiita.com/kon/items/8c5396d8de42a4c34818)
- [homebrew-caskを使って簡単にMacの環境構築をしよう!](http://nanapi.co.jp/blog/2014/03/05/homebrew-cask/)
- [Homebrew-fileでhomebrewでインストールしたパッケージの管理をする](http://www.task-notes.com/entry/20150316/1426474800)

# Homebrewの使い方

ここからは, Homebrewの使い方を見て行きましょう.
なお, Homebrewのバージョンは2015年3月時点の最新版の0.9.5とします.

```
$ brew
Example usage:
  brew [info | home | options ] [FORMULA...]
  brew install FORMULA...
  brew uninstall FORMULA...
  brew search [foo]
  brew list [FORMULA...]
  brew update
  brew upgrade [FORMULA...]
  brew pin/unpin [FORMULA...]

Troubleshooting:
  brew doctor
  brew install -vd FORMULA
  brew [--env | config]

Brewing:
  brew create [URL [--no-fetch]]
  brew edit [FORMULA...]
  open https://github.com/Homebrew/homebrew/blob/master/share/doc/homebrew/Formula-Cookbook.md

Further help:
  man brew
  brew home
```

なお, Homebrewにおいては, ライブラリやコマンドは｢Formula｣というビルド手順書単位で管理することになります.
例えばPerlをインストールするのであれば, 後述の`install`コマンドに対して, Perlのビルド手順書が書かれた`perl`というFormulaを指定することで実現することができます.

Formulaでは, ライブラリやツールをビルドする手順に加えて, 依存となるライブラリやツールの指定もできますので, 導入したいライブラリやコマンドのインストールを指定するだけで, その依存となるライブラリやツールも自動的にインストールすることができます.
Homebrewで利用できるFormulaについては, 後述の`search`コマンドで検索したり, `info`コマンドで詳細を確認することができます.

## `install`

Homebrewを利用して, Formulaからツールをインストールします.
例えばemacsをインストールするのであれば, ｢emacs｣というFormulaを利用して, `brew install emacs`でOKです.

```
$ brew install emacs
==> Downloading https://homebrew.bintray.com/bottles/emacs-24.4.mavericks.bottle.3.tar.gz
####################################################################### 100.0%
==> Pouring emacs-24.4.mavericks.bottle.3.tar.gz
==> Caveats
To have launchd start emacs at login:
    ln -sfv /usr/local/opt/emacs/*.plist ~/Library/LaunchAgents
Then to load emacs now:
    launchctl load ~/Library/LaunchAgents/homebrew.mxcl.emacs.plist

WARNING: launchctl will fail when run under tmux.
==> Summary
🍺  /usr/local/Cellar/emacs/24.4: 3914 files, 104M
```

`which`で`emacs`コマンドの位置を確認すると, 次のように表示されるはずです.

```
$ which emacs
/usr/local/bin/emacs
```

Homebrewは, `/usr/local/Cellar`以下にFormula単位でライブラリやコマンドをインストールし, これらの`bin`ディレクトリに配置されているファイルのエイリアス(シンボリックリンク)をパスが通っている`/usr/local/bin`以下に配置することで, Formulaを利用してインストールしたライブラリやコマンドの管理を実現しています.

## `info`

Homebrewが扱えるFormulaの詳細を確認できます.

```
$ brew info emacs
emacs: stable 24.4 (bottled), devel 24.4-dev, HEAD
https://www.gnu.org/software/emacs/
/usr/local/Cellar/emacs/24.4 (3914 files, 104M) *
  Poured from bottle
From: https://github.com/Homebrew/homebrew/blob/master/Library/Formula/emacs.rb
==> Dependencies
Build: xz ✔, pkg-config ✔
Optional: d-bus ✘, gnutls ✘, librsvg ✘, imagemagick ✘, mailutils ✘, glib ✔
==> Options
--cocoa
	Build a Cocoa version of emacs
--keep-ctags
	Don't remove the ctags executable that emacs provides
--with-d-bus
	Build with d-bus support
--with-glib
	Build with glib support
--with-gnutls
	Build with gnutls support
--with-imagemagick
	Build with imagemagick support
--with-librsvg
	Build with librsvg support
--with-mailutils
	Build with mailutils support
--with-x11
	Build with x11 support
--devel
	Install development version 24.4-dev
--HEAD
	Install HEAD version
==> Caveats
To have launchd start emacs at login:
    ln -sfv /usr/local/opt/emacs/*.plist ~/Library/LaunchAgents
Then to load emacs now:
    launchctl load ~/Library/LaunchAgents/homebrew.mxcl.emacs.plist

WARNING: launchctl will fail when run under tmux.
```

インストール時に表示する`Caveats`も表示されるので, 見逃してしまった場合は`info`コマンドで確認しましょう.

なお, `info`コマンドは, Homebrewで管理できるが, まだインストールしていないコマンドに対しても実行することができます.

```
ruby: stable 2.2.1 (bottled), HEAD
https://www.ruby-lang.org/
Not installed
From: https://github.com/Homebrew/homebrew/blob/master/Library/Formula/ruby.rb
==> Dependencies
Build: pkg-config ✔
Required: libyaml ✔, openssl ✔
Recommended: readline ✔
Optional: gdbm ✔, gmp ✘, libffi ✔
==> Options
--universal
        Build a universal binary
--with-doc
        Install documentation
--with-gdbm
        Build with gdbm support
--with-gmp
        Build with gmp support
--with-libffi
        Build with libffi support
--with-suffix
        Suffix commands with '22'
--with-tcltk
        Install with Tcl/Tk support
--without-readline
        Build without readline support
--HEAD
        Install HEAD version
```

未インストールの場合, このように3行目がインストール先ではなく｢Not installed｣と表示されます.

### `search`

Homebrewで扱えるFormulaを検索します.
`brew search perl`のように引数を与えた場合, その引数で指定した文字列に関係のあるFormulaが表示されます.

```
$ brew search perl
perl         perl-build   perlmagick
```

また, `brew search`のみで実行した場合, Homebrewで管理できるツールやライブラリの一覧を表示します.

```
$ brew search
a2ps                            apachetop                       autojump                        bibtex2html                     cadaver
a52dec                          ape                             automake                        bibtexconv                      cadubi
aacgain                         apg                             automoc4                        bibutils                        cairo
aalib                           apgdiff                         automysqlbackup                 bigdata                         cairomm
aamath                          apib                            autopano-sift-c                 bigloo                          calabash
    ... 後略 ...
```

### `list`

現在Homebrewで管理しているツールやライブラリの一覧を表示します.

```
$ brew list
ansible                         giflib                          libyaml                         ricty
autoconf                        git                             lua                             ruby-build
automake                        glib                            luajit                          scons
bison                           go                              lv                              selenium-server-standalone
brew-cask                       gobject-introspection           mercurial                       sqlite
    ... 後略 ...
```

### `update`

Homebrew本体や, Formulaの更新を行うコマンドです.

```
$ brew update
Updated Homebrew from 01463d23 to 49d0e7a6.
==> New Formulae
git-plus
==> Updated Formulae
curl               encfs              jenkins            markdown           peco/peco/peco     vegeta
ecj                git                libxmp             multimarkdown      syncthing
elasticsearch      heroku-toolbelt    lnav               openconnect        trafficserver
```

### `upgrade`

Homebrewで管理しているツールやライブラリの更新をします.

```
$ brew upgrade
==> Upgrading 1 outdated package, with result:
git 2.3.4
==> Upgrading git
==> Downloading https://homebrew.bintray.com/bottles/git-2.3.4.yosemite.bottle.tar.gz
####################################################################### 100.0%
==> Pouring git-2.3.4.yosemite.bottle.tar.gz
==> Caveats
The OS X keychain credential helper has been installed to:
  /usr/local/bin/git-credential-osxkeychain

The "contrib" directory has been installed to:
  /usr/local/share/git-core/contrib

Bash completion has been installed to:
  /usr/local/etc/bash_completion.d

zsh completion has been installed to:
  /usr/local/share/zsh/site-functions
==> Summary
🍺  /usr/local/Cellar/git/2.3.4: 1362 files,  31M
```

### `uninstall`

Homebrewでインストールしたツールやライブラリを削除します.

```
$ brew uninstall emacs
Uninstalling /usr/local/Cellar/emacs/24.4...
```
