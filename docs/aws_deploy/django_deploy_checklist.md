# Djangoプロジェクトのローンチ時のチェックリスト

## メンテナンスモード
- https://github.com/fabiocaccamo/django-maintenance-mode
を使うといい。主な設定は公式を参照。

- project直下にtemplates/admin/base.htmlを作り、下記のようにuser-toolsを上書きすることで、管理者画面で容易にスイッチできる。

### admin/base.html
```html
<div id="user-tools">
    {% block welcome-msg %}
    ...
    {% endblock %}
    {% block userlinks %}
        ...
        {% if user.has_usable_password %}
        ...
        {% endif %}
        <a href="{% url 'admin:logout' %}">{% translate 'Log out' %}</a>
        {% if user.is_superuser %}/
            {% if maintenance_mode %}
                <a href="{% url 'maintenance_mode_off' %}">メンテナンス終了</a>
            {% else %}
                <a href="{% url 'maintenance_mode_on' %}">メンテナンス開始</a>
            {% endif %}
        {% endif %}
    {% endblock %}
</div>
```

- 503.htmlは、[project_root]/templates/503.htmlに配置(templates直下が大事)

## サイトマップの設定
- Djangoの標準機能で実装できる
  - [django公式](https://docs.djangoproject.com/en/3.1/ref/contrib/sitemaps/)
  - [sitemap公式](https://www.sitemaps.org/ja/protocol.html)
- 本番ではデフォルトで、example.com が入っているので、adminからドメインを変更する。

## robots.txtの設定
- indexされたくないものをDisallowに追加する。
  - adminはデフォルトでnoindexになっている
  - wagtailの管理者はnoindexになっていないので注意
  - そのほかチャット部分などは、noindex

## loggerの設定（エラー時のメールの設定）
### settings/production.py
```python
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "standard": {
            "format": "[%(asctime)s] %(levelname)s [%(name)s:%(lineno)s] %(message)s",
            "datefmt": "%d/%b/%Y %H:%M:%S",
        },
    },
    "filters": {
        "require_debug_false": {
            "()": "django.utils.log.RequireDebugFalse",
        },
    },
    "handlers": {
        "null": {
            "level": "DEBUG",
            "class": "logging.NullHandler",
        },
        "file": {
            "level": "INFO",
            "class": "logging.handlers.RotatingFileHandler",
            "filename": "/var/log/[project_name]/debug.log",  # パスは環境に合わせて、ファイルは作る
            "maxBytes": 1024 * 1024 * 512,
            "backupCount": 10,
            "formatter": "standard",
        },
        "console": {"level": "INFO", "class": "logging.StreamHandler", "formatter": "standard"},
        "mail_admins": {
            "level": "ERROR",
            "class": "django.utils.log.AdminEmailHandler",
            "include_html": True,
            "filters": ["require_debug_false"],
        },
    },
    "loggers": {
        "django.security.DisallowedHost": {"handlers": ["null"], "propagate": False},
        "django": {"handlers": ["file", "console", "mail_admins"], "level": "DEBUG", "propagate": True,},  # NOQA: E231
        "main": {"handlers": ["file", "console", "mail_admins"], "level": "DEBUG", "propagate": True,},  # NOQA: E231
    },
}
```
これに加えて、[ADMINS](https://docs.djangoproject.com/en/3.1/ref/settings/#admins)を設定する必要がある

※テスト実行時には、メールは飛ばないことが確認されている（2022年1月現在）

※グーグルアカウントのメールでは、メールが飛ばないことが確認されている（2022年1月現在）

## djangoのセキュリティチェック

1. [django公式のデプロイチェックリスト](https://docs.djangoproject.com/en/3.1/howto/deployment/checklist/)
2. githubにapi-keyなどが上がっていないか確認する

## エラーページの作成
- 400.html
- 403.html
- 404.html
- 500.html
- 503.html

すべてプロジェクト直下のtemplates直下に配置する

## サイト名
管理者画面からデフォルトで存在するサイトテーブルの値をexample.comから使用する独自ドメインに変更する。
