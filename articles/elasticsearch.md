---
title: "Elasticsearchの基本構成要素とユースケース（実例）をまとめる"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [elasticsearch, kibana, fluentd, kubernetes]
published: false
---

## 本記事について
タイトルの通り、Elasticsearchを知らない人に向けた、Elasticsearchの基本構成要素とユースケースの紹介記事です。

## Elasticsearchの概要・特徴
* Javaで書かれている全文検索ソフトウェア
* 索引型検索による高速検索
* クラスタ構成・シャーディングによる高可用性
* REST APIによるアクセスが可能
* データはJSON形式
* Query DSLというJSON形式の言語で検索が可能
* ログ収集・可視化などの様々なエコシステムと連携が可能

:::message
**索引型検索**
文書を高速に検索できるような索引を事前に構成しておき、その索引をキーに目的のデータを探す方式です。Elasticsearchでは、[転置インデックス](https://gihyo.jp/dev/serial/01/search-engine/0003)の仕組みが利用されています。
:::

:::message
**シャーディング**
格納するデータ量が増大するにつれて、サーバーのCPU、メモリ、I/O負荷が上昇し、性能が徐々に低下することがあります。このとき、保持するデータを分割し、複数台のサーバーに分散配置することで 1台あたりの負荷を減らすことができます。この仕組みを一般的にシャーディングと呼んでいます。
:::

https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html

## Elasticsearch の論理的構成要素

まずは Elasticsearchの論理的構成要素から見ていきます。
論理的構成要素は、主に Elasticsearchを利用する上で必要な知識となってきます。
論理的構成要素さえ分かれば、Elasticsearchをある程度使えるようになります。

![](/images/elasticsearch/elasticsearch-0.drawio.png)
*Elasticsearchの論理的構成要素*

|  用語      | 概説              | 
| -------- | --------------------------- | 
| インデックス    | ・RDB でいうテーブルのようなもの<br>・ドキュメントを保存する場所 | 
| ドキュメント | ・RDB でいうレコードのようなもの<br>・JSONオブジェクト<br>・1つ以上のフィールドで構成される | 
| フィールド    | ・RDB でいうカラムのようなもの<br>・ドキュメント内の Key=Value の組<br>・さまざまなデータ型がサポートされている  | 
| マッピング    | ・インデックスの設定項目の1つ<br>・ドキュメントのデータ構造、ドキュメント内の各フィールドの型を定義している | 

:::details ドキュメントタイプ（Document Type）について
古い情報だとドキュメントタイプという構成要素が出てきますが、最新のバージョンでは完全に廃止となっています。
ドキュメントタイプは 1つのインデックスの中に複数のマッピングを持たせられるというものでしたが、この仕組みは分かりづらく、インデックスを分けたほうがよいという考えが主流になり、廃止されることになりました。
:::

## Elasticsearch の物理的構成要素

次に Elasticsearchの物理的構成要素を見ていきます。
物理的構成要素は、主に Elasticsearchを構築・管理する上で必要な知識となってきます。
物理的構成要素を理解すれば、Elasticsearchを利用したシステムを構築できるでしょう。

![](/images/elasticsearch/elasticsearch-1.drawio.png)
*Elasticsearchの物理的構成要素*

|  用語      | 概説                     | 
| -------- | --------------------------- | 
| クラスタ    | ・協調して動作するノードグループ | 
| ノード | ・Elasticsearchが動作する各サーバー<br>・様々な役割を持たせられる | 
| プライマリシャード | ・インデックスのデータを分割したもの   | 
| レプリカシャード | ・プライマリシャードの複製<br>・自動的にプライマリシャードのあるノードとは異なるノードに配置される<br>・プライマリシャードが消失した際、自動でプライマリシャードに昇格する   | 

### Node roles

Elasticsearchでは、様々なノードのロールが用意されており、そこから選択して、ノードの役割を決定します。

https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#node-roles

代表的なものを、設定ファイル（`elasticsearch.yml`）の記述と共にいくつか紹介します。

#### Master-eligible node
```yml:elasticsearch.yml
node.roles: [ master ]
```
* Masterノードの役割
    * ノードの参加と離脱の管理（ヘルスチェック）
    * クラスタメタデータの管理
    * シャードの割当と再配置

Elasticsearchクラスタには、必ず1台のMasterノードが必要で、Master-eligibleノードの中から、選定アルゴリズムによって決定されます。
つまり、Masterノードが停止してしまった際にも選定アルゴリズムによって、Master-eligibleノードから別のMasterノードが選定され、クラスタ構成が保たれるということです。

https://www.elastic.co/guide/en/elasticsearch/reference/8.6/modules-discovery-voting.html

#### Data node
```yml:elasticsearch.yml
node.roles: [ data ]
```

Dataノードはシャードを保持し、CRUDを実行します。検索基盤として高パフォーマンスを維持するためには、非常に高性能な Dataノードが必要になります。

* ストレージ
    * 高速なI/O処理が必要なため、SSD推奨
    * RAIDを構成する場合
        * ストライピングでパフォーマンスを上げる
        * ミラーリングは不要（Elasticsearch 自体の機能であるレプリカで対応）
* CPU
    * 検索（search）や集計（aggregation）処理には多くの計算リソースが必要
* メモリ
    * 主に以下を考慮する
        * OS のページキャッシュ
        * Elasticsearch の JVM に割り当てるヒープサイズ

https://www.elastic.co/jp/blog/elasticsearch-caching-deep-dive-boosting-query-speed-one-cache-at-a-time

#### Coordinating only node
```yml:elasticsearch.yml
node.roles: [ ]
```

Elasticsearch のノードは Coordinating と呼ばれる共通の役割を持っています。

* Coordinating は、クライアントリクエストのハンドリング処理のこと
    * 具体的には…
        * 格納すべきシャードへのルーティング
        * 各シャードからのレスポンスの集約およびクライアントへの応答

これはすべてのノードに必要であるため、自動的に割り当てられる無効にできないロールと捉えるとよいでしょう。つまり、`node.roles`で何も指定しなかった場合、Coordinating only というロードバランサーの様な役割を担うノードになります。

## Elasticsearchのユースケース（実例）
実際に私がプロジェクトで構築したシステムを参考にユースケースを見ていきましょう。

![](/images/elasticsearch/elasticsearch-usecase.drawio.png)
*Elasticsearchのユースケース（実例）*

### Relational Database の検索における wrapper としての役割

上図の Web/APIサーバー群 → 全文検索サーバー群 について。
ここでは、一部のデータに対して RDBだけでなく、Elasticsearchにも CRUD処理を実施しています。

ここで、RDBの特徴について今一度確認しましょう。通常、RDB はテーブル設計段階で正規化されているはずです。正規化はデータを整合的に管理する上で必要ですが、同時に検索のパフォーマンスを悪化させる可能性もあります。`join`されまくっているクエリが遅いのは皆さん経験したことあるでしょう。

例えば、顧客の検索を実装したいとき、顧客に関連するテーブルが多く、データ量も多いとなった場合、Elasticsearchを検討すべきです。顧客に関連する情報をすべて保持した Elasticsearch のインデックスを作成すれば、検索パフォーマンスは圧倒的に改善されるでしょう。

要するに RDB の検索面での wrapper のような働きをしてくれるわけですね。

### ログの収集と可視化を実装する役割

上図の Web/APIサーバー群 → ログ収集サーバー について。
ここでは、Kubernetes環境で分散された各コンテナのログについて、サイドカーの fluentd で取り込み、ログ収集サーバーに送信しています。そして、ログ収集サーバー側の fluentd（td-agent）で受け取ったログから Elaticsearch のインデックスを作成し、Kibana を用いた GUI でログを解析します。分散したログを集約し、ユーザビリティの高い GUI で横断検索ができるという点がポイントです。

## 概要をつかんだところで…

### Elastic Stack実践ガイド[Elasticsearch/Kibana編]
https://book.impress.co.jp/books/1119101078

この書籍は情報が古い以外は特に文句がなく、体系的に学ぶためにとても有用です（今ならKindle Unlimitedで無料で読めます）。

### Elasticsearch Guide
https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html

一次情報。リファレンスとして。
