## Formatterの比較

FormatterについてもLinterと同様に、Pythonの代表的なものとDeMiAで標準で使用しているものをいくつか紹介・比較していきます。

## [autopep8](https://github.com/hhatto/autopep8)

おそらくPythonのFormatterでもっともメジャーなもので、pycodestyleのLintingに準拠して整形を行うFormatterです。

Linterがスタンダードということもあり、こちらもスタンダードなPEP 8準拠のFormatterという位置付けで、設定も柔軟に可能です。

ちなみに作者は兵庫の方です。

## [yapf](https://github.com/google/yapf)

Googleの開発するFormatterで、Yet Another Python Formatterの略です。

設定できる項目が非常に多いのが特徴で、[clang-format](https://clang.llvm.org/docs/ClangFormat.html)というフォーマットに準拠しており、Lintingの結果のみならずスタイルに対して独自に言及することが特徴です。

## [black](https://github.com/psf/black)

比較的新しいながら最近勢いづいているFormatterです。一応PEP 8準拠ではあるのですが、PEP 8のみならず独自のスタイルによる整形を行う比較的厳しいFormatterです。

設定できる項目が１行の文字数くらいでほとんど自由度がないのですが、逆に何も設定せずともクオリティの高い整形を行ってくれることで人気を集めています。

DjangoのソースコードにもこのFormatterが使用されることになり、相性もよく、現在DeMiAの標準のFormatterになっています。

## [isort](https://github.com/PyCQA/isort)

厳密にはFormatterではないのですが、同じ文脈で使用されることが多いので合わせて紹介しておきます。

このパッケージはPythonのimport文を他ファイル、外部パッケージ、標準ライブラリなどでまとめた上でソートして並べてくれるものです。

Djangoなどは特にimport文が何十行にもなってしまうこともしばしばあり、ごちゃつきがちなので便利なツールです。こちらもDeMiAで標準で使用されています。
