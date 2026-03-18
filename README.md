> [日本語版](README.ja.md)

# azure-blob-storage-site-deploy-e2e

> **For developers**: See the [dev repository](https://github.com/nuitsjp/azure-blob-storage-site-deploy-dev) for setup, test execution, and development workflow details.

An E2E test repository for [azure-blob-storage-site-deploy](https://github.com/nuitsjp/azure-blob-storage-site-deploy).

## Workflow structure

`.github/workflows/deploy.yml` handles the following events:

| Event | Condition | Job | Deployed prefix |
|---|---|---|---|
| `push` | Push to `main` branch | `deploy` | `main/` |
| `pull_request` | opened / synchronize / reopened | `deploy` | `pr-<number>/` |
| `pull_request` | closed | `cleanup` | Deletes `pr-<number>/` |
| `release` | published (pre-release / draft excluded) | `deploy-release-latest` | `release-latest/` |

## Azure OIDC setup

Three federated identity credentials are required in Azure Entra ID to allow GitHub Actions to authenticate:

| Name | Subject | Purpose |
|---|---|---|
| `github-main-branch` | `repo:<owner>/<repo>:ref:refs/heads/main` | Push to main branch |
| `github-pull-request` | `repo:<owner>/<repo>:pull_request` | Pull request events |
| `github-environment-production` | `repo:<owner>/<repo>:environment:production` | GitHub Release published |

### Why `environment: production`

With the `release` event, the OIDC token subject becomes `repo:...:ref:refs/tags/<tag-name>`. Registering a new credential for every tag is operationally expensive. By setting `environment: production` on the `deploy-release-latest` job, the subject is fixed to `repo:...:environment:production`, so a single credential registration covers all future releases.
