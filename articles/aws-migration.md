---
title: "AWS 移行戦略 7R について簡単にまとめてみた"
emoji: "🦈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws]
published: false
---

## 概要
AWSにおける移行戦略7Rを簡単にまとめてみました。

## 移行戦略
### 概要
| 移行戦略   | 戦略の概要（オンプレのシステムをどうするか）               | 難易度 | 
| ---------- | ---------------------------------------------------------- | ------ | 
| Rehost     | IaaS（EC2）中心の構成に移行する。                            | 易     | 
| Relocate   | [VMware Cloud on AWS](https://aws.amazon.com/jp/vmware/) に移行する。   | 易     | 
| Replatform | クラウドのメリットを享受できるマネージドサービスを中心とした<br>構成に移行する。 | 中     | 
| Repurchase | 同じ価値を提供できる製品（SaaSなど）を購入し、乗り換える。<br>既存システムは廃止する。   | 中     | 
| Refactor   | アプリケーションのコードを含めて、すべての設計を見直し、<br>クラウドネイティブな構成に作り直す。 | 難     | 
| Retire     | 廃止する。                                                   | -      | 
| Retain     | 現状維持。手を加えずに稼働させ続ける。                                     | -      | 

### 検討フロー
![](/images/aws-migration/flow.drawio.png)
:::message
あくまで「検討の余地あり」なので、矢印の先一択ではないです
:::
### Rehost（リホスト）
![](/images/aws-migration/rehost.drawio.png)

移行ツールとして、[AWS Application Migration Service(AWS MGN)](https://aws.amazon.com/jp/application-migration-service/)があります。
アンマネージドサービスであるEC2を中心とした構成に乗り換えるため、オンプレミスとの構成や知識のギャップは少なくなります。
ただし、アプリケーション自体がステートフルなままだと、EC2をスケーリングできないなどの制約があるため、クラウドを利用する旨味が薄くなります。

### Replatform（リプラットフォーム）
![](/images/aws-migration/replatform.drawio.png)

DBサーバーは、[AWS Database Migration Service](https://aws.amazon.com/jp/dms/) を利用して、マネージドサービスであるRDSやAuroraに移行します。異なるDBエンジンの場合は[AWS Schema Conversion Tool](https://docs.aws.amazon.com/ja_jp/SchemaConversionTool/latest/userguide/CHAP_Welcome.html)で変換します。
ファイルサーバーは、[AWS DataSync](https://aws.amazon.com/jp/datasync/) を利用します。マネージドサービスであるS3やEFSなどにデータを移行できます。

![](/images/aws-migration/beanstalk.drawio.png)

また、[Elastic Beanstalk](https://aws.amazon.com/jp/elasticbeanstalk/)などのPaaSにアプリケーションを移行する場合も、Replatform戦略といえるでしょう。

### Refactor（リファクタリング）
コンテナ化が良い例ではないでしょうか。

![](/images/aws-migration/refactor.png)
*[AWS ソリューション構成例 - コンテナを利用した Web サービス](https://aws.amazon.com/jp/cdp/ec-container/)*

コンテナ化までとは行かずとも、Auto Scaling + EC2 構成でもアプリケーションをステートレスにする必要があります。セッション情報などをElastiCacheなどの外部ストレージに保存するようにアプリケーションを変更する必要があります。
このように、アプリケーションの基礎部分の設計から見直すため、移行の難易度およびコストは高くなります。

## 移行計画のサポートツール

### AWS Cloud Adoption Readiness Tool (CART)
[AWS Cloud Adoption Readiness Tool (CART)](https://cloudreadiness.amazonaws.com/#/cart)はクラウド導入支援ツールです。
質問に答えることで、クラウド移行の準備状況と、それに対する大まかな推奨事項のレポートが作成されます。AWSアカウントが無くても利用できるのがポイントです。
https://dev.classmethod.jp/articles/try-migrationreadinessassessment/

### AWS Application Discovery Service
[AWS Application Discovery Service](https://aws.amazon.com/jp/application-discovery/)はオンプレミスの情報を収集して、移行計画をサポートするサービスです。
収集した情報は AWS Migration Hub で確認でき、そのまま移行管理にも利用できます。


![](/images/aws-migration/discovery.png)
*[Qiita-【初心者】AWS Application Discovery Service を使ってみる](https://qiita.com/mksamba/items/cb4797763eba25e53ec7)*

https://qiita.com/mksamba/items/cb4797763eba25e53ec7


## 参考
* [AWS への移行:ベストプラクティスと戦略](https://pages.awscloud.com/rs/112-TZM-766/images/Migrating-to-AWS_Best-Practices-and-Strategies_eBook.pdf)
