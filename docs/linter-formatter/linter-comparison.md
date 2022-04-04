## Linterの比較

ここでは、例としてPythonのLinterについて、現在DeMiAで標準として取り入れられているものや、有名なものを取り上げて、比較を行なっていきます。

なお、以下で紹介するものを実際に全て導入する必要はありません。むしろ過剰にLinterを導入すればLinter同士で競合したり、警告が増えすぎて読みきれず本質的でなくなったりしますので注意してください。基本的にはDeMiAの開発ではDeMiA標準のものを、自分の開発では自分の好きなものを、他の場所で開発を行う時にはその場所の標準を使ってもらえれば構いません。

## [pycodestyle(PEP 8)](https://pypi.org/project/pycodestyle/)
Pythonには、[PEP (Python Enhancement Proposals)](https://www.python.org/dev/peps/)という、機能やアップデートの紹介、ポリシーなどを紹介する文書があるのですが、その8番目の文書は[Style Guide for Python Code](https://www.python.org/dev/peps/pep-0008/)、すなわちPythonのプログラムの記法について説明しています。

pycodestyleは、このPEPの8番目で紹介されている記法に則っているか、つまり公式の言うことに従っているかの判定を行います。もともとはLinter自体もPEP 8という名前で呼ばれていたのですが、[ややこしいからドキュメントと同じ名前をつけるな](https://github.com/PyCQA/pycodestyle/issues/466)という提議の末、pycodestyleに名前を変えたという経緯があります。

PEP 8に書かれている事項はPythonの文法を勉強している中で自然と身に付いているものが多く、公式だけあってクセも少ないので、使いやすいLinterであると言えるでしょう。

## [pyflakes](https://github.com/PyCQA/pyflakes#design-principles)

pyflakesはプログラムの記法のスタイルについては一切言及せず、実行するとエラーになるような箇所への指摘のみを行うLinterです。ほかのLinterにあるような特定の警告・エラーを無視するような設定ができなかったりと機能が限定されていますが、そのぶん高速に動作します。エラー検出のみに特化したミニマムなLinterということになります。

## [flake8](https://flake8.pycqa.org/en/latest/)

DeMiAで標準で使用しているLinterです。PEP 8のスタイルへの警告とpyflakesのエラー検出を組み合わせたLinterで、pyflakes単体ではできなかった設定変更なども可能になっており、使い勝手が向上しています。

## [Pylint](https://pylint.pycqa.org/en/latest/)
このLinterもPEP 8の記法に準拠しているのですが、pycodestyleに比べると圧倒的に基準が厳しく、普通に実行できるような箇所でも警告やエラー検出をたびたびしてきます。一方で、その厳しさをクリアできればPEP 8で定められているような標準の記法そのものを実現できるとも言えるでしょう。

ちなみにdef-initのソースコードをPylintに読ませてみると、2022/2/3の執筆時点では394件の問題が検出されました。ただしこれはそもそもコーディング時点でPylintを使用していないので、最初からPylintを意識しながら書けばもっと警告は減らせるかと思います。



