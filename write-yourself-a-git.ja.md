# Write yourself a Git!

［訳註: このファイルは https://wyag.thb.lt の翻訳です。<time datetime="2020-11-21T15:26:49">2020年11月22日</time>に作成され、最後の変更は<time datetime="2020-11-24T16:17:17">2020年11月25日</time>に行われました。］

## 導入 <!-- Introduction -->

この記事は、 [Git バージョン管理システム](https://git-scm.com/)をボトムアップで、すなわち、最も基本的なレベルから始め、そこから前進することによって説明する試みです。この試みは簡単すぎるようには思えないし、何度も成功かどうか疑わしい結果を収めてきたものです。しかし、簡単な方法があります: Git の内部を理解するために必要なことは、 Git を最初から再実装することだけです。 <!-- This article is an attempt at explaining the Git version control system from the bottom up, that is, starting at the most fundamental level moving up from there. This does not sound too easy, and has been attempted multiple times with questionable success. But there’s an easy way: all it takes to understand Git internals is to reimplement Git from scratch. -->

実行しないでください。 <!-- No, don’t run. -->

これは冗談ではないし、まったく複雑なことでもありません: この記事を上から下まで読んでコードを書けば（あるいは、[このリポジトリ](https://github.com/thblt/write-yourself-a-git)をクローンしさえすれば。ただし、自分で書くべきです。本当に。）最後には `wyag` というプログラムが出来上がります。そのプログラムは、 Git の全ての基本機能 (`init`, `add`, `rm`, `status`, `commit`, `log`, ..) を `git` 自体との完全な互換性を持つように実装しています。実際、この記事の最後のコミットは `git` ではなく `wyag` で作られています。そして、その全てが503行の非常にシンプルな Python コードになります。 <!-- It’s not a joke, and it’s really not complicated: if you read this article top to bottom and write the code (or just clone the repository — but you should write the code yourself, really), you’ll end up with a program, called wyag, that will implement all the fundamental features of git: init, add, rm, status, commit, log… in a way that is perfectly compatible with git itself. The last commit of this article was actually created with wyag, not git. And all that in exactly 503 lines of very simple Python code. -->

「しかし、それには Git は複雑すぎるのではないか」？　私の考えでは、 Git が複雑であるというのは誤解です。 Git が大きなプログラムで、多機能であることは真実です。しかし、そのプログラムのコアは実際には非常にシンプルで、 Git プログラムの見掛け上の複雑さは、それがしばしばひどく反直感的である（そして、ブログ投稿 [Git is a burrito](https://byorgey.wordpress.com/2009/01/12/abstraction-intuition-and-the-monad-tutorial-fallacy/) はおそらく助けにならない）という事実にまず起因しています。しかし、 Git を最も混乱させているのは、そのコアモデルの極端なシンプルさ*と*パワーかもしれません。コアのシンプルさと強力なアプリケーションの組み合わせは、基本的な抽象（モナドとかどう？）の本質的なシンプルさから種々のアプリケーションを得るために必要な精神的飛躍のため、物事を掴みにくくすることがよくあります。 <!-- But isn’t Git too complex for that? That Git is complex is, in my opinion, a misconception. Git is a large program, with a lot of features, that’s true. But the core of that program is actually extremely simple, and its apparent complexity stems first from the fact it’s often deeply counterintuitive (and Git is a burrito blog posts probably don’t help). But maybe what makes Git the most confusing is the extreme simplicity and power of its core model. The combination of core simplicity and powerful applications often makes thing really hard to grasp, because of the mental jump required to derive the variety of applications from the essential simplicity of the fundamental abstraction (monads, anyone?) -->

Git を実装することで、その基本を全て赤裸々に曝け出させることができます。 <!-- Implementing Git will expose its fundamentals in all their naked glory. -->

**何を期待していますか？**　この記事は、とても単純化されたバージョンの Git コアコマンドを実装し、詳細に（何か不明な点があれば[報告してください]()！）説明します。コードをシンプルにかつ要点を外さないように保っているので、 `wyag` は本物の git コマンドラインほど強力ではありませんが、何が欠けているのかは明らかで、やってみたい人は誰でもそれを実装することができます。よく言われるように、“wyag をフル機能の git ライブラリと CLI にアップグレードするのは読者の演習として残されています。” <!-- What to expect? This article will implement and explain in great details (if something is not clear, please report it!) a very simplified version of Git core commands. I will keep the code simple and to the point, so wyag won’t come anywhere near the power of the real git command-line — but what’s missing will be obvious, and trivial to implement by anyone who wants to give it a try. “Upgrading wyag to a full-featured git library and CLI is left as an exercise to the reader”, as they say. -->

より正確には、私達は: <!-- More precisely, we’ll implement: -->

- `add` () [git man page](https://git-scm.com/docs/git-add)
- `cat-file` ([wyag source]()) [git man page](https://git-scm.com/docs/git-cat-file)
- `checkout` ([wyag source]()) [git man page](https://git-scm.com/docs/git-checkout)
- `commit` () [git man page](https://git-scm.com/docs/git-commit)
- `hash-object` ([wyag source]()) [git man page](https://git-scm.com/docs/git-hash-object)
- `init` ([wyag source]()) [git man page](https://git-scm.com/docs/git-init)
- `log` ([wyag source]()) [git man page](https://git-scm.com/docs/git-log)
- `ls-tree` () [git man page](https://git-scm.com/docs/git-ls-tree)
- `merge` () [git man page](https://git-scm.com/docs/git-merge)
- `rebase` () [git man page](https://git-scm.com/docs/git-rebase)
- `rev-parse` () [git man page](https://git-scm.com/docs/git-rev-parse)
- `rm` () [git man page](https://git-scm.com/docs/git-rm)
- `show-ref` () [git man page](https://git-scm.com/docs/git-show-ref)
- `tag` ([wyag source]()) [git man page](https://git-scm.com/docs/git-tag)

を実装します。 <!-- More precisely, we’ll implement: -->

この記事を理解するために多くのことを知っている必要はありません: 基本的な Git （明白です）、基本的な Python 、基本的なシェルだけです。 <!-- You’re not going to need to know much to follow this article: just some basic Git (obviously), some basic Python, some basic shell. -->

- まず、最も基本的な **git コマンド**（エキスパートレベルのものではないですが、 `init`, `add`, `rm`, `commit` あるいは `checkout` を使ったことが無ければ迷うことになるでしょう。）にある程度精通していることを想定しています。 <!-- First, I’m only going to assume some level of familiarity with the most basic git commands — nothing like an expert level, but if you’ve never used init, add, rm, commit or checkout, you will be lost. -->
- 言語的には、 wyag は **Python** で実装されます。繰り返しますが、あまり派手なものは使いませんし、 Python は擬似コードのように見えるので、簡単に理解することができます。（皮肉にも最も複雑なパートはコマンドライン引数をパーズするロジックです。それを理解する必要はまったくありません。）ですが、プログラミングは知っているが Python は一度も使ったことが無いという場合は、インターネットのどこかでクラッシュコースを見付けて Python に慣れておくことをお勧めします。 <!-- Language-wise, wyag will be implemented in Python. Again, I won’t use anything too fancy, and Python looks like pseudo-code anyways, so it will be easy to follow (ironically, the most complicated part will be the command-line arguments parsing logic, and you don’t really need to understand that). Yet, if you know programming but have never done any Python, I suggest you find a crash course somewhere in the internet just to get acquainted with the language. -->
- `wyag` と `git` はターミナルプログラムです。 Unix ターミナルの使い方は知っていると思います。繰り返しますが、あなたが l77t h4x0r である必要はありませんが、あなたのツールボックスには `cd`, `ls`, `rm`, `tree` やこれらの仲間達が含まれているべきです。 <!-- wyag and git are terminal programs. I assume you know your way inside a Unix terminal. Again, you don’t need to be a l77t h4x0r, but cd, ls, rm, tree and their friends should be in your toolbox. -->

---

# Warning

**Windows ユーザーへの注意** <!-- Note for Windows users -->

`wyag` は、 Python インタープリターがある任意の Unix-like システム上で動くはずですが、 Windows 上でどう振る舞うかについてはまったく分かりません。テストスイートは bash 互換シェルを要求しますが、これは WSL によって提供されると思います。 Windows ユーザーからのフィードバックがあれば有り難いです！ <!-- wyag should run on any Unix-like system with a Python interpreter, but I have absolutely no idea how it will behave on Windows. The test suite absolutely requires a bash-compatible shell, which I assume the WSL can provide. Feedback from Windows users would be appreciated! -->

---

## 入門 <!-- Getting started -->

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

私達は（git の `init`, `commit` などのような）サブコマンドを扱う必要があります。 argparse のスラングでは、これらは「サブパーザー」と呼ばれます。現時点で必要なのは、私達の CLI がその中のいくつかを使用するということと、全ての呼び出しが実際にはその中の一つ（単に `git` と呼ぶのではなく、 `git COMMAND` と呼びます。）を*要求する*ということを宣言することだけです。 <!-- We’ll need to handle subcommands (as in git: init, commit, etc.) In argparse slang, these are called “subparsers”. At this point we only need to declare that our CLI will use some, and that all invocation will actually require one — you don’t just call git, you call git COMMAND. -->

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

## リポジトリの作成: init <!-- Creating repositories: init -->

時系列順*でも*論理順*でも*最初の Git コマンドが `git init` であることは明らかなので、 `wyag init` を作るところから始めます。これを達成するには、まず、ごく基本的なリポジトリの抽象が必要です。 <!-- Obviously, the first Git command in chronological and logical order is git init, so we’ll begin by creating wyag init. To achieve this, we’re going to first need some very basic repository abstraction. -->

### Repository オブジェクト <!-- The Repository object -->

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

### init コマンド <!-- The init command -->

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

### repo_find() 関数 <!-- The repo_find() function -->

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

## オブジェクトの読み書き: hash-object と cat-file <!-- Reading and writing objects: hash-object and cat-file -->

### オブジェクトって何？ <!-- What are objects? -->

さて、リポジトリが出来たので、今度はその中に何かを入れるのが順当です。また、リポジトリは退屈でしたが、 Git 実装を書くというのは単に大量の `mkdir` を書けばよいというものではありません。**オブジェクト**の話をして、 `git hash-object` と `git cat-file` を実装しましょう。 <!-- Now that we have repositories, putting things inside them is in order. Also, repositories are boring, and writing a Git implementation shouldn’t be just a matter of writing a bunch of mkdir. Let’s talk about objects, and let’s implement git hash-object and git cat-file. -->

これら2つのコマンドを知らない人も居るかもしれません。現に、日常の git ツールボックスには含まれていないし、非常に低レベルなもの（ git 用語でいう「配管 (plumbing) 」）です。これらのコマンドがすることは非常にシンプルです: `hash-object` は既存のファイルを git オブジェクトに変換し、 `cat-file` は既存の git オブジェクトを標準出力に印字します。 <!-- Maybe you don’t know these two commands — they’re not exactly part of an everyday git toolbox, and they’re actually quite low-level (“plumbing”, in git parlance). What they do is actually very simple: hash-object converts an existing file into a git object, and cat-file prints an existing git object to the standard output. -->

では、**実のところ、 Git オブジェクトとは何なのでしょう？**　 Git は「内容アドレスファイルシステム」です 。つまり、通常のファイルシステムでは、ファイル名は恣意的であり、ファイルの内容と無関係であるのに対し、 Git が保存するファイル名は、その内容から数学的に導き出される、ということです。これには非常に重要な含意があります: 仮にテキストファイルが1バイト変わったら、その内部名も変わるということです。簡単に言えば、ファイルを*変更している*のではなく、別の場所に新しいファイルを作っているということです。オブジェクトとは正に git リポジトリ内のファイルのことであり、そのパスはその内容によって決定されます。 <!-- Now, what actually is a Git object? At its core, Git is a “content-addressed filesystem”. That means that unlike regular filesystems, where the name of a file is arbitrary and unrelated to that file’s contents, the names of files as stored by Git are mathematically derived from their contents. This has a very important implication: if a single byte of, say, a text file, changes, its internal name will change, too. To put it simply: you don’t modify a file, you create a new file in a different location. Objects are just that: files in the git repository, whose path is determined by their contents. -->

---

# Warning

**Git は（本物の）キーバリューストアではない** <!-- Git is not (really) a key-value store -->

あの素晴らしき [Pro Git](https://git-scm.com/book/id/v2/Git-Internals-Git-Objects) を含むいくつかのドキュメントが、 Git を「キーバリューストア」と呼んでいます。これは誤りではないですが、誤解を招くかもしれません。実際には、 Git よりも通常のファイルシステムの方がキーバリューストアにより近いです。 Git はデータからキーを計算するので、むしろ*バリューバリューストア*と呼ばれるべきものです。 <!-- Some documentation, including the excellent Pro Git, call Git a “key-value store”. This is not incorrect, but may be misleading. Regular filesystems are actually closer to a key-value store than Git is. Because it computes keys from data, Git should rather be called a value-value store. -->

---

Git はオブジェクトを使って非常に多くのものを保存します: まず第一に、バージョン管理している実際のファイル（たとえば、ソースコード）です。コミットもオブジェクトです。タグもです。いくつかの注目すべき例外（後で見ます！）を除いて、 Git では、ほとんど全てのものがオブジェクトとして保存されます。 <!-- Git uses objects to store quite a lot of things: first and foremost, the actual files it keeps in version control — source code, for example. Commit are objects, too, as well as tags. With a few notable exceptions (which we’ll see later!), almost everything, in Git, is stored as an object. -->

パスはその内容の [SHA-1 ハッシュ](https://en.wikipedia.org/wiki/Cryptographic_hash_function)を計算することで計算されます。より正確には、 Git は、そのハッシュを小文字の16進文字列としてレンダリングし、2つの部分（最初の2文字と残り）に分けます。最初の2文字はディレクトリ名として使い、残りはファイル名として使います。（これは、多くのファイルシステムで、1つのディレクトリに過剰に多くのファイルがあることが嫌われていて、クロールが低速になることがあるためです。 Git の方法では、256個の中間ディレクトリが作成できるので、ディレクトリあたりのファイル数は平均で256分の1になります。） <!-- The path is computed by calculating the SHA-1 hash of its contents. More precisely, Git renders the hash as a lowercase hexadecimal string, and splits it in two parts: the first two characters, and the rest. It uses the first part as a directory name, the rest as the file name (this is because most filesystems hate having too many files in a single directory and would slow down to a crawl. Git’s method creates 256 possible intermediate directories, hence dividing the average number of files per directory by 256) -->

---

# Note

**ハッシュ関数って何？** <!-- What is a hash function? -->

簡単に言えば、ハッシュ関数は一方向性のある数学関数の一種です: 値からハッシュを計算するのは簡単ですが、あるハッシュがどの値から生成されるかを計算する方法はありません。ハッシュ関数のごく単純な例は `strlen` 関数です。文字列の長さを計算するのは非常に簡単で、与えられた文字列の長さが変わることもありません（勿論、文字列自体が変わっていない場合！）が、その長さだけを与えられても、元の文字列を得ることはできません。*暗号学的*ハッシュ関数は、ある与えられたハッシュを生成する入力を計算することが実際には不可能なほどに難しいという性質を付け加えて、同じものをさらに複雑にしただけのものです。（ `strlen` では、 `strlen(i) == 12` となる入力 `i` を生成するには、単に12文字のランダムな文字を入力すればよいです。 SHA-1 のようなアルゴリズムでは、ずっと長く、実際には不可能なほど長い時間が掛かります。［原註: [SHA-1 で衝突が発見された](https://shattered.io/)ことを知っているかもしれません。実際には Git はもう SHA-1 を使っていません: SHA-1 ではなく、衝突することが知られている2つの PDF ファイル以外の全ての既知の入力に同じハッシュを適用する、[強化された変種](https://github.com/git/git/blob/26e47e261e969491ad4e3b6c298450c061749c9e/Documentation/technical/hash-function-transition.txt#L34-L36)を使っています。］ <!-- Simply put, a hash function is a kind of unidirectional mathematical function: it is easy to compute the hash of a value, but there’s no way to compute which value produced a hash. A very simple example of a hash function is the strlen function. It’s really easy to compute the length of a string, and the length of a given string will never change (unless the string itself changes, of course!) but it’s impossible to retrieve the original string, given only its length. Cryptographic hash functions are just a much more complex version of the same, with the added property that computing an input meant to produce a given hash is hard enough to be practically impossible. (With strlen, producing an input i with strlen(i) == 12, you just have to type twelve random characters. With algorithms such as SHA-1. it would take much, much longer — long enough to be practically impossible[footnote: You may know that collisions have been discovered in SHA-1. Git actually doesn’t use SHA-1 anymore: it uses a hardened variant which is not SHA, but which applies the same hash to every known input but the two PDF files known to collide.]. -->

---

オブジェクトストレージシステムの実装を始める前に、正確なストレージフォーマットを理解しなければなりません。オブジェクトは、そのタイプ（ `blob`, `commit`, `tag` または `tree` ）を指定するヘッダーで始まります。このヘッダーの後は、 ASCII スペース (0x20) 、 ASCII 数字でバイト単位のオブジェクトのサイズ、 null (0x00) （ナルバイト）、オブジェクトのコンテンツと続きます。 Wyag のリポジトリのあるコミットオブジェクトの最初の48バイトは、このようになっています: <!-- Before we start implementing the object storage system, we must understand their exact storage format. An object starts with a header that specifies its type: blob, commit, tag or tree. This header is followed by an ASCII space (0x20), then the size of the object in bytes as an ASCII number, then null (0x00) (the null byte), then the contents of the object. The first 48 bytes of a commit object in Wyag’s repo look like this: -->

```
00000000  63 6f 6d 6d 69 74 20 31  30 38 36 00 74 72 65 65  |commit 1086.tree|
00000010  20 32 39 66 66 31 36 63  39 63 31 34 65 32 36 35  | 29ff16c9c14e265|
00000020  32 62 32 32 66 38 62 37  38 62 62 30 38 61 35 61  |2b22f8b78bb08a5a|
```

最初の行には、タイプヘッダー、スペース (`0x20`) 、 ASCII でのサイズ (1086) 、およびナルセパレーター `0x00` があります。最初の行の最後の4バイトは、オブジェクトのコンテンツの始まりであり、「 tree 」という単語です。これについては、コミットについて話すときにさらに議論します。 <!-- In the first line, we see the type header, a space (0x20), the size in ASCII (1086) and the null separator 0x00. The last four bytes on the first line are the beginning of that object’s contents, the word “tree” — we’ll discuss that further when we’ll talk about commits. -->

オブジェクト（ヘッダーとコンテンツ）は `zlib` で圧縮されて保存されます。 <!-- The objects (headers and contents) are stored compressed with zlib. -->

### 総称的なオブジェクトのオブジェクト <!-- A generic object object -->

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

### オブジェクトの読み込み <!-- Reading objects -->

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

まだ `object_find` 関数は導入していません。実のところ、これはプレースホルダーであり、このようなものです: <!-- We haven’t introduced the object_find function yet. It’s actually a placeholder, and looks like this: -->

``` py
def object_find(repo, name, fmt=None, follow=True):
    return name
```

この奇妙で小さな関数がある理由は、 Git にはオブジェクトを参照する方法が*たくさん*あるからです: 完全なハッシュ、短いハッシュ、タグ……。 `object_find()` が私達の名前解決関数となります。[後で]()実装するというだけで、これは一時的なプレースホルダーです。つまり、本物を実装するまで、オブジェクトを参照する方法は完全なハッシュによる方法しか無いということです。 <!-- The reason for this strange small function is that Git has a lot of ways to refer to objects: full hash, short hash, tags… object_find() will be our name resolution function. We’ll only implement it later, so this is a temporary placeholder. This means that until we implement the real thing, the only way we can refer to an object will be by its full hash. -->

## 後書き <!-- Final words -->

### License

This document is part of wyag &LT;https://wyag.thb.lt&GT;. Copyright (c) 2018 Thibault Polge &LT;thibault@thb.lt&GT;. All rights reserved

Wyag is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

Wyag is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with Wyag. If not, see http://www.gnu.org/licenses/.
