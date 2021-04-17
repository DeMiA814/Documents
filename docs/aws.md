# AWS デプロイ手順

## 目次

- [準備](#準備)
  - [`requirements.txt` の生成](#requirementstxt-の生成)
  - [EC2 インスタンスのチェックリスト](#EC2-インスタンスのチェックリスト)
- [デプロイ](#デプロイ)
  - [必要なパッケージのインストール](#必要なパッケージのインストール)
  - [リポジトリのクローン](#リポジトリのクローン)
  - [プロジェクトのセットアップ](#プロジェクトのセットアップ)
  - [Gunicorn の設定](#Gunicorn-の設定)
  - [Nginx の設定](#Nginx-の設定)
- [Tips](#Tips)
  - [SSL 化](#SSL-化)
  - [静的ファイルに対する gzip 圧縮の有効化](#静的ファイルに対する-gzip-圧縮の有効化)
  - [Basic 認証](#Basic-認証)
  - [2 回目以降のデプロイの自動化](#2-回目以降のデプロイの自動化)

## 準備

### `requirements.txt` の生成

開発環境で Poetry を使っていない場合は[チェックリスト](#EC2-インスタンスのチェックリスト)まで飛ばしてよい．

サーバでは Poetry を使えないため，Pip で依存パッケージをインストールできるように `requirements.txt` を生成しておく:

```console
$ poetry export --without-hashes > requirements.txt
```

**NOTE**: `--no-hashes` オプションはなくても構わないが，`requirements.txt` のサイズは大きくなる．

### EC2 インスタンスのチェックリスト

- セキュリティグループで 80 番と 443 番のポートでのアクセスを許可する．
- SSH ログインのための秘密鍵の権限を制限する:

  ```console
  $ chmod 400 "秘密鍵のパス"
  ```

## デプロイ

**NOTE**: Ubuntu 20.04 を前提とする．

### 必要なパッケージのインストール

```console
$ sudo apt update
$ sudo apt install -y python3-venv nginx
```

追加で以下のパッケージが必要になることがある:

| パッケージ名                     | 理由                                                   |
| :------------------------------- | :----------------------------------------------------- |
| `build-essential`, `python3-dev` | 一部のビルドが必要な Python パッケージをビルドするため |
| `libpq-dev`                      | Psycopg2 をビルドするのに必要                          |

### リポジトリのクローン

```console
$ git clone https://DeMiA814@github.com/DeMiA814/"プロジェクト名".git -b "ブランチ名"
$ cd "プロジェクト名"
```

### プロジェクトのセットアップ

```console
$ echo 'export DJANGO_SETTINGS_MODULE="プロジェクト名.settings.production"' >> .bashrc
$ exec "$SHELL"
$ python3 -m venv .venv
$ source .venv/bin/activate
$ pip install -r requirements.txt --no-cache
$ pip install -U gunicorn
$ python manage.py compilescss
$ python manage.py collectstatic --no-input -i '*.scss'
$ python manage.py migrate
$ deactivate
```

### Gunicorn の設定

```console
$ sudo editor /etc/systemd/system/gunicorn.service
$ sudo systemctl enable --now gunicorn
```

設定ファイルは以下のようになる:

```ini
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/"プロジェクト名"
Environment=DJANGO_SETTINGS_MODULE="プロジェクト名.settings.production"
ExecStart=/home/ubuntu/"プロジェクト名"/.venv/bin/gunicorn --access-logfile - --bind=0.0.0.0:8000 "プロジェクト名".wsgi:application

[Install]
WantedBy=multi-user.target
```

**NOTE**: 上記のプロジェクトが新レイアウトで作られた場合の設定である．旧レイアウトのプロジェクトでは，単に `Environment` の行を削除すればよい．

### Nginx の設定

```console
$ sudo ufw allow 'Nginx full'
$ sudo editor /etc/nginx/sites-available/"プロジェクト名"
$ sudo ln -s /etc/nginx/sites-available/"プロジェクト名" /etc/nginx/sites-enabled
$ sudo systemctl restart nginx
```

設定ファイルは以下のようになる:

```nginx
server {
  server_name "ドメイン名 or IP";
  listen 80;

  proxy_set_header Host $host;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header X-Real-IP $remote_addr;

  # favicon へのリクエストをログに残さない
  location = /favicon.ico {
    access_log off;
    log_not_found off;
  }
  location / {
    proxy_redirect off;
    proxy_pass http://localhost:8000;
  }
  # 静的ファイルをAWS S3にアップロードする場合は必要ない
  location /static {
    alias /home/ubuntu/<app_name>/staticfiles/;
    access_log off;
    add_header Cache-Control "public; max_age=2592000; must-revalidate";
  }
  # 必要に応じて (static同様AWS S3を使う場合は不要)
  location /media {
    alias /home/ubuntu/<app_name>/media/;
    access_log off;
    add_header Cache-Control "public; max_age=2592000; must-revalidate";
  }
}
```

## Tips

### SSL 化

**NOTE**: ドメインを取得している必要がある．

```console
$ sudo apt-get install -y certbot python3-certbot-nginx
$ sudo certbot --nginx
```

SSL 化完了後，`/etc/nginx/sites-available/"プロジェクト名"` を編集して HTTP/2 を有効化する (option):

```diff
- listen 443 ssl;
+ listen 443 ssl http2;
```

編集後，Nginx を再起動する:

```console
$ sudo systemctl restart nginx
```

### 静的ファイルに対する gzip 圧縮の有効化

`/etc/nginx/nginx.conf` の `gzip_types` の行のアンコメントし，Nginx を再起動する．

**NOTE**: AWS S3 等を用いている場合は効果がない．

### Basic 認証

#### Basic 認証に使われるファイルの生成

(例) `/etc/nginx/htpasswd` に保存する場合:

```console
$ echo "ユーザー名:$(openssl passwd -apr1 "パスワード")" | sudo tee /etc/nginx/htpasswd
$ chmod a-wx /etc/nginx/htpasswd
```

#### Nginx の設定

```nginx
server {
  ...
  auth_basic "クライアント側に伝えるメッセージ";
  auth_basic_user_file /etc/nginx/htpasswd;
  ...
}
```

### 2 回目以降のデプロイの自動化

次のスクリプトをホームディレクトリに置く:

```sh
#!/bin/sh
set -e
cd ~/"プロジェクト名"
git pull origin "ブランチ名"
source ./.venv/bin/activate
pip install -U -r requirements.txt --no-cache
python manage.py compilescss
python manage.py collectstatic --no-input -i '*.scss'
python manage.py migrate
sudo systemctl restart gunicorn
```

以降，プロジェクトの更新だけの単純なデプロイは以下のコマンドで完了する:

```console
$ sh "スクリプト名"
```

### 複数の人が SSH ログインできるようにする

**NOTE**: 複数人が同時に SSH ログインしないように注意する．

#### キーペアの作成

ログインしたい人がキーペアを作成する:

```console
$ ssh-keygen -t ed25519 -f "鍵のファイル名" # ローカル環境で
```

例えば，鍵のファイル名を `foo` とすると，`foo` という名前の秘密鍵と `foo.pub` という

#### 公開鍵の追加

すでにログインできる人が `.pub` ファイルを受け取り，ファイルの内容全体をコピーし，SSH 環境の `~/.ssh/authorized_keys` に内容を追記する:

```console
$ echo 'コピーした内容' >> ~/.ssh/authorized_keys
```
