# Django フロントエンド テストフロードキュメント

## 目次

- 結合テスト
  - 概要
  - 使用ツール
  - 手順
  - サンプルコード
- 負荷テスト
  - 概要

## 結合テスト

### 概要

テストやスクレイピングで使用される Selenium を用いたテストを行う。実際にユーザーが従う導線を自動化する。テスト中の様子はブラウザドライバーが立ち上がるので確認することができる。

### 使用ツール

1.[django.contrib.staticfiles.testing.StaticLiveServerTestCase](https://docs.djangoproject.com/en/3.2/topics/testing/tools/)

2.[Selenium](https://selenium-python.readthedocs.io/index.html)(1.に含まれている)

### 手順

1. 必要な情報を用意する

- テストするページの URL
- 想定されるユーザーの動きのシナリオ

2. 必要なディレクトリ、ファイルの作成

```
myapp/uitests
├── __init__.py
├── __pycache__(勝手にできる)
├── base.py
└── test_〇〇.py
```

3. テスト用に用意された`SeleniumTests`か`SeleniumTestsWithAuth`を継承したクラスの中で`test_〇〇`というメソッドを定義して用意したシナリオを記述する

### サンプルコード

base.py ではテストコードを書くときにベースになるクラスを定義している。

#### _uitest/base.py_

```python
from myapp.models import User
from django.contrib.staticfiles.testing import StaticLiveServerTestCase
from selenium.webdriver.chrome.webdriver import WebDriver
import chromedriver_binary
```

#### _uitests/base.py_

```python
class SeleniumTests(StaticLiveServerTestCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.selenium = WebDriver()
        cls.selenium.implicitly_wait(10)

    @classmethod
    def tearDownClass(cls):
        cls.selenium.close()
        cls.selenium.quit()
        super().tearDownClass()
```

`SeleniumTests`はログインしていなくても利用できる部分のテストで使用する。

#### _uitests/base.py_

```python
class SeleniumTestsWithAuth(SeleniumTests, TestsWithAuthMixin):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.setup_test_data()
```

`SeleniumTestsWithAuth`はログインなしではアクセスできない部分のテストで使用する。

#### _uitests/base.py_

```python
class TestsWithAuthMixin:
    """
    ログインに必要な情報やフォームの情報はプロジェクトによって変わると思うので適宜変更してください
    """

    @classmethod
    def setup_test_data(cls):
        cls._username = "test太郎"
        cls._email = "test@test.com"
        cls._password = "thisistest"
        cls.user = User.objects.create_user(
            username=cls._username, email=cls._email, password=cls._password
        )
        cls._username_field_name = "username"
        cls._password_field_name = "password"
        cls._login_url = "/login"

    def login(self):
        self.selenium.get("%s%s" % (self.live_server_url, self._login_url))
        username_input = self.selenium.find_element_by_name(self._username_field_name)
        username_input.send_keys(self._username)
        password_input = self.selenium.find_element_by_name(self._password_field_name)
        password_input.send_keys(self._password)
        self.selenium.find_element_by_xpath('//input[@type="submit"]').click()

```

実際のテスト部分。例えばログイン部分のテストをする場合

#### _uitests/test_login.py_

```python
import time
from myapp.uitests.base import SeleniumTests
from myapp.models import User

#テストしたい部分は非ログイン状態から始まるので`SeleniumTests`を継承する
class LoginTests(SeleniumTests):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls._username = "test太郎"
        cls._email = "test@test.com"
        cls._password = "thisistest"
        cls.user = User.objects.create_user(
            username=cls._username, email=cls._email, password=cls._password
        )

    def test_login(self):
        self.selenium.get("%s%s" % (self.live_server_url, "/login"))
        username_input = self.selenium.find_element_by_name("username")
        username_input.send_keys(self._username)
        password_input = self.selenium.find_element_by_name("password")
        password_input.send_keys(self._password)
        self.selenium.find_element_by_xpath('//input[@type="submit"]').click()
        time.sleep(10)
```

ブログ一覧ページでテストしたい場合

```python
import time
from myapp.models import Blog
from myapp.uitests.base import SeleniumTestsWithAuth
from faker import Faker

fake = Faker("ja_JP")

# ログインした状態から始めたいので`SeleniumTestsWithAuth`を継承している
class BlogViewTests(SeleniumTestsWithAuth):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
		# ブログ一覧ページでテストするときに必要なデータをセットアップしている。
        for i in range(10):
            Blog.objects.create(user=cls.user, title=f"ブログ{i}", content=fake.text())

    def test_blog_list(self):
        self.login()
        time.sleep(10)
		# 以下具体的な導線の記述

```

ユーザーの導線は selenium で記述していく必要がある。基本的には`find_〇〇()`系で要素を取得、`click()`でクリック、`send_keys()`で入力を再現していくことになる。この[記事](https://qiita.com/mochio/items/dc9935ee607895420186#)に selenium の操作メソッドがまとめられている。

```
(.venv) $ python managa.py test myapp/uitests/
```

か

```
(.venv) $ python managa.py test myapp.uitests
```

を実行することで定義した`test_〇〇()`が全て実行される。<br>
また個別で実行したいときは以下のようにコマンドを打てばよい。

```
(.venv) $ python managa.py test myapp.uitests.BlogViewTests
```

```
(.venv) $ python managa.py test myapp.uitests.BlogViewTests.test_blog_list
```

## 負荷テスト

### 概要

負荷テストでは大量のデータを使用する場合が想定できるが結合テストのセットアップ時にデータ生成を行う。`StaticLiveServerTestCase`を継承したクラス内で`fixtures`というメンバに json ファイル名を代入することでモックのデータを生成することができる。
