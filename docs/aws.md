# AWS デプロイ手順

## Ubuntu 18.04 版

必要なパッケージをインストールする:

```console
$ sudo apt-get install -y build-essential python3 python3-venv python3-dev nginx
```

プロジェクトをクローンしてくる:

```console
$ git clone https://AkaDeMiA13@github.com/AkaDeMiA13/<app_name>.git
$ cd <app_name>
```

プロジェクトをセットアップする:

```console
$ python3 -m venv .venv
$ source .venv/bin/activate
$ pip install -r requirements.txt
$ pip install -U gunicorn
$ python manage.py migrate
$ python manage.py compilescss
$ python manage.py collectstatic --no-input -i '*.scss'
$ deactivate
```

ウェブサーバとプロキシを設定する:

```console
$ sudo ufw allow 'Nginx full'
$ sudo editor /etc/systemd/system/gunicorn.service
$ sudo systemctl enable --now gunicorn.service
$ sudo editor /etc/nginx/sites-available/<app_name>
$ sudo ln -s /etc/nginx/sites-available/<app_name> /etc/nginx/sites-enabled/
$ sudo systemctl restart nginx
```

- `gunicorn.service`:

  ```ini
  [Unit]
  Description=gunicorn daemon
  After=network.target

  [Service]
  User=ubuntu
  Group=www-data
  WorkingDirectory=/home/ubuntu/<app_name>
  ExecStart=/home/ubuntu/<app_name>/.venv/bin/gunicorn --access-logfile - --bind=0.0.0.0:8000 <app_name>.wsgi:application

  [Install]
  WantedBy=multi-user.target
  ```

- `/etc/nginx/sites-available/<app_name>`:

  ```nginx
  server {
    server_name <domain>;
    listen 80;

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;

    location / {
      proxy_redirect off;
      proxy_pass http://localhost:8000;
    }
    location /static {
      alias /home/ubuntu/<app_name>/staticfiles/;
    }
    # 必要に応じて
    location /media {
      alias /home/ubuntu/<app_name>/media/;
    }
    # 無くてもよい
    location /favicon.ico {
      alias /path/to/favicon.ico;
    }
  }
  ```

### 次回以降のデプロイを自動化するスクリプト

ホームディレクトリ直下に配置する:

```bash
#!/bin/sh
cd ~/<app_name>
git pull
source ./.venv/bin/activate
pip install -U -r requirements.txt --no-cache
python manage.py compilescss
python manage.py collectstatic --no-input -i '*.scss'
sudo systemctl restart gunicorn
```

### SSL 化

**注**: ドメインを取得している必要がある．

```console
$ sudo apt-get install -y certbot python3-certbot-nginx
$ sudo certbot --nginx
```

Certbotが起動して，対話形式でSSL化することができる．

### Basic 認証

Basic 認証に必要なファイルを作成する:

```console
$ echo "<ユーザー名>:$(openssl passwd -apr1 <パスワード>)" | sudo tee /path/to/basicauth/userfile
$ chmod a-wx /path/to/basicauth/userfile
```

Nginx の設定:

```nginx
server {
  ...
  auth_basic "<クライアントに伝えるメッセージ>";
  auth_basic_user_file /path/to/basicauth/userfile;
  ...
}
```
