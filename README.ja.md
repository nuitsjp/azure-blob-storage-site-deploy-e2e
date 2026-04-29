> [English](README.md)

# azure-blob-storage-site-deploy-e2e

> **開発者向け**: セットアップ・テスト実行・開発ワークフローについては [devリポジトリ](https://github.com/nuitsjp/azure-blob-storage-site-deploy-dev) を参照してください。

[azure-blob-storage-site-deploy](https://github.com/nuitsjp/azure-blob-storage-site-deploy) のE2Eテスト用リポジトリです。

## ワークフロー構成

`.github/workflows/deploy.yml` が以下のイベントに対応しています。

| イベント | 条件 | 実行ジョブ | デプロイ先プレフィックス |
|---|---|---|---|
| `push` | `main` ブランチへのプッシュ | `deploy` | `main/` |
| `pull_request_target` | opened / synchronize / reopened | `deploy` | `pr-<番号>/` |
| `pull_request_target` | closed | `cleanup` | `pr-<番号>/` を削除 |
| `release` | published（prerelease / draft を除外） | `deploy-release-latest` | `release-latest/` |

## Fork PR 対応

このリポジトリへの Fork からの PR でも `deploy` ジョブが動作し、ステージング環境（`pr-<番号>/`）が自動公開されます。Fork PR では GitHub の仕様により `secrets` が一切露出しないため、`on: pull_request` のままでは Azure OIDC ログインが失敗します。これを解消するため、本ワークフローでは以下の設計を採用しています。

### 基本方針

- `on: pull_request_target` で起動 — secrets / OIDC は **upstream リポジトリのコンテキスト**で利用可能
- Fork のコードは実行しない — `actions/checkout@v4`（無印）で base ブランチをチェックアウトし、`.github/` / `action.yml` / scripts はすべて upstream のものだけを動かす（任意コード実行リスクなし）
- Fork からは静的コンテンツのみ取り込む — Fork PR 時のみ `actions/checkout@v4` で fork の `head.sha` を `__fork__/` に別チェックアウトし、`source_dir` を `__fork__/docs` に切り替えてアップロード
- `persist-credentials: false` — fork 側 checkout に対してクレデンシャル汚染を遮断

### OIDC subject

`pull_request_target` で発行される OIDC トークンの subject は upstream リポジトリ参照（`repo:<owner>/<repo>:pull_request`）になるため、後述の `github-pull-request` 資格情報がそのまま機能します。Fork ごとの追加登録は不要です。

### 残るリスクと運用

- Fork PR で配信される HTML / JS には XSS 等のリスクが残ります。**公開ステージングである前提で PR レビュー時に内容を確認**してください
- 現状は信頼チェック（`author_association` ゲート / environment required reviewers）を入れていません。外部コントリビューターを広く受け入れる場合は workflow 側に追加してください

## Azure OIDC 設定

GitHub Actions から Azure にログインするため、Azure Entra ID のフェデレーション資格情報（Federated Identity Credential）を以下の3件登録します。

| 名前 | Subject | 用途 |
|---|---|---|
| `github-main-branch` | `repo:<owner>/<repo>:ref:refs/heads/main` | mainブランチへのpush |
| `github-pull-request` | `repo:<owner>/<repo>:pull_request` | PR / Fork PR イベント |
| `github-environment-production` | `repo:<owner>/<repo>:environment:production` | GitHub Release公開時 |

### `environment: production` を使う理由

`release` イベントのOIDCトークンのsubjectは `repo:...:ref:refs/tags/<タグ名>` となります。タグ名ごとに資格情報を登録すると運用が煩雑になるため、`deploy-release-latest` ジョブに `environment: production` を設定しています。これによりsubjectが `repo:...:environment:production` に固定され、1件の登録で全リリースに対応できます。
