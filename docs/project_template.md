# プロジェクトテンプレート(Django用)

# [gitignore]
## giboインストール
giboで.gitignoreを自動作成できるので推奨

macOS
```
$ brew install gibo
```
Linux
```
$ curl -L https://raw.github.com/simonwhitaker/gibo/master/gibo -so ~/.local/bin/gibo && chmod +x ~/.local/bin/gibo
```
## .gitignore自動作成
```
$ gibo update
$ gibo dump Python >> .gitignore
```


# [gitattributes]
### 例 : .gitattributes
```.gitattributes
* text=auto eol=lf
```


# [editorconfig]
使用しているエディターでEditorConfigの拡張をインストール

### 例 : .editorconfig
```.editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.py]
indent_style = space
indent_size = 4

[*.html]
indent_style = space
indent_size = 2

[*.{css,scss}]
indent_style = space
indent_size = 2

[*.{js,ts}]
indent_style = space
indent_size = 2

[*.{yml,yaml}]
indent_style = space
indent_size = 2
```


# [pre-commit]
## pre-commitをインストール
pip install pre-commit

### 例 : .pre-commit-config.yaml
```yaml:.pre-commit-config.yaml
repos:
  - repo: https://github.com/PyCQA/isort
    rev: 5.6.4
    hooks:
      - id: isort
        exclude: migrations/
  - repo: https://github.com/psf/black
    rev: 20.8b1
    hooks:
      - id: black
        exclude: migrations/
  - repo: https://gitlab.com/PyCQA/flake8
    rev: 3.8.4
    hooks:
      - id: flake8
        exclude: migrations/
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: ""
    hooks:
      - id: prettier
        exclude_types: [html, json]
```

pre-commitでは、以下の4つのフォーマッターを使用している。
- isort : importのソート
- black : エラーの修正
- flake8 : コードフォーマッタ
- prettier : コードフォーマッタ

それぞれ設定ファイルを書く。

## flake8
### 例 : .flake8
```.flake8
[flake8]
max-line-length = 100
extend-ignore = E203
```

## isort, black
### インストール
pip install setuptools

### 例 : pyproject.toml
```pyproject.toml
[tool.black]
line-length = 100

[tool.isort]
include_trailing_comma = true
line_length = 100
multi_line_output = 3
known_django = "django"
sections = "FUTURE,STDLIB,DJANGO,THIRDPARTY,FIRSTPARTY,LOCALFOLDER"
```

## prettier
### 例 : .prettierrc.toml
```.prettierrc.toml
printWidth = 100
semi = true
tabWidth = 2
trailingComma = "es5"
```

### 例 : .prettierignore
```.prettierignore
*.html
*.min.css
*.min.js
fixtures/
```

## secret key作成
settings.pyのSECRET_KEYを削除

### scripts/gen_secret_key.py
```python:gen_secret_key.py
#!/usr/bin/env python

from django.core.management.utils import get_random_secret_key

secret_key = get_random_secret_key()
print("SECRET_KEY = '{}'".format(secret_key))
```



