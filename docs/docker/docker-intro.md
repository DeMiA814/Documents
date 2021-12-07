# Dockerとは

Dockerとは、コンテナ仮想環境を開発・配置・実行するためのオープンソースプラットフォームです。より具体的には、各開発者のローカル環境に依存せず、独立した仮想環境をOS上に立ち上げ、その中で処理を実行することで誰でも全く同一の環境での開発を可能にし、また、作成したコンテナを他の開発者や環境、例えば各人の端末からサーバーへそのまま移行したり、コンテナごと削除したりすることで、環境構築やリセットの手間を省いてしまおうという代物です。

よく比較対象として上がるものとして、Virtual BoxやParallels、WSL2などが有名な仮想マシンと、Djangoの開発でいつも使っているような仮想環境があります。
これらとの違いとして、まず仮想マシンはホストとなるOSの上に仮想化を制御するソフトウェアがあり、その中にゲストのOSを丸ごと導入してその中で処理を実行します。これにより、ホストOSには完全に依存せず、自由度の高い操作を行うことができる一方で、使わない機能も含めてOSそのものを導入しなければならないうえ、ホストOSが裏で動いているため、非効率的であり多くの計算資源を必要とする、端的に言えば非常に重いという特徴があります。
一方、仮想環境はこれとは真逆で、処理にはホストOSのカーネルを用いるのでホストの環境に依存しています。ライブラリやモジュールをそれぞれの仮想環境にそれぞれ導入でき、またこれを切り替えられるというアプリケーションの階層の独立化であり、非常に軽量ですが、仮想環境でできることはホストOSでできることの内に限られています。

これらに対してDockerは、処理にはホストOSのカーネルを利用するのでゲストOSの導入が不要で軽量でありながら、間に入るDocker Engineが緩衝材となりこれらの処理を独立化することで、ホストの環境に依存せず、疑似的にOSを仮想化したかのような処理を行うことができます。すなわち、Docker Engineが異なるホストOSへの処理の対応はよしなにしてくれるので、アプリから見ればどこでも同じDocker Engineの処理を呼び出せばよい、ということになります。（ただし何でもできるというわけではない、詳しくは後述）

![https://knowledge.sakura.ad.jp/13265/](https://knowledge.sakura.ad.jp/images/2018/01/VM_Container-680x387.jpg)


# Dockerのメリットとデメリット
dockerは適切に使用すれば有用である一方で、デメリットも伴います。どのような場面で導入すべきかはプロジェクト初期の時点で検討されるべきです。

- メリット
	- 環境構築のコストが省ける
		上で述べたように、dockerを用いると全く同じ環境を他の人と共有できるため、誰かが最初に構築した環境をチームで共有すれば環境構築にかけるコストを低減できます。
		
	- アプリやフレームワークの拡張性
		設定を共有できるというのは自分たち独自の環境に限った話でなく、例えばPythonを使いたければPythonが導入済みの環境、PostgreSQLならPostgreSQL用の環境といったように、多くのアプリやフレームワークが公式にdockerをサポートしていたり、世界中のエンジニアの作った環境を、設定１行で導入できるため、これらを各自で導入する手間、および環境依存性のリスクがカットできます。

	- 設定を集約的に記述できる
		アプリケーションを開発していると環境固有のローカル設定を記述したり、サーバーの設定とプロキシの設定を別の場所に書いたりと、情報が散乱してデプロイしたその人でないと書き換えができない、といった状況になることが間々ありますが、Dockerを使えば自然と設定の記述が追いやすい構造になるので、可読性が上がり属人性が排除できます。ただし、これはDocker固有のメリットというわけでなく、むしろAnsibleやterraformなどInfrastructure as Codeが得意とする領域の話であり、Dockerにおいては副次的なメリットという位置付けです。

	- 現在のアプリケーション開発における流行
	現状、Dockerは国内外多くの企業の間のブームであり、いまだ加熱傾向にあります。そのため多くのサービスがDockerをサポートしており、この傾向は今後しばらくは続くと予想されます。例えば先に挙げた公式のサポートするDockerコンテナの共有や、テスト、デプロイ自動化などのサービスがDockerの使用を推奨していたり、そちらの方が導入が楽といったことがあり、こういった背景から既存のサービスやナレッジからの恩恵を受けやすいというメリットがあります。

- デメリット
	- 学習コスト
		Dockerを導入するためには、プロジェクトのチーム全体に適切な運用をするための素養が求められます。とはいってもさほど難しい技術でもないのですが、Dockerを使ったことのないメンバーがいた場合に却ってコストがかかってしまわないか、単純な開発コストだけでなくその後の運用も天秤にかけて、技術選定において判断を行う必要があります。

	- 不要な導入コストになる可能性
		Dockerが有用になるのは上記のメリットが当てはまる時、具体的には環境構築やアプリの構成が複雑な時や、他のアプリと連携する拡張性が求められる時、継続的に開発を進めていくためにCI/CDの仕組みを整えたりインフラの監視を行う必要があるときなどです。それ以外の時、例えばHTML/CSSとJavaScriptでホームページを作るのにDockerを使うのは不必要なコストを割くだけと言えます。これも先のデメリットに繋がりますが、開発におけるコストとその後の運用で得られるメリットを比較し、Dockerを導入する必要があるのかを検討しましょう。

	- 機能の制限や環境固有の問題
		コンテナはVMと異なり実際に別のOSを動かしているわけではなく、処理を行なっている実態はホストOSである以上、稀にそのホストOS固有の問題があったり、機能に制限があったりすることがあります。例えば最近だと、MySQLのコンテナがM1Macの環境で動かないなどの問題がありました。こういった問題に柔軟に対応できるのは、全ての設定を自ら行わなくてはならない従来のデプロイの長所とも言えます。

# Dockerの導入

WindowsとMacについては、比較的簡単に導入することができます。CLIから利用するDocker Engine単体をインストールする方法と、GUIのDocker Desktopを一緒にインストールする方法がありますが、Docker Desktopはそれなりに有用なので、特に理由がなければ後者をおすすめします。
https://docs.docker.com/engine/install/ からDocker Desktopをインストールすれば、本体も自動的についてきます。

一方Linuxの環境へのインストールはやや煩わしく、Docker DesktopはないのでCLIをインストールすることになります。プラットフォームによって具体的な方法は異なってくるため、上記リンク先から各環境に応じた指示に従って導入していく形になります。
さらに、Dockerで作成した複数のコンテナを一括で管理するためのDocker Composeというツールが事実上必須になってくるのですが、WindowsとMacOSのDockerにはこれが自動的に付属してくる一方、Linux環境ではGitHubから自力でインストールしなければならないという点にも注意してください。詳しくは下記リンクの導入に従ってください。
https://docs.docker.jp/compose/install.html

# Dockerの使い方

## Dockerの構成

Dockerの使い方を例を交えて示す上で、まずDockerが具体的にどのような仕組みで機能しているのかを説明します。Dockerの設定は全て**Dockerfile**という形式のファイルに記述され、この中にどのようなコンテナを作成するのかを定義していくことになります。

そして、コンテナを作成するためにまず、buildという処理を行ってこのDockerfileから**Dockerイメージ**というファイル群を生成します。このDockerイメージはDockerの環境の実行に必要なアプリなどをまとめたものであり、直接編集することのできないイミュータブルな存在です。そのため、Dockerfileを編集した際には再度buildを行いイメージを生成し直すことになります。（作った環境がリセットされるわけではありません）
また、このイメージは自分で一から構成する他に、インターネット上に公開されている他のイメージをベースに設定を追加していくということができます（むしろ実際の開発ではこちらの方が主）。詳しい内容については[Docker Hub](https://hub.docker.com/)から参照することができます。また、ここでは外部への公開だけでなく、チームでのイメージの共有などGitHubのような使い方もできます。

このイメージをrunすることによって作られるのが、実際に開発を進めていくことになる書き込み可能な仮想環境、**Dockerコンテナ**になります。実際に接続してみればわかりますが、多くのコンテナのベースとなっているのはDebianまたはAlpine Linuxの環境なので、基本的にはLinuxサーバーにSSHするのと同じような感覚で操作することができます。
ただし、読み書き可能とはいってもコンテナ内で直接編集した内容はそのコンテナを消去してしまえば消えてしまうため、コンテナ内に入って直接編集したりコマンドを打つことは基本的にはしません。ではどうするのかというと、コンテナには本家Linuxと同様に外部からストレージをマウントすることができ、この中の変更はコンテナが消えても保存されるので、これを利用して開発したアプリをコンテナにマウントしてから実行、という形を取ることになります。
![enter image description here](https://post-output.com/wp-content/uploads/2021/03/Container.jpg)

そして、このコンテナを複数生成した時に、それらの間にネットワークを作り、相互に利用できるようにするのが**Docker Compose**という機能です。　例えば下図の例では、インターネット上の1337番ポートへのリクエストをDocker Compose内のネットワークの80番ポートへフォワードしています。コンテナ内の80番ポートはnginxにつながっており、nginxはこれを8000番にあるgunicorn・Djangoにプロキシし、レスポンスを返すという仕組みになっています。
![enter image description here](https://storage.googleapis.com/zenn-user-upload/qwazyqc1ie3k2d4e7clgltckw7zx)
先にDockerのメリットとして言及した設定を集約的に記述できるというのはつまり、このようなネットワークを形成するにあたってホスト内でいくつもアプリを立ち上げる必要がなく、全ての設定を一箇所にまとめて記述し、管理することができるということです。

## Dockerの使用例

以下では、実際の例を交えながらDockerの使用方法について説明していきます。使用するプロジェクトは[インターン向けチャットアプリ](https://github.com/DeMiA814/intern)、まずは上のリポジトリを通常通りクローンしてください。

### Dockerファイルの作成

まずは、Dockerfileを記述してコンテナの設定をしていきます。拡張子なしで「Dockerfile」という名前のファイルを作成してください。これがデフォルトで読み込まれるDockerfileになります。場合によっては「Dockerfile.development」のようにサフィックスをつけて適用するDockerfileを区別することもできます。
作成する場所はどこでも良いですが、アプリのディレクトリ直下、あるいはDocker関連の設定を集約するディレクトリを別にしてその中に作成することをお勧めします。以下ではクローンしたinternディレクトリ直下に作成することにします。

DjangoのプロジェクトをDockerで運用するにあたって共通して役に立つ設定をまとめたものが以下になります。これを作成したDockerfileの中にコピペしてください。
```
FROM python:3.9-slim

ENV PYTHONUNBUFFERE=1 \
	PYTHONDONTWRITEBYCODE=1

WORKDIR /app

COPY  requirements.txt　.
RUN pip install --upgrade pip && pip install -r requirements.txt

COPY docker-entrypoint.sh /usr/local/bin
ENTRYPOINT ["docker-entrypoint.sh"]
```
上から順に説明していきます。
まずFROMはなんのイメージをベースにこのイメージを作成するかを指定します。ローカルにあるイメージから指定された名前を探し、該当がなければDocker Hub上のイメージを探しにいきます。バージョンやサイズなども指定できます。なんのイメージもベースにしない最小構成で作成する場合はFROM scratchと指定します。
ENV命令はコンテナ内に指定した環境変数を渡す命令で、ここでは標準出力やエラー出力のバッファをなくすPYTHONUNBUFFEREとパッケージ利用時にキャッシュファイルの生成を無くすPYTHONDONTWRITEBYCODEを指定しています。特に後者は、放っておくとコンテナのサイズが肥大化していくので、指定しておくことをお勧めします。
WORKDIRはrunやこのあとのCOPYなどの命令をどの回想で実行するかを指定しています。ホスト側ではなくコンテナ内部でのディレクトリであることに注意してください。ここでは後ほど定義する/appディレクトリを指定しています。
RUNは見た通り、起動時に実行する命令を記述しています。コンテナはそのものが仮想環境としての役割を果たしているので、直に依存パッケージをインストールします。
COPYはホストからコンテナ内へのファイルのコピー、ENTRYPOINTはコンテナ起動時に必ず実行される処理を指定します。/usr/local/binを起点に指定のファイルを探すので、そこにファイルをコピーしています。

次に、internディレクトリにこの起動時実行ファイルを作成します。docker-entrypoint.shを作成し、そこに処理を書き込んでください。
例えばDjangoでは、
```
#!/bin/bash

python manage.py migrate
# python manage.py sass static/scss/main.scss static/css/main.css
python manage.py collectstatic --noinput

exec  "$@"
```
とでも指定しておけば、これらの処理をサーバー起動時に自動的に実行してくれます。（今回SCSSはないですが）
ちなみに１行目に書いてあるのはShebangといい、使用するインタプリタを指定しています。これがないとエラーが起きる時があるので気をつけましょう。
最後の行のはコンテナの永続化であり、これがないとentrypointの処理が終了した時点でコンテナが終了してしまいます。

### DjangoのPostgres設定

次に、Djangoの設定を変更します。とはいっても、ここでは例示のためにPostgreSQLを用いるために設定を記述するだけで、Dockerを導入するにあたって特別Djangoに設定しなければならない事柄はありません。
ただし、PostgreSQLのパスワードなどの情報を直接書き込むのはセキュリティ上の欠陥ですし、かといってこれまで通りgitignoreして別ファイルに書くと、なにか編集した場合DjangoとDockerの両方を編集せねばならず、デプロイを簡潔にするDockerの哲学に反し、保守性を欠きます。そこで、ここでは環境変数で共通の値を両方に渡し、その環境変数を管理することにしましょう。

Djangoには環境変数を読み込むためのパッケージdjango-environが用意されていますのでこれを利用します。同時に、PostgreSQLとの接続に必要なpsycopg2-binaryをrequirements.txtに追記しておいてください。本番環境を想定する場合、これに加えてgunicornも追記します。仮想環境を作成してこれらをインストールしてrequirements.txtに出力、でも構いません。Poetryを利用している場合もこのようにファイルに出力しておく必要があります。

settings.pyで`import  environ`、`import os`と、ファイル一番下に
```
env = environ.Env()
env.read_env(os.path.join(BASE_DIR,'.env'))
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": env("POSTGRES_DB"),
        "USER": env("POSTGRES_USER"),
        "PASSWORD": env("POSTGRES_PASSWORD"),
        "HOST": "db",
        "PORT": 5432,
    }
}
```
を追記し、データベースをPostgreSQLに指定します。ここで重要なのは、PORTを指定するということです。これにより、同IP内の指定されたポートにデータベースアクセスをする、つまり先程のDocker Composeの図におけるapp serverコンテナとDBコンテナの間の接続を行います。
そして、BASE_DIR以下に.envファイルを作成し、その中にDBのパスワードなどを記述します。
```
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_INITDB_ARGS="--auth=md5"
```
４行目は認証方式を指定しています（デフォルトのものだとこの後エラーが起きる可能性があります）

以上でDjango関連の設定は終了です。次にDocker Composeの設定をしていきます。

### docker-compose.ymlの設定

どこでもいいので（今回はintern以下に作っています）、docker-compose.ymlというファイルを作ります。Docker Compose周りの設定はこのymlファイルに記述していくことになります。ひとまず設定は以下の通り
```
version:  '3'
  
services:
	db:
		image:  postgres
		env_file:
			- .env
		expose:
			- 5432
		volumes:
			- ./postgres_data:/var/lib/postgresql/data
		user:  ${POSTGRES_USER}
		healthcheck:
			test:  /usr/bin/pg_isready
			interval:  10s
			timeout:  5s
			retries:  5
	web:
		build:  .
		command:  python3 manage.py runserver 0.0.0.0:8000
		volumes:
			- .:/app
		ports:
			- 8000:8000
		depends_on:
			db:
				condition:  service_healthy
```
まずversionというのはComposeファイルのバージョンを指します。現状はいつも3で構いません。

servicesとはCompose内で立てるコンテナ群のことで、その中の要素がそれぞれのコンテナを指します。
各コンテナのなかでは、まず元となるイメージかDockerfileのいずれかを指定します。イメージの場合はimageキーにそのイメージの名前を指定し、Dockerfileの場合はbuildキーにDockerfileのある階層を指定します。今回dbコンテナは公開されているPostgresのイメージを、webコンテナは先ほど書いたDockerfileを指定しています。これらを同時に指定することはできません。

env_fileでは環境変数の入ったファイルを指定して環境変数をコンテナ内に渡すことができ、ここでは先ほど書いたDBのパスワード等を渡しています。
また、ここでは使っていませんが、environmentキーで環境変数を直に記述して渡すこともできます。例えばDJANGO_SETTINGS_MODULEを渡して読み込む設定ファイルの制御を行うことなどもできるでしょう。

exposeとportsはいずれもポートに関する設定であり、exposeはCompose内でのみ公開するポートを指定、portsは<Compose外>:<Compose内>という記法で外からのリクエストをフォワードします。従って、この場合Compose外から5432にアクセスすることはできませんが、8000にアクセスすればCompose内の8000にフォワードされ、Djangoのサーバーに届きます。DBは、Djangoの設定で指定したポートをexposeしてください。

volumesは<マウントしたいディレクトリのパス>:<コンテナ内のパス>でコンテナに指定したディレクトリをマウントすることができます。dbではコンテナ内の`/var/lib/postgresql/data`にマウントしてデータを永続化することを忘れないでください。webコンテナには素ではPythonしか入っていないので、Djangoアプリをマウントしています。

commandはコンテナ起動時に実行されるコマンドであり、順序的にはDockerfileのentrypointの後になります。

depends_onはコンテナの起動順序であり、ここではDBコンテナが正常に立ち上がったのを確認してからwebコンテナを立ち上げるように書いています。ただし、コンテナの起動とアプリの起動は別物で、例えばDBの初回起動時などは初期化の時間が長く、その間にwebコンテナが立ち上がってエラーが起きたりします。これを防ぐため、conditionでより詳細な条件を指定することができます。今回の場合、dbコンテナのhealthcheckのpg_isreadyコマンドでDBが立ち上がったかを判定し、service_healthyでこれを見てwebコンテナの起動を制御します。ちなみにコンテナを終了する際はこれと逆順に停止します。

### サーバーの起動

これで一旦構成は完成ですので、ここで一度動作確認をしてみましょう。Docker単体ではDockerfileからイメージを作成するのが`docker build`、処理を実行するのが`docker run`というコマンドですが、Composeでは`docker-compose up --build`というコマンドで全コンテナのリビルドから立ち上げまでを行ってくれます。今回の構成ではコンテナを立ち上げれば自動的にサーバーも起動する設定になっていますので直接何かコマンドを打つことはあまりないですが、例えばmakemigrationsやcreatesuperuserをする場合などは、`docker-compose exec <コンテナ名> <コマンド>`で実行できるほか、`docker-compose exec <コンテナ名> bash`でコンテナのbashを立ち上げて中に入ることができます。起動しているコンテナの削除は`docker-compose down`で行えます。
ymlと同じ階層で`docker-compose up --build`と打ってみてください。ビルドが走り、コンテナが立ち上がると思います。ここまでの流れが正しくできていれば、 http://localhost:8000/ にアクセスすればアプリが立ち上がっているはずです。

### gunicornとnginxの追加

これで普段開発するための環境は整いましたが、デプロイする際にはこれをgunicorn+nginxにする必要があります。最後にこの設定を行います。
gunicornへの移行は単純で、docker-compose.ymlのwebコンテナのcommandを`gunicorn intern.wsgi:application --bind 0.0.0.0:8000`にするだけです。
nginxについては、nginxのイメージが公開されているのでこれを利用します。設定については、nginx用のディレクトリを作成し、この中にnginx.confファイルを作成して、そこに以下の内容を記述します。
```
upstream  application  {
	server web:8000;
}

server  {
	listen  80;
	location  /  {
		proxy_pass http://application;
		proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header Host  $host;
		proxy_redirect off;
	}
	location /static/ {
		alias /staticfiles/;
	}
	location /media/ {
		alias /media/;
	}
}
```
概ね普段のnginxと記法は同じですが、まずupstreamの定義ではコンテナ名を使用すると名前解決してくれます。そのため、今回の場合webコンテナのポート8000へのアクセスをapplicationとして定義しています。
そして、その後のproxy_passにおいては、このupstream名を使用しないとエラーが出るので注意してください。
staticfilesとmediaについては、nginxコンテナ内の/staticfiles/と/media/を参照することにします。

intern以下にnginxディレクトリを作成し、同じくdocker-compose.ymlのservices以下に追記しましょう。
```
services:
	nginx:
		image:  nginx:latest
		volumes:
			- ./staticfiles:/staticfiles
			- ./media:/media
			- ./nginx/nginx.conf:/etc/nginx/conf.d/nginx.conf
		ports:
			- 80:80
		depends_on:
			- web
```
staticfilesとmediaはymlからの相対パスか絶対パスを指定してください。
また、webコンテナの`ports: - 8000:8000`を`expose: 8000`にして、nginxを介さず直にwebコンテナへアクセスすることを禁止します。
以上で全環境の構築は終了です。ctrl+Cでコンテナを終了し、`docker compose up --build`でリビルドしてhttp://localhost:8000/ に再度アクセスし、異常がないことを確認してください。`docker compose up`に`-d`オプションをつければ、サーバーのログを流さずバックグラウンドで起動することも可能です。

### まとめ

これらのファイルを他の人と共有するか、Docker Hubを通してイメージを共有することで、全く同じ環境で開発を進めることができます。
最初はやや煩雑に感じるかもしれませんが、環境構築を誰か一人が行えば良い点、設定を一箇所に集中して書ける点については実感してもらえたかと思います。クローンしてきたリポジトリのdocker-introブランチに完成版をあげておきますので、何かあればそちらを参照してください。
