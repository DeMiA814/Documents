## Linter、Formatterとは

まずそもそもLinter、Formatterとは何かということを説明します。

### Lint（リント）
Lintとは、書いたプログラムの記法がある一定の規則に則っているかをチェックしてくれるプログラムのことを指します。これはプログラムを実行したらエラーが出るような箇所だけではなく、「１行の長さが長すぎる」「使ってないimport文がある」など、実行はできるがパフォーマンスや可読性の観点から改善した方がいい箇所に対しても警告を表示します。このような警告に従うことで、より堅牢性・保守性に優れたプログラムになり、潜在的なバグの危険性を回避できます。

もともとはC言語のコードの解析を行うツールの名称なのですが、大々的に広まった結果、言語に関わらずコードの解析を行うプログラムをLintと呼ぶようになりました。解析すること自体をLintingと言ったり、動詞的に用いられることもあるワードです。

### Linter
Linterとは、上記のLintの警告やエラーをエディタ上に反映してくれるツールのことです。各言語に特化したものがそれぞれ存在し、言語のパッケージ管理ツールからインストールするものや、エディタの拡張機能としてインストールするものがあります。

警告を出す記法の基準やその厳しさなどはそれぞれLinterによって異なり、多くの場合は自分で設定することもできます。

### Formatter
Lintの解析結果に従って、標準の記法に沿うようにコードを自動的に整形してくれるツールです。デフォルトではFormatを実行すると整形を行ってくれる形になっていますが、エディタの設定やのちに説明するpre-commitというツールを導入すれば、ファイルの保存時やペースト時、gitへのcommit時に自動的にFormatを実行してもらうことも可能です。

## Linter、Formatterの導入

ここでは例としてPythonのLinter、FormatterのVSCodeへの導入方法について解説します。他の言語やエディターについてもパッケージ管理ツールなどの違いはありますが、概ね同じやり方で導入できますので参考にしてみてください。

まず、Linterをインストールする必要がありますが、これはPythonであれば基本的に普段使っているpipからインストールできます。例えばDeMiAで標準で使用しているflake8というLinterであれば
```
$ pip install flake8
```
で導入できます。Formatterについても同様です。
次に、VSCodeを開いた状態で、Windowsであれば「Ctrl+,」、macであれば「⌘ Cmd+,」を押すとVSCodeの設定を開けます。ここで、導入したLinterの名前で検索するとそれに関わる設定が出てきますので、python.linting.<Lintert名>Enabledという項目がTrueになっていればそのLinterでLintしてくれます。
また、これはデフォルトでTrueに設定されていますが、そもそもPythonでLintingを行うかを設定するpython.linting.enabledと、保存時に自動でLintingを行うpython.linting.lintOnSaveがTrueになっていることも確認してください。

Formatterについても概ね同様で、python.formatting.providerという項目で導入したFormatterを指定すれば設定は完了です。保存時に自動整形するeditor.formatOnSaveがTrueになっていることを確認してください。

## pre-commit

DeMiAでは、VSCodeによる自動整形とあわせてpre-commitというツールも使用しています。

このツールは、gitへのcommit時に指定したテストや整形などを行ってくれるというもので、このLinter、Formatterの文脈でしばしば使用されます。

使い方はこれも同様にコマンドラインから
```
$ pip install pre-commit
```
でインストールしたのち、gitの管理しているディレクトリに`.pre-commit-config.yaml`というファイルを作成して設定を記述します。例えばDeMiAでPythonプロジェクトのテンプレートとして使用している設定は以下のような感じです。

```
repos:
	- repo:  https://github.com/PyCQA/isort
	  rev:  5.10.1
	  hooks:
		- id:  isort
	- repo:  https://github.com/psf/black
	  rev:  21.11b1
	  hooks:
		- id:  black
	- repo:  https://github.com/PyCQA/flake8
	  rev:  4.0.1
	  hooks:
		- id:  flake8
	- repo:  https://github.com/pre-commit/mirrors-prettier
	  rev:  v2.5.0
	  hooks:
		- id:  prettier
	      exclude_types: [html]
	- repo:  https://github.com/pappasam/toml-sort
	  rev:  v0.19.0
	  hooks:
		- id:  toml-sort
		  args: [--all,  --in-place]
```

- repo
実行する処理のソースコードのリポジトリを指定します。
- rev
ブランチ名やリリースバージョンなどで実行するコードを指定します。
- hooks
実行する処理はhookという単位で提供されており、ここではどの処理を実行するかを指定します。

具体的にどのような処理をしているかはこの後紹介しますので今理解する必要はありません、今はひとまずpre-commitがこういったものだということを知っておいてください。
