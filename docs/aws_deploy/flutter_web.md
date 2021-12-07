# Flutter for web のデプロイ

## 目次

- Flutter for web のビルドとアップロード
- EC2 環境の設定

## Flutter for web のビルド

プロジェクトのビルド

```console
$ flutter build web
```

<プロジェクト>/build/ 以下に web ディレクトリが生成される。
このディレクトリをデプロイする。

```console
$ sftp -i <秘密鍵> ubuntu@<IPアドレス>
```

対話シェルが開始されるのでリモートサーバーに送信したいディレクトリを選択する。

```console
sftp> put -r <プロジェクト>/build/web
```

これでサーバーのルートに先ほどビルドしたディレクトリが送信される。

## EC2 環境の設定

OS は ubuntu を選択した想定。

```console
$ sudo apt update
```

サーバーには Apach を使用。

```console
$ sudo apt install apach2
```

Apach の設定ファイルを編集。

```console
$ sudo editor /etc/apache2/sites-available/000-default.conf
```

```conf
<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port t>
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        ServerAdmin webmaster@localhost
        # DocumentRoot /var/www/html これを以下に変更
        DocumentRoot /var/www/web

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log

```

編集した設定ファイルに合わせてシンボリックリンクを作成。

```console
$ sudo ln -s ~/web /var/www/
```

この状態で

```console
$ sudo systemctl restart apache2.service
```

サーバーを置いている IP アドレスにアクセスすると Flutter アプリケーションを確認できる。
