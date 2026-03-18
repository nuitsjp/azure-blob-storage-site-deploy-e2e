> [English](README.md)

# azure-blob-storage-site-deploy-e2e

> **開発者向け**: セットアップ・テスト実行・開発ワークフローについては [devリポジトリ](https://github.com/nuitsjp/azure-blob-storage-site-deploy-dev) を参照してください。

[azure-blob-storage-site-deploy](https://github.com/nuitsjp/azure-blob-storage-site-deploy) のE2Eテスト用リポジトリです。

## ワークフロー構成

`.github/workflows/deploy.yml` が以下のイベントに対応しています。

| イベント | 条件 | 実行ジョブ | デプロイ先プレフィックス |
|---|---|---|---|
| `push` | `main` ブランチへのプッシュ | `deploy` | `main/` |
| `pull_request` | opened / synchronize / reopened | `deploy` | `pr-<番号>/` |
| `pull_request` | closed | `cleanup` | `pr-<番号>/` を削除 |
| `release` | published（prerelease / draft を除外） | `deploy-release-latest` | `release-latest/` |

## Azure OIDC 設定

GitHub Actions から Azure にログインするため、Azure Entra ID のフェデレーション資格情報（Federated Identity Credential）を以下の3件登録します。

| 名前 | Subject | 用途 |
|---|---|---|
| `github-main-branch` | `repo:<owner>/<repo>:ref:refs/heads/main` | mainブランチへのpush |
| `github-pull-request` | `repo:<owner>/<repo>:pull_request` | PRイベント |
| `github-environment-production` | `repo:<owner>/<repo>:environment:production` | GitHub Release公開時 |

### `environment: production` を使う理由

`release` イベントのOIDCトークンのsubjectは `repo:...:ref:refs/tags/<タグ名>` となります。タグ名ごとに資格情報を登録すると運用が煩雑になるため、`deploy-release-latest` ジョブに `environment: production` を設定しています。これによりsubjectが `repo:...:environment:production` に固定され、1件の登録で全リリースに対応できます。
