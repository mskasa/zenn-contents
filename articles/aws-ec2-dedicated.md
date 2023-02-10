---
title: "AWS EC2 Dedicated Instances(ハードウェア専有インスタンス)とDedicated Hostsの違いについて"
emoji: "🦈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws, ec2]
published: true
---

## 本記事の目的
* Dedicated Instances(ハードウェア専有インスタンス)の概要理解
* Dedicated Hostsの概要理解
* 自動配置とアフィニティについての整理

:::message
細かい内容は公式ドキュメントに記載されているので、本記事の対象外とします。
あくまで、概要理解を本記事の目的とします。
:::

## Dedicated Instances(ハードウェア専有インスタンス)
* 名前の通り、ハードウェア（ホスト）を専有するインスタンス
* EC2を1つ起動する度に、AWSからハードウェア（ホスト）が1つ割り当てられるイメージ
* コンプライアンス対応(他組織との同居を禁止する場合)などでよく利用される

![](/images/aws-ec2-dedicated/instances.drawio.png)

:::message
同一アカウントかつハードウェア専有インスタンスではない他のインスタンスとはハードウェア（ホスト）を共有できる
:::

## Dedicated Hosts
* 名前の通り、専有のホスト
* AWSからハードウェア（ホスト）を借りて、専有するイメージ
* BYOL(保有するソフトウェアライセンスを持ち込み、効率的に使用したい場合)などでよく利用される

![](/images/aws-ec2-dedicated/hosts.drawio.png)

### 自動配置とアフィニティ

#### 自動配置
* ホスト側で設定する
    * 無効の場合、当該ホストのIDを指定して起動したインスタンスのみが受け入れられる
    * 有効の場合、ターゲット未指定でもインスタンスタイプ設定が一致するインスタンスであれば受け入れられる

#### アフィニティ
* インスタンス側で設定する
    * 設定値`Host`の場合、特定のホストで起動したインスタンスが停止した場合、常に同じホストで再開される
    * 設定値`Off`の場合、特定のホストで起動したインスタンスが停止した場合、ベストエフォートで同じホストで再開される

## 所感
AWS SAP の勉強中に混乱したのでまとめてみました。
あくまで試験対策レベルなので、もし現場で利用する機会があれば追記しようと思います。
間違いあれば指摘お願いします。
以上です。

## 参考
https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/dedicated-instance.html
https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/dedicated-hosts-overview.html
