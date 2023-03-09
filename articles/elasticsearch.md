---
title: "Elasticsearchの基本概念とユースケースをまとめる"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [elasticsearch, kibana, fluentd, kubernetes]
published: false
---

## 本記事の目的

## Elasticsearchの概要

## Elasticsearchの論理的な概念

まずは Elasticsearchの論理的な概念（見かけ上の

![](/images/elasticsearch/elasticsearch-0.drawio.png)

|  用語      | 概要                        | 
| -------- | --------------------------- | 
| インデックス    | RDBでいうテーブルのイメージ | 
| ドキュメント | RDBでいうレコードのイメージ | 
| フィールド    | RDBでいうカラムのイメージ   | 

### インデックス
* ドキュメントを保存する場所のこと
* 検索を効率的に行うために転置インデックスを構成したり、さまざまなデータ形式で保存されています

https://gihyo.jp/dev/serial/01/search-engine/0003

### ドキュメント
* ドキュメントはJSONオブジェクトです
* ドキュメントは1つ以上のフィールドで構成されます

```json:example
{
    "name" : "Taro",
    "age" : 30,
    "email" : "taro@gmail.com"
}
```

### フィールド
* ドキュメント内の`key=value`の組をフィールドと呼びます
* さまざまなデータ型がサポートされており、マッピングを定義することで型を指定します
    * マッピング
        * インデックスの設定項目の1つ
        * ドキュメント内の各フィールドのデータ構造や型を記述した情報のこと
        * Elasticsearchが推測して自動的にマッピングを定義してくれるため、事前定義は必ずしも必要ではありません

https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html

:::details ドキュメントタイプ（Document Type）
古い情報だとドキュメントタイプという概念が出てきますが、最新のバージョンでは完全に廃止となっています。
ドキュメントタイプは 1つのインデックスの中に複数のマッピングを持たせられるというものでしたが、この仕組みは分かりづらく、インデックスを分けたほうがよいという考えが主流になり、廃止されることになりました。
:::

## Elasticsearchの物理的な概念

Elasticsearch が実際にどういった物理的なリソースで構成されているかを見ていきます。

![](/images/elasticsearch/elasticsearch-1.drawio.png)

|  用語      | 概要                        | 
| -------- | --------------------------- | 
| クラスタ    | 協調して動作するノードグループ | 
| ノード | Elasticsearchが動作する各サーバー | 
| プリマリシャード | インデックスのデータを分割したもの   | 
| レプリカシャード | プライマリシャードの複製   | 

### クラスタ

クラスタ構成

### ノード

Elasticsearchでは、様々なノードのロールが用意されており、そこから選択して、ノードの役割を決定します。

https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#node-roles

代表的なものをいくつか紹介します。

#### Master(Master-eligible)ノード
```yml:elasticsearch.yml
node.roles: [ master ]
```

#### Data ノード
```yml:elasticsearch.yml
node.roles: [ data ]
```

Dataノードはシャードを保持し、CRUDを実行します。高スペックが必要（特にメモリ）になるので注意しましょう。

#### Coordinating only ノード
```yml:elasticsearch.yml
node.roles: [ ]
```

Elasticsearch のノードは Coordinating と呼ばれる共通の役割を持っています。

Coordinating は、クライアントリクエストのハンドリング処理です。
具体的には、格納すべきシャードへのルーティングであったり、各シャードからのレスポンスを集約してクライアントに応答したりといった処理です。

これはすべてのノードに必要であるため、自動的に割り当てられる無効にできないロールと捉えるとよいでしょう。つまり、`node.roles`で何も指定しなかった場合、Coordinating only というロードバランサーの様な役割を担うノードになります。

### シャード
:::message
**シャーディング**
格納するデータ量が増大するにつれて、サーバーのCPU、メモリ、I/O負荷が上昇して、性能が徐々に低下することがあります。このとき、保持するデータを分割し、複数台のサーバーに分散配置することで1台あたりの負荷を減らすことができます。この仕組みは一般的にシャーディングと呼ばれており、Elasticsearchにおいても実現されています。
:::

#### プライマリシャード

#### レプリカシャード



## Masterノードの選定

## シャード分割とレプリカ
p.65


Document Typeは廃止
https://www.uipath.com/ja/community-blog/knowledge-base/elasticsearch-index-template

## Elasticsearchのユースケース例
実際に私がプロジェクトで構築したシステムを参考にユースケースを見ていきましょう。

![](/images/elasticsearch/elasticsearch-usecase.drawio.png)

### Relational Database の検索における wrapper としての役割

上図の Web/APIサーバー群 → 全文検索サーバー群 について。
ここでは、一部のデータに対して RDBだけでなく、Elasticsearchにも CRUD処理を実施しています。

ここで、RDBの特徴について今一度確認しましょう。通常、RDB はテーブル設計段階で正規化されているはずです。正規化はデータを整合的に管理する上で必要ですが、同時に検索のパフォーマンスを悪化させる可能性もあります。`join`されまくっているクエリが遅いのは皆さん経験したことあるでしょう。

例えば、顧客の検索を実装したいとき、顧客に関連するテーブルが多く、データ量も多いとなった場合、Elasticsearchを検討すべきです。顧客に関連する情報をすべて保持した Elasticsearch のインデックスを作成すれば、検索パフォーマンスは圧倒的に改善されるでしょう。

要するに RDB の検索面での wrapper のような働きをしてくれるわけですね。

### ログの収集と可視化を実装する役割

上図の Web/APIサーバー群 → ログ収集サーバー について。
ここでは、Kubernetes環境で分散された各コンテナのログについて、サイドカーの fluentd で取り込み、ログ収集サーバーに送信しています。そして、ログ収集サーバー側の fluentd（td-agent）で受け取ったログから Elaticsearch のインデックスを作成し、Kibana を用いた GUI でログを解析します。分散したログを集約し、ユーザビリティの高い GUI で横断検索ができるという点がポイントです。

:::message
スタンダードな全文検索システムとしての事例はこちら↓
https://www.elastic.co/jp/customers/nikkei-1
:::