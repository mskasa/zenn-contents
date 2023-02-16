---
title: "AWS 認証周りのサービスとユースケースのまとめ"
emoji: "🦈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws]
published: false
---

## 1. AWS Directory Service
AWS Directory Service は、AWS上でディレクトリサービスをマネージドで提供する。
ディレクトリサービスは、ネットワークに接続しているリソースやユーザーを管理する。
代表例として、Microsoft の Active Directory がある。

### 1-1. オンプレミスの Active Directory の認証をそのまま使いたい
例えば、下記サービスを既存のユーザーで利用したい場合など。

| サービス名        | サービス概要                                                   | 
| ----------------- | -------------------------------------------------------------- | 
| Amazon WorkSpaces | マネージド型の仮想デスクトップサービス                         | 
| Amazon WorkDocs   | 完全マネージド型の企業向けストレージおよびファイル共有サービス | 
| Amazon WorkMail   | 企業向け E メールおよびカレンダーのマネージド型サービス        | 
| Amazon QuickSight | 企業の各部署が蓄積している膨大なデータの分析                   | 

この場合、AWS Directory Service の AD Connector を使用する。



### AWSに Active Directory を移行したい
単純にマネージドサービスを利用することでクラウドの恩恵を受けたい

ともにVPCで起動するマネージドサービス

Simple AD

Samba 4 Active Directory Compatible Server

最大5000ユーザー

Managed Microsoft AD

Windows Server 2012 R2によって動作

5000を超えるユーザー

他ドメインとの信頼関係

MFA（多要素認証）

## AWS IAM Identity Center(旧 AWS Single Sign-On)

### AWSアカウントに SSO したい

## Amazon Cognito

アプリケーションに認証機能を短期間で実装したい

