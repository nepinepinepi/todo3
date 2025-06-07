# TodoApp — Serverless CRUD on AWS

![License](https://img.shields.io/github/license/your-org/todo-app)

> **完全オフラインでテスト可能**なサーバーレス TODO 管理アプリケーションです。バックエンドは AWS のマネージドサービス、フロントエンドは Vuetify SPA で構築しています。

---

## 📑 目次

1. [概要](#概要)
2. [アーキテクチャ](#アーキテクチャ)
3. [技術スタック](#技術スタック)
4. [前提条件](#前提条件)
5. [ローカル開発 (オフライン)](#ローカル開発-オフライン)
6. [自動テスト](#自動テスト)
7. [デプロイ](#デプロイ)
8. [ディレクトリ構成](#ディレクトリ構成)
9. [環境変数](#環境変数)
10. [ライセンス](#ライセンス)
11. [参考](#参考)

---

## 概要

このリポジトリは、サーバーレス構成で TODO の **Create / Read / Update / Delete** を提供するアプリケーションです。以下を主な目的としています。

* スケーラブルで低コストな本番環境デプロイ
* **クラウド通信なしで完結**するローカルテスト環境

## アーキテクチャ

```
┌─────────────┐
│  Browser    │
│ (Vue SPA)   │
└─────┬───────┘
      │ CDN
      ▼
┌─────────────┐           ┌──────────────────────┐
│ CloudFront  │◀─────────▶│   S3 (Static SPA)    │
└─────┬───────┘           └──────────────────────┘
      │ HTTPS (api.todo.example.com)
      ▼
┌─────────────┐    Lambda Proxy   ┌──────────────┐
│ API Gateway │ ────────────────▶ │   Lambda     │
└─────────────┘                   └──────┬───────┘
                                         │ AWS SDK
                                         ▼
                                   ┌──────────────┐
                                   │  DynamoDB    │
                                   └──────────────┘
```

* **ドメイン管理**: Route 53 で `todo.example.com` を管理。
* **SPA ホスティング**: S3 + CloudFront (HTTPS, OAI) で配信。
* **API**: API Gateway (HTTP API) + Lambda (Node.js 20)。
* **データストア**: DynamoDB (単一テーブル設計)。

## 技術スタック

| レイヤ     | 技術                                  | 補足                           |
| ------- | ----------------------------------- | ---------------------------- |
| インフラ    | AWS SAM CLI / CloudFormation        | IaC; SAM テンプレートは `infra/` 配下 |
| ローカルモック | LocalStack 4.3, DynamoDB Local 2.x  | Docker で自動起動                 |
| バックエンド  | Node.js 20 / TypeScript, AWS Lambda | api/\*                       |
| フロント    | Vue 3, Vite, Vuetify 3              | frontend/\*                  |
| テスト     | Jest + supertest (unit/integration) | backend                      |
|         | Playwright (e2e)                    | SPA 対応                       |
|         | Vitest (frontend unit)              |                              |

## 前提条件

* **Node.js** >= 20
* **Docker** >= 24  (compose v2 同梱)
* **AWS SAM CLI** >= 1.122  (`brew install aws/tap/aws-sam-cli`)
* **AWS CLI** (deploy 時のみ)

> 📝 **AWS 資格情報は不要**: ローカルテストは 100% オフラインです。

## ローカル開発 (オフライン)

```bash
# 1. リポジトリを取得
$ git clone https://github.com/your-org/todo-app.git
$ cd todo-app

# 2. 依存関係をインストール
$ make bootstrap       # または `npm ci && cd frontend && npm ci`

# 3. モックサービスを起動 (LocalStack + DynamoDB Local)
$ make up              # docker compose up -d

# 4. Lambda/API をローカルで起動
$ make start-api       # sam local start-api --template infra/template.yaml

# 5. SPA を起動
$ cd frontend && npm run dev

# ブラウザで http://localhost:5173 にアクセス
```

> **LocalStack 4.x** は API Gateway / Lambda / Route 53 をフルサポートし、
> DynamoDB Local は Java 17 対応の 2.x 系を使用しています。

### docker-compose.yml 抜粋

```yaml
version: "3.8"
services:
  localstack:
    image: localstack/localstack:4.3
    ports:
      - "4566:4566"           # Edge Port
      - "53:53"/udp           # Route53
    environment:
      - SERVICES=lambda,dynamodb,apigateway,route53
      - DEBUG=0
  dynamodb-local:
    image: public.ecr.aws/aws-dynamodb-local/dynamodb-local:latest
    command: -jar DynamoDBLocal.jar -inMemory -sharedDb
    ports:
      - "8000:8000"
```

## 自動テスト

| レベル   | 実行コマンド                     | 内容                                     |
| ----- | -------------------------- | -------------------------------------- |
| Unit  | `npm run test:unit`        | pure function/unit test (jest, vitest) |
| Intg. | `npm run test:int`         | LocalStack + supertest 経由で API 叩く      |
| e2e   | `npm run test:e2e`         | Playwright; SPA + API をローカル統合          |
| 全体    | `npm test` または `make test` | 上記すべて                                  |

* 各ジョブは **クラウドへ通信しません**。LocalStack & DynamoDB Local が提供するエンドポイント (`http://localhost:4566`, `http://localhost:8000`) に対して実行します。
* Lambda ハンドラは `sam build` で transpile → `sam local invoke` でユニット + 依存リソース分離テスト。

## デプロイ

```bash
# 初回のみスタック名とリージョンを対話入力
sam deploy -g -t infra/template.yaml
```

* デプロイ後、`sam sync --watch` でホットデプロイ可。
* DNS: Route 53 で `api.todo.example.com` レコードを CloudFront/API GW へ CNAME/Alias。

## ディレクトリ構成

```
.
├── backend/          # Lambda ソース (TypeScript)
├── frontend/         # Vue 3 + Vite + Vuetify
├── infra/            # SAM テンプレート & IaC
├── tests/            # unit / integration / e2e
├── docker-compose.yml
├── Makefile          # 開発用ショートカット
└── README.md
```

## 環境変数

| 変数名                 | 用途                     | デフォルト                   |
| ------------------- | ---------------------- | ----------------------- |
| `AWS_ENDPOINT_URL`  | LocalStack エンドポイント     | `http://localhost:4566` |
| `DYNAMODB_ENDPOINT` | DynamoDB Local URL     | `http://localhost:8000` |
| `TABLE_NAME`        | DynamoDB テーブル名         | `Todos`                 |
| `STAGE`             | 環境識別子 (dev/stage/prod) | `dev`                   |

## ライセンス

MIT © 2025 Your Name

## 参考

* LocalStack [https://github.com/localstack/localstack](https://github.com/localstack/localstack)
* AWS SAM CLI [https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/using-sam-cli-local.html](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/using-sam-cli-local.html)
* DynamoDB Local [https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.DownloadingAndRunning.html](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.DownloadingAndRunning.html)
