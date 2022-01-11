# SEO対策など

## google アナリティクスの設定
設定する前に梅木に相談すること。
1. グーグルアナリティクスに入り、左下の管理から、流れにしたがって追加
1. グローバルサイトタグをコピーしてbase.htmlのhead内に埋め込むか、タグマネージャーを使う
1. 必要に応じてコンバージョンの設定を行う

## googleサーチコンソールの設定
設定する前に梅木に相談すること。
1. googleサーチコンソールに入り、TXTレコードを取得
1. 利用しているネームサーバーに、そのTXTレコードを埋め込む
1. 多くの場合、AWSのRoute53だが、メールを併用しようと思うと各ドメインサイトのネームサーバーを使わなければならないこともある。（MXレコードが秘匿されているため）

## メタディスクリプション
### base.htmlのhead内
```html
<meta name="description" content="Webサイトやアプリなどを制作する京都大学の学生によるITベンチャー。低価格かつ業界随一の早さで提供し、エンジニア業界に革命を起こす。">
```

## キーワード設定
### base.htmlのhead内
```html
<meta name=”keywords” content=”ウェブサイト,アプリ,開発,制作”>
```

## ogp
```html
<meta property="og:title" content="株式会社DeMiA" />
<meta property="og:type" content="website" />
<meta property="og:description" content="京都大学の学生によるITベンチャーです。ウェブサイトやアプリケーションなどの開発を行っています。" />
<meta property="og:url" content="https://demia.co.jp" />
<meta property="og:image" content="https://demia.co.jp{% static 'main/img/icon_white_back.png' %}" />
<meta property="twitter:image" content="https://demia.co.jp{% static 'main/img/icon_white_back.png' %}" />
<meta property="fb:app_id" content="[id]" />
<meta name="twitter:card" content="summary">
<meta name="twitter:site" content="@DeMiA_Inc"
```

## ファビコン
```html
<link rel="shortcut icon" href="{% static 'main/img/demia_favicon.png' %}">
```

## canonical
```html
<link rel="canonical" href="https://demia.co.jp">
```
