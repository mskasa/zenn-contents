---
title: "Elasticsearch"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [elasticsearch]
published: false
---


## 論理的な概念

![](/images/elasticsearch/elasticsearch-0.drawio.png)

|  用語      | 概要                        | 
| -------- | --------------------------- | 
| インデックス    | RDBでいうテーブルのイメージ | 
| ドキュメント | RDBでいうレコードのイメージ | 
| フィールド    | RDBでいうカラムのイメージ   | 

### インデックス
* ドキュメントを保存する場所のこと
* 検索を効率的に行うために転置インデックス[^1]を構成したり、さまざまなデータ形式で保存されています
* 後述するシャードという単位に分割されて格納されます

[^1]: [転置インデックス](https://gihyo.jp/dev/serial/01/search-engine/0003)

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
* Elasticsearchにおける転置インデックスは、フィールドごとに作成、管理されています
* さまざまなデータ型がサポートされており、マッピングを定義することで型を指定します

https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html

#### マッピング
* インデックスの設定項目の1つ
* ドキュメント内の各フィールドのデータ構造や型を記述した情報のこと
* Elasticsearchが推測して自動的にマッピングを定義してくれるため、事前定義は必ずしも必要ではありません

:::message
**ドキュメントタイプ（Document Type）**
古い情報だとドキュメントタイプという概念が出てきますが、最新のバージョンでは完全に廃止となっています。
ドキュメントタイプは 1つのインデックスの中に複数のマッピングを持たせられるというものでしたが、この仕組みは分かりづらく、インデックスを分けたほうがよいという考えが主流になり、廃止されることになりました。
:::

## 物理的な概念

![](/images/elasticsearch/elasticsearch-1.drawio.png)

|  用語      | 概要                        | 
| -------- | --------------------------- | 
| クラスタ    | 協調して動作するノードグループ | 
| ノード | Elasticsearchが動作する各サーバー | 
| プリマリシャード | インデックスのデータを分割したもの   | 
| レプリカシャード | シャードの複製   | 

### ノード
|  ノードの種別      | 役割の概要                 | 
| -------- | --------------------------- | 
| Master(Master-eligible)    | ・Masterノード<br>・Master候補ノード | 
| Data | ・シャードの保持<br>・クエリへの応答<br>・etc... | 
| Ingest | ・データの変換や加工処理<br>・LogstashやFluentdのようなイメージ<br>・[LogstashとElasticsearchのIngestノード、どちらを使うべき？ - Elastic Blog](https://www.elastic.co/jp/blog/should-i-use-logstash-or-elasticsearch-ingest-nodes)  | 
| Coordinating only | ・ロードバランサー  | 
| Machine Learning | ・様々な機械学習ジョブを実行できる<br>・OSS版では設定不可  | 






### シャード
:::message
**シャーディング**
格納するデータ量が増大するにつれて、サーバーのCPU、メモリ、I/O負荷が上昇して、性能が徐々に低下することがあります。このとき、保持するデータを分割し、複数台のサーバーに分散配置することで1台あたりの負荷を減らすことができます。この仕組みは一般的にシャーディングと呼ばれており、Elasticsearchにおいても実現されています。
:::

#### プライマリシャード

#### レプリカシャード



## Masterノードの選定

## シャード分割とレプリカ


Document Typeは廃止
https://www.uipath.com/ja/community-blog/knowledge-base/elasticsearch-index-template