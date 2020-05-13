# はじめに
このドキュメントは、djangoでリアルタイムチャットシステムを実装するための参考資料を集めたものです。\
他の用途でwebsocketを用いる場合は、実装の多くがドキュメントに書いてある方法と同一ですが、\
細部は異なるのでご注意ください。

著者の実行環境は以下。
* ubuntu 18.04
* python 3.7.3
* django 3.0.5
* channels 2.3.1

# 1. websocketについて
初心者向け:\
https://qiita.com/south37/items/6f92d4268fe676347160 \
詳しい:\
https://developer.mozilla.org/ja/docs/Web/API/WebSockets_API

# 2. 必要な設定など
* django ver3.x(2.xでも可能)
* daphne(またはuvicorn)等のASGIサーバー
* redis(またはMemcached)等のキャッシュサーバー
* channels(djangoでwebsocketを扱うためのモジュール)

```
sudo apt update
sudo apt install redis-server 
pip install django channels channels-redis daphne asgiref
```

# 3. サンプルコード
channelsのチュートリアルを参照するのがよいです。\
チュートリアルではdockerでredisを実行していますが、dockerは必要ではありません。\
`redis-server`でredisサーバーを起動します。起動中は他のコマンドを実行できなくなるので、
末尾に&をつけてバックグラウンドで実行するなり、tmux使うなり、ターミナルを複数開くなりして対応してください。\
また、django3.xでasgiを使う場合は`manage.py runserver`は用いないでください。代わりに、
`daphne myproject.asgi:application`で実行します。\
サンプルコードを含むチュートリアルは、以下です。

**※注意：サンプルコードは完全に動作するコードを記述したものではありません。
実装したいアプリケーションの仕様に合わせて適宜書き換えてください。**\
**※完全に動作する実装例は京大ノートのリポジトリを参照してください。**\
https://github.com/AkaDeMiA13/DeMiA-Lecture_app/ \
**実装上重要なファイルは、
`myproject/settings.py routing.py asgi.py consumer.py`, 
`Lecture_app/viws.py`, 
`Lecture_app/templates/Lecture_app/mobile/chat.html`
です。**

* django側の設定(下のHTML側のコードも参照してください。)\
https://channels.readthedocs.io/en/latest/installation.html \
https://channels.readthedocs.io/en/latest/tutorial/index.html \
チュートリアルは初めてchannelsを触る人には少し読みにくいので、適宜以下の資料を参照してください。\
https://qiita.com/massa142/items/cbd508efe0c45b618b34 \
https://qiita.com/masa0209/items/24e303f4e06f02c55e99 

* HTML側のコード

```
<html>
    <head>
        <meta charset="utf-8">
    </head>
    <body>
    <div id="chat-log">
        <!-- チャットのログが出力される -->
    </div>
    <div class="sendarea" id="sendarea">
        <!-- メッセージの送信 -->
        <input type="text" id="chat-message-input">
        <input type="button" value="Send" id="chat-message-submit">
    </div>

    <script>
        let roomName = 'room_id'; //room_idは任意 

        // WebSocketのコネクションを確立
        // 参考: https://developer.mozilla.org/ja/docs/Web/API/WebSocket
        let chatSocket = new WebSocket(
            `ws://${window.location.host}/ws/chat/${roomName}/`
        );

        // サーバーからメッセージを受信した時の処理
        chatSocket.onmessage = function (e) {
            let data = JSON.parse(e.data);
            // dataの構造はサーバー側のレスポンスに依存します。
            let message = data["message"];
            let user = data["user"];

            // UIに表示
            let log_box = document.querySelector("#chat-log");

            let element = document.createElement('div');
            element.innerHTML = user + ':' + message;
            log_box.appendChild(element);
        };

        // 送信ボタンをクリックしたらwebsocketでサーバーにメッセージを送るようにする。
        document.querySelector("#chat-message-submit").onclick = function (e) {
            let messageInputDom = document.querySelector("#chat-message-input");
            let message = messageInputDom.value;
            // サーバーにメッセージを送信
            chatSocket.send(JSON.stringify({
                message,
            }));
            messageInputDom.value = "";
        };

    </script>
    </body>
</html>
```
