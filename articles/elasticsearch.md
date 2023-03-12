---
title: "Elasticsearchの構成要素とユースケース（実例）"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [elasticsearch, kibana, fluentd, kubernetes]
published: false
---

## 本記事について
Elasticsearchを知らない人に向けて Elasticsearchの構成要素とユースケースについて解説します。
本記事後半にて、私が実際にプロジェクトで構築したシステムを実例として紹介します。

## Elasticsearchの概要・特徴
* Javaで書かれている全文検索ソフトウェア
* 索引型検索を利用
* クラスタ構成・シャーディングによって高可用性を実現
* REST APIによるアクセスが可能
* データはJSON形式
* Query DSLというJSON形式の言語で検索が可能
* ログ収集・可視化などの様々なエコシステムと連携可能

:::message
**索引型検索**
文書を高速に検索できるような索引を事前に構成しておき、その索引をキーに目的のデータを探す方式。Elasticsearchでは、[転置インデックス](https://gihyo.jp/dev/serial/01/search-engine/0003)の仕組みが利用されている。
:::

:::message
**シャーディング**
格納するデータ量が増大するにつれて、サーバーのCPU、メモリ、I/O負荷が上昇し、性能が徐々に低下することがある。このとき、保持するデータを分割し、複数台のサーバーに分散配置することで 1台あたりの負荷を減らすことができる。この仕組みを一般的にシャーディングと呼ぶ。
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
古い情報だとドキュメントタイプという構成要素があるが、最新のバージョンでは完全に廃止となっている。
ドキュメントタイプとは、1つのインデックスの中に複数のマッピングを持たせられるというものだが、この仕組みは分かりづらく、インデックスを分けたほうがよいという考えが主流になり、廃止されることになった。
:::

## Elasticsearch の物理的構成要素

次に Elasticsearchの物理的構成要素を見ていきます。
物理的構成要素は、主に Elasticsearchを構築・管理する上で必要な知識となってきます。

![](/images/elasticsearch/elasticsearch-1.drawio.png)
*Elasticsearchの物理的構成要素*

|  用語      | 概説                     | 
| -------- | --------------------------- | 
| クラスタ    | ・協調して動作するノードグループ | 
| ノード | ・Elasticsearchが動作する各サーバー<br>・様々な役割を持たせられる（後述の Node roles を参照） | 
| プライマリシャード | ・インデックスのデータを分割したもの   | 
| レプリカシャード | ・プライマリシャードの複製<br>・自動的にプライマリシャードのあるノードとは異なるノードに配置される<br>・プライマリシャードが消失した際、自動でプライマリシャードに昇格する   | 

### Node roles

Elasticsearchでは、様々なノードのロールが用意されており、それらを組み合わせてノードの役割を決定します。

https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#node-roles

代表的なものを、設定ファイル（`elasticsearch.yml`）の記述と共にいくつか紹介します。

#### Master-eligible node
```yml:elasticsearch.yml
node.roles: [ master ]
```

Master-eligible node とは、Master候補のノードという意味です。

* Master node の役割
    * ノードの参加と離脱の管理（ヘルスチェック）
    * クラスタメタデータの管理
    * シャードの割当と再配置

Elasticsearchクラスタには、必ず1台のMaster node が必要で、Master-eligible node の中から、選定アルゴリズムによって決定されます。
何らかの理由で Master node が停止してしまった際にも、選定アルゴリズムによって 1台のMaster-eligible node が Master node に昇格し、クラスタ構成が保たれます。

https://www.elastic.co/guide/en/elasticsearch/reference/8.6/modules-discovery-voting.html

#### Data node
```yml:elasticsearch.yml
node.roles: [ data ]
```

Data node はシャードを保持し、CRUDを実行します。検索基盤として高パフォーマンスを維持するためには、非常に高性能な Data node が必要になります。

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

* Coordinating とは、クライアントリクエストのハンドリング処理のこと
    * 具体的には…
        * 格納すべきシャードへのルーティング
        * 各シャードからのレスポンスの集約およびクライアントへの応答

この処理はすべてのノードに必要であるため、Coordinating は、自動的に割り当てられる且つ無効にできないロールと捉えるとよいでしょう。つまり、`node.roles`で何も指定しなかった場合、Coordinating only node というロードバランサーの様な役割を担うノードになります。

## Elasticsearch のユースケース（実例）
実際に私がプロジェクトで構築したシステムを参考にユースケースを見ていきましょう。

![](/images/elasticsearch/elasticsearch-usecase.drawio.png)
*Elasticsearchのユースケース（実例）*

### Relational Database の検索における wrapper としての役割

上図の Web/APIサーバー群 → 全文検索サーバー群 について。
ここでは、一部のデータに対して RDBだけでなく、Elasticsearchにも CRUD処理を実施しています。

さて、RDBの特徴について今一度確認しましょう。通常、RDB はテーブル設計段階で正規化されているはずです。正規化はデータを整合的に管理する上で必要ですが、同時に検索のパフォーマンスを悪化させる可能性もあります。`join`されまくっているクエリが遅いのは当たり前ですね。

例えば、顧客の検索を実装したいとき、顧客に関連するテーブルが多く、データ量も多いとなった場合、Elasticsearchを検討すべきです。顧客に関連する情報をすべて保持した Elasticsearch のインデックスを作成すれば、検索パフォーマンスは圧倒的に改善されるでしょう。

要するに RDB の検索面での wrapper のような働きをしてくれるわけですね。

### ログの収集と可視化を実装する役割

上図の Web/APIサーバー群 → ログ収集サーバー について。
ここでは、Kubernetes環境で分散された各コンテナのログをサイドカーの fluentd で取り込み、ログ収集サーバーに送信しています。そして、ログ収集サーバー側の fluentd（td-agent）で、受け取ったログを Elaticsearch に格納し、Kibana で解析します。分散したログを集約し、ユーザビリティの高い GUI で横断検索できるという点がポイントです。

## 参考

### Elastic Stack実践ガイド[Elasticsearch/Kibana編]
https://book.impress.co.jp/books/1119101078

この書籍は情報が古い以外は特に文句がなく、体系的に学ぶためにとても有用です（今ならKindle Unlimitedで無料で読めます）。

### Elasticsearch Guide
https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html

一次情報。リファレンスとして。
