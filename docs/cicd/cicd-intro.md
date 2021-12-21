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

jobsは一連の処理のリストであり、buildはjobのidです。runs-onではテストを行う環境を指定しており、Linux以外にもWindows、Macともに使用可能です。max-parallelは並行に実行するjobの最大個数を指定しており、デフォルトから変えることは基本的にありません。matrixではこのあと使用することのできる変数を定義しています。

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
同様にproduction.pyを作成して以下を記述し、
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
		"HOST":  "db",
		"PORT":  5432,
	}
}
```
settings.pyのDATABASEを以下に変更します。
```
DATABASES  =  {
		'default':  {
		'ENGINE':  'django.db.backends.sqlite3',
		'NAME':  os.path.join(BASE_DIR,  'db.sqlite3'),
	}
}
```
settings.pyのBASE_DIRを`BASE_DIR  =  Path(__file__).resolve(strict=True).parent.parent.parent`に、wsgi.pyとmanage.pyの読み込む設定ファイルを`os.environ.setdefault('DJANGO_SETTINGS_MODULE',  'intern.settings.production')`に変更すれば、本番環境の設定で実行できるようになりました。

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

## AWS CodePipeline, CodeBuild, CodeDeploy

EC2やECSを利用したデプロイを行う際に、デプロイ設定をAWS内で完結できるというメリットがあります。

例では引き続き先に使用したプロジェクトとEC2インスタンスをそのまま利用することとしますが、実際のデプロイではインスタンスの立ち上げから自動化することができます。

まず、前準備としてAWSの権限を設定します。
必要な権限は以下の通り
 - **CodeBuild、CodeDeploy、CodePipeline、aws:codestar-connections**
	サービスの使用に前提として必要となる権限
- **ssm:CreateAssociation on resource**
	 複数のサービスの関連づけに必要
- **iam:CreateRole**
	サービスに対して他のサービスの操作を行うための権限であるサービスロールを付与するのに必要です。管理者にこれらの権限を事前にもらってください。

まず、IAMのコンソールに移ってロールを作成します。
まずはEC2用のロールを作成するので、ユースケースのなかからEC2を選択し、ポリシーの選択ではAmazonEC2RoleforAWSCodeDeployにチェックをつけて次に進んでください。タグは空欄で結構で、ロール名にはわかりやすい名前を入力してください。
次にCodeDeployのロールを設定します。ユースケースはCodeDeploy、ポリシーはAWSCodeDeployRoleとAmazonS3FullAccessで後は同様です。

次に、CodeDeployのコンソールを開いてアプリケーションの作成を選択してください。
- **アプリケーション名**
適当なわかりやすい名前をつけておいてください
- **コンピューティングプラットフォーム**
デプロイする先を選択します。今回はEC2/オンプレミスを選択します。

作成したアプリケーションに対してデプロイグループを作成します。
- **デプロイグループ名**
わかりやすい名前。
- **サービルロール**
作成したCodeDeploy用のサービスロールを指定します。
- **デプロイタイプ**
Blue/Greenのほうがデプロイ時に中断がなくて安全ですが、ロードバランサが必要になります。今回LBは使用していないのでインプレースを指定します。
- **環境設定**
デプロイ先を指定します。今回はEC2ですので、EC2 Instanceを指定し、デプロイするインスタンスを指定します。
- **エージェント設定**
後述のエージェントを更新する頻度を設定できます。デフォルトで結構です。
- **デプロイ設定** 
デプロイ先のインスタンスが複数ある時に、一度にデプロイするのか段階的に行うのかの設定を行えます。今回はデプロイ先はひとつなのでAllAtOnceに指定します。
- **Load Balance**
ロードバランサについて設定できます。今回LBは使用しないのでロードバランシングを有効にするのチェックボックスを外します。

次に、CodePipelineで新規のパイプラインを作成します。CodePipelineのコンソールから作成ページに映ってください。
各設定は前から順に、
 - **パイプライン名**
	 適当なわかりやすい名前をつけておいてください。
- **サービスロール**
	新規サービスロールを選択しておけば自動的にロールが作成されます。
- **ソースプロバイダー**
	デプロイするソースコードの取得先を選択します。デフォルトではAWSのソースコード管理ツールであるCodeCommitが選択されていますが、GitHubが選択できるのでこちらに変更します。
　GitHubに接続するボタンができるので、そこから認証し、リポジトリとブランチを選択します。
　検出オプションは、ウェブフックにしておけば自動的にGitHub側に指定したブランチへのpushで作動するウェブフックを作成してくれるので、このままで結構です。
- **ビルドステージのプロバイダー**
	ビルドコンテンツを取得先を選択します。今回はCodeBuildを選択します。
- **ビルドプロジェクト**
	CodeBuildのビルドプロジェクトを選択します。ここではまだプロジェクトを作成していないので、プロジェクトの作成を押してください。別ウィンドウで作成画面が開きます。
		- **プロジェクト名**
			例によって適当なわかりやすい名前をつけておいてください。
		 - **環境**
			ビルド・テストの環境を設定します。OS以外は基本一番上を選択すれば大丈夫です。サービルロールも新しいロールの作成のままで大丈夫です。
		- **buildspec.yml**
			のちに作成しますが、ビルドの際の実行コマンドなどの設定はこのyamlに記述します。デフォルトではプロジェクトルートにあるbuildspec.ymlという名前のファイルを参照しますが、個々の設定で変更できます。今回はデフォルトです。
		- **ログ**
			S3、CloudWatchにログを出力できます。しなくてもログ自体は参照できるので、必要ない場合は余計な料金が発生しないようチェックを外しておきましょう。
- **環境変数**
	ビルドに渡す環境変数を設定できます。
- **ビルドタイプ**
	ビルドを複数に分割する際に使用します。今回は単一ビルドです。
- **デプロイプロバイダー**
	デプロイを実行するプロバイダーを選択します。今回はCodeDeployを選択し、アプリケーションとデプロイグループから先ほど作成したCodeDeployを設定します。

以上が設定できたらパイプラインを作成してCodePipelineの設定は完了です。

最後にEC2の設定です。まずはEC2のコンソールからアクション>セキュリティ>IAMロールを変更と進んで、先ほど作成したEC2用のロールをアタッチします。
次に、インスタンスにSSH接続し、エージェントのインストールを行います。コマンドはUbuntuサーバーであれば以下、
https://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/codedeploy-agent-operations-install-ubuntu.html
Amazon Linuxであれば以下に従ってください。
https://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/codedeploy-agent-operations-install-linux.html
デフォルトではrootユーザーを使用して実行する設定になっているのですが、これによりエラーが起きることがあります。その場合は、下記の手順に従ってユーザーを切り替えてください。
https://aws.amazon.com/jp/premiumsupport/knowledge-center/codedeploy-agent-non-root-profile/

次に、プロジェクトルートにCodeBuildとDeployの処理を記述するbuildspec.ymlとappspec.ymlを作成します。Gitに入りさえすればいいのでこの作業はサーバー内でやる必要はありません。まずbuildspec.ymlには以下のように記述してください。
```
version:  0.2
phases:
	install:
		commands:
			- python -m pip install --upgrade pip
			- pip install -r requirements.txt
			- pip install black
	build:
		commands:
			- black .
			- python manage.py test --settings=intern.settings.settings
artifacts:
	files:
		- "**/*"	
	base-directory:  $CODEBUILD_SRC_DIR
	name:  apply-artifacts
	discard-paths:  no
```
- version
CodeBuildのバージョンです。0.2が推奨されています。
- phases
処理を実行するシーケンスをそれぞれ記述します。install、pre-build、build、post-buildなどがあります。
- command
実行するコマンドを記述します。installでビルド環境に依存パッケージを導入し、buildでテストやその他必要な処理を実行するのが一般的です。
- artifacts
ビルドしたファイルの保存について設定します。今回の設定では自動的にS3に専用のバケットが用意されそこにアップロードされ、CodeDeployではそれを参照するようになっています。
filesはアップロードするファイルを指定することができ、上記のように書けば全ファイルをアップロードすることになります。
base-directoryはアップロードするディレクトリを指し、上記の環境変数はビルドしたところと同階層を指します。
nameはartifactの名前を指定し、APIからこの処理を実行するときや、既存のartifactに対してビルドプロジェクトを作成する時に使用されます。
discard-pathsはyesにするとビルド出力内の階層構造がなくなり、全てのファイルが出力のルートに並ぶことになります。

次に、appspec.ymlは以下のようになります。
```
version:  0.0
	os:  linux
	files:
		- source:  ./
		destination:  /home/ubuntu/intern-aws
	hooks:
		BeforeInstall:
			- location:  scripts/before.sh
			timeout:  300
		AfterInstall:
			- location:  scripts/after.sh
			timeout:  300
```
- version
CodeDeployのバージョンを指定します。2021月12月現在指定できるのは0.0のみです。
- os
デプロイ先のOSを指定します。linuxとwindowsが指定できます。
- files
デプロイするディレクトリをsourceで指定し、destinationでデプロイ先の設置場所を指定します。デプロイ先とファイルがかぶっているとエラーが起きますので、今回は先程のGitHub Actionsで作成したinternディレクトリとは別にintern-awsディレクトリを作成することにします。
- hooks
デプロイ時の処理を指定します。BeforeInstall、AfterInstall、AfterAllowTrafficなどがあり、処理自体はここではなく別のスクリプトファイルに書き込んでそれを呼び出す形になります。同階層にscriptsディレクトリを作ってその中にスクリプトを置いておくのが通例です。
![enter image description here](https://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/images/lifecycle-event-order-ecs.png)

最後に、呼び出すbefore.shとafter.shを作成して完了です。before.shは以下のようになります。
```
#!/bin/bash
cd /home/ubuntu/intern-aws
docker-compose down
```
after.shは以下のようになります。
```
#!/bin/bash
cd /home/ubuntu/intern-aws
sudo chown -R $USER postgres_data
docker-compose up --build -d
```
migrateやcollectstaticはコンテナ起動時に自動的に走るように設定してあるので今回はこれだけですが、コンテナを使用しない場合はそれらの処理はここで呼び出すことになります。

以上でセットアップは終了ですが、このままだとGitHub Actionsも同時に走ってしまうので、workflowのデプロイのステップをコメントアウトした上で、pushしてみましょう。CodePipelineのコンソールから実行状況が確認できます。
これ以降は、特に設定することなくpushのたびにテストが走りEC2が更新されることになります。やや煩雑に感じたかもしれませんが、実用上はIAMのロールなどは１つCI/CD用のものを作成してそれを使い回すことになるかと思いますので、実際にはもう少し手順を簡略化できるでしょう。
