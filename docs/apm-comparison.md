﻿# APMとは
ここでは、監視ツールの比較をするにあたって、そもそもAPMとはなんなのか定義します。

APMとは、Application Performance Managementの略で、アプリケーションやシステムのトラフィックやリソースの使用状況などのパフォーマンスを監視し、それを構成するサーバーやネットワークなどのインフラを管理することです。

ここでは、そのAPMを実現するために使われるAPMツールについての比較を行います。

# 監視ツールの比較

## DataDog

機能の特徴としては全ユーザーをひと画面に表示し、アクセスの多いホストを強調表示したり、条件付けをしてアクセスの傾向を見るなどのマーケティングよりの分析や、コンテナの監視、CIの可視化、サーバーレスの監視といった高度な機能など多様な機能が挙げられます。
また、セキュリティ面での機能も充実しており、脅威のリアルタイム検知や自動で暗号化を行ってくれる機能、システム全体のセキュリティスキャンなどが存在しています。
全体的に、監視の機能をビジネスの方向に生かそうとしているという印象を受けました。
![enter image description here](https://embed-fastly.wistia.com/deliveries/3a4e174fd73571f42ed182cf6194ec7b959f108e.webp?image_crop_resized=1280x800)
![enter image description here](https://imgix.datadoghq.com/img/blog/announcing-security-monitoring/datadog-security-monitoring-hero-rev2.png?fit=crop&w=1200&h=630)
導入にはホストにAgentと呼ばれるソフトをインストールする方法とAWS CloudWatchと連携する方法があり、前者であれば金銭コストを削減できる一方で人的・時間的コストがかかり、後者であれば導入は楽になりますが費用が嵩む可能性があります。
なお、料金は5ホストまでは無料ですが、これはコンテナのモニタリングやアラートなどの高度な設定は有料になり1ホストにつき15or23ドルになります。
https://www.datadoghq.com/ja/pricing/

## New Relic
New Relicはアラートの判別やインシデントの解析にAIを使用したオブザベーションプラットフォームです。
New Relicでは、純粋なトラフィック時間や実行時間などアプリケーションのパフォーマンスだけをAPMとして分離しており、インフラやログのトラッキングとはまた別のものとなっています。そのため、APMを行うにはAgentをサーバーにインストールする必要があり、ロギングを行うためにはCloudWatchを連携する必要があるようです。（基本的な機能自体に大きな差はありません）
Datadogがビジネス寄りだったのに対してこちらはより純粋な監視に特化した印象です。
![enter image description here](https://newrelic.com/sites/default/files/styles/1200w/public/2021-05/product_apm_marquee_sept2020.png?itok=6DKS0DuD)

Datadogとの違いとしては、Datadogにあるログの中のセンシティブなデータを検知して自動的に暗号化する機能などがない一方で、Datadogにはできないモバイルネイティブのアプリのパフォーマンス監視ができるようです。
![enter image description here](https://newrelic.com/sites/default/files/styles/1200w/public/2021-07/products_mobile_marquee_sept2020.png?itok=6zLXh_jz)

料金は従量課金が基本で、最初の１ユーザーに対して、月に100GBのデータ取り込み、0.25ドル/GB、1000件のインシデントまでは無料で、それ以上になるとそれぞれ0.5ドル/件の料金が発生するようです。また、ユーザーを増やすにも料金が発生し、その場合には機能に応じて問い合わせが必要になるようです。
https://newrelic.com/jp

## Site24x7
前にあげた二つと比較するとやや機能は劣りますが、それでも基本的なパフォーマンス監視やトラフィック、リソースの監視、コンテナの監視、ロギングやモバイルネイティブアプリの監視などの機能は整っています。
![enter image description here](https://www.site24x7.jp/help/images/dashboard-msp.jpg)
日本へのサービス拡充に力を入れている印象があり、日本語ページも豊富な他、日本語のリーフレットなども用意してくれているようです。
導入には他と同様サーバーへAgentをインストールする必要がありますが、CloudWatchとの連携は必須ではないようです。
Site24x7の強みは料金で、機能制限ありで5ホストまでは無料、10ホストまでは2000円、40ホストで7000円と、他と比べて安価な値段設定がされていることです。

## Zabbix
監視アプリケーション全体でいうと最も利用者が多いと言われているツールです。
これまで紹介してきたツールが全てSaaS型であり、サービスのサーバーにある監視アプリを利用してきたのに対して、このZabbixは監視サーバーを自分で構築し、他のサーバーの監視を行うオンプレミス型になります。
下の図でいう監視サーバーをこれまでのサービスでは提供者が用意してくれてたのに対して、このツールは自分で用意するということです。
![enter image description here](https://knowledge.sakura.ad.jp/images/2017/12/WS000244.png)
明らかに導入コストが大きくなりますが、その分料金は無料で全機能が配布されています。（~~**GPL**ですが~~）
まずは監視用のサーバーを立てて、ここにZabbix本体をインストールします。
https://knowledge.sakura.ad.jp/12446/
インストールはコマンドから行え、それ以降の操作はGUI上で行うことができます。
これができたら監視対象となるサーバーに他と同様Agentを導入し、それを本体に登録すれば監視が行えるようになります。
https://knowledge.sakura.ad.jp/13655/
なお、Zabbixで行えるのはリソースやネットワークの監視であり、アプリケーションパフォーマンスの監視は行えないので注意してください。

# まとめ
以上が候補となる監視ツールになります。これ以外にもツールは存在しているのですが、機能が少なく他の下位互換であったり、料金が高額だったりするのでここでは割愛します。基本的な機能はどれも似通っており、差となるのは料金と高度な機能とUIです。いずれについても実際に使ってみないとなんともいえないところですので、一度無料枠などを活用して試してみるのも手かもしれません。
