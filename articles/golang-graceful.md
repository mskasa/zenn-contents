---
title: "Go言語における Graceful Shutdown 解説とサンプル（HTTPサーバー、ジョブ/バッチ処理の場合）"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go, kubernetes]
published: false
---

## 概要

## Graceful Shutdown とは

## HTTPサーバー
一般的なHTTPサーバーソフトウェア（Nginx、Apache、etc…）は Graceful Shutdown をサポートしています。
また、多くのWebフレームワーク（Spring、Django、Express、etc…）でもサポートしていたり、実装に関する詳細なドキュメントを提供していたりします。

Go の net/http標準パッケージもその例に漏れず、Graceful Shutdown をサポートしています。

サンプルコードは以下になります。

## ジョブ/バッチ処理

どのように終了すればよいかはジョブの仕様によりけりといったところでしょうが、トランザクションの途中で落ちるのは避けたいという要求は共通しているのではないでしょうか？

サンプルコードは以下になります。

## Kubernetes環境において

