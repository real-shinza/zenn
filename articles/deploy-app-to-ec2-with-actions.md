---
title: "GitHub Actions から EC2 にデプロイして Discord ボットを常時稼働させる手順"
emoji: "🌥"
type: "tech"
topics: ["aws", "ec2", "github", "githubactions", "pm2"]
published: true
published_at: "2026-01-08 17:00"
---

# はじめに

### この記事の目的

この記事は、個人開発で作成した Discord ボットを GitHub Actions から EC2 に自動デプロイし、常時稼働させるまでの作業内容を記録することを目的として執筆しています。
実際に構築した環境をもとに、デプロイから運用までの環境構築手順を整理します。

### この記事で分かること

- GitHub Actions を使って、EC2 にアプリケーションを自動デプロイする方法  
- EC2 上で Discord ボットを常時稼働させるための環境構築と運用の流れ  
- pm2 や Docker を利用したシンプルな運用構成

# この記事の前提

### 構成と方針

この記事では、以下の方針で内容を整理しています。

- Discord アプリの作成方法やボットの実装詳細については扱わない
- ボットの一部機能として Docker を利用しているが、詳細は扱わない
- 運用コストをできるだけ抑えることを優先する

:::message
運用コストを抑えるため、ECS や Secrets Manager などのサービスは使用せず、EC2 上で必要なサービスを直接動かす構成にしています。
:::

### 対象範囲

以下の内容については、この記事では扱いません。

- Discord ボットの作成方法
- Discord ボットの実装詳細
- Discord ボットが持つ各機能の実装や利用方法

### 事前準備

以下が完了していることを前提とします。

- AWS アカウントが作成されていること
- Discord ボットのソースコードが作成されていること
- Discord ボットのトークンを取得済みであること
- GitHub リポジトリが作成され、ソースコードも push 済みであること

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
| OS | Amazon Linux 2023 | EC2 の実行環境 |
| プロセス管理 | pm2 | Discord ボットのプロセス管理 |

# EC2 インスタンスの作成

Discord ボットを常時稼働させるため、AWS EC2 インスタンスを作成します。
ここではコストを抑えつつ、最低限の構成を前提とします。

### インスタンスを作成

1. EC2 ダッシュボードを開く
2. 「インスタンスを起動」をクリック
3. インスタンス作成画面へ移動
  ![](/images/deploy-app-to-ec2-with-actions/ec2-instances.png)

### AMI（OS）の選択

インスタンスで使用する OS を選択します。
以下の組み合わせを選択しました。

- Amazon マシンイメージ: `Amazon Linux 2023 kernel-6.1 AMI`
- アーキテクチャ: `64ビット (x86)`

![](/images/deploy-app-to-ec2-with-actions/ec2-launch-instance-ami.png)

### インスタンスタイプの選択

インスタンスタイプを選択します。

:::message
EC2 では、選択するインスタンスタイプによって性能や料金が異なります。
主に違いが出るのは、以下の点です。

- CPU
- メモリ容量
- 1時間あたりの料金
:::

今回は `t3.micro` を選択しました。
理由は以下の通りです。

- ボットアプリのメモリ使用量が 0.5GiB ではやや余裕がなく、安定運用が難しかった
- `t3.micro` は メモリが 1GiB あり、余裕を持って動かせる
- CPU 性能はそれほど必要なく、コストとのバランスが良かった

![](/images/deploy-app-to-ec2-with-actions/ec2-launch-instance-type.png)

### キーペア

SSH 接続用のキーペアを設定します。
今回は以下の内容で新しいキーペアを作成しました。

- キーペアのタイプ： `ED25519`
- プライベートキーファイル形式：`.pem`

![](/images/deploy-app-to-ec2-with-actions/ec2-launch-instance-create-keypair.png)

:::message
キーペアを新規作成すると秘密鍵がダウンロードされるので、大切に保管して下さい。
秘密鍵は後ほど使用します。
:::

![](/images/deploy-app-to-ec2-with-actions/ec2-launch-instance-keypair.png)

### ネットワーク設定

ネットワーク設定を行います。
ここでは新しいセキュリティグループを作成し、SSH 接続に必要な最小限の設定のみ行います。

インバウンドルールでは、EC2 インスタンスへ SSH 接続するためにポート 22 (SSH) の通信を許可します。

:::message
0.0.0.0/0 は推奨しません。
HTTP / HTTPS が不要な場合は、追加しなくても問題ありません。
:::

![](/images/deploy-app-to-ec2-with-actions/ec2-launch-instance-network.png)

### ストレージを設定

ストレージを設定します。
コストを抑えつつ Discord ボットを安定稼働させる構成を選択します。

今回の設定は以下の通りです。

- ボリュームサイズ: 8 GiB
- ボリュームタイプ: 汎用 SSD（gp3）

![](/images/deploy-app-to-ec2-with-actions/ec2-launch-instance-storage.png)

### インスタンスを起動

設定内容を確認し、「インスタンスを起動」 をクリックします。
問題がなければ、数十秒ほどでインスタンスが起動します。

![](/images/deploy-app-to-ec2-with-actions/ec2-launch-instance-summary.png)

# EC2 インスタンスの初期セットアップ

EC2 インスタンスが起動したら、AWS マネジメントコンソールからインスタンスへ接続し、Discord ボットを動かすための初期セットアップを行います。

### EC2 インスタンスへ接続

EC2 インスタンス一覧、またはインスタンス詳細画面から右上にある「接続」をクリックします。

![](/images/deploy-app-to-ec2-with-actions/ec2-instance-detail.png)

ログインができるとブラウザ上で以下の画面が表示されます。

![](/images/deploy-app-to-ec2-with-actions/ec2-instance-connect.png)

### パッケージの更新

まずはシステムパッケージを最新の状態に更新します。

```bash
sudo dnf update -y
```

### Node.js をインストール

Discord ボットは Node.js で動作するため、Node.js をインストールします。

```bash
sudo dnf install -y nodejs
```

インストール後、以下のコマンドを実行し、バージョンが表示されれば問題ありません。

```bash
node -v
npm -v
```

### pm2 をインストール

Discord ボットを常時稼働させるため、pm2 をインストールします。
pm2 は Node.js アプリケーションのプロセス管理ツールです。

```bash
sudo npm install -g pm2
```

インストール後、以下のコマンドを実行し、バージョンが表示されれば問題ありません。

```bash
pm2 -v
```

### Docker をインストール

Discord ボットの一部機能で Docker を使用するため、Docker をインストールします。

```bash
sudo dnf install -y docker
```

Docker サービスの起動と自動起動設定

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

インストール後、以下のコマンドを実行し、バージョンが表示されれば問題ありません。

```bash
docker --version
```

### ec2-user を docker グループに追加

sudo を使わず Docker を操作できるように設定します。

```bash
sudo usermod -aG docker ec2-user
```

設定を反映するため、一度ログアウトします。

```bash
exit
```

再ログイン後、以下のコマンドを実行し、エラーが出なければ問題ありません。

```bash
docker ps
```

### Docker Compose をインストール

Docker Compose（スタンドアロン版）をインストールします。
Discord ボットの一部機能で、コンテナを管理するためにインストールしています。

```bash
sudo mkdir -p /usr/local/bin
sudo curl -SL https://github.com/docker/compose/releases/download/v5.0.1/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

インストール後、以下のコマンドを実行し、バージョンが表示されれば問題ありません。

```bash
docker-compose version
```

# Elastic IP（EIP）の設定

EC2 インスタンスを作成した直後は、パブリック IP アドレスは再起動のたびに変更される可能性があります。
GitHub Actions から EC2 に SSH 接続してデプロイを行う構成では、IP アドレスが変わらないことが非常に重要です。
そのため、EC2 インスタンスに対して Elastic IP（EIP）を割り当てます。

### Elastic IP を設定する理由

EIP を設定する主な理由は以下の通りです。

- EC2 再起動後も IP アドレスが変わらない
- GitHub Actions の SSH 接続先を固定できる

:::message
GitHub Actions を使った自動デプロイ構成では、EIP の設定はほぼ必須になります。
:::

### Elastic IP アドレスを割り振る 

1. AWS マネジメントコンソールで EC2 を開く
2. 左メニューから 「Elastic IP」 を選択
3. 「Elastic IP アドレスを割り当てる」 をクリック

![](/images/deploy-app-to-ec2-with-actions/ec2-elasticip-addresses.png)

### Elastic IP アドレスの設定

Elastic IP アドレス割当てでは、EC2 と同じリージョンで設定を行います。
設定後、画面下部の「割り当てる」 をクリックします。

![](/images/deploy-app-to-ec2-with-actions/ec2-elasticip-address-settings.png)

### EC2 インスタンスに Elastic IP を関連付ける

1. Elastic IP 一覧から、割り当てた IP を選択
2. 「Elastic IP アドレスを関連付ける」をクリック
  ![](/images/deploy-app-to-ec2-with-actions/ec2-elasticip-address-detail.png)
3. リソースタイプ：インスタンス
4. 対象の EC2 インスタンスを選択
5. 「関連付ける」 をクリック
  ![](/images/deploy-app-to-ec2-with-actions/ec2-elasticip-address-detail-associate.png)

### 設定後の確認

EC2 インスタンス詳細画面を開き、以下を確認します。

- Elastic IP が表示されている
- 再起動しても IP アドレスが変わらない

これで、固定 IP アドレスの設定は完了です。

![](/images/deploy-app-to-ec2-with-actions/ec2-instance-detail-elasticip.png)

### 料金についての注意点

Elastic IP には、以下の注意点があります。

- EC2 に関連付けられている間は無料
- 未使用（関連付けされていない）状態では課金対象

:::message
不要になった Elastic IP は、必ず解放して下さい。
:::

# GitHub Actions によるデプロイワークフロー

GitHub Actions を利用して、EC2 にアプリケーションをデプロイする設定を行います。
GitHub Actions はデプロイ操作のみを行い、アプリケーションのビルド・実行は EC2 上で行います。

### デプロイの全体像

デプロイの流れは次の通りです。

1. main ブランチに push
2. GitHub Actions が起動
3. .env ファイルを生成
4. rsync で EC2 にファイルを同期
5. EC2 上でビルド・再起動
6. Docker コンテナを更新

### workflow.yml を作成

`.github/workflows` ディレクトリ配下に YAML ファイルを作成します。
GitHub Actions から EC2 へデプロイするワークフローは以下の通りです。
※ 以下は記事用に一部簡略化・修正した例です。

```yaml:workflow.yml
name: Deploy to AWS EC2

on:
  push:
    branches:
      - main  # main ブランチに push されたら実行

jobs:
  deploy:
    runs-on: ubuntu-latest

    # リポジトリのコードを取得
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    # SSH 接続用の秘密鍵を一時的に配置
    - name: Set up SSH key
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
        chmod 600 ~/.ssh/id_rsa

    # Secrets から本番用 .env ファイルを生成
    - name: Create env file
      run: echo "${{ secrets.ENV_PRODUCTION }}" > .env

    # アプリケーションコードを EC2 に同期
    - name: Upload files to EC2
      run: |
        rsync -avz --delete \
          -e "ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no" \
          --exclude '.git' \
          --exclude '.github' \
          . \
          ec2-user@${{ secrets.SSH_HOST }}:/home/ec2-user/app/

    # EC2 上でビルド・再起動を実行
    - name: Build and restart application on EC2
      run: |
        ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no ec2-user@${{ secrets.SSH_HOST }} << 'EOF'
          set -e
          cd /home/ec2-user/app
          npm ci
          npx tsc
          pm2 reload app || pm2 start build/index.js --name app
          docker-compose pull
          docker-compose up -d --remove-orphans
        EOF
```

### 各ステップの解説

- **SSH 接続の準備**
  GitHub Actions から EC2 に接続するため、Secrets に登録した秘密鍵を一時的に配置します。
  この鍵は ジョブ終了後に破棄されるため、EC2 側に秘密情報が残ることはありません。
  ※ Secrets へ秘密情報を登録する方法は後述します。
- **`.env` ファイルの生成**
  本番用の環境変数は GitHub Actions の Secrets から生成します。
  当然ですが、`.env` ファイルには認証情報などがあるため Git 管理しません。
  また、AWS の料金を抑える目的もあり、この方法を採用しています。
  ※ 本来は AWS Secrets Manager などのサービスを使用した方がよいです。
- **rsync によるファイル同期**
  rsync を使用して、アプリケーションコードを EC2 に同期します。
  `.git` や `.github` は除外します。それ以外にも不要なファイルがあれば除外して下さい。
- **EC2 上でのビルドと再起動**
  EC2 上で以下の処理を実行します。
  - 依存関係のインストール（npm ci）
  - TypeScript のコンパイル
  - pm2 プロセスの再起動
  - Docker コンテナの更新

# GitHub Secrets を設定

GitHub Actions から EC2 にデプロイするために必要な GitHub Secrets を設定します。
以下の情報を GitHub Secrets として管理します。

- SSH 秘密鍵
- EC2 の接続先（Elastic IP）
- 本番用環境変数（.env）

### GitHub Secrets の設定画面を開く

1. 対象の GitHub リポジトリを開く
2. `Settings` タブをクリック
3. 左メニューから `Secrets and variables` > `Actions` を選択
4. `New repository secret` をクリック

![](/images/deploy-app-to-ec2-with-actions/github-settings-actions.png)

### GitHub Secrets を設定する

今回、使用する Secrets は以下の3つです。

| 名前 | 用途 |
|---|---|
| SSH_PRIVATE_KEY	| EC2 に接続するための秘密鍵 |
| SSH_HOST | EC2 の Elastic IP |
| ENV_PRODUCTION | 本番用 `.env` の中身 |

![](/images/deploy-app-to-ec2-with-actions/github-repository-secrets.png)

# 実際にデプロイを実行

ここまでの設定でデプロイの準備ができました。
実際に GitHub Actions を使ってデプロイを実行します。

### デプロイのトリガー

今回の構成では、以下をトリガーとしてデプロイが実行されます。
ローカル環境で、変更内容を `main` ブランチに push します。

```bash
git add .
git commit -m "deploy commit"
git push origin main
```

この push をきっかけに、GitHub Actions が自動的に起動します。
特別な操作は不要で、通常の Git 操作だけでデプロイが実行されます。

### GitHub Actions の実行状況を確認

push 後、GitHub リポジトリの Actions タブを開くと、デプロイ用のワークフローが実行されていることを確認できます。
また、各ステップの詳細なログも確認することができます。

YAML ファイルの内容通りに実行されていきます。

- リポジトリのチェックアウト
- SSH 鍵の設定
- rsync によるファイル同期
- EC2 上でのビルド・再起動
- Docker コンテナの更新

エラーが発生した場合は、このログに原因が表示されます。

![](/images/deploy-app-to-ec2-with-actions/github-actions-deploy.png)
※ キャプチャは記事内容と若干異なります。

### デプロイ完了

ワークフローが **Success（緑のチェック）** になっていれば、GitHub Actions 側のデプロイ処理は成功です。

この時点で、EC2 上では以下の処理が完了しています。

- 最新のコードが配置されている
- Node.js アプリケーションがビルドされている
- pm2 によって Discord ボットが起動・更新されている
- docker-compose によるコンテナが起動・更新されている

実際に Discord を確認すると、開発したボットがオンラインになり、ボットの機能が使用できる状態になっていました。
※ Discord ボット自体はお見せできません。
![](/images/deploy-app-to-ec2-with-actions/discord-bot-online.png)

# EC2 再起動時にも Discord ボットを自動起動させる設定

EC2 は以下のようなタイミングで再起動が発生する可能性があります。

- 手動で EC2 を再起動した場合
- AWS 側のメンテナンス
- 何らかの障害や OS 再起動

pm2 の設定を行っていないと Discord ボットは停止したままになってしまいます。
そのため、EC2 起動時に pm2 が自動的に立ち上がり、管理しているプロセスを復元する設定を行います。

### pm2 startup を実行

pm2 には、OS 起動時に自動起動するための startup 機能が存在します。
まず、以下のコマンドを実行します。

```bash
pm2 startup
```

### 表示される sudo コマンドを実行

pm2 startup を実行すると、sudo コマンドが表示されます。
表示された内容をそのままコピーして実行して下さい。
※ 以下は例です。

```bash
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u ec2-user --hp /home/ec2-user
```

### pm2 のプロセス一覧を保存

pm2 で起動しているプロセス情報を保存します。
そうすることで EC2 再起動時、pm2 は以前起動していた Discord ボットを自動で復元します。

```bash
pm2 save
```

### EC2 を再起動して動作確認

設定が完了したら、実際に EC2 を再起動して確認します。

```bash
sudo reboot
```

再起動後、再度 EC2 に接続し、以下を実行します。

```bash
pm2 list
```

Discord ボットが online 状態になっていれば成功です。
これで EC2 再起動後も Discord ボットは自動で立ち上がるようになりました。

:::message
`pm2 startup` を実行しただけでは、プロセスは復元されません。
必ず `pm2 save` を実行して、現在のプロセス一覧を保存して下さい。
:::

# 最後に

今回は、個人開発していた Discord ボットを GitHub Actions から EC2 にデプロイし、常時稼働させるまでの手順についてまとめました。

最初にも記述しましたが、この記事はあくまで作業内容の記録を目的としていますので、あまり親切な内容ではなかったかもしれません。

ですが、EC2 や GitHub Actions について、何かしら参考になる部分があれば嬉しいです。

まだ理解が浅い部分もあるため、もし認識違いや改善できそうな点があれば、教えていただけると助かります。

また、もし需要がありそうであれば、今後 Discord ボットの実装についても記事にしたいと考えています。
