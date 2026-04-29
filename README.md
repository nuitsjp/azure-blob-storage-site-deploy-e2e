> [日本語版](README.ja.md)

# azure-blob-storage-site-deploy-e2e

> **For developers**: See the [dev repository](https://github.com/nuitsjp/azure-blob-storage-site-deploy-dev) for setup, test execution, and development workflow details.

An E2E test repository for [azure-blob-storage-site-deploy](https://github.com/nuitsjp/azure-blob-storage-site-deploy).

## Workflow structure

`.github/workflows/deploy.yml` handles the following events:

| Event | Condition | Job | Deployed prefix |
|---|---|---|---|
| `push` | Push to `main` branch | `deploy` | `main/` |
| `pull_request_target` | opened / synchronize / reopened | `deploy` | `pr-<number>/` |
| `pull_request_target` | closed | `cleanup` | Deletes `pr-<number>/` |
| `release` | published (pre-release / draft excluded) | `deploy-release-latest` | `release-latest/` |

## Fork PR support

This repository supports `deploy` jobs for PRs from forks: a staging environment (`pr-<number>/`) is automatically published. Because GitHub does not expose `secrets` to fork PRs, a plain `on: pull_request` workflow would fail at the Azure OIDC login step. The workflow uses the following design to address this.

### Design

- Triggered by `on: pull_request_target` — secrets / OIDC are available **in the context of the upstream repository**
- Fork code is never executed — `actions/checkout@v4` (default) checks out the base branch, and all of `.github/`, `action.yml`, and scripts run from the upstream side only (no arbitrary code execution risk)
- Only static content is taken from the fork — only when handling a fork PR, `actions/checkout@v4` checks out the fork's `head.sha` into `__fork__/` and `source_dir` is switched to `__fork__/docs` for upload
- `persist-credentials: false` — prevents credential leakage from the fork-side checkout

### OIDC subject

The OIDC token issued under `pull_request_target` has its subject scoped to the upstream repository (`repo:<owner>/<repo>:pull_request`), so the `github-pull-request` credential below works as-is. No per-fork credential registration is required.

### Remaining risks and operations

- HTML / JS served from a fork PR can still carry XSS or similar risks. **Review the content during PR review**, on the assumption that the staging environment is publicly accessible
- Trust checks (`author_association` gating / environment-required reviewers) are not currently configured. Add them in the workflow if you intend to broadly accept external contributors

## Azure OIDC setup

Three federated identity credentials are required in Azure Entra ID to allow GitHub Actions to authenticate:

| Name | Subject | Purpose |
|---|---|---|
| `github-main-branch` | `repo:<owner>/<repo>:ref:refs/heads/main` | Push to main branch |
| `github-pull-request` | `repo:<owner>/<repo>:pull_request` | Pull request / fork PR events |
| `github-environment-production` | `repo:<owner>/<repo>:environment:production` | GitHub Release published |

### Why `environment: production`

With the `release` event, the OIDC token subject becomes `repo:...:ref:refs/tags/<tag-name>`. Registering a new credential for every tag is operationally expensive. By setting `environment: production` on the `deploy-release-latest` job, the subject is fixed to `repo:...:environment:production`, so a single credential registration covers all future releases.
