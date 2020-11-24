# Write yourself a Git!

［訳註: このファイルは https://wyag.thb.lt の翻訳です。<time datetime="2020-11-21T15:26:49">2020年11月22日</time>に作成され、<time datetime="2020-11-21">2020年11月22日</time>に変更されました。］

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

## 後書き <!-- Final words -->

### License

This document is part of wyag &LT;https://wyag.thb.lt&GT;. Copyright (c) 2018 Thibault Polge &LT;thibault@thb.lt&GT;. All rights reserved

Wyag is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

Wyag is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with Wyag. If not, see http://www.gnu.org/licenses/.
