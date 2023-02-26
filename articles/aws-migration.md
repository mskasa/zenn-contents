---
title: "AWS 移行戦略 7R について簡単にまとめてみた"
emoji: "🦈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws, 移行, サーバー移行, データ移行]
published: true
published_at: 2023-02-27 07:00
---

## 概要
AWS における移行戦略 7R を簡単にまとめてみました。
ざっくりと理解することを目的としているので、内容は浅いです。ご容赦ください。

## 移行戦略
### 概要
| 移行戦略   | 戦略の概要（オンプレのシステムをどうするか）               | 難易度 | 
| ---------- | ---------------------------------------------------------- | ------ | 
| Rehost     | IaaS（EC2）中心の構成に移行する。                            | 易     | 
| Relocate   | [VMware Cloud on AWS](https://aws.amazon.com/jp/vmware/) に移行する。   | 易     | 
| Replatform | マネージドサービスを中心とした構成に移行する。 | 中     | 
| Repurchase | 同じ価値を提供できる製品（SaaSなど）を購入し、乗り換える。   | 中     | 
| Refactor   | すべての設計を見直し、クラウドネイティブな構成に作り直す。 | 難     | 
| Retire     | 廃止する。                                                   | -      | 
| Retain     | 現状維持。現手を加えずに稼働させ続ける。                                    | -      | 

### 検討フロー
![](/images/aws-migration/flow.drawio.png)
:::message
あくまで「検討の余地あり」なので、矢印の先一択ではないです
:::

### 具体例
#### Rehost（リホスト）
![](/images/aws-migration/rehost.drawio.png)

移行ツールとして、[AWS Application Migration Service(AWS MGN)](https://aws.amazon.com/jp/application-migration-service/) があります。
アンマネージドサービスである EC2 を中心とした構成に乗り換えるため、オンプレミスの構成とギャップが小さいというメリットがあります。
ただし、アプリケーション自体がステートフルなままだと、EC2 をスケーリングできないなどの制約があり、クラウドを利用する旨味が薄くなります。

#### Replatform（リプラットフォーム）
![](/images/aws-migration/replatform.drawio.png)

DBサーバーは、[AWS Database Migration Service](https://aws.amazon.com/jp/dms/) を利用して、マネージドサービスである RDS や Aurora に移行します。異なる DBエンジンの場合は [AWS Schema Conversion Tool](https://docs.aws.amazon.com/ja_jp/SchemaConversionTool/latest/userguide/CHAP_Welcome.html) で変換します。
ファイルサーバーは、[AWS DataSync](https://aws.amazon.com/jp/datasync/) を利用して、マネージドサービスである S3 や EFS などにデータを移行できます。

![](/images/aws-migration/beanstalk.drawio.png)

また、[Elastic Beanstalk](https://aws.amazon.com/jp/elasticbeanstalk/) などの PaaS にアプリケーションを移行する場合も、Replatform戦略といえるでしょう。

#### Refactor（リファクタリング）
コンテナ化が良い例ではないでしょうか。

![](/images/aws-migration/refactor.png)
*[AWS ソリューション構成例 - コンテナを利用した Web サービス](https://aws.amazon.com/jp/cdp/ec-container/)*

コンテナ化までとは行かずとも、Auto Scaling + EC2 構成でもアプリケーションをステートレスにする必要があります。セッション情報などを ElastiCache などの外部ストレージに保存するようにアプリケーションを変更する必要があります。
アプリケーションの基礎部分の設計から見直すため、移行の難易度およびコストは高くなる一方、クラウドのメリットを最大限に活かすことができます。

## 移行計画のサポートツール

### AWS Cloud Adoption Readiness Tool (CART)
[AWS Cloud Adoption Readiness Tool (CART)](https://cloudreadiness.amazonaws.com/#/cart) はクラウド導入支援ツールです。
質問に答えることで、クラウド移行の準備状況と、それに対する大まかな推奨事項のレポートが作成されます。AWSアカウントが無くても利用できるのがポイントです。
https://dev.classmethod.jp/articles/try-migrationreadinessassessment/

### AWS Application Discovery Service
[AWS Application Discovery Service](https://aws.amazon.com/jp/application-discovery/) はオンプレミスの情報を収集して、移行計画をサポートするサービスです。
収集した情報は後述する AWS Migration Hub で確認でき、そのまま移行管理にも利用できます。また、収集した情報を Kinesis Data Firehose 経由で S3 に送信し、Athena で SQLクエリを利用した分析が可能です。

![](/images/aws-migration/discovery.png)
*[Qiita-【初心者】AWS Application Discovery Service を使ってみる](https://qiita.com/mksamba/items/cb4797763eba25e53ec7)*

https://qiita.com/mksamba/items/cb4797763eba25e53ec7

### AWS Migration Hub
[AWS Migration Hub](https://aws.amazon.com/jp/migration-hub/) は移行を管理するサービスです。前述の AWS Application Discovery Service で収集した情報を可視化し、最適な移行サービス（3rd Party製含む）を選択することができます。移行開始後は、移行の進捗状況をトラッキングすることができます。詳しくは下記の記事を参照ください。古い記事ですが、イメージを掴む分には問題ないです。

https://dev.classmethod.jp/articles/20170815-migration-hub/

## 参考
* [AWS への移行:ベストプラクティスと戦略](https://pages.awscloud.com/rs/112-TZM-766/images/Migrating-to-AWS_Best-Practices-and-Strategies_eBook.pdf)
