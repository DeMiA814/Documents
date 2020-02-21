# JavaScript スタイルガイド

<!-- ## 概要
本文書は[Google JavaScript Style Guide](https://google.github.io/styleguide/jsguide.htm)と[Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript)を参考にしている． -->

## ソースファイル

### ファイル名

ファイル名には小文字英数字(`[0-9a-z]`)とアンダースコア(`_`)，ハイフン(`-`)を使うことができる．ファイルの拡張子は`.js`でなければならない．

## スタイル

### 命名規則

- 変数名と関数名は`obj`や`someVariable`のように，最初の単語を除き，頭文字を大文字にして単語を結合する．
- クラス名は`Base`や`SomeClass`のように，すべての単語の頭文字を大文字にする．
- `Boolean`型を返す関数は`is`または`has`を接頭辞として付ける，または三人称単数形の動詞を用いる．
- 変数名は意味が明確になるようにする．略称が一般的なものは省略しても構わない．

### インデント

インデントは 2 文字のスペースで行う．

### 変数

- 変数の宣言には`const`を使用する．

  理由: 変数に再代入させないようにするため．

  ```js
  // bad
  var a = 1;
  var b = 2;

  // good
  const a = 1;
  const b = 2;
  ```

  `const`は変数への再代入を禁止するだけなので，プロパティを変更することはできる．

  ```js
  const a = {};
  a.b = "c";
  console.log(a); // Expected `{ b: 'c' }`
  ```

- 再代入が必要な変数は`let`を使って宣言する．

  理由: `let`はブロックスコープであるのに対し，`var`は関数スコープである．一般に変数の生存期間が短いほどバグのリスクは下がる．

  ```js
  // bad
  var count = 1;
  if (true) {
    count += 1;
  }

  // good
  let count = 1;
  if (true) {
    count += 1;
  }
  ```

### オブジェクト

- オブジェクトの作成にはリテラル構文を使用する．

  ```js
  // bad
  const item = new Object();

  // good
  const item = {};
  ```

- オブジェクトのメソッドの短縮形を使用する．

  ```js
  // bad
  const atom = {
    value: 1,

    addValue: function(value) {
      return atom.value + value;
    }
  };

  // good
  const atom = {
    value: 1,

    addValue(value) {
      return atom.value + value;
    }
  };
  ```

- 識別子として無効なものにのみクォートをつける．

  理由: 一般に読みやすくなると考えられている．

  ```js
  // bad
  const bad = {
    foo: 3,
    bar: 4,
    "data-blah": 5
  };

  // good
  const good = {
    foo: 3,
    bar: 4,
    "data-blah": 5
  };
  ```

### 配列

- 配列の生成にはリテラル構文を用いる．

  ```js
  // bad
  const items = new Array();

  // good
  const items = [];
  ```

- 要素の追加には`Array.prototype.push`を用いる．

  ```js
  const someStack = [];

  // bad
  someStack[someStack.length()] = "foo";

  // good
  someStack.push("foo");
  ```

### 関数パラメータへの再代入

関数パラメータに再代入してはならない．

理由: 呼び出し元の変数が変更されることがある．

### 比較演算子

`==`と`!=`の代わりに`===`と`!==`を用いる．

## フォーマット

### フォーマッタの使用

フォーマッタには[Prettier](https://prettier.io)を使用する．
