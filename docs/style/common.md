# 共通ルール

以下の設定は Editorconfig で設定することができる．

## ソースファイルの形式

### 文字コード

ソースファイルの文字コードは原則として UTF-8(BOM なし)とする．

#### 理由

UTF-8 は最も広く使われている文字コードであり，ほとんどの環境で扱うことができる．

### 改行文字

改行文字は原則として`LF(0x0A)`とする．

#### 理由

Git 上では改行文字が`LF`で管理される．

#### Git for Windows 向けの設定

Windows では Git でコミットしたときに改行文字が`CRLF`に変換されることがある．`.gitattributes`に以下を追記:

```
* text=auto eol=lf
```

### ファイル末尾の改行

すべてのソースファイルは必ず改行によって終端されなければならない．

#### 理由

`git diff`を実行したときにファイル末尾の改行が存在しないときに"No newline at end of file"が表示されるため，これを抑制するために必要．これは[POSIX](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap03.html#tag_03_206)において，1 行が 0 文字以上の文字と改行文字で構成されると規定されていることに由来する．

### 行末の空白文字

必要な場合を除き，行末に空白文字を含んではいけない．

#### 理由

ほとんどのソースファイルでは行末の空白文字は意味を持たない．

#### 例外

- Markdown: 行末にスペースを 2 つ含めることで強制改行(HTML における`<br>`)を表す．`\`で代用可能．
