# Write yourself a Git!

［訳註: このファイルは https://wyag.thb.lt の翻訳です。<time datetime="2020-11-21T15:26:49">2020年11月22日</time>に作成され、最後の変更は<time datetime="2021-10-26T14:36:18">2021年10月26日</time>に行われました。］

## 導入
<!-- Introduction -->

この記事は、 [Git バージョン管理システム](https://git-scm.com/)をボトムアップで、すなわち、最も基本的なレベルから始め、そこから前進することによって説明する試みです。この試みは簡単すぎるようには思えないし、何度も成功かどうか疑わしい結果を収めてきたものです。しかし、簡単な方法があります: Git の内部を理解するために必要なことは、 Git を最初から再実装することだけです。 <!-- This article is an attempt at explaining the Git version control system from the bottom up, that is, starting at the most fundamental level moving up from there. This does not sound too easy, and has been attempted multiple times with questionable success. But there’s an easy way: all it takes to understand Git internals is to reimplement Git from scratch. -->

実行しないでください。 <!-- No, don’t run. -->

これは冗談ではないし、まったく複雑なことでもありません: この記事を上から下まで読んでコードを書けば（あるいは、[このリポジトリ](https://github.com/thblt/write-yourself-a-git)をクローンしさえすれば。ただし、自分で書くべきです。本当に。）最後には `wyag` というプログラムが出来上がります。そのプログラムは、 Git の全ての基本機能（ `init` 、`add` 、 `rm` 、 `status` 、 `commit` 、 `log` ……）を `git` 自体との完全な互換性を持つように実装しています。実際、この記事の最後のコミットは `git` ではなく `wyag` で作られています。［訳註: この翻訳のコミットは `git` で作られています。この翻訳では `cmd_commit` はまだ実装されていません。］そして、その全てが503行の非常にシンプルな Python コードになります。 <!-- It’s not a joke, and it’s really not complicated: if you read this article top to bottom and write the code (or just clone the repository — but you should write the code yourself, really), you’ll end up with a program, called wyag, that will implement all the fundamental features of git: init, add, rm, status, commit, log… in a way that is perfectly compatible with git itself. The last commit of this article was actually created with wyag, not git. And all that in exactly 503 lines of very simple Python code. -->

しかし、それには Git は複雑すぎるのではないか？　私の考えでは、 Git が複雑であるというのは誤解です。 Git が大きなプログラムで、多機能であることは真実です。しかし、そのプログラムのコアは実際には非常にシンプルで、 Git プログラムの見掛け上の複雑さは、それがしばしばひどく反直感的である（そして、ブログ投稿 [Git is a burrito](https://byorgey.wordpress.com/2009/01/12/abstraction-intuition-and-the-monad-tutorial-fallacy/) はおそらく助けにならない）という事実にまず起因しています。しかし、 Git を最も混乱させているのは、そのコアモデルの極端なシンプルさ*と*パワーかもしれません。コアのシンプルさと強力なアプリケーションの組み合わせは、基本的な抽象（モナドとかどう？）の本質的なシンプルさから種々のアプリケーションを得るために必要な精神的飛躍のため、物事を掴みにくくすることがよくあります。 <!-- But isn’t Git too complex for that? That Git is complex is, in my opinion, a misconception. Git is a large program, with a lot of features, that’s true. But the core of that program is actually extremely simple, and its apparent complexity stems first from the fact it’s often deeply counterintuitive (and Git is a burrito blog posts probably don’t help). But maybe what makes Git the most confusing is the extreme simplicity and power of its core model. The combination of core simplicity and powerful applications often makes thing really hard to grasp, because of the mental jump required to derive the variety of applications from the essential simplicity of the fundamental abstraction (monads, anyone?) -->

Git を実装することで、その基本を全て赤裸々に曝け出させることができます。 <!-- Implementing Git will expose its fundamentals in all their naked glory. -->

**何を期待していますか？**　この記事は、とても単純化されたバージョンの Git コアコマンドを実装し、詳細に（何か不明な点があれば[報告してください]()！［訳註: この翻訳の問題点を原作者に報告しないでください。］）説明します。コードをシンプルにかつ要点を外さないように保っているので、 `wyag` は本物の git コマンドラインほど強力ではありませんが、何が欠けているのかは明らかで、やってみたい人は誰でもそれを実装することができます。よく言われるように、“wyag をフル機能の git ライブラリと CLI にアップグレードするのは読者の演習として残されています。” <!-- What to expect? This article will implement and explain in great details (if something is not clear, please report it!) a very simplified version of Git core commands. I will keep the code simple and to the point, so wyag won’t come anywhere near the power of the real git command-line — but what’s missing will be obvious, and trivial to implement by anyone who wants to give it a try. “Upgrading wyag to a full-featured git library and CLI is left as an exercise to the reader”, as they say. -->

より正確には、私達は: <!-- More precisely, we’ll implement: -->

- `add` () [git man page](https://git-scm.com/docs/git-add)
- `cat-file` ([wyag source](#cat-file-コマンド)) [git man page](https://git-scm.com/docs/git-cat-file)
- `checkout` ([wyag source](#checkout-コマンド)) [git man page](https://git-scm.com/docs/git-checkout)
- `commit` () [git man page](https://git-scm.com/docs/git-commit)
- `hash-object` ([wyag source](#hash-object-コマンド)) [git man page](https://git-scm.com/docs/git-hash-object)
- `init` ([wyag source](#init-コマンド)) [git man page](https://git-scm.com/docs/git-init)
- `log` ([wyag source](#log-コマンド)) [git man page](https://git-scm.com/docs/git-log)
- `ls-tree` () [git man page](https://git-scm.com/docs/git-ls-tree)
- `merge` () [git man page](https://git-scm.com/docs/git-merge)
- `rebase` () [git man page](https://git-scm.com/docs/git-rebase)
- `rev-parse` () [git man page](https://git-scm.com/docs/git-rev-parse)
- `rm` () [git man page](https://git-scm.com/docs/git-rm)
- `show-ref` () [git man page](https://git-scm.com/docs/git-show-ref)
- `tag` ([wyag source](#tag-コマンド)) [git man page](https://git-scm.com/docs/git-tag)

を実装します。 <!-- More precisely, we’ll implement: -->

この記事を理解するために多くのことを知っている必要はありません: 基本的な Git （明白です）、基本的な Python 、基本的なシェルだけです。 <!-- You’re not going to need to know much to follow this article: just some basic Git (obviously), some basic Python, some basic shell. -->

- まず、最も基本的な **git コマンド**（エキスパートレベルのものではないですが、 `init` 、 `add` 、 `rm` 、 `commit` あるいは `checkout` を使ったことが無ければ迷うことになるでしょう。）にある程度精通していることを想定しています。 <!-- First, I’m only going to assume some level of familiarity with the most basic git commands — nothing like an expert level, but if you’ve never used init, add, rm, commit or checkout, you will be lost. -->
- 言語的には、 wyag は **Python** で実装されます。繰り返しますが、あまり派手なものは使いませんし、 Python は擬似コードのように見えるので、簡単に理解することができます。（皮肉にも最も複雑なパートはコマンドライン引数をパーズするロジックです。それを理解する必要はまったくありません。）ですが、プログラミングは知っているが Python は一度も使ったことが無いという場合は、インターネットのどこかでクラッシュコースを見付けて Python に慣れておくことをお勧めします。 <!-- Language-wise, wyag will be implemented in Python. Again, I won’t use anything too fancy, and Python looks like pseudo-code anyways, so it will be easy to follow (ironically, the most complicated part will be the command-line arguments parsing logic, and you don’t really need to understand that). Yet, if you know programming but have never done any Python, I suggest you find a crash course somewhere in the internet just to get acquainted with the language. -->
- `wyag` と `git` はターミナルプログラムです。 Unix ターミナルの使い方は知っていると思います。繰り返しますが、あなたが l77t h4x0r である必要はありませんが、あなたのツールボックスには `cd` 、 `ls` 、 `rm` 、 `tree` やこれらの仲間達が含まれているべきです。 <!-- wyag and git are terminal programs. I assume you know your way inside a Unix terminal. Again, you don’t need to be a l77t h4x0r, but cd, ls, rm, tree and their friends should be in your toolbox. -->

---

**Warning**

**Windows ユーザーへの注意** <!-- Note for Windows users -->

`wyag` は、 Python インタープリターがある任意の Unix-like システム上で動くはずですが、 Windows 上でどう振る舞うかについてはまったく分かりません。テストスイートは bash 互換シェルを要求しますが、これは WSL によって提供されると思います。 Windows ユーザーからのフィードバックがあれば有り難いです！ <!-- wyag should run on any Unix-like system with a Python interpreter, but I have absolutely no idea how it will behave on Windows. The test suite absolutely requires a bash-compatible shell, which I assume the WSL can provide. Feedback from Windows users would be appreciated! -->

---

## 入門
<!-- Getting started -->

Python 3 （私は 3.6.5 を使っています。問題に当たったらこれ以上のバージョンを使ってみてください。 Python 2 では全く動作しません。）とあなたのお気に入りのテキストエディターが必要です。サードパーティパッケージや virtualenvs 、通常の Python インタープリター以外のものは必要ありません: 必要なものは全て Python 標準ライブラリにあります。 <!-- You’re going to need Python 3 (I used 3.6.5: if you encounter issues try using at least this version. Python 2 won’t work at all) and your favorite text editor. We won’t need third party packages or virtualenvs, or anything besides a regular Python interpreter: everything we need is in Python’s standard library. -->

私達はコードを2つのファイルに分割します: <!-- We’ll split the code into two files: -->

- 実行可能形式。 `wyag` と呼びます。 <!-- An executable, called wyag; -->
- Python ライブラリ。 `libwyag.py` と呼びます。 <!-- A Python library, called libwyag.py; -->

さて、全てのソフトウェアプロジェクトは大量のボイラープレートから始まります。これを乗り越えましょう。 <!-- Now, every software project starts with a boatload of boilerplate, so let’s get this over with. -->

バイナリの作成から始めます。テキストエディターで `wyag` という名前の新しいファイルを作成して、次の数行をコピーしてください: <!-- We’ll begin by creating the binary. Create a new file called wyag in your text editor, and copy the following few lines: -->

``` py
#!/usr/bin/env python3

import libwyag
libwyag.main()
```

そして、実行可能にします: <!-- Then make it executable: -->

``` sh
chmod +x wyag
```

出来ました！ <!-- You’re done! -->

今度はライブラリです。ライブラリは `libwyag.py` という名前で、 `wyag` 実行可能形式と同じディレクトリに無ければなりません。テキストエディターで空っぽの `libwyag.py` を開くことから始めます。 <!-- Now for the library. it must be called libwyag.py, and be in the same directory as the wyag executable. Begin by opening the empty libwyag.py in your text editor. -->

まず、大量の import が必要です。（単に各 import をコピーするか、それらを全て1行にマージしてください。） <!-- We’re first going to need a bunch of imports (just copy each import, or merge them all in a single line) -->

- Git は CLI アプリケーションなので、コマンドライン引数をパーズするためのものが必要です。 Python は [argparse](https://docs.python.org/3/library/argparse.html) というクールなモジュールを提供しています。このモジュールは私達の仕事の 99 % を行えます。 <!-- Git is a CLI application, so we’ll need something to parse command-line arguments. Python provides a cool module called argparse that can do 99% of the job for us. -->

  ``` py
  import argparse
  ```

- base ライブラリが提供するよりも少し多くのコンテナー型、とくに `OrderedDict` が必要です。 [collections](https://docs.python.org/3/library/collections.html#collections.OrderedDict) にあります。 <!-- We’ll need a few more container types than the base lib provides, most notably an OrderedDict. It’s in collections. -->

  ``` py
  import collections
  ```

- Git は基本的に Microsoft の INI 形式の構成ファイル形式を使います。 [configparser](https://docs.python.org/3/library/configparser.html) モジュールはそれらのファイルを読み書きできます。 <!-- Git uses a configuration file format that is basically Microsoft’s INI format. The configparser module can read and write these files. -->

  ``` py
  import configparser
  ```

- Git は SHA-1 関数をかなり広い範囲で使います。 Python では、これは [hashlib](https://docs.python.org/3/library/hashlib.html) にあります。 <!-- Git uses the SHA-1 function quite extensively. In Python, it’s in hashlib. -->

  ``` py
  import hashlib
  ```

- [os](https://docs.python.org/3/library/os.html) と [os.path](https://docs.python.org/3/library/os.path.html) は魅力的なファイルシステム抽象化ルーチンを提供します。 <!-- os and os.path provide some nice filesystem abstraction routines. -->

  ``` py
  import os
  ```

- 正規表現を*少しだけ*使います: <!-- we use just a bit of regular expressions: -->

  ``` py
  import re
  ```

- （ `sys.argv` にある）実際のコマンドライン引数にアクセスするために、 [sys](https://docs.python.org/3/library/sys.html) も必要です。 <!-- We also need sys to access the actual command-line arguments (in sys.argv): -->

  ``` py
  import sys
  ```

- Git は全てを zlib で圧縮します。[これも Python にあります](https://docs.python.org/3/library/zlib.html): <!-- Git compresses everything using zlib. Python has that, too: -->

  ``` py
  import zlib
  ```

import は以上です。コマンドライン引数を扱うことが多くなってきます。 Python はシンプルでありながらそれなりに強力なパーズライブラリ `argparse` を提供しています。これは素晴らしいライブラリですが、インターフェイスは最も直感的とまではいかないかもしれません。必要なら[ドキュメント](https://docs.python.org/3/library/argparse.html)を参照してください。 <!-- Imports are done. We’ll be working with command-line arguments a lot. Python provides a simple yet reasonably powerful parsing library, argparse. It’s a nice library, but its interface may not be the most intuitive ever; if need, refer to its documentation. -->

``` py
argparser = argparse.ArgumentParser(description="The stupid content tracker")
```

私達は（git の `init` 、 `commit` などのような）サブコマンドを扱う必要があります。 argparse のスラングでは、これらは「サブパーザー」と呼ばれます。現時点で必要なのは、私達の CLI がその中のいくつかを使用するということと、全ての呼び出しが実際にはその中の一つ（単に `git` と呼ぶのではなく、 `git COMMAND` と呼びます。）を*要求する*ということを宣言することだけです。 <!-- We’ll need to handle subcommands (as in git: init, commit, etc.) In argparse slang, these are called “subparsers”. At this point we only need to declare that our CLI will use some, and that all invocation will actually require one — you don’t just call git, you call git COMMAND. -->

``` py
argsubparsers = argparser.add_subparsers(title="Commands", dest="command")
argsubparsers.required = True
```

`dest="command"` 引数は、選択されたサブパーザーの名前が `command` というフィールドにある文字列として返されることを表します。なので、必要なことは、この文字列を読み取り、適宜正しい関数を呼び出すことだけです。慣例では、これらの関数の前に `cmd_` を付けます。 `cmd_*` 関数は、パーズされた引数を唯一のパラメーターに取り、実際のコマンドを実行する前にそれを処理し、検証する責任を負います。 <!-- The dest="command" argument states that the name of the chosen subparser will be returned as a string in a field called command. So we just need to read this string and call the correct function accordingly. By convention, I’ll prefix these functions by cmd_. cmd_* functions take the parsed arguments as their unique parameter, and are responsible for processing and validating them before executing the actual command. -->

``` py
def main(argv=sys.argv[1:]):
    args = argparser.parse_args(argv)

    if   args.command == "add"         : cmd_add(args)
    elif args.command == "cat-file"    : cmd_cat_file(args)
    elif args.command == "checkout"    : cmd_checkout(args)
    elif args.command == "commit"      : cmd_commit(args)
    elif args.command == "hash-object" : cmd_hash_object(args)
    elif args.command == "init"        : cmd_init(args)
    elif args.command == "log"         : cmd_log(args)
    elif args.command == "ls-tree"     : cmd_ls_tree(args)
    elif args.command == "merge"       : cmd_merge(args)
    elif args.command == "rebase"      : cmd_rebase(args)
    elif args.command == "rev-parse"   : cmd_rev_parse(args)
    elif args.command == "rm"          : cmd_rm(args)
    elif args.command == "show-ref"    : cmd_show_ref(args)
    elif args.command == "tag"         : cmd_tag(args)
```

## リポジトリの作成: init
<!-- Creating repositories: init -->

時系列順*でも*論理順*でも*最初の Git コマンドが `git init` であることは明らかなので、 `wyag init` を作るところから始めます。これを達成するには、まず、ごく基本的なリポジトリの抽象が必要です。 <!-- Obviously, the first Git command in chronological and logical order is git init, so we’ll begin by creating wyag init. To achieve this, we’re going to first need some very basic repository abstraction. -->

### Repository オブジェクト
<!-- The Repository object -->

リポジトリの抽象が必要であることは明白です: 私達は、 git コマンドを実行する時はほとんどいつでも、リポジトリに何かするか、リポジトリを作成するか、リポジトリから何かを読み取るか、あるいはリポジトリを修正するかしようとしています。 <!-- We’ll obviously need some abstraction for a repository: almost every time we run a git command, we’re trying to do something to a repository, to create it, read from it or modify it. -->

git では、リポジトリは2つの物から成ります: バージョン管理されているファイルがある「ワークツリー」と、 Git が自身のデータを保存する「 git ディレクトリ」です。多くの場合、ワークツリーは通常のディレクトリであり、 git ディレクトリは、 `.git` という名前の、ワークツリーの子ディレクトリです。 <!-- A repository, in git, is made of two things: a “work tree”, where the files meant to be in version control live, and a “git directory”, where Git stores its own data. In most cases, the worktree is a regular directory and the git directory is a child directory of the worktree, called .git. -->

Git は*より多くの*ケース（ bare リポジトリや分離された gitdir など）をサポートしますが、これは私達には必要ありません: 基本的な `worktree/.git` のアプローチを取ります。リポジトリオブジェクトは2つのパス（ワークツリーと gitdir ）を保持するだけです。 <!-- Git supports much more cases (bare repo, separated gitdir, etc) but we won’t need them: we’ll stick the basic approach of worktree/.git. Our repository object will then just hold two paths: the worktree and the gitdir. -->

新しい `Repository` オブジェクトを作るには、少しだけチェックが必要です: <!-- To create a new Repository object, we only need to make a few checks: -->

- ディレクトリが存在し、そこに `.git` という名前のサブディレクトリが含まれていることを確認しなければなりません。 <!-- We must verify that the directory exists, and contains a subdirectory called .git. -->
- `.git/config` （単なる INI ファイルです。）にある設定を読み込み、 `core.repositoryformatversion` が 0 になるように制御します。このフィールドについては、すぐに詳しく説明します。 <!-- We read its configuration in .git/config (it’s just an INI file) and control that core.repositoryformatversion is 0. More on that field in a moment. -->

コンストラクターは省略可能なパラメーター `force` を取ります。このパラメーターは、これらのチェックを全て無効にします。後で作る `repo_create()` 関数がリポジトリを作るために `Repository` オブジェクトを使うためのものです。そのため、（まだ）無効なファイルシステム上の位置からでもリポジトリを作る方法が必要です。 <!-- The constructor takes an optional force which disables all check. That’s because the repo_create() function which we’ll create later uses a Repository object to create the repo. So we need a way to create repository even from (still) invalid filesystem locations. -->

``` py
class GitRepository(object):
    """A git repository"""

    worktree = None
    gitdir = None
    conf = None

    def __init__(self, path, force=False):
        self.worktree = path
        self.gitdir = os.path.join(path, ".git")

        if not (force or os.path.isdir(self.gitdir)):
            raise Exception("Not a Git repository %s" % path)

        # Read configuration file in .git/config
        self.conf = configparser.ConfigParser()
        cf = repo_file(self, "config")

        if cf and os.path.exists(cf):
                self.conf.read([cf])
        elif not force:
            raise Exception("Configuration file missing")

        if not force:
            vers = int(self.conf.get("core", "repositoryformatversion"))
            if vers != 0:
                raise Exception("Unsupported repositoryformatversion %s" % vers)
```

私達はリポジトリ内の**多く**のパスを操作することになります。これらのパスを計算したり、必要に応じて不足しているディレクトリ構造を作ったりするために、いくつかユーティリティ関数を作ると良いです。まずは一般的なパス構築関数です: <!-- We’re going to be manipulating lots of paths in repositories. We may as well create a few utility functions to compute those paths and create missing directory structures if needed. First, just a general path building function: -->

``` py
def repo_path(repo, *path):
    """Compute path under repo's gitdir."""
    return os.path.join(repo.gitdir, *path)
```

次の2つの関数 `repo_file()` と `repo_dir()` は、それぞれファイルまたはディレクトリのパスを返します。オプションによっては作成もします。これらの関数の違いは、ファイル版は最後のコンポーネントまでのディレクトリしか作成しないというところです。 <!-- The two next functions, repo_file() and repo_dir(), return and optionally create a path to a file or a directory, respectively. The difference between them is that the file version only creates directories up to the last component. -->

``` py
def repo_file(repo, *path, mkdir=False):
    """Same as repo_path, but create dirname(*path) if absent.  For
example, repo_file(r, \"refs\", \"remotes\", \"origin\", \"HEAD\") will create
.git/refs/remotes/origin."""

    if repo_dir(repo, *path[:-1], mkdir=mkdir):
        return repo_path(repo, *path)

def repo_dir(repo, *path, mkdir=False):
    """Same as repo_path, but mkdir *path if absent if mkdir."""

    path = repo_path(repo, *path)

    if os.path.exists(path):
        if (os.path.isdir(path)):
            return path
        else:
            raise Exception("Not a directory %s" % path)

    if mkdir:
        os.makedirs(path)
        return path
    else:
        return None
```

新しいリポジトリを**作成する**には、ディレクトリ（存在しない場合は作り、その他の場合は空であることを確認する。）から始め、次のパスを作成します: <!-- To create a new repository, we start with a directory (which we create if doesn’t already exist, or check for emptiness otherwise) and create the following paths: -->

- `.git` は git ディレクトリ自身です。次を含みます: <!-- .git is the git directory itself, which contains: -->
  - `.git/objects/` はオブジェクトストアです。[次の節で]()導入します。 <!-- .git/objects/ : the object store, which we’ll introduce in the next section. -->
  - `.git/refs/` はリファレンスストアです。これから議論します。2つのサブディレクトリ `heads` と `tags` を含みます。 <!-- .git/refs/ the reference store, which we’ll discuss. It contains two subdirectories, heads and tags. -->
  - `.git/HEAD` は現在の HEAD への参照です。（詳細は後ほど！） <!-- .git/HEAD, a reference to the current HEAD (more on that later!) -->
  - `.git/config` はリポジトリの構成ファイルです。 <!-- .git/config, the repository’s configuration file. -->
  - `.git/description` はリポジトリの概要ファイルです。 <!-- .git/description, the repository’s description file. -->

``` py
def repo_create(path):
    """Create a new repository at path."""

    repo = GitRepository(path, True)

    # First, we make sure the path either doesn't exist or is an
    # empty dir.

    if os.path.exists(repo.worktree):
        if not os.path.isdir(repo.worktree):
            raise Exception ("%s is not a directory!" % path)
        if os.listdir(repo.worktree):
            raise Exception("%s is not empty!" % path)
    else:
        os.makedirs(repo.worktree)

    assert(repo_dir(repo, "branches", mkdir=True))
    assert(repo_dir(repo, "objects", mkdir=True))
    assert(repo_dir(repo, "refs", "tags", mkdir=True))
    assert(repo_dir(repo, "refs", "heads", mkdir=True))

    # .git/description
    with open(repo_file(repo, "description"), "w") as f:
        f.write("Unnamed repository; edit this file 'description' to name the repository.\n")

    # .git/HEAD
    with open(repo_file(repo, "HEAD"), "w") as f:
        f.write("ref: refs/heads/master\n")

    with open(repo_file(repo, "config"), "w") as f:
        config = repo_default_config()
        config.write(f)

    return repo
```

構成ファイルはとてもシンプルです。1つのセクション (`[core]`) と次の3つのフィールドを持つ INI-like ファイルです: <!-- The configuration file is very simple, it’s a INI-like file with a single section ([core]) and three fields: -->

- `repositoryformatversion = 0`: gitdir フォーマットのバージョンです。 0 は初期フォーマットを意味し、 1 は拡張機能と同じです。 1 を超える場合、 git はパニックします。 wyag は 0 のみを受け入れます。 <!-- repositoryformatversion = 0: the version of the gitdir format. 0 means the initial format, 1 the same with extensions. If > 1, git will panic; wyag will only accept 0. -->
- `filemode = false`: 作業ツリーでのファイルモードのトラッキングを無効にします。 <!-- filemode = false: disable tracking of file mode changes in the work tree. -->
- `bare = false`: このリポジトリに作業ツリーがあるということを示します。 Git は省略可能な `worktree` キーをサポートします。このキーは、 `..` でない場合、作業ツリーの場所を示します。 wyag ではサポートしません。 <!-- bare = false: indicates that this repository has a worktree. Git supports an optional worktree key which indicates the location of the worktree, if not ..; wyag doesn’t. -->

これは Python の configparser ライブラリを使って作ります: <!-- We create this file using Python’s configparser lib: -->

``` py
def repo_default_config():
    ret = configparser.ConfigParser()

    ret.add_section("core")
    ret.set("core", "repositoryformatversion", "0")
    ret.set("core", "filemode", "false")
    ret.set("core", "bare", "false")

    return ret
```

### init コマンド
<!-- The init command -->

これでリポジトリを読み取ったり作ったりするコードが出来たので、 `wyag init` コマンドを作って、このコードをコマンドラインから使えるようにしましょう。 `wyag init` は、丁度 `git init` のように振る舞いますが、勿論、可能なカスタマイズはかなり少ないです。 `wyag init` の構文は: <!-- Now that we have code to read and create repositories, let’s make this code usable from the command line by creating the wyag init command. wyag init behaves just like git init — with much less possible customization, of course. The syntax of wyag init is going to be: -->

```
wyag init [path]
```

となります。 <!-- Now that we have code to read and create repositories, let’s make this code usable from the command line by creating the wyag init command. wyag init behaves just like git init — with much less possible customization, of course. The syntax of wyag init is going to be: -->

完全なリポジトリ作成ロジックはすでにあります。コマンドを作るために必要なものは後2つだけです: <!-- We already have the complete repository creation logic. To create the command, we’re only going to need two more things: -->

1.  コマンドの引数を処理するために argparse サブパーザーを作る必要があります。 <!-- We need to create an argparse subparser to handle our command’s argument. -->

    ``` py
    argsp = argsubparsers.add_parser("init", help="Initialize a new, empty repository.")
    ```

    `init` の場合、省略可能な位置指定引数（リポジトリを初期化する場所のパス）が1つあります。デフォルトは `.` です: <!-- In the case of init, there’s a single, optional, positional argument: the path where to init the repo. It defaults to .: -->

    ``` py
    argsp.add_argument("path",
                       metavar="directory",
                       nargs="?",
                       default=".",
                       help="Where to create the repository.")
    ```

1.  argparse が返すオブジェクトから引数の値を読み取り、正しい値で実際の関数を呼び出す「ブリッジ」関数も必要です。 <!-- We also need a “bridge” function that will read argument values from the object returned by argparse and call the actual function with correct values. -->

    ``` py
    def cmd_init(args):
        repo_create(args.path)
    ```

これで終わりです！　この手順を踏めば、どこにでも git リポジトリを `wyag init` できるようになるはずです。 <!-- And we’re done! If you’ve followed these steps, you should now be able to wayg init a git repository anywhere. -->

### repo_find() 関数
<!-- The repo_find() function -->

リポジトリを実装しているうちに後々必要になる関数があります: Git の機能［訳註: 原文の「 init 」を取り除きました。おかしかったら後で直します。］は大抵既存のリポジトリを使います。リポジトリは現在のディレクトリにあることが多いですが、代わりに親ディレクトリにあることもあります。（リポジトリのルートが `~/Documents/MyProject` で、現在のディレクトリが `~/Documents/MyProject/src/tui/frames/mainview/` のように。）これから作る関数 `repo_find()` は、現在のディレクトリから始めて `/` まで遡ってリポジトリを探します。リポジトリとして識別するには `.git` ディレクトリの存在を確認します。 <!-- While we’re implementing repositories, there’s a function we’re going to need a lot later: Almost all Git functions init use an existing repository. It’s often in the current directory, but it may be in a parent instead: your repository’s root may be in ~/Documents/MyProject, but you may currently be in ~/Documents/MyProject/src/tui/frames/mainview/. The repo_find() function we’ll now create will look for a repository, starting at current directory and recursing back until /. To identify something as a repo, it will check for the presence of a .git directory. -->

``` py
def repo_find(path=".", required=True):
    path = os.path.realpath(path)

    if os.path.isdir(os.path.join(path, ".git")):
        return GitRepository(path)

    # If we haven't returned, recurse in parent, if w
    parent = os.path.realpath(os.path.join(path, ".."))

    if parent == path:
        # Bottom case
        # os.path.join("/", "..") == "/":
        # If parent==path, then path is root.
        if required:
            raise Exception("No git directory.")
        else:
            return None

    # Recursive case
    return repo_find(parent, required)
```

これでリポジトリの完成です！ <!-- And we’re done with repositories! -->

## オブジェクトの読み書き: hash-object と cat-file
<!-- Reading and writing objects: hash-object and cat-file -->

### オブジェクトって何？
<!-- What are objects? -->

さて、リポジトリが出来たので、今度はその中に何かを入れるのが順当です。また、リポジトリは退屈でしたが、 Git 実装を書くというのは単に大量の `mkdir` を書けばよいというものではありません。**オブジェクト**の話をして、 `git hash-object` と `git cat-file` を実装しましょう。 <!-- Now that we have repositories, putting things inside them is in order. Also, repositories are boring, and writing a Git implementation shouldn’t be just a matter of writing a bunch of mkdir. Let’s talk about objects, and let’s implement git hash-object and git cat-file. -->

これら2つのコマンドを知らない人も居るかもしれません。現に、日常の git ツールボックスには含まれていないし、非常に低レベルなもの（ git 用語でいう「配管 (plumbing) 」）です。これらのコマンドがすることは非常にシンプルです: `hash-object` は既存のファイルを git オブジェクトに変換し、 `cat-file` は既存の git オブジェクトを標準出力に印字します。 <!-- Maybe you don’t know these two commands — they’re not exactly part of an everyday git toolbox, and they’re actually quite low-level (“plumbing”, in git parlance). What they do is actually very simple: hash-object converts an existing file into a git object, and cat-file prints an existing git object to the standard output. -->

では、**実のところ、 Git オブジェクトとは何なのでしょう？**　 Git は「内容アドレスファイルシステム」です。つまり、通常のファイルシステムでは、ファイル名は恣意的であり、ファイルの内容と無関係であるのに対し、 Git が保存するファイル名は、その内容から数学的に導き出される、ということです。これには非常に重要な含意があります: 仮にテキストファイルが1バイト変わったら、その内部名も変わるということです。簡単に言えば、ファイルを*変更している*のではなく、別の場所に新しいファイルを作っているということです。オブジェクトとは正に git リポジトリ内のファイルのことであり、そのパスはその内容によって決定されます。 <!-- Now, what actually is a Git object? At its core, Git is a “content-addressed filesystem”. That means that unlike regular filesystems, where the name of a file is arbitrary and unrelated to that file’s contents, the names of files as stored by Git are mathematically derived from their contents. This has a very important implication: if a single byte of, say, a text file, changes, its internal name will change, too. To put it simply: you don’t modify a file, you create a new file in a different location. Objects are just that: files in the git repository, whose path is determined by their contents. -->

---

**Warning**

**Git は（本物の）キーバリューストアではない** <!-- Git is not (really) a key-value store -->

あの素晴らしき [Pro Git](https://git-scm.com/book/id/v2/Git-Internals-Git-Objects) を含むいくつかのドキュメントが、 Git を「キーバリューストア」と呼んでいます。これは誤りではないですが、誤解を招くかもしれません。実際には、 Git よりも通常のファイルシステムの方がキーバリューストアにより近いです。 Git はデータからキーを計算するので、むしろ*バリューバリューストア*と呼ばれるべきものです。 <!-- Some documentation, including the excellent Pro Git, call Git a “key-value store”. This is not incorrect, but may be misleading. Regular filesystems are actually closer to a key-value store than Git is. Because it computes keys from data, Git should rather be called a value-value store. -->

---

Git はオブジェクトを使って非常に多くのものを保存します: まず第一に、バージョン管理している実際のファイル（たとえば、ソースコード）です。コミットもオブジェクトです。タグもです。いくつかの注目すべき例外（後で見ます！）を除いて、 Git では、ほとんど全てのものがオブジェクトとして保存されます。 <!-- Git uses objects to store quite a lot of things: first and foremost, the actual files it keeps in version control — source code, for example. Commit are objects, too, as well as tags. With a few notable exceptions (which we’ll see later!), almost everything, in Git, is stored as an object. -->

パスはその内容の [SHA-1 ハッシュ](https://en.wikipedia.org/wiki/Cryptographic_hash_function)を計算することで計算されます。より正確には、 Git は、そのハッシュを小文字の16進文字列としてレンダリングし、2つの部分（最初の2文字と残り）に分けます。最初の2文字はディレクトリ名として使い、残りはファイル名として使います。（これは、多くのファイルシステムで、1つのディレクトリに過剰に多くのファイルがあることが嫌われていて、クロールが低速になることがあるためです。 Git の方法では、256個の中間ディレクトリが作成できるので、ディレクトリあたりのファイル数は平均で256分の1になります。） <!-- The path is computed by calculating the SHA-1 hash of its contents. More precisely, Git renders the hash as a lowercase hexadecimal string, and splits it in two parts: the first two characters, and the rest. It uses the first part as a directory name, the rest as the file name (this is because most filesystems hate having too many files in a single directory and would slow down to a crawl. Git’s method creates 256 possible intermediate directories, hence dividing the average number of files per directory by 256) -->

---

**Note**

**ハッシュ関数って何？** <!-- What is a hash function? -->

簡単に言えば、ハッシュ関数は一方向性のある数学関数の一種です: 値からハッシュを計算するのは簡単ですが、あるハッシュがどの値から生成されるかを計算する方法はありません。ハッシュ関数のごく単純な例は `strlen` 関数です。文字列の長さを計算するのは非常に簡単で、与えられた文字列の長さが変わることもありません（勿論、文字列自体が変わっていない場合！）が、その長さだけを与えられても、元の文字列を得ることはできません。*暗号学的*ハッシュ関数は、ある与えられたハッシュを生成する入力を計算することが実際には不可能なほどに難しいという性質を付け加えて、同じものをさらに複雑にしただけのものです。（ `strlen` では、 `strlen(i) == 12` となる入力 `i` を生成するには、単に12文字のランダムな文字を入力すればよいです。 SHA-1 のようなアルゴリズムでは、ずっと長く、実際には不可能なほど長い時間が掛かります。［原註: [SHA-1 で衝突が発見された](https://shattered.io/)ことを知っているかもしれません。実際には Git はもう SHA-1 を使っていません: SHA-1 ではなく、衝突することが知られている2つの PDF ファイル以外の全ての既知の入力に同じハッシュを適用する、[強化された変種](https://github.com/git/git/blob/26e47e261e969491ad4e3b6c298450c061749c9e/Documentation/technical/hash-function-transition.txt#L34-L36)を使っています。］ <!-- Simply put, a hash function is a kind of unidirectional mathematical function: it is easy to compute the hash of a value, but there’s no way to compute which value produced a hash. A very simple example of a hash function is the strlen function. It’s really easy to compute the length of a string, and the length of a given string will never change (unless the string itself changes, of course!) but it’s impossible to retrieve the original string, given only its length. Cryptographic hash functions are just a much more complex version of the same, with the added property that computing an input meant to produce a given hash is hard enough to be practically impossible. (With strlen, producing an input i with strlen(i) == 12, you just have to type twelve random characters. With algorithms such as SHA-1. it would take much, much longer — long enough to be practically impossible[footnote: You may know that collisions have been discovered in SHA-1. Git actually doesn’t use SHA-1 anymore: it uses a hardened variant which is not SHA, but which applies the same hash to every known input but the two PDF files known to collide.]. -->

---

オブジェクトストレージシステムの実装を始める前に、正確なストレージフォーマットを理解しなければなりません。オブジェクトは、そのタイプ（ `blob` 、 `commit` 、 `tag` または `tree` ）を指定するヘッダーで始まります。このヘッダーの後は、 ASCII スペース (0x20) 、 ASCII 数字でバイト単位のオブジェクトのサイズ、 null (0x00) （ナルバイト）、オブジェクトのコンテンツと続きます。 Wyag のリポジトリのあるコミットオブジェクトの最初の48バイトは、このようになっています: <!-- Before we start implementing the object storage system, we must understand their exact storage format. An object starts with a header that specifies its type: blob, commit, tag or tree. This header is followed by an ASCII space (0x20), then the size of the object in bytes as an ASCII number, then null (0x00) (the null byte), then the contents of the object. The first 48 bytes of a commit object in Wyag’s repo look like this: -->

```
00000000  63 6f 6d 6d 69 74 20 31  30 38 36 00 74 72 65 65  |commit 1086.tree|
00000010  20 32 39 66 66 31 36 63  39 63 31 34 65 32 36 35  | 29ff16c9c14e265|
00000020  32 62 32 32 66 38 62 37  38 62 62 30 38 61 35 61  |2b22f8b78bb08a5a|
```

最初の行には、タイプヘッダー、スペース (`0x20`) 、 ASCII でのサイズ (1086) 、およびナルセパレーター `0x00` があります。最初の行の最後の4バイトは、オブジェクトのコンテンツの始まりであり、「 tree 」という単語です。これについては、コミットについて話すときにさらに議論します。 <!-- In the first line, we see the type header, a space (0x20), the size in ASCII (1086) and the null separator 0x00. The last four bytes on the first line are the beginning of that object’s contents, the word “tree” — we’ll discuss that further when we’ll talk about commits. -->

オブジェクト（ヘッダーとコンテンツ）は `zlib` で圧縮されて保存されます。 <!-- The objects (headers and contents) are stored compressed with zlib. -->

### 総称的なオブジェクトのオブジェクト
<!-- A generic object object -->

オブジェクトには複数のタイプがありますが、これらは全て同一の保存・検索メカニズムと同一の一般的なヘッダーフォーマットを共有しています。様々なタイプのオブジェクトの詳細に飛び込む前に、共通の機能を抽象化しておく必要があります。最も簡単な方法は、2つの未実装メソッド（ `serialize()` と `deserialize()` ）を持つ、総称的な `GitObject` を作ることです。後ほど、この総称クラスをサブクラス化し、オブジェクトフォーマットごとに実際にこれらの関数を実装します。 <!-- Objects can be of multiple types, but they all share the same storage/retrieval mechanism and the same general header format. Before we dive into the details of various types of objects, we need to abstract over these common features. The easiest way is to create a generic GitObject with two unimplemented methods: serialize() and deserialize(). Later, we’ll subclass this generic class, actually implementing these functions for each object format. -->

``` py
class GitObject (object):

    repo = None

    def __init__(self, repo, data=None):
        self.repo=repo

        if data != None:
            self.deserialize(data)

    def serialize(self):
        """This function MUST be implemented by subclasses.

It must read the object's contents from self.data, a byte string, and do
whatever it takes to convert it into a meaningful representation.  What exactly that means depend on each subclass."""
        raise Exception("Unimplemented!")

    def deserialize(self, data):
        raise Exception("Unimplemented!")
```

### オブジェクトの読み込み
<!-- Reading objects -->

オブジェクトを読み込むには、そのハッシュを知る必要があります。そして、このハッシュからパスを（上で説明した公式（最初の2文字、それからディレクトリ区切り文字 `/` 、それから残りの部分）で）計算し、 gitdir の「 objects 」ディレクトリの中でそれを探します。つまり、 `e673d1b7eaa0aa01b5bc2442d570a765bdaae751` へのパスは `.git/objects/e6/73d1b7eaa0aa01b5bc2442d570a765bdaae751` です。 <!-- To read an object, we need to know its hash. We then compute its path from this hash (with the formula explained above: first two characters, then a directory delimiter /, then the remaining part) and look it up inside of the “objects” directory in the gitdir. That is, the path to e673d1b7eaa0aa01b5bc2442d570a765bdaae751 is .git/objects/e6/73d1b7eaa0aa01b5bc2442d570a765bdaae751. -->

そして、そのファイルをバイナリファイルとして読み込み、 `zlib` を使って展開します。 <!-- We then read that file as a binary file, and decompress it using zlib. -->

展開されたデータから2つのヘッダーコンポーネントを抜き出します: オブジェクトタイプとサイズです。タイプから実際に使うクラスを決定します。サイズは Python の整数に変換し、一致するかどうかを確認します。 <!-- From the decompressed data, we extract the two header components: the object type and its size. From the type, we determine the actual class to use. We convert the size to a Python integer, and check if it matches. -->

全て終わったら、そのオブジェクトフォーマットに適したコンストラクターを呼び出すだけです。 <!-- When all is done, we just call the correct constructor for that object’s format. -->

``` py
def object_read(repo, sha):
    """Read object object_id from Git repository repo.  Return a
    GitObject whose exact type depends on the object."""

    path = repo_file(repo, "objects", sha[0:2], sha[2:])

    with open (path, "rb") as f:
        raw = zlib.decompress(f.read())

        # Read object type
        x = raw.find(b' ')
        fmt = raw[0:x]

        # Read and validate object size
        y = raw.find(b'\x00', x)
        size = int(raw[x:y].decode("ascii"))
        if size != len(raw)-y-1:
            raise Exception("Malformed object {0}: bad length".format(sha))

        # Pick constructor
        if   fmt==b'commit' : c=GitCommit
        elif fmt==b'tree'   : c=GitTree
        elif fmt==b'tag'    : c=GitTag
        elif fmt==b'blob'   : c=GitBlob
        else:
            raise Exception("Unknown type %s for object %s".format(fmt.decode("ascii"), sha))

        # Call constructor and return object
        return c(repo, raw[y+1:])
```

<a id="foo"></a>
`object_find` 関数はまだ導入していません。実のところ、これはプレースホルダーであり、このようなものです: <!-- We haven’t introduced the object_find function yet. It’s actually a placeholder, and looks like this: -->

``` py
def object_find(repo, name, fmt=None, follow=True):
    return name
```

この奇妙で小さな関数がある理由は、 Git にはオブジェクトを参照する方法が*たくさん*あるからです: 完全なハッシュ、短いハッシュ、タグ……。 `object_find()` が私達の名前解決関数となります。[後で]()実装するというだけで、これは一時的なプレースホルダーです。つまり、本物を実装するまで、オブジェクトを参照する方法は完全なハッシュによる方法しか無いということです。 <!-- The reason for this strange small function is that Git has a lot of ways to refer to objects: full hash, short hash, tags… object_find() will be our name resolution function. We’ll only implement it later, so this is a temporary placeholder. This means that until we implement the real thing, the only way we can refer to an object will be by its full hash. -->

### オブジェクトの書き込み
<!-- Writing objects -->

オブジェクトの書き込みは読み込みの逆です: ハッシュを計算し、ヘッダーを挿入し、全てを zlib 圧縮し、その結果を所定の位置に書き込みます。あまり多く説明する必要は無いと思いますが、ハッシュはヘッダーが追加された**後で**計算されるということには注意してください。 <!-- Writing an object is reading it in reverse: we compute the hash, insert the header, zlib-compress everything and write the result in place. This really shouldn’t require much explanation, just notice that the hash is computed after the header is added. -->

``` py
def object_write(obj, actually_write=True):
    # Serialize object data
    data = obj.serialize()
    # Add header
    result = obj.fmt + b' ' + str(len(data)).encode() + b'\x00' + data
    # Compute hash
    sha = hashlib.sha1(result).hexdigest()

    if actually_write:
        # Compute path
        path=repo_file(obj.repo, "objects", sha[0:2], sha[2:], mkdir=actually_write)

        with open(path, 'wb') as f:
            # Compress and write
            f.write(zlib.compress(result))

    return sha
```

### blob の扱い
<!-- Working with blobs -->

blob は、実際のフォーマットが無いため、4つの Git オブジェクトタイプの中で最もシンプルです。 blob はユーザーのコンテンツです: git に置かれている全てのファイルは blob として保存されます。 blob は、基本的なオブジェクトストレージメカニズムを超える構文や制約が無い（単なる不特定多数のデータです。）ので、簡単に操作できます。 `GitBlob` クラスの作成は些細なことです。 `serialize` 関数と `deserialize` 関数は、入力を変更せずに保存したり返したりするだけです。 <!-- Of the four Git object types, blobs are the simplest, because they have no actual format. Blobs are user content: every file you put in git is stored as a blob. That make them easy to manipulate, because they have no actual syntax or constraints beyond the basic object storage mechanism: they’re just unspecified data. Creating a GitBlob class is thus trivial, the serialize and deserialize functions just have to store and return their input unmodified. -->

``` py
class GitBlob(GitObject):
    fmt=b'blob'

    def serialize(self):
        return self.blobdata

    def deserialize(self, data):
        self.blobdata = data
```

### cat-file コマンド
<!-- The cat-file command -->

これで `wyag cat-file` を作成できるようになりました。 `git cat-file` の基本的な構文は2つの位置指定引数だけです: タイプとオブジェクト識別子: <!-- With all that, we can now create wyag cat-file. The basic syntax of git cat-file is just two positional arguments: a type and an object identifier: -->

```
git cat-file TYPE OBJECT
```

サブパーザーは非常にシンプルです: <!-- The subparser is very simple: -->

``` py
argsp = argsubparsers.add_parser("cat-file",
                                help="Provide content of repository objects")

argsp.add_argument("type",
                   metavar="type",
                   choices=["blob", "commit", "tag", "tree"],
                   help="Specify the type")

argsp.add_argument("object",
                   metavar="object",
                   help="The object to display")
```

そして、これらの関数は何も説明する必要は無いでしょう。すでに書かれたロジックを呼び出すだけです: <!-- And the functions themselves shouldn’t need any explanation, they just call into already written logic: -->

``` py
def cmd_cat_file(args):
    repo = repo_find()
    cat_file(repo, args.object, fmt=args.type.encode())

def cat_file(repo, obj, fmt=None):
    obj = object_read(repo, object_find(repo, obj, fmt=fmt))
    sys.stdout.buffer.write(obj.serialize())
```

### hash-object コマンド<!-- The hash-object command -->

`hash-object` は基本的には `cat-file` の反対です: ファイルを読み込み、オブジェクトとしてハッシュを計算し、（ -w フラグが渡された場合）リポジトリに保存するか、単にハッシュを印字するかします。 <!-- hash-object is basically the opposite of cat-file: it reads a file, computes its hash as an object, either storing it in the repository (if the -w flag is passed) or just printing its hash. -->

`git hash-object` の基本的な構文はこのようなものです: <!-- The basic syntax of git hash-object looks like this: -->

```
git hash-object [-w] [-t TYPE] FILE
```

これは: <!-- Which converts to: -->

``` py
argsp = argsubparsers.add_parser(
    "hash-object",
    help="Compute object ID and optionally creates a blob from a file")

argsp.add_argument("-t",
                   metavar="type",
                   dest="type",
                   choices=["blob", "commit", "tag", "tree"],
                   default="blob",
                   help="Specify the type")

argsp.add_argument("-w",
                   dest="write",
                   action="store_true",
                   help="Actually write the object into the database")

argsp.add_argument("path",
                   help="Read object from <file>")
```

に変換されます。 <!-- Which converts to: -->

実際の実装は非常にシンプルです。いつものように、小さなブリッジ関数を作ります: <!-- The actual implementation is very simple. As usual, we create a small bridge function: -->

``` py
def cmd_hash_object(args):
    if args.write:
        repo = GitRepository(".")
    else:
        repo = None

    with open(args.path, "rb") as fd:
        sha = object_hash(fd, args.type.encode(), repo)
        print(sha)
```

そして、実際の実装です。引数 repo は省略可能です: <!-- and the actual implementation. The repo argument is optional: -->

``` py
def object_hash(fd, fmt, repo=None):
    data = fd.read()

    # Choose constructor depending on
    # object type found in header.
    if   fmt==b'commit' : obj=GitCommit(repo, data)
    elif fmt==b'tree'   : obj=GitTree(repo, data)
    elif fmt==b'tag'    : obj=GitTag(repo, data)
    elif fmt==b'blob'   : obj=GitBlob(repo, data)
    else:
        raise Exception("Unknown type %s!" % fmt)

    return object_write(obj, repo)
```

### packfile はどう？
<!-- What about packfiles? -->

今実装したのは「緩いオブジェクト」と呼ばれるものです。 Git には packfile と呼ばれる2つ目のオブジェクトストレージメカニズムがあります。 packfile は、緩いオブジェクトと比べて効率的ですが、より複雑でもあります。そして、 `wyag` では実装する価値がありません。簡単に言えば、 packfile は（ `tar` のような）緩いオブジェクトを集めたものでありながら、一部はデルタとして（別のオブジェクトの変形として）保存されているようなものです。 packfile は wyag がサポートするには複雑すぎます。 <!-- What we’ve just implemented is called “loose objects”. Git has a second object storage mechanism called packfiles. Packfiles are much more efficient, but also much more complex, than loose objects. And aren’t worth implementing in wyag. Simply put, a packfile is a compilation of loose objects (like a tar) but some are stored as deltas (as a transformation of another object). Packfiles are way too complex to be supported by wyag. -->

packfile は `.git/objects/pack/` に保存されます。拡張子は `.pack `であり、拡張子が `.idx` である同名のインデックスファイルが付属しています。 packfile を緩いオブジェクトフォーマットに変換したい（たとえば、既存のリポジトリで `wyag` を使って遊びたい）場合、次のような解決策があります。 <!-- The packfile is stored in .git/objects/pack/. It has a .pack extension, and is accompanied by an index file of the same name with the .idx extension. Should you want to convert a packfile to loose objects format (to play with wyag on an existing repo, for example), here’s the solution. -->

まず、 packfile を gitdir の外に*移動させ*ます。コピーするぐらいなら移動させるべきです。 <!-- First, move the packfile outside the gitdir. Copying it, you have to move it. -->

``` sh
mv .git/objects/pack/pack-d9ef004d4ca729287f12aaaacf36fee39baa7c9d.pack .
```

`.idx` は無視できます。それから、ワークツリーから、それを `cat` して、その結果を `git unpack-objects` にパイプします: <!-- You can ignore the .idx. Then, from the worktree, just cat it and pipe the result to git unpack-objects: -->

``` sh
cat pack-d9ef004d4ca729287f12aaaacf36fee39baa7c9d.pack | git unpack-objects
```

## コミット履歴の読み取り: log
<!-- Reading commit history: log -->

### コミットのパーズ
<!-- Parsing commits -->

オブジェクトを読み書きできるようになったので、コミットについて考えましょう。コミットオブジェクト（非圧縮、ヘッダーを除く）はこのような見た目をしています: <!-- Now that we can read and write objects, we should consider commits. A commit object (uncompressed, without headers) looks like this: -->

```
tree 29ff16c9c14e2652b22f8b78bb08a5a07930c147
parent 206941306e8a8af65b66eaaaea388a7ae24d49a0
author Thibault Polge <thibault@thb.lt> 1527025023 +0200
committer Thibault Polge <thibault@thb.lt> 1527025044 +0200
gpgsig -----BEGIN PGP SIGNATURE-----

 iQIzBAABCAAdFiEExwXquOM8bWb4Q2zVGxM2FxoLkGQFAlsEjZQACgkQGxM2FxoL
 kGQdcBAAqPP+ln4nGDd2gETXjvOpOxLzIMEw4A9gU6CzWzm+oB8mEIKyaH0UFIPh
 rNUZ1j7/ZGFNeBDtT55LPdPIQw4KKlcf6kC8MPWP3qSu3xHqx12C5zyai2duFZUU
 wqOt9iCFCscFQYqKs3xsHI+ncQb+PGjVZA8+jPw7nrPIkeSXQV2aZb1E68wa2YIL
 3eYgTUKz34cB6tAq9YwHnZpyPx8UJCZGkshpJmgtZ3mCbtQaO17LoihnqPn4UOMr
 V75R/7FjSuPLS8NaZF4wfi52btXMSxO/u7GuoJkzJscP3p4qtwe6Rl9dc1XC8P7k
 NIbGZ5Yg5cEPcfmhgXFOhQZkD0yxcJqBUcoFpnp2vu5XJl2E5I/quIyVxUXi6O6c
 /obspcvace4wy8uO0bdVhc4nJ+Rla4InVSJaUaBeiHTW8kReSFYyMmDCzLjGIu1q
 doU61OM3Zv1ptsLu3gUE6GU27iWYj2RWN3e3HE4Sbd89IFwLXNdSuM0ifDLZk7AQ
 WBhRhipCCgZhkj9g2NEk7jRVslti1NdN5zoQLaJNqSwO1MtxTmJ15Ksk3QP6kfLB
 Q52UWybBzpaP9HEd4XnR+HuQ4k2K0ns2KgNImsNvIyFwbpMUyUWLMPimaV1DWUXo
 5SBjDB/V/W2JBFR+XKHFJeFwYhj7DD/ocsGr4ZMx/lgc8rjIBkI=
 =lgTX
 -----END PGP SIGNATURE-----

Create first draft
```

このフォーマットは [RFC 2822](https://www.ietf.org/rfc/rfc2822.txt) で定義されているメールメッセージの簡略版です。一連のキーバリューペアで始まり、コミットメッセージで終わります。キーとバリューの区切り文字にはスペースを使い、コミットメッセージは複数行に跨る可能性があります。値は複数行に跨ることができ、続く行はスペースで始まります。パーザーはこのスペースを取り除かなければなりません。 <!-- The format is a simplified version of mail messages, as specified in RFC 2822. It begins with a series of key-value pairs, with space as the key/value separator, and ends with the commit message, that may span over multiple lines. Values may continue over multiple lines, subsequent lines start with a space which the parser must drop. -->

これらのフィールドを見てみましょう: <!-- Let’s have a look at those fields: -->

- `tree` は、ツリーオブジェクトへの参照であり、すぐ後で見ることになるオブジェクトタイプです。ツリーは、ブロブの ID をファイルシステム上の位置に割り当て、作業ツリーの状態を記述します。簡単に言えば、コミットの実際の内容（ファイルとその行き先）です。 <!-- tree is a reference to a tree object, a type of object that we’ll see soon. A tree maps blobs IDs to filesystem locations, and describes a state of the work tree. Put simply, it is the actual content of the commit: files, and where they go. -->
- `parent` はこのコミットの親コミットへの参照です。このフィールドは繰り返されることもあります: たとえばマージコミットには複数の親があります。存在しないこともあります: リポジトリの最初のコミットには明らかに親がありません。 <!-- parent is a reference to the parent of this commit. It may be repeated: merge commits, for example, have multiple parents. It may also be absent: the very first commit in a repository obviously doesn’t have a parent. -->
- コミットの作者がコミットできる人であるとは限らないため、 `author` と `committer` は分けられています。（ GitHub ユーザーにとっては明らかでないかもしれませんが、多くのプロジェクトで電子メール経由で Git を実行しています。） <!-- author and committer are separate, because the author of a commit is not necessarily the person who can commit it (This may not be obvious for GitHub users, but a lot of projects do Git through e-mail) -->
- `gpgsig` はこのオブジェクトの PGP 署名です。 <!-- gpgsig is the PGP signature of this object. -->

このフォーマットのシンプルなパーザーを書くところから始めます。コードは明白です。これから作ろうとしている関数の名前 `kvlm_parse()` は紛らわしいかもしれません: タグが全く同じフォーマットなので、この関数は両方のオブジェクトタイプに使うため、 `commit_parse()` という名前ではありません。私は KVLM を「メッセージ付きのキーバリューリスト (Key-Value List with Message) 」という意味で使っています。 <!-- We’ll start by writing a simple parser for the format. The code is obvious. The name of the function we’re about to create, kvlm_parse(), may be confusing: it isn’t called commit_parse() because tags have the very same format, so we’ll use it for both objects types. I use KVLM to mean “Key-Value List with Message”. -->

``` py
def kvlm_parse(raw, start=0, dct=None):
    if not dct:
        dct = collections.OrderedDict()
        # You CANNOT declare the argument as dct=OrderedDict() or all
        # call to the functions will endlessly grow the same dict.

    # We search for the next space and the next newline.
    spc = raw.find(b' ', start)
    nl = raw.find(b'\n', start)

    # If space appears before newline, we have a keyword.

    # Base case
    # =========
    # If newline appears first (or there's no space at all, in which
    # case find returns -1), we assume a blank line.  A blank line
    # means the remainder of the data is the message.
    if (spc < 0) or (nl < spc):
        assert(nl == start)
        dct[b''] = raw[start+1:]
        return dct

    # Recursive case
    # ==============
    # we read a key-value pair and recurse for the next.
    key = raw[start:spc]

    # Find the end of the value.  Continuation lines begin with a
    # space, so we loop until we find a "\n" not followed by a space.
    end = start
    while True:
        end = raw.find(b'\n', end+1)
        if raw[end+1] != ord(' '): break

    # Grab the value
    # Also, drop the leading space on continuation lines
    value = raw[spc+1:end].replace(b'\n ', b'\n')

    # Don't overwrite existing data contents
    if key in dct:
        if type(dct[key]) == list:
            dct[key].append(value)
        else:
            dct[key] = [ dct[key], value ]
    else:
        dct[key]=value

    return kvlm_parse(raw, start=end+1, dct=dct)
```

ここで `OrderedDict` を使っているのは、私達が実装した `cat-file` はコミットオブジェクトをパーズしてからシリアライズしなおすことで印字するため、定義された順序と全く同じ順序のフィールドが必要だからです。また、 Git ではコミットやタグに現れるキーの順序も重要なようです。 <!-- We use an OrderedDict here because cat-file, as we’ve implemented it, will print a commit object by parsing it and re-serializing it, so we need fields to be in the exact same order they were defined. Also, in Git, the order keys appear in commit and tag object seem to matter. -->

似たようなオブジェクトを書く必要があるので、 `kvlm_serialize()` をツールキットに追加しましょう。 <!-- We’re going to need to write similar objects, so let’s add a kvlm_serialize() function to our toolkit. -->

``` py
def kvlm_serialize(kvlm):
    ret = b''

    # Output fields
    for k in kvlm.keys():
        # Skip the message itself
        if k == b'': continue
        val = kvlm[k]
        # Normalize to a list
        if type(val) != list:
            val = [ val ]

        for v in val:
            ret += k + b' ' + (v.replace(b'\n', b'\n ')) + b'\n'

    # Append message
    ret += b'\n' + kvlm[b'']

    return ret
```

### コミットオブジェクト
<!-- The Commit object -->

パーザーが出来たので、 `GitCommit` クラスが作れます: <!-- Now we have the parser, we can create the GitCommit class: -->

``` py
class GitCommit(GitObject):
    fmt=b'commit'

    def deserialize(self, data):
        self.kvlm = kvlm_parse(data)

    def serialize(self):
        return kvlm_serialize(self.kvlm)
```

### log コマンド
<!-- The log command -->

Git が提供するものよりもずっとシンプルなバージョンの `log` を実装します。最も重要なことは、ログの表現を*全く*扱わないということです。代わりに、 Graphviz データをダンプし、ユーザーが `dot` を使って実際のログをレンダリングできるようにします。 <!-- We’ll implement a much, much simpler version of log than what Git provides. Most importantly, we won’t deal with representing the log at all. Instead, we’ll dump Graphviz data and let the user use dot to render the actual log. -->

``` py
argsp = argsubparsers.add_parser("log", help="Display history of a given commit.")
argsp.add_argument("commit",
                   default="HEAD",
                   nargs="?",
                   help="Commit to start at.")
```

``` py
def cmd_log(args):
    repo = repo_find()

    print("digraph wyaglog{")
    log_graphviz(repo, object_find(repo, args.commit), set())
    print("}")

def log_graphviz(repo, sha, seen):

    if sha in seen:
        return
    seen.add(sha)

    commit = object_read(repo, sha)
    assert (commit.fmt==b'commit')

    if not b'parent' in commit.kvlm.keys():
        # Base case: the initial commit.
        return

    parents = commit.kvlm[b'parent']

    if type(parents) != list:
        parents = [ parents ]

    for p in parents:
        p = p.decode("ascii")
        print ("c_{0} -> c_{1};".format(sha, p))
        log_graphviz(repo, p, seen)
```

これで、私達の log コマンドをこのように使うことができます: <!-- You can now use our log command like this: -->

```
wyag log e03158242ecab460f31b0d6ae1642880577ccbe8 > log.dot
dot -O -Tpdf log.dot
```

### コミットの分析
<!-- Anatomy of a commit -->

ここで、いくつかのことに気付いたかもしれません。 <!-- You may have noticed a few things right now. -->

まず第一に、私達はコミットを弄って、コミットオブジェクトをざっと見て回って、コミット履歴のグラフを構築してきましたが、ワークツリー内の単一のファイルや、ブロブには一度も触れていませんでした。*コミットの内容について考えずに*、コミットに多くのことを行ってきました。これは重要です: ワークツリーの内容はコミットの一部でしかありません。しかし、コミットは全てのもの（内容、作者、そして親もです。）から成ります。コミットの ID （ SHA-1 ハッシュ）がコミットオブジェクト全体から計算されるということを覚えていれば、コミットは不変であるということの意味が分かるでしょう: 作者、親コミット、あるいは1つのファイルを変更する場合、実際には新しく、異なったオブジェクトを作っています。全てのコミットは、最初のコミットに至るまで、場所と、リポジトリ全体との関係に縛り付けられています。言い換えれば、与えられたコミット ID は、ファイルの内容を識別するだけではなく、そのコミットを履歴全体とリポジトリ全体に縛り付けもするということです。 <!-- First and foremost, we’ve been playing with commits, browsing and walking through commit objects, building a graph of commit history, without ever touching a single file in the worktree or a blob. We’ve done a lot with commits without considering their contents. This is important: work tree contents are just a part of a commit. But a commit is made of everything: its contents, its authors, and also its parents. If you remember that the ID (the SHA-1 hash) of a commit is computed from the whole commit object, you’ll understand what it means that commits are immutable: if you change the author, the parent commit or a single file, you’ve actually created a new, different object. Each and every commit is bound to its place and its relationship to the whole repository up to the very first commit. To put it otherwise, a given commit ID not only identifies some file contents, but it also binds the commit to its whole history and to the whole repository. -->

コミットの視点から見ると時間が逆行しているということにも注目する価値があります: 私達は、午後の気晴らしとしての慎ましやかな出発から、数行のコードで始まり、いくつかの初期コミットがあって、そして現在の状態（数百万行のコード、数十人の貢献者、その他）まで進歩するプロジェクトの歴史について考えることに慣れています。しかし、各コミットは未来を全く認識しておらず、過去のみにリンクしています。コミットには「記憶」はありますが、予感はありません。 <!-- It’s also worth noting that from the point of view of a commit, time runs backwards: we’re used to considering the history of a project from its humble beginnings as an evening distraction, starting with a few lines of code, some initial commits, and progressing to its present state (millions of lines of code, dozens of contributors, whatever). But each commit is completely unaware of its future, it’s only linked to the past. Commits have “memory”, but no premonition. -->

---

**Note**

テリー・プラチェットのディスクワールドでは、トロールは、自分達が未来から過去に向かって進んでいると信じています。その信念の背後にある論法は、歩くときに見えるものは*目の前*にあるものであるというものです。時間の中で知覚しうるものは過去だけであり、それは記憶しているからです。したがって、それはあなたが向かっている場所です。 Git はディスクワールドのトロールによって書かれたものです。 <!-- In Terry Pratchett’s Discworld, trolls believe they progress in time from the future to the past. The reasoning behind that belief is that when you walk, what you can see is what’s ahead of you. Of time, all you can perceive is the past, because you remember; hence it’s where you’re headed. Git was written by a Discworld troll. -->

---

では、何がコミットを作るのでしょうか？　要約すると: <!-- So what makes a commit? To sum it up: -->

- ツリーオブジェクト（これから議論します。すなわち、ワークツリー、ファイル、およびディレクトリの内容です。） <!-- A tree object, which we’ll discuss now, that is, the contents of a worktree, files and directories; -->
- 0個または1個以上の親 <!-- Zero, one or more parents; -->
- 作者の ID （名前と電子メールアドレス） <!-- An author identity (name and email); -->
- コミッターの ID （名前と電子メールアドレス） <!-- A committer identity (name and email); -->
- 省略可能な PGP 署名 <!-- An optional PGP signature -->
- メッセージ <!-- A message; -->

この全体が SHA-1 識別子にハッシュされます。 <!-- All this hashed together in a SHA-1 identifier. -->

---

**Note**

**待った、それは Git をブロックチェーンにするのでは？**
<!-- Wait, does that make Git a blockchain? -->

暗号通貨のせいで、最近はブロックチェーンが誇大広告になっています。そう、*ある意味では* Git はブロックチェーンです: Git は、各要素がその構造の履歴全体に関聯付けられていることを保証する方法で、暗号学的手段によって結び付けられたブロック（コミット）の列です。ですが、この比較を真剣に考えすぎないでください。 GitCoin は必要ありません。本当に。 <!-- Because of cryptocurrencies, blockchains are all the hype these days. And yes, in a way, Git is a blockchain: it’s a sequence of blocks (commits) tied together by cryptographic means in a way that guarantee that each single element is associated to the whole history of the structure. Don’t take the comparison too seriously, though: we don’t need a GitCoin. Really, we don’t. -->

---

## コミットデータの読み取り: checkout
<!-- Reading commit data: checkout -->

コミットが特定の状態のファイルとディレクトリよりもずっと多くのものを保持しているのは良いことですが、本当に便利なものではありません。おそらくツリーオブジェクトも実装しはじめる時期に来たので、コミットを作業ツリーにチェックアウトできるようにしましょう。 <!-- It’s all well that commits hold a lot more than files and directories in a given state, but that doesn’t make them really useful. It’s probably time to start implementing tree objects as well, so we’ll be able to checkout commits into the work tree. -->

### ツリーには何があるの？
<!-- What’s in a tree? -->

非形式的には、ツリーは作業ツリーの内容を記述し、ブロブをパスに関聯付けます。ファイルモード、（作業ツリーに対する相対）パス、そして SHA-1 から成る3要素タプルの配列です。典型的なツリーの内容はこのような見た目をしています:
<!-- Informally, a tree describes the content of the work tree, that it, it associates blobs to paths. It’s an array of three-element tuples made of a file mode, a path (relative to the worktree) and a SHA-1. A typical tree contents may look like this: -->

|  モード  |                    SHA-1                   |     パス     |
|----------|--------------------------------------------|--------------|
| `100644` | `894a44cc066a027465cd26d634948d56d13af9af` | `.gitignore` |
| `100644` | `94a9ed024d3859793618152ea559a168bbcbb5e2` | `LICENSE`    |
| `100644` | `bab489c4f4600a38ce6dbfd652b90383a4aa3e45` | `README.md`  |
| `100644` | `6d208e47659a2a10f5f8640e0155d9276a2130a9` | `src`        |
| `040000` | `e7445b03aea61ec801b20d6ab62f076208b7d097` | `tests`      |
| `040000` | `d5ec863f17f3a2e92aa8f6b66ac18f7b09fd1b38` | `main.c`     |

モードは単なるファイルの[モード](https://en.wikipedia.org/wiki/File_system_permissions)、パスはファイルの場所です。 SHA-1 はブロブか別のツリーオブジェクトのどちらかを参照しています。ブロブならパスはファイル、ツリーならパスはディレクトリです。このツリーをファイルシステム内でインスタンス化するには、最初のパス (`.gitignore`) に関聯付けられたオブジェクトを読み込み、そのタイプを確認します。これはブロブなので、このブロブの内容で、 `.gitignore` という名前のファイルを作るだけです。 `LICENSE` と `README.md` については同様です。しかし、 `src` に関聯付けられているオブジェクトはブロブではなく、別のツリーです: ディレクトリ `src` を作り、そのディレクトリの中で、新しいツリーを使って同じ操作を繰り返します。 <!-- Mode is just the file’s mode, path is its location. The SHA-1 refers to either a blob or another tree object. If a blob, the path is a file, if a tree, it’s directory. To instantiate this tree in the filesystem, we would begin by loading the object associated to the first path (.gitignore) and check its type. Since it’s a blob, we’ll just create a file called .gitignore with this blob’s contents; and same for LICENSE and README.md. But the object associated with src is not a blob, but another tree: we’ll create the directory src and repeat the same operation in that directory with the new tree. -->

---

**Warning**

**パスは単一のファイルシステムエントリーです** <!-- A path is a single filesystem entry -->

パスは丁度1つのオブジェクトを識別します。2つでも3つでもありません。5段階にネストしたディレクトリがあるなら、再帰的に互いを参照するツリーオブジェクトが5つ必要になります。 `dir1/dir2/dir3/dir4/dir5` のように、単一のツリーエントリーに完全なパスを入れるショートカットはできません。 <!-- The path identifies exactly one object. Not two, not three. If you have five levels of nested directories, you’re going to need five tree objects recursively referring to one another. You cannot take the shortcut of putting a full path in a single tree entry, like dir1/dir2/dir3/dir4/dir5. -->

---

### ツリーのパーズ
<!-- Parsing trees -->

タグやコミットとは違い、ツリーオブジェクトはバイナリオブジェクトです。でも、フォーマットは実際には非常にシンプルです。ツリーは、このフォーマットのレコードを連結したものです: <!-- Unlike tags and commits, tree objects are binary objects, but their format is actually quite simple. A tree is the concatenation of records of the format: -->

```
[mode] space [path] 0x00 [sha-1]
```

- `[mode]` は、最大6バイトの、ファイルモードの ASCII 表現です。たとえば、 100644 はバイト値 49 (ASCII 「1」) 、 48 (ASCII 「0」) 、 48 、 54 、 52 、 52 にエンコードされます。 <!-- [mode] is up to six bytes and is an ASCII representation of a file mode. For example, 100644 is encoded with byte values 49 (ASCII “1”), 48 (ASCII “0”), 48, 54, 52, 52. -->
- それに 0x20 、 ASCII スペースが続きます。 <!-- It’s followed by 0x20, an ASCII space; -->
- ナル (0x00) で終わるパスが続きます。 <!-- Followed by the null-terminated (0x00) path; -->
- 20バイトの、バイナリエンコーディングされた、オブジェクトの SHA-1 が続きます。なぜバイナリなのか？　神のみぞ知る。 <!-- Followed by the object’s SHA-1 in binary encoding, on 20 bytes. Why binary? God only knows. -->

パーザーは非常にシンプルなものになります。まず、単一のレコード（葉、単一のパス）のための、小さなオブジェクトラッパーを作ります: <!-- The parser is going to be quite simple. First, create a tiny object wrapper for a single record (a leaf, a single path): -->

``` py
class GitTreeLeaf(object):
    def __init__(self, mode, path, sha):
        self.mode = mode
        self.path = path
        self.sha = sha
```

ツリーオブジェクトは同じ基本的なデータ構造の繰り返しに過ぎないので、パーザーは2つの関数で書きます。1つは単一のレコードを展開するパーザーです。パーズされたデータと、入力データの中で到達した位置を返します: <!-- Because a tree object is just the repetition of the same fundamental data structure, we write the parser in two functions. First, a parser to extract a single record, which returns parsed data and the position it reached in input data: -->

``` py
def tree_parse_one(raw, start=0):
    # Find the space terminator of the mode
    x = raw.find(b' ', start)
    assert(x-start == 5 or x-start==6)

    # Read the mode
    mode = raw[start:x]

    # Find the NULL terminator of the path
    y = raw.find(b'\x00', x)
    # and read the path
    path = raw[x+1:y]

    # Read the SHA and convert to an hex string
    sha = hex(
        int.from_bytes(
            raw[y+1:y+21], "big"))[2:] # hex() adds 0x in front,
                                           # we don't want that.
    return y+21, GitTreeLeaf(mode, path, sha)
```

そして、「本物の」パーザーです。入力データが無くなるまで前のものをループで呼び出すだけのものです。 <!-- And the “real” parser which just calls the previous one in a loop, until input data is exhausted. -->

``` py
def tree_parse(raw):
    pos = 0
    max = len(raw)
    ret = list()
    while pos < max:
        pos, data = tree_parse_one(raw, pos)
        ret.append(data)

    return ret
```

最後になりましたが、シリアライザーが必要です: <!-- Last but not least, we’ll need a serializer: -->

``` py
def tree_serialize(obj):
    #@FIXME Add serializer!
    ret = b''
    for i in obj.items:
        ret += i.mode
        ret += b' '
        ret += i.path
        ret += b'\x00'
        sha = int(i.sha, 16)
        # @FIXME Does
        ret += sha.to_bytes(20, byteorder="big")
    return ret
```

あとは、これを全て組み合わせてクラスにするだけです: <!-- And now we just have to combine all that into a class: -->

``` py
class GitTree(GitObject):
    fmt=b'tree'

    def deserialize(self, data):
        self.items = tree_parse(data)

    def serialize(self):
        return tree_serialize(self)
```

ついでに wyag に `ls-tree` コマンドを追加しましょう。簡単なので、やらない理由はありません。 <!-- While we’re at it, let’s add the ls-tree command to wyag. It’s so easy there’s no reason not to. -->

``` py
argsp = argsubparsers.add_parser("ls-tree", help="Pretty-print a tree object.")
argsp.add_argument("object",
                   help="The object to show.")

def cmd_ls_tree(args):
    repo = repo_find()
    obj = object_read(repo, object_find(repo, args.object, fmt=b'tree'))

    for item in obj.items:
        print("{0} {1} {2}\t{3}".format(
            "0" * (6 - len(item.mode)) + item.mode.decode("ascii"),
            # Git's ls-tree displays the type
            # of the object pointed to.  We can do that too :)
            object_read(repo, item.sha).fmt.decode("ascii"),
            item.sha,
            item.path.decode("ascii")))
```

### checkout コマンド
<!-- The checkout command -->

実装を分かりやすくするために、実際の git コマンドを過度に単純化します。また、セーフガードもいくつか追加します。私達のバージョンの checkout は次のように動作します: <!-- We’re going to oversimplify the actual git command to make our implementation clear and understandable. We’re also going to add a few safeguards. Here’s how our version of checkout will work: -->

- 2つの引数を取ります: コミットとディレクトリです。 Git の checkout に必要なのはコミットだけです。 <!-- It will take two arguments: a commit, and a directory. Git checkout only needs a commit. -->
- それから、**ディレクトリが空の場合かつ空の場合に限り**、ツリーをディレクトリにインスタンス化します。 Git にはデータの削除を避けるためのセーフガードがたくさんありますが、あまりにも複雑で、 wyag で再現しようとするのは安全ではありません。 wyag の目的は git のデモンストレーションであって、作業用の実装を作ることではないので、この制限は許容範囲内のものです。 <!-- It will then instantiate the tree in the directory, if and only if the directory is empty. Git is full of safeguards to avoid deleting data, which would be too complicated and unsafe to try to reproduce in wyag. Since the point of wyag is to demonstrate git, not to produce a working implementation, this limitation is acceptable. -->

始めましょう。いつものように、サブパーザーが必要です: <!-- Let’s get started. As usual, we need a subparser: -->

``` py
argsp = argsubparsers.add_parser("checkout", help="Checkout a commit inside of a directory.")

argsp.add_argument("commit",
                   help="The commit or tree to checkout.")

argsp.add_argument("path",
                   help="The EMPTY directory to checkout on.")
```

ラッパー関数です: <!-- A wrapper function: -->

``` py
def cmd_checkout(args):
    repo = repo_find()

    obj = object_read(repo, object_find(repo, args.commit))

    # If the object is a commit, we grab its tree
    if obj.fmt == b'commit':
        obj = object_read(repo, obj.kvlm[b'tree'].decode("ascii"))

    # Verify that path is an empty directory
    if os.path.exists(args.path):
        if not os.path.isdir(args.path):
            raise Exception("Not a directory {0}!".format(args.path))
        if os.listdir(args.path):
            raise Exception("Not empty {0}!".format(args.path))
    else:
        os.makedirs(args.path)

    tree_checkout(repo, obj, os.path.realpath(args.path).encode())
```

そして、実際の作業を行う関数です: <!-- And a function to do the actual work: -->

``` py
def tree_checkout(repo, tree, path):
    for item in tree.items:
        obj = object_read(repo, item.sha)
        dest = os.path.join(path, item.path)

        if obj.fmt == b'tree':
            os.mkdir(dest)
            tree_checkout(repo, obj, dest)
        elif obj.fmt == b'blob':
            with open(dest, 'wb') as f:
                f.write(obj.blobdata)
```

## 参照、タグ、そしてブランチ
<!-- Refs, tags and branches -->

### 参照って何？
<!-- What’s a ref? -->

Git の参照（ reference あるいは ref ）は、 git が保持するものの中でも最もシンプルな種類のものです。 `.git/refs` のサブディレクトリにあり、オブジェクトのハッシュの16進表現を ASCII でエンコードしたテキストファイルです。実際、これぐらいシンプルです: <!-- Git references, or refs, are probably the most simple type of things git holds. They live in subdirectories of .git/refs, and are text files containing a hexadecimal representation of an object’s hash, encoded in ASCII. They’re actually as simple as this: -->

```
6071c08bcb4757d8c89a30d9755d2466cef8c1de
```

参照は別の参照を参照することもあり、そうなると間接的にオブジェクトを参照するだけになりますが、その場合はこのようになります: <!-- Refs can also refer to another reference, and thus only indirectly to an object, in which case they look like this: -->

```
ref: refs/remotes/origin/master
```

---

**Note**

**直接参照と間接参照** <!-- Direct and indirect references -->

今後は、 `ref: path/to/other/ref` 形式の参照を**間接**参照と呼び、 SHA-1 オブジェクト ID の参照を**直接参照**と呼びます。
<!-- From now on, I will call a reference of the form ref: path/to/other/ref an indirect reference, and a ref with a SHA-1 object ID a direct reference. -->

---

この節は参照の用途について記述します。今のところ、重要なことはこれだけです: <!-- This whole section will describe the uses of refs. For now, all that matter is this: -->

- 参照はテキストファイルです。 `.git/refs` 階層にあります。 <!-- they’re text files, in the .git/refs hierarchy; -->
- 参照はオブジェクトの sha-1 識別子か別のブランチへの参照を保持しています。 <!-- they hold the sha-1 identifier of an object, or a reference to another branch. -->

参照を扱うためには、まず、シンプルな再帰的ソルバーが必要になります。このソルバーは、参照の名前を取り、再帰参照（上で例示したような、その内容が `ref:` で始まる参照）を最後まで辿り、 SHA-1 を返します: <!-- To work with refs, we’re first going to need a simple recursive solver that will take a ref name, follow eventual recursive references (refs whose content begin with ref:, as exemplified above) and return a SHA-1: -->

``` py
def ref_resolve(repo, ref):
    with open(repo_file(repo, ref), 'r') as fp:
        data = fp.read()[:-1]
        # Drop final \n ^^^^^
    if data.startswith("ref: "):
        return ref_resolve(repo, data[5:])
    else:
        return data
```

小さな関数を2つ作って show-refs コマンドを実装しましょう。1つ目は、参照を集めて辞書として返す、つまらない再帰関数です: <!-- Let’s create two small functions, and implement the show-refs command. First, a stupid recursive function to collect refs and return them as a dict: -->

``` py
def ref_list(repo, path=None):
    if not path:
        path = repo_dir(repo, "refs")
    ret = collections.OrderedDict()
    # Git shows refs sorted.  To do the same, we use
    # an OrderedDict and sort the output of listdir
    for f in sorted(os.listdir(path)):
        can = os.path.join(path, f)
        if os.path.isdir(can):
            ret[f] = ref_list(repo, can)
        else:
            ret[f] = ref_resolve(repo, can)

    return ret
```

``` py
argsp = argsubparsers.add_parser("show-ref", help="List references.")

def cmd_show_ref(args):
    repo = repo_find()
    refs = ref_list(repo)
    show_ref(repo, refs, prefix="refs")

def show_ref(repo, refs, with_hash=True, prefix=""):
    for k, v in refs.items():
        if type(v) == str:
            print ("{0}{1}{2}".format(
                v + " " if with_hash else "",
                prefix + "/" if prefix else "",
                k))
        else:
            show_ref(repo, v, with_hash=with_hash, prefix="{0}{1}{2}".format(prefix, "/" if prefix else "", k))
```

### タグって何？
<!-- What’s a tag? -->

参照の最もシンプルな用途はタグです。タグは、単なるオブジェクト（大抵はコミット）のユーザー定義名です。タグのごく一般的な用途はソフトウェアリリースの識別です: たとえば、プログラムのバージョン 12.78.52 の最後のコミットをマージしたところなので、最近のコミット（ `6071c08` と呼びましょう。）がバージョン 12.78.52 であるとします。この関聯付けを明示するには、次のようにするだけです: <!-- The most simple use of refs is tags. A tag is just a user-defined name for an object, often a commit. A very common use of tags is identifying software releases: You’ve just merged the last commit of, say, version 12.78.52 of your program, so your most recent commit (let’s call it 6071c08) is your version 12.78.52. To make this association explicit, all you have to do is: -->

``` sh
git tag v12.78.52 6071c08
# the object hash ^here^^ is optional and defaults to HEAD.
```

これで、 `6071c08` を指す、 `v12.78.52` という名前の新しいタグが作られます。タグ付けはエイリアス化のようなものです: タグは既存のオブジェクトを参照する新しい方法を導入します。このタグが作られた後は、名前 `v12.78.52` は `6071c08` を参照しています。たとえば、これら2つのコマンドは完全に等価です: <!-- This creates a new tag, called v12.78.52, pointing at 6071c08. Tagging is like aliasing: a tag introduces a new way to refer to an existing object. After the tag is created, the name v12.78.52 refers to 6071c08. For example, these two commands are now perfectly equivalent: -->

``` sh
git checkout v12.78.52
git checkout 6071c08
```

---

**Note**

バージョンはタグの一般的な用途ですが、 Git にあるほとんど全てのものと同様に、タグには事前定義された意味論はありません: タグは、意味してほしいものを意味し、指したいオブジェクトを指すことができ、*ブロブ*をタグ付けすることさえできます！ <!-- Versions are a common use of tags, but like almost everything in Git, tags have no predefined semantics: they mean whatever you want them to mean, and can point to whichever object you want, you can even tag blobs! -->

---

### タグオブジェクトのパーズ
<!-- Parsing tag objects -->

すでに、タグは実際には参照だと推測しているでしょう。タグは `.git/refs/tags/` 階層にあります。唯一の注目する価値のある点は、2つのフレーバー（軽量タグとタグオブジェクト）があることです。 <!-- You’ve probably guessed already that tags are actually refs. They live in the .git/refs/tags/ hierarchy. The only point worth noting is that they come in two flavors: lightweight tags and tags objects. -->

<dl>
<dt>「軽量」タグ <!-- “Lightweight” tags --></dt>
<dd>は、コミット、ツリー、またはブロブへの単なる通常の参照です。 <!-- are just regular refs to a commit, a tree or a blob. --></dd>
<dt>タグオブジェクト <!-- Tag objects --></dt>
<dd>は、 `tag` タイプのオブジェクトを指す通常の参照です。軽量タグと異なり、タグオブジェクトには、作者、日付、省略可能な PGP 署名、そして省略可能な註釈があります。フォーマットはコミットオブジェクトと同じです。 <!-- are regular refs pointing to an object of type tag. Unlike lightweight tags, tag objects have an author, a date, an optional PGP signature and an optional annotation. Their format is the same as a commit object. --></dd>
</dl>

タグオブジェクトは実装する必要すらありません。 `GitCommit` オブジェクトを再利用し、 `fmt` フィールドを変更するだけです: <!-- We don’t even need to implement tag objects, we can reuse GitCommit and just change the fmt field: -->

``` py
class GitTag(GitCommit):
    fmt = b'tag'
```

これでタグがサポートされました。 <!-- And now we support tags. -->

### tag コマンド
<!-- The tag command -->

tag コマンドを追加しましょう。 Git では、このコマンドは2つのことを行います: 新しいタグを作るか、（デフォルトで）既存のタグの一覧を表示するかです。なので、次のように実行できます: <!-- Let’s add the tag command. In Git, it does two things: it creates a new tag or list existing tags (by default). So you can invoke it with: -->

``` sh
git tag                  # List all tags
git tag NAME [OBJECT]    # create a new *lightweight* tag NAME, pointing
                         # at HEAD (default) or OBJECT
git tag -a NAME [OBJECT] # create a new tag *object* NAME, pointing at
                         # HEAD (default) or OBJECT
```

これを argparse に翻訳すると次のようになります。 `--list` と `[-a] name [object]` の間の相互排除を無視していることに注意してください。これは argparse には複雑すぎるようです。 <!-- This translates to argparse as follows. Notice we ignore the mutual exclusion between -[HACK: for https://github.github.com/gfm/#html-comment]-list and [-a] name [object], which seems too complicated for argparse. -->

``` py
argsp = argsubparsers.add_parser(
    "tag",
    help="List and create tags")

argsp.add_argument("-a",
                    action="store_true",
                    dest="create_tag_object",
                    help="Whether to create a tag object")

argsp.add_argument("name",
                    nargs="?",
                    help="The new tag's name")

argsp.add_argument("object",
                    default="HEAD",
                    nargs="?",
                    help="The object the new tag will point to")
```

`cmd_tag` 関数は、 `name` が与えられているかどうかによって、振る舞い（一覧または作成）をディスパッチします。 <!-- The cmd_tag function will dispatch behavior (list or create) depending on whether or not name is provided. -->

``` py
def cmd_tag(args):
    repo = repo_find()

    if args.name:
        tag_create(args.name,
                   args.object,
                   type="object" if args.create_tag_object else "ref")
    else:
        refs = ref_list(repo)
        show_ref(repo, refs["tags"], with_hash=False)
```

### ブランチって何？
<!-- What’s a branch? -->

見て見ぬ振りをしてきたことに本気で対処する時です: ほとんどの Git ユーザーと同様に、 wyag は、ブランチとは何なのかをまだ全く理解していません。現在のところ、 wyag は、リポジトリを無秩序なオブジェクトの束として扱い、その一部をコミットとして扱っており、コミットはブランチの中にグループ化されているという事実や、 `HEAD` コミット（*すなわち*、**現在の**ブランチの**先頭の**コミット）が存在するという事実を全く表現していません。 <!-- It’s time to address the elephant in the room: like most Git users, wyag still doesn’t have any idea what a branch is. It currently treats a repository as a bunch of disorganized objects, some of them commits, and has no representation whatsoever of the fact that commits are grouped in branches, and that at every point in time there’s a commit that’s HEAD, ie, the head commit of the current branch. -->

では、ブランチとは何なのでしょう？　実はその答えは意外とシンプルですが、ただ驚くことになるかもしれません: **ブランチはコミットへの参照です**。ブランチはコミット名の一種だと言うこともできます。この点では、ブランチはタグと全く同じものです。タグは `.git/refs/tags` にある参照であり、ブランチは `.git/refs/heads` にある参照です。 <!-- Now, what’s a branch? The answer is actually surprisingly simple, but it may also end up being simply surprising: a branch is a reference to a commit. You could even say that a branch is a kind of a name for a commit. In this regard, a branch is exactly the same thing as a tag. Tags are refs that live in .git/refs/tags, branches are refs that live in .git/refs/heads. -->

勿論、ブランチとタグには違いがあります: <!-- There are, of course, differences between a branch and a tag: -->

1.  ブランチはコミットへの参照ですが、タグはあらゆるオブジェクトを参照できます。 <!-- Branches are references to a commit, tags can refer to any object; -->
1.  しかし、最も重要なことは、ブランチの参照はコミットごとに更新されるということです。あなたがコミットするたびに、 Git は実際にはこれを行っています: <!-- But most importantly, the branch ref is updated at each commit. This means that whenever you commit, Git actually does this: -->
    1.  現在のブランチの ID を親として使い、新しいコミットオブジェクトが作られます。 <!-- a new commit object is created, with the current branch’s ID as its parent; -->
    1.  コミットオブジェクトがハッシュされ、保存されます。 <!-- the commit object is hashed and stored; -->
    1.  ブランチの参照が新しいコミットのハッシュを参照するように更新されます。 <!-- the branch ref is updated to refer to the new commit’s hash. -->

以上です。 <!-- That’s all. -->

しかし、**現在の**ブランチはどうでしょうか？　実はもっと簡単です。 `refs` 階層の外側、 `.git/HEAD` にある参照ファイルで、**間接**参照（つまり、 `ref: path/to/other/ref` 形式のものであり、単純なハッシュではありません。）です。 <!-- But what about the current branch? It’s actually even easier. It’s a ref file outside of the refs hierarchy, in .git/HEAD, which is an indirect ref (that is, it is of the form ref: path/to/other/ref, and not a simple hash). -->

**デタッチされたHEAD** <!-- Detached HEAD -->

適当なコミットをチェックアウトすると、 git は「 HEAD がデタッチされた状態 (detached HEAD state) 」であることを警告します。これは、もはやどのブランチにも居ないという意味です。この場合、 `.git/HEAD` は**直接**参照です: 内容は SHA-1 です。 <!-- When you just checkout a random commit, git will warn you it’s in “detached HEAD state”. This means you’re not on any branch anymore. In this case, .git/HEAD is a direct reference: it contains a SHA-1. -->

### オブジェクトの参照: `object_find` 関数
<!-- Referring to objects: the object_find function -->

#### 名前の解決
<!-- Resolving names -->

4つの引数を取って、2つ目を変更せずに返し、他の3つは無視するという[馬鹿げた object_find 関数](#foo)を作った時のことを覚えていますか？　それをより便利なものに置き換える時が来ました。小さいけど便利な、実際の Git の名前解決アルゴリズムのサブセットを実装します。新しい `object_find()` は2つのステップで動作します: 1つ目は、名前が与えられると完全な sha-1 ハッシュを返すというものです。たとえば、 `HEAD` なら現在のブランチの先頭のコミットのハッシュを返したりします。より正確には、この名前解決関数はこのように動作します: <!-- Remember when we’ve created the stupid object_find function that would take four arguments, return the second unmodified and ignore the other three? It’s time to replace it by something more useful. We’re going to implement a small, but usable, subset of the actual Git name resolution algorithm. The new object_find() will work in two steps: first, given a name, it will return a complete sha-1 hash. For example, with HEAD, it will return the hash of the head commit of the current branch, etc. More precisely, this name resolution function will work like this: -->

- `name` が HEAD の場合、単に `.git/HEAD` を解決します。 <!-- If name is HEAD, it will just resolve .git/HEAD; -->
- `name` が完全なハッシュの場合、そのハッシュが変更せずに返されます <!-- If name is a full hash, this hash is returned unmodified. -->
- `name` が短いハッシュのように見える場合、完全なハッシュがその短いハッシュで始まるオブジェクトを集めます。 <!-- If name looks like a short hash, it will collect objects whose full hash begin with this short hash. -->
- 最後に、 name に一致するタグとブランチを解決します。 <!-- At last, it will resolve tags and branches matching name. -->

最後の2つのステップが値を*集める*方法に注意してください: 最初の2つは絶対参照なので、安全に結果を返すことができます。しかし、短いハッシュやブランチ名は曖昧なことがあるので、全ての可能なその名前の意味を列挙し、1つより多く見付かった場合はエラーを発生させたいです。 <!-- Notice how the last two steps collect values: the first two are absolute references, so we can safely return a result. But short hashes or branch names can be ambiguous, we want to enumerate all possible meanings of the name and raise an error if we’ve found more than 1. -->

**短いハッシュ** <!-- Short hashes -->

利便性のために、 Git では、名前の先頭部分でハッシュを参照できるようになっています。たとえば、 `5bd254aa973646fa16f66d702a5826ea14a3eb45` は `5bd254` として参照されることがあります。これを「短いハッシュ」と呼びます。 <!-- For convenience, Git allows to refer to hashes by a prefix of their name. For example, 5bd254aa973646fa16f66d702a5826ea14a3eb45 can be referred to as 5bd254. This is called a “short hash”. -->

``` py
def object_resolve(repo, name):
    """Resolve name to an object hash in repo.

This function is aware of:

 - the HEAD literal
 - short and long hashes
 - tags
 - branches
 - remote branches"""
    candidates = list()
    hashRE = re.compile(r"^[0-9A-Fa-f]{1,16}$")
    smallHashRE = re.compile(r"^[0-9A-Fa-f]{1,16}$")

    # Empty string?  Abort.
    if not name.strip():
        return None

    # Head is nonambiguous
    if name == "HEAD":
        return [ ref_resolve(repo, "HEAD") ]


    if hashRE.match(name):
        if len(name) == 40:
            # This is a complete hash
            return [ name.lower() ]
        elif len(name) >= 4:
            # This is a small hash 4 seems to be the minimal length
            # for git to consider something a short hash.
            # This limit is documented in man git-rev-parse
            name = name.lower()
            prefix = name[0:2]
            path = repo_dir(repo, "objects", prefix, mkdir=False)
            if path:
                rem = name[2:]
                for f in os.listdir(path):
                    if f.startswith(rem):
                        candidates.append(prefix + f)

    return candidates
```

2つ目のステップは、タイプの引数が与えられた場合、見付けたオブジェクトを、要求されたタイプのオブジェクトまで辿るというものです。些細なケースだけを扱えばよいので、非常にシンプルな反復プロセスです: <!-- The second step is to follow the object we found to an object of the required type, if a type argument was provided. Since we only need to handle trivial cases, this is a very simple iterative process: -->

- タグがあり、 fmt が他の何かの場合、そのタグを辿ります。 <!-- If we have a tag and fmt is anything else, we follow the tag. -->
- コミットがあり、 fmt がツリーの場合、そのコミットのツリーオブジェクトを返します。 <!-- If we have a commit and fmt is tree, we return this commit’s tree object -->
- その他全ての状況では、中断します。 <!-- In all other situations, we abort. -->

（タグ自体をタグ付けすることができるため、ステップ数が不定なので、このプロセスは反復的です。） <!-- (The process is iterative because it may take an undefined number of steps, since tags themselves can be tagged) -->

``` py
def object_find(repo, name, fmt=None, follow=True):
    sha = object_resolve(repo, name)

    if not sha:
        raise Exception("No such reference {0}.".format(name))

    if len(sha) > 1:
        raise Exception("Ambiguous reference {0}: Candidates are:\n - {1}.".format(name,  "\n - ".join(sha)))

    sha = sha[0]

    if not fmt:
        return sha

    while True:
        obj = object_read(repo, sha)

        if obj.fmt == fmt:
            return sha

        if not follow:
            return None

        # Follow tags
        if obj.fmt == b'tag':
            sha = obj.kvlm[b'object'].decode("ascii")
        elif obj.fmt == b'commit' and fmt == b'tree':
            sha = obj.kvlm[b'tree'].decode("ascii")
        else:
            return None
```

#### rev-parse コマンド
<!-- The rev-parse command -->

`git rev-parse` コマンドは多くのことを行いますが、そのユースケースの1つはリビジョン（コミット）の参照の解決です。 `object_find` の「追跡 (follow) 」機能をテストするために、このインターフェイスに省略可能な `wyag-type` 引数を追加します。 <!-- The git rev-parse commands does a lot, but one of its use cases is solving revision (commits) references. For the purpose of testing the “follow” feature of object_find, we’ll add an optional wyag-type arguments to its interface. -->

``` py
argsp = argsubparsers.add_parser(
    "rev-parse",
    help="Parse revision (or other objects )identifiers")

argsp.add_argument("--wyag-type",
                   metavar="type",
                   dest="type",
                   choices=["blob", "commit", "tag", "tree"],
                   default=None,
                   help="Specify the expected type")

argsp.add_argument("name",
                   help="The name to parse")
```

``` py
def cmd_rev_parse(args):
    if args.type:
        fmt = args.type.encode()

    repo = repo_find()

    print (object_find(repo, args.name, args.type, follow=True))
```

## ステージングエリアとインデックスファイル
<!-- The staging area and the index file -->

コミットの作成は、大量の（1つだけではなく、再帰的であることを覚えておいてください。正確にディレクトリごとに1つずつ必要です。）新しいツリーオブジェクトとルートツリーオブジェクトを指すコミットオブジェクトを作り、 `HEAD` をそのコミットを指すように更新するだけのようです。このようにすることもできますが、 Git はもっと複雑な道を選びました: インデックスファイルの形式で実装されたステージングエリアです。 <!-- Creating commits seems to be simply a matter of creating a bunch of new tree object (not just one, remember they’re recursive — you need exactly one per directory), a commit object pointing at the root tree object, and updating HEAD to point to that commit — and yes, we could do it like this, but Git chooses a much more complicated road: the staging area, implemented in the form of the index file. -->

Git でコミットするには、まず `git add` や `git rm` で変更を「ステージ」し、*それから*コミットするということはご存じの通りです: コミットのようなオブジェクトを使ってステージングエリアを表現するのが論理的に思えますが、 Git は全く異なった道を選び、インデックスファイルの形式の、全く異なったメカニズムを使っています。インデックスファイルは、 HEAD コミットからの変更の集合を表現しており、コミットするには、これを組み合わせて新しいコミットにしなければなりません。 <!-- You certainly know that to commit in Git, you first “stage” some changes, using git add and git rm, then commit. It would seem logical to use a commit-like object to represent the staging area, but Git goes a completely different way, and uses a completely different mechanism, in the form of the index file. The index file represents a set of changes from the HEAD commit, to commit, they must be combined to create a new commit. -->

### インデックスのパーズ
<!-- Parsing the index -->

インデックスファイルは、 Git リポジトリが保持しうるデータの中で最も複雑なものです。完全なドキュメントは Git ソースツリーの `Documentation/technical/commit-graph-format.txt` にあり、 [GitHub ミラー上で](https://github.com/git/git/blob/master/Documentation/technical/index-format.txt)読むことができます。インデックスは3つの部分から成ります: <!-- The index file is by far the most complicated piece of data a Git repository can hold. Its complete documentation can be found in Git source tree at Documentation/technical/commit-graph-format.txt, you can read it on the Github mirror. The index is made of three parts: -->

- 署名といくつかの基本的な情報（最も重要なのはそのインデックスが保持しているエントリーの個数です。）を持つ、古典的なヘッダー <!-- A classic header with a signature and a few basic info, most importantly the number of entries it holds; -->
- ソートされた、それぞれが変更を表現する、一連のエントリー <!-- A series of entries, sorted, each representing a change; -->
- 一連の省略可能な拡張機能（無視します。） <!-- A series of optional extensions, which we’ll ignore. -->

私達が表現する必要があるのはエントリーアイテムだけです。実際にはかなり多くのものを保持しています: <!-- The only thing we need to represent is an entry item. It actually holds quite a lot of stuff: -->

``` py
class GitIndexEntry(object):
    ctime = None
    """The last time a file's metadata changed.  This is a tuple (seconds, nanoseconds)"""

    mtime = None
    """The last time a file's data changed.  This is a tuple (seconds, nanoseconds)"""

    dev = None
    """The ID of device containing this file"""
    ino = None
    """The file's inode number"""
    mode_type = None
    """The object type, either b1000 (regular), b1010 (symlink), b1110 (gitlink). """
    mode_perms = None
    """The object permissions, an integer."""
    uid = None
    """User ID of owner"""
    gid = None
    """Group ID of ownner (according to stat 2.  Isn'th)"""
    size = None
    """Size of this object, in bytes"""
    obj = None
    """The object's hash as a hex string"""
    flag_assume_valid = None
    flag_extended = None
    flag_stage = None
    flag_name_length = None
    """Length of the name if < 0xFFF (yes, three Fs), -1 otherwise"""

    name = None
```

## 後書き
<!-- Final words -->

### License

This document is part of wyag &LT;https://wyag.thb.lt&GT;. Copyright (c) 2018 Thibault Polge &LT;thibault@thb.lt&GT;. All rights reserved

Wyag is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

Wyag is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with Wyag. If not, see http://www.gnu.org/licenses/.
