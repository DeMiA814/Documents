# Poetry

## 概要
パッケージを管理する上で`pip`でパッケージのリストを更新するには明示的に`pip freeze`しなければならないため，それを忘れてしまうことが頻繁に起こる．このことを回避するために`poetry`を導入する．

## インストール
以下のコマンドでインストールできる:

```shell
$ pip install --user poetry
```

インストール後パスを通す．`--user`オプションをつけた場合，`pip`はデフォルトで`~/.local/bin`にインストールする．

## 使い方
### 新規プロジェクトを作る
```shell
$ poetry new <project_name>
```

### 既存のプロジェクトを`poetry`で管理する
```shell
$ poetry init
```

### 環境を作る
```shell
$ poetry install
```

このコマンドは依存パッケージのインストールと仮想環境の作成を行う．

### 依存パッケージの追加
```shell
$ poetry add <package_name>
```

`-D`または`--dev`オプションをつけると，開発用のパッケージとして登録される．

### パッケージの削除
```shell
$ poetry remove <package_name>
```

### `poetry`の設定
```shell
$ poetry config <config name>
```

`poetry`はデフォルトでは仮想環境をソースツリー外に作る．ソースツリー内に仮想環境を作るには以下の設定をする．

```shell
$ poetry config settings.virtualenvs.in-project true
```
