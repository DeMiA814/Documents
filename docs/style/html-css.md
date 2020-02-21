# HTML/CSS スタイルガイド

## ソースファイル

### ファイル名

ファイル名には小文字英数字(`[0-9a-z]`)とアンダースコア(`_`)，ハイフン(`-`)を使うことができる．拡張子はファイルに応じて指定する．

## 共通スタイル

フォーマッタには Prettier を使用する．Prettier によって，ルールの多くは意識する必要がない．

*注*: Djangoテンプレートを用いている場合，HTMLファイルのフォーマット後に大きく崩れることがある:

```django
<!-- フォーマット前 -->
{% block sidebar %}{% endblock %}
{% block navbar %}{% endblock %}

<!-- フォーマット後 -->
{% block sidebar %}{% endblock %} {% block navbar
%}{% endblock %}
```

この場合，`navbar`のブロックの直前にコメントを挿入することでフォーマットを抑制する必要がある．

### 命名規則

HTML 要素には原則として小文字英数字とアンダースコア，ハイフンのみが使用できる．

```html
<!-- bad -->
<a href="/" class="linkToHome">Home</a>

<!-- good -->
<img src="image.png" alt="Image" class="header-image" />
```

### インデント

インデントはスペース 2 文字で行う．

### プロトコル

image や media，script などを指定するときは可能な限り HTTPS を使用する．

#### 理由

多くのブラウザで HTTPS なページから HTTP のコンテンツへのアクセスがブロックされるようになっている．

```html
<!-- bad: プロトコルの省略 -->
<script src="//ajax.googleapis.com/ajax/libs/jquery/3.4.1/jquery.min.js"></script>
<!-- bad： httpの利用 -->
<script src="http://ajax.googleapis.com/ajax/libs/jquery/3.4.1/jquery.min.js"></script>

<!-- good -->
<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.4.1/jquery.min.js"></script>
```

### 引用符

引用符にはダブルクォート(`"`)を使用する．

```html
<!-- bad -->
<a href="google.com">Google</a>

<!-- good -->
<a href="google.com">Google</a>
```

## HTML スタイル

### 読み込み

特別な理由がない限り，CSS は`head`要素内，JavaScript は`body`要素の末尾で読み込む．_注_: `head`要素内で JavaScript は`defer`属性を付けて読み込むこと．

#### 理由

CSS はページの見た目に関わるため，レンダリングされる前に読み込まれる`head`要素内で読み込むべきである．JavaScript はすぐには実行されないことが多いため，ページのレンダリング後に読み込まれるべきである．

例:

```html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    ...
    <link
      rel="stylesheet"
      href="https://stackpath.bootstrapcdn.com/bootstrap/4.4.1/css/bootstrap.min.css"
    />
    ...
  </head>
  <body>
    ...
    <script src="https://code.jquery.com/jquery-3.4.1.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/popper.js@1.16.0/dist/umd/popper.min.js"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.4.1/js/bootstrap.min.js"></script>
  </body>
</html>
```

### type 属性

stylesheet と script の type 属性は省略する．HTML5 ではデフォルトで解釈される．

```html
<!-- bad -->
<link rel="stylesheet" href="style.css" type="text/css" />
<script
  src="https://code.jquery.com/jquery-3.4.1.min.js"
  type="text/javascript"
></script>

<!-- good -->
<link rel="stylesheet" href="style.css" />
<script src="https://code.jquery.com/jquery-3.4.1.min.js"></script>
```

## CSS スタイル

### プロパティの順番

意味毎に並べる（要検討）．

### セクション

可能であれば，セクションのかたまり毎にコメントを書く．

例:

```css
/* Header */

.header {
}

/* Footer */

.footer {
}

/* Gallery */

.gallery {
}
```
