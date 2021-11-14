# 開発環境で HTTPS

## 背景

WebRTC などの一部の機能のテストには HTTPS が必須であるが，一般的に開発用サーバは HTTPS でのアクセスができない．

## 採用ツール

[stunnel](https://www.stunnel.org)（スタンネル）を使用し，開発サーバを TLS に対応させる．stunnel は汎用のツールであるため，使用プログラミング言語によらず TLS 化することができる．

証明書の発行には [mkcert](https://github.com/FiloSottile/mkcert) を使用する．

## インストール

### mkcert

- Ubuntu

  ```console
  $ sudo curl -L https://github.com/FiloSottile/mkcert/releases/download/v1.4.3/mkcert-v1.4.3-linux-amd64 -o /usr/local/bin/mkcert
  ```

- macOS

  ```console
  $ brew install mkcert
  ```

インストール後，ルート証明書を発行・登録する．

```console
$ mkcert -install
```

**TODO**: WSL で証明書を発行後，Windows 側にもその証明書を登録しなければならない．

### stunnel

- Ubuntu

  ```console
  $ sudo apt update
  $ sudo apt install stunnel4
  ```

- macOS

  ```
  $ brew install stunnel
  ```

## 設定

### TLS 証明書の発行

```console
$ mkcert -cert-file cert.pem -key-file key.pem localhost 127.0.0.1 ::1
```

### 設定ファイルの作成

stunnel の設定ファイルは BOM 付き UTF-8 でなければならないため，最初に BOM を書き込む:

```console
$ printf '\xef\xbb\xbf; BOM is before the semicolon\n' >stunnel.conf
```

次に，stunnel.conf に以下を追記する:

```ini
[https]
accept = 8001
connect = 8000
cert = cert.pemの絶対パス
key = key.pemの絶対パス
TIMEOUTclose = 0
```

stunnel を起動:

```console
$ stunnel stunnel.conf
```

これで，`127.0.0.1:8000` がサーバになっているときに，[https://127.0.0.1:8001](https://127.0.0.1:8001) にアクセスできるようになる．
