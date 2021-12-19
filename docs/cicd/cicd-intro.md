# CI/CDツールの導入

ここでは、実際のプロジェクトにおいてのツールの導入に際して参考とできるよう、主要なCI/CDツールの導入を紹介していきます。

## GitHub Actions
普段からGitHubを利用しているだけあって、導入は非常に簡単です。
ここでは、この後使用するツールとの相性と、既にテストコードが用意されていることを考慮して、例として[インターンのチャットアプリのDocker導入済みブランチ](https://github.com/DeMiA814/intern/tree/docker-intro)を用います。ただし、GitHub Actionsの設定はリポジトリで単位で行う必要があるため、上記のリポジトリを各人のアカウントにフォークしてから作業を開始するようにしてください。ブランチの詳細についてはDocker導入ドキュメントを参照してください。

まず、フォークしてきたリポジトリのdocker-introブランチから新しくブランチを切って、そのブランチをデフォルトブランチに設定してください。これにより、このブランチが本番環境であるように仮定します。やや煩雑ですが、実際のプロジェクトでは普通元からデフォルトブランチが本番環境になっているかと思いますので、この操作は不要です。

次に、リポジトリのActionsタブを選択し、作成するWorkflowのテンプレートを選択します。Actionsでは１つのブランチに対していくつものトリガーや対象となるブランチを設定することができ、その１つ１つの処理をWorkflowと呼びます。Djangoのテンプレートが用意されているので、ここではそれを使いましょう。Djangoを選択するとyamlの編集画面に遷移します。テンプレートは以下のようになっています。

```
name: Django CI

on:
  push:
    branches: [ cicd-intro ]
  pull_request:
    branches: [ cicd-intro ]

jobs:
  build:

    runs-on: ubuntu-latest
    strategy:
      max-parallel: 4
      matrix:
        python-version: [3.7, 3.8, 3.9]

    steps:
    - uses: actions/checkout@v2
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v2
      with:
        python-version: ${{ matrix.python-version }}
    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: Run Tests
      run: |
        python manage.py test
```
上から順に、まずnameはWorkflowの表示名、onは処理のトリガー、jobsはその中身を指します。onでは例として作成したcicd-introブランチへのpushとpull requestが指定されており、他にもファイルの作成や削除、時間指定などさまざまなトリガーを設定できます。

jobsは一連の処理のリストであり、buildはjobのidです。runs-onではテストを行う環境を指定しており、Linux以外にもWindows、Macともに使用可能です。max-parallelは並行に実行するjobの最大個数を指定しており、デフォルトから帰ることは基本的にありません。matrixではこのあと使用することのできる変数を定義しています。

stepsでは一連の処理を記述しており、最初のusesではActionsのバージョンを、その後のusesではパブリックリポジトリであるsetup-pythonを環境に指定し、Pythonを導入しています。その後の処理は見ての通りテストに使うコマンドを指定しています。

このテンプレートのままテストを実行した場合どうなるのか確認してみましょう。**自分で作ったプロジェクトを使う場合は、本番環境のDBでテストしないようtestコマンドにsettingsオプションを付けて読み込む設定を切り替えるのを忘れないでください。** Start Commitからyamlをcommitします。このyamlはリポジトリ直下の.github/workflows/以下に保存されます。

再度Actionsタブを開くと、このcommitに対してテストが行われていることがわかると思います。ここで、自分のプロジェクトを使った場合は成功するかもしれませんが、docker-introブランチをそのまま用いている場合はPOSTGRES_USERという環境変数がないことでエラーが起きると思います。これは、Docker環境でDBにPostgresを使うにあたって.envファイルにその設定を書き、それをdocker-compose.ymlやsettings.pyから共通で呼び出す形にしていたことによるものです。これを回避するために、test用のsettingsを用意しましょう。こうすることにより他に設定しなければいけない箇所が出てきたときにも対応が楽になります。（実際、環境変数を解決しても、この後の実装ではホスト名をdbで名前解決できなくなるので、これを127.0.0.1に書き換える必要がある）

intern以下にsettingsディレクトリを作り、そこにsettings.pyを移します。同じ場所にunittest.pyを作成して以下を記述してください。

```
from  .settings import  *
import os
import environ

env = environ.Env()
env.read_env(os.path.join(BASE_DIR,  ".env"))
DATABASES  = {
	"default": {
		"ENGINE":  "django.db.backends.postgresql",
		"NAME":  env("POSTGRES_DB"),
		"USER":  env("POSTGRES_USER"),
		"PASSWORD":  env("POSTGRES_PASSWORD"),
		"HOST":  "127.0.0.1",
		"PORT":  5432,
	}
}
```
なお、これはテストにPostgresを使う場合の例ですので、SQLiteで構わないという場合は
```
DATABASES  =  {
		'default':  {
		'ENGINE':  'django.db.backends.sqlite3',
		'NAME':  os.path.join(BASE_DIR,  'db.sqlite3'),
	}
}
```
と設定してください。
settings.pyのBASE_DIRを`BASE_DIR  =  Path(__file__).resolve(strict=True).parent.parent.parent`に、wsgi.pyとmanage.pyの読み込む設定ファイルを`os.environ.setdefault('DJANGO_SETTINGS_MODULE',  'intern.settings.settings')`に変更すればテストを実行できる形になりました。

つぎに、Actionsで作成したyamlを修正します。
このyamlファイルにはdocker-composeと同じ文法でコンテナを用意できるので、これを利用してPostgresコンテナを作成します。
strategyとstepsの間に以下を記述します。
（ちなみに、Dockerコマンドも通常通り使うことができるのですが、docker-compose upはなぜかエラーで動きませんでした）
```
services:
	postgres:
		image:  postgres:latest
		env:
			POSTGRES_USER:  postgres
			POSTGRES_PASSWORD:  postgres
			POSTGRES_DB:  postgres
		ports:
			- 5432:5432
		options:  --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
```
また、テストのコマンドを作成した設定に合わせて修正します。
```
- name:  Run Tests
	run:  |
		python manage.py test --settings=intern.settings.unittest
	env:
		POSTGRES_USER:  postgres
		POSTGRES_PASSWORD:  postgres
		POSTGRES_DB:  postgres
```
さらに、現在標準で使っているblackのformattingに文法が準拠しているかのテストも行うようにしましょう。
```
- name:  Lint with black
	run:  |
		pip install black
		black --check .
```
以上でテストを実行できる状態になりましたので、これまでの変更をpushしてみると、Actionsでテストが正常に実行されることが確認できると思います。

次にデプロイについても自動化してみましょう。stepsは上から順に実行されエラーが出れば止まるので、テストの下にデプロイについて書けばテストを通ったコードのみデプロイされることになります。
デプロイ自体はsshしてからgit pullなどをすればいいのですが、そうなるとsshの鍵などをどのように管理すべきかという問題が生じてきます。そのままこのyamlに書き込むわけにはいきません。
このような場合には、GitHubの機能であるSecretsを使います。SecretsではGitHub内で使用できる環境変数を設定できるのですが、この環境変数は一度設定すると確認することはできず削除か上書きしかできなくなります。

まずは通常通りEC2でサーバーを立ち上げてsshし、リポジトリをcloneします。IaCを使えばここら辺も自動化できるのですが、ここでは割愛します。
次に、サーバーにDockerとDocker Composeをインストールします。コマンドは以下の通り。
```
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
    
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

sudo curl -L https://github.com/docker/compose/releases/download/v2.2.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

あとは、internディレクトリ以下に.envファイルを作成してPostgres関連の環境変数を定義、ここでは
```
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_INITDB_ARGS="--auth-host=md5"
```
として、同階層で`sudo docker-compose up --build -d`を実行すればサーバーの立ち上げが完了します。もし権限関係でエラーが起きた場合、コンテナ内外でユーザーが食い違っていることが原因なので、/etc/passwdと/etc/groupをマウントしたり、chmodかchownするなどしてください。

次に、GitHubに先ほど述べたSecretsを登録します。sshとpullに必要な情報は
- sshの秘密鍵
- EC2のユーザー名
- EC2のホスト

ですので、これらを登録します。GitHubのリポジトリのSettings>SecretsからNew Repository Secretsを選択し、これらを（例ではSECRET_KEY、EC2_USER、EC2_HOSTという名前で）登録します。
これでyaml内から環境変数の形でこれらの情報が使えるようになったので、以下のように新しくstepを一番後ろに作成します。
```
- name:  Deploy
  run:  |
	echo "$SECRET_KEY" > secret_key
	chmod 600 secret_key
	ssh -oStrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} -i secret_key "cd <プロジェクトのリポジトリ> \
	&& git pull origin <ブランチ名> \
	&& docker-compose restart"
  env:
	SECRET_KEY:  ${{ secrets.SECRET_KEY }}
	EC2_USER:  ${{ secrets.EC2_USER }}
	EC2_HOST:  ${{ secrets.EC2_HOST }}
```
ここまでできたらpushしてみてください、Actionsから自動でデプロイができたのが確認できたと思います。これ以降は、指定した本番環境ブランチにmergeかpushがあるたびにテストが走り、それが通れば自動でサーバーに反映されることになります。

