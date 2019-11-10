# Git での開発

## `.gitignore` の利用
`.gitignore` を設定することで，自動生成されるファイルや，エディタの設定ファイルのような環境固有のファイルなどのように，バージョン管理すべきでないファイルをGitで管理しないようにすることができる．

簡単な例:

```
# `#`から改行までコメント．
# すべてのサブディレクトリ下にある，名前が一致するファイルやディレクトリを無視する．
file

# 末尾に`/`をつけることでディレクトリであることを表す．
directory/

# ワイルドカードも使うことができる．拡張子が`.obj`のファイルを無視する．
*.obj
```

### `gibo`による`.gitignore`の自動生成
手動で`.gitignore`を作成するのは非常に手間である．したがって`.gitignore`を生成するスクリプトである，`gibo`の利用を推奨する．

#### インストール
**TODO**: Windows向けのインストール方法

- macOS

  ```shell
  $ brew install gibo
  ```

- Linux

  ```shell
  $ curl -L https://raw.github.com/simonwhitaker/gibo/master/gibo -so ~/.local/bin/gibo && chmod +x ~/.local/bin/gibo
  ```

#### 使い方
```shell
$ gibo update
$ gibo dump Python >> .gitignore
```

## Gitの使い方
**TODO**: 詳細を書く
