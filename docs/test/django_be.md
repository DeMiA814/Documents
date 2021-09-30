# Django バックエンド テストフロードキュメント

## 目次

- 単体テスト
  - 概要
  - 使用ツール
  - 手順
  - サンプルコード
- 結合テスト
  - 概要
  - 使用ツール
  - 手順
  - サンプルコード

## 単体テスト

### 概要

単体テストでのテストの単位は関数一つ一つ。django では主に models、forms で定義された関数のテストが想定される。

### 使用ツール

1. [django.test.TestCase](https://docs.djangoproject.com/en/3.2/topics/testing/overview/)

#### django.test.TestCase

django 標準のテストツール。単体テストと統合テストで使用する。

### サンプルコード

#### 階層構造

```
myapp/tests
├── __init__.py
├── forms
│   ├── __init__.py
│   ├── test_blog.py
│   └── test_user.py
├── models
│   ├── __init__.py
│   ├── test_blog.py
│   └── test_user.py
└── views
    ├── __init__.py
    ├── base.py
    ├── test_auth.py
    └── test_blog.py
```

django では標準で tests.py が生成されるがそれを tests パッケージに変更している。またファイル名は test\_〇〇 のような形式にする必要がある。基本的には各テストコードで最終的には`assert〇〇`系の TestCase のメソッドを使用して出力が期待通りかを確認する。

#### _models.py_

```python
from django.db import models
from django.contrib.auth.models import AbstractUser

# Create your models here.


class User(AbstractUser):
    is_private = models.BooleanField(default=False)


class Blog(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name="blog")
    title = models.CharField(max_length=30, default="")
    content = models.CharField(max_length=140, default="")

    @property
    def is_private(self) -> bool:
        return self.user.is_private

```

今回このようなモデルの定義で`is_private`のようなゲッターを定義したとする。

#### _models/test_blog.go_

```python
from faker import Faker
from myapp.models import Blog, User
from django.test import (
    TestCase,
)

fake = Faker('ja_JP')
# ちなみにFakerは偽データ生成用のpackage

class BlogModelTests(TestCase):
    def setUp(self):
      """
      ここでテストに必要な状態をセットアップ
      """
        self.private_user = User(
            username='秘密太郎',
            is_private=True,
        )
        self.public_user = User(
            username='公開太郎'
        )
        self.private_user.save()
        self.public_user.save()

    def test_private_blog(self):
        blog = Blog(
            user=self.private_user,
            title=fake.name(),
            content=fake.text(),
        )
        self.assertTrue(blog.is_private)

    def test_public_blog(self):
        blog = Blog(
            user=self.public_user,
            title=fake.name(),
            content=fake.text(),
        )
        self.assertFalse(blog.is_private)

```

`setUp()`は TestCase を継承したクラス内で定義すると`test_◯◯()`の前に実行される。テストごとに状態を持つ必要がある場合はここで設定することができる。

#### _forms.py_

```python
from django.db.models import fields
from .models import User
from django import forms
from django.core.exceptions import ValidationError
from django.contrib.auth.forms import UserCreationForm,AuthenticationForm


class CustomUserCreationForm(UserCreationForm):
    class Meta:
        model = User
        fields = ('username', 'email', 'password1', 'password2')

    def clean(self):
        cleaned_data = super().clean()
        name = cleaned_data.get('username')
        if len(name) < 2:
            raise ValidationError('名前は2文字以上にしてください')
        if not name[-2:] == '太郎':
            raise ValidationError('末尾は"太郎"で終わってください')
        return cleaned_data

class CustomLoginForm(AuthenticationForm):
    pass
```

例えば以上のような form を実装した時は

#### _forms/test_user.py_

```python
from myapp.forms import CustomUserCreationForm
from django.core.exceptions import ValidationError
from myapp.models import User
from django.test import TestCase

class UserFormTests(TestCase):
    def test_valid_username(self):
        form = CustomUserCreationForm({
            'username':'iii太郎',
            'email':'test@test.com',
            'password1':'thisistest',
            'password2':'thisistest',
        })
        self.assertTrue(form.is_valid())

    def test_invalid_username(self):
        form1 = CustomUserCreationForm({
            'username':'あ',
            'email':'test@test.com',
            'password1':'thisistest',
            'password2':'thisistest',
        })
        form2 = CustomUserCreationForm({
            'username':'あああ',
            'email':'test@test.com',
            'password1':'thisistest',
            'password2':'thisistest',
        })
        self.assertFalse(form1.is_valid())
        self.assertRaisesMessage(ValidationError,'名前は2文字以上にしてください',form1.clean)
        self.assertFalse(form2.is_valid())
        self.assertRaisesMessage(ValidationError,'末尾は"太郎"で終わってください',form2.clean)
```

というふうになる。

## 結合テスト

### 概要

Django では urls,views をテストすることで実現できる。

### 使用ツール

1. [django.test.TestCase](https://docs.djangoproject.com/en/3.2/topics/testing/overview/)
<!-- 2. [django.test.Client](https://docs.djangoproject.com/en/3.2/topics/testing/tools/#the-test-client) -->

#### django.test.TestCase

django 標準のテストツール。単体テストと統合テストで使用する。

<!-- #### django.test.Client

django 標準のテストツール。統合テストで使用する。
わざわざ web で実行しなくても views のテストが可能。 -->

### サンプルコード

以下のような認証周りの views が実装されているとする。

#### _views.py_

```python
from django.shortcuts import render
from django.contrib.auth.decorators import login_required
from django.utils.decorators import method_decorator
from django.contrib.auth.views import LoginView
from django.views.generic.edit import CreateView
from django.views.generic import ListView
from .forms import CustomUserCreationForm, CustomLoginForm
from django.urls import reverse_lazy
from .models import Blog

# Create your views here.
def index(request):
    return render(request, "myapp/index.html")


class Signup(CreateView):
    form_class = CustomUserCreationForm
    template_name = "myapp/signup.html"
    success_url = reverse_lazy("login")


class Login(LoginView):
    form_class = CustomLoginForm
    template_name = "myapp/login.html"

```

signup と login のテストを実装する。

#### _views/test_auth.py_

```python
from django.http.response import HttpResponse
from django.test import TestCase, Client
from myapp.models import User


class SignupTests(TestCase):
    def test_get(self):
        res = self.client.get("/signup")
        self.assertEqual(res.status_code, 200)
        self.assertTemplateUsed(res, "myapp/signup.html")

    def test_valid_post(self):
        params = {
            "username": "test太郎",
            "email": "test@test.com",
            "password1": "thisistest",
            "password2": "thisistest",
        }
        res = self.client.post("/signup", params)
        self.assertRedirects(
            res,
            "/login",
            status_code=302,
            target_status_code=200,
            fetch_redirect_response=True,
        )

    def test_invalid_post(self):
        params = {
            "username": "test",
            "email": "test@test.com",
            "password1": "thisistest",
            "password2": "thisistest",
        }
        res = self.client.post("/signup", params)
        self.assertEqual(res.status_code, 200)
        self.assertTemplateUsed(res, "myapp/signup.html")
        self.assertFormError(res, "form", None, errors=['末尾は"太郎"で終わってください'])


class LoginTests(TestCase):
    def setUp(self):
        self.existing_user = User(username="exisiting太郎")
        self.existing_user.set_password("thisistest")
        self.existing_user.save()

    def test_get(self):
        res = self.client.get("/login")
        self.assertEqual(res.status_code, 200)
        self.assertTemplateUsed(res, "myapp/login.html")

    def test_valid_post(self):
        params = {
            "username": self.existing_user.username,
            "password": "thisistest",
        }
        res = self.client.post("/login", params)
        self.assertRedirects(
            res,
            "/top",
            status_code=302,
            target_status_code=200,
            fetch_redirect_response=True,
        )

    def test_invalid_post(self):
        params = {
            "username": "not_existing",
            "password": "thisistest",
        }
        res = self.client.post("/login", params)
        self.assertEqual(res.status_code, 200)
        self.assertTemplateUsed(res, "myapp/login.html")
```

基本的には GET と POST のテストをすることになる。POST 時のデータは辞書の形で post()メソッドの第二引数に入れる必要がある。最低でもステータスコードの`assertEqual()`はテストする。今回のケースではその他に使用されるテンプレート、リダイレクト、エラーを assert している。<br>
次にログインが必要な場合の統合テストについて考える。

#### _views.py_

```python
from django.contrib.auth.decorators import login_required
from django.utils.decorators import method_decorator
from django.views.generic import ListView
from .models import Blog

@method_decorator(login_required, name="dispatch") # ログインが必要
class BlogList(ListView):
    model = Blog
    context_object_name = "blogs"
    template_name = "myapp/top.html"

```

#### _views/test_blog.py_

```python
from django.test.testcases import TestCase
from myapp.models import Blog
from .base import TestsWithAuthMixin #次のサンプルコードで出てくる
from faker import Faker

fake = Faker("ja_JP")


class BlogViewTestsMixin(TestsWithAuthMixin):
    @classmethod
    def setup_test_data(cls):
        super().setup_test_data()
        for i in range(10):
            Blog.objects.create(user=cls.user, title=f"ブログ{i}", content=fake.text())

class BlogViewTests(TestCase, BlogViewTestsMixin):
    """
    ここがテストの本体の部分
    """
    @classmethod
    def setUpTestData(cls):
        cls.setup_test_data()

    def test_get(self):
        self.login()
        res = self.client.get("/top")
        self.assertEqual(res.status_code, 200)
        for i in range(10):
            self.assertContains(res, f"ブログ{i}")
```

#### _views/test_base.py_

```python
from myapp.models import User

class TestsWithAuthMixin:
    @classmethod
    def setup_test_data(cls):
        cls._username = "test太郎"
        cls._email = "test@test.com"
        cls._password = "thisistest"
        cls.user = User.objects.create_user(
            username=cls._username, email=cls._email, password=cls._password
        )

    def login(self):
        return self.client.login(username=self._username, password=self._password)

```

ここで大事なのは`TestsWithAuthMixin`。TestCase のメンバである client には login()というメソッドが存在しこのメソッドに存在するユーザーの情報を入力することでログイン状態を実現できる。今回はブログ一覧のみでログインで必要だが通常複数のページのテストでログイン状態を維持しなければならないことが想定される。今回のように`base.py`で Mixin を実装しておけば各 views テストケースで利用できる。
