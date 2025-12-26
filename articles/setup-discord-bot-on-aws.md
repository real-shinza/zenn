---
title: "DiscordボットをGitHubからAWSのEC2にデプロイして常時稼働させる為の手順"
emoji: "🌥"
type: "tech"
topics: ["discord", "aws", "ec2", github, githubactions]
published: false
---

# はじめに

### この記事の目的
この記事は作業内容を記録することを目的として執筆しています。
EC2 上に Discord ボットをデプロイし、常時稼働させるための準備手順をまとめます。

### 構成と方針
この記事では、以下の方針で構成しています。

- Discord アプリの作成方法やボットの実装詳細については扱わない
- 運用コストをできるだけ抑えることを優先する
- ボットの一部機能として Docker を利用しているが、詳細は扱わない

:::message
ECS や Lambda などのマネージドサービスは使用せず、  
EC2 上で必要なサービスを直接動かす構成を採用しています。
:::

# この記事の前提

### 対象範囲

以下の内容については、本記事の対象外とします。

- Discord ボットの作成方法
- Discord ボットの実装詳細
- Discord ボットが持つ各機能の実装や利用方法

### 事前準備
以下が完了していることを前提とします。

- AWS アカウントの作成されていること
- Discord ボットの作成されていること
- Discord ボットのトークンを取得済みであること
- GitHub リポジトリが作成されていること

### 使用技術
この記事で扱う Discord ボットおよびインフラ構成は以下の通りです。

| 項目 | 使用技術 | 用途 |
|---|---|---|
| 言語 | TypeScript (Node.js) | Discord ボットの実装 |
| Discord API | discord.js | Discord との通信 |
| コンテナ | Docker | ボットの一部機能で利用 |
| リポジトリ管理 | GitHub | ソースコードの管理 |
| CI/CD | GitHub Actions | EC2 への自動デプロイ |
| サーバー | AWS EC2 | ボット・関連サービスの常時稼働 |
| OS | Amazon Linux | EC2 の実行環境 |
| プロセス管理 | pm2 | Discord ボットのプロセス管理 |

# EC2 インスタンスの作成

# セキュリティグループの設定

# GitHub Actions から EC2 へのデプロイ概要
