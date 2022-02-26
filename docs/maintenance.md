# プロジェクト保守

本ドキュメントでは，プロジェクトおよびサーバーの保守について述べる．

## 頻度

プロジェクトの保守は 1 ヶ月に 1 回とする．これは，Django のマイナーアップデートの間隔が 1 ヶ月であることから決定した．

## 手順

### 依存パッケージ更新

各プロジェクトのパッケージ更新は次のように行う:

- Python

  ```console
  $ poetry update
  $ poetry export --without-hashes >requirements.txt
  ```

  メジャーアップデートがある場合は，`pyproject.toml` を直接編集してパッケージのバージョンを変更し，`poetry update` を実行する．

- Go

  ```console
  $ go get -u
  $ go mod tidy
  ```

### ローカルテスト

パッケージ更新後，各種テストを行う．特に，パッケージのメジャーアップデートがある場合，そのパッケージに破壊的変更が存在することがあるため，リリースノート等をよく読む必要がある．

### サーバー保守

サーバーのシステムパッケージを更新する:

```console
$ sudo apt update
$ sudo apt upgrade
```

不要なパッケージやキャッシュを削除する:

```console
$ sudo apt autoremove --purge
$ sudo apt clean
```

#### データベースのスナップショット

データベースに RDS 等のマネージドサービスを使わず，EC2 に直接インストールしている場合は，手動でスナップショットを作成する．以下，PostgreSQL の場合を説明する．

スナップショットを作成するには以下を実行する:

```console
$ pg_dump database名 > スナップショットファイル
```

スナップショットから復元する場合は次のコマンドを実行する:

```console
$ psql database名 < スナップショットファイル
```

Django を使用している場合は，`manage.py` の `dumpdata` サブコマンドでもスナップショットを作成できる:

```console
$ python manage.py dumpdata > スナップショットファイル
```

復元には `loaddata` サブコマンドを使用する:

```console
$ python manage.py loaddata スナップショットファイル
```
