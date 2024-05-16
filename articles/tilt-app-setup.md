---
title: "Tiltの特徴と環境構築手順~アプリケーション構築まで~"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [kubernetes,docker,tilt,kind]
published: false
---

## 概要
皆さんは。。。

- Tiltの特徴について
  - Docker Composeと比較して説明します
- Tiltのセットアップ
  - a
- Tiltでアプリケーションを構築
  - a

## Tiltとは
Tiltは、Kubernetes環境での開発を効率化するためのツールです。Tiltfileを用いてデプロイプロセスをスクリプト化し、開発者に迅速なフィードバックと一貫した開発環境を提供します。

## Docker Compose との比較
開発環境でのコンテナ管理をサポートするツールとして、まず名前が上がるのがDocker Composeでしょう。そこで、Docker Composeと比較して、Tiltのメリットとデメリットを見ていきます。

### メリット
#### 環境の一貫性
プロダクションでKubernetesを利用している場合、Tiltの恩恵は大きいと思います。同じKubernetesマニフェスト、Kustomize、Helmチャートを使用できるため、環境間の一貫性が保たれ、環境依存のバグを減らせます。

#### 迅速なフィードバック
Tiltはコード変更を検知して自動的にコンテナを再ビルドし、Kubernetesクラスターに再デプロイします。これにより、コードの変更がすぐに反映され、迅速なフィードバックが得られます。開発サイクルが短縮され、生産性が向上します。

#### リアルタイムのログと状態監視
Tiltはリアルタイムでログやアプリケーションの状態を監視し、開発者にフィードバックを提供します。これにより、問題の検出やデバッグが迅速に行えます。

### デメリット
#### 必要な知識が多い
Kubernetes自体の知識はもちろんですが、Tiltfileの記述や設定も学習する必要があります。Tiltfileについては現時点では英語のドキュメントがほとんどです。

#### セットアップが複雑
セットアップや環境構築が複雑で、特にKubernetesクラスタの構築や管理が必要です。こちらも英語のリファレンスがほとんどです。

### どっち使えばいい？（個人的見解）
プロダクションでKubernetesを利用しているのであればTiltをオススメしたいです。そうでなければ、無理にTiltを使う必要はなく、シンプルなDocker Composeで良いと思います。

## 事前準備
### Go
- [インストール](https://go.dev/doc/install)
```sh
brew install go
```
- 確認
```sh
go version
```

### docker
- [インストール](https://docs.docker.com/desktop/)
```sh
brew install --cask docker
```
- 確認
```sh
docker -v
```

### kind
- [インストール](https://kind.sigs.k8s.io/docs/user/quick-start/)
```sh
brew install kind
```
- 確認
```sh
kind version
```

### kubectl
- [インストール](https://kubernetes.io/ja/docs/tasks/tools/#kubectl)
```sh
brew install kubectl
```
- 確認
```sh
kubectl version --client
```

### Tilt
- [インストール](https://docs.tilt.dev/install.html)
```sh
brew install tilt-dev/tap/tilt
```
- 確認
```sh
tilt version
```

### ctlptl
- [インストール](https://github.com/tilt-dev/ctlptl)
```sh
brew install tilt-dev/tap/ctlptl
```
- 確認
```sh
ctlptl version
```

## アプリケーションの構築

## DBの構築

## docker registry ui の導入

## まとめ
現場でTiltを使ってローカル開発環境を構築したので、紹介記事を書いてみました。日本語のリファレンスが少ないため、この記事が誰かの足掛かりになれば幸いです。
以上です。
