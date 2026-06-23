# lablumen-shared

Centralized **reusable GitHub Actions workflows** for the LabLumen platform — the single source of
truth for CI/CD logic. Consuming repos (`lablumen-app`) call these as thin callers via `uses:`.

> Make this repo **public** (or grant org Actions access) so other repos can `uses:` these workflows.

## Workflows (all `on: workflow_call`)

| Workflow | Purpose | Key inputs | Secrets | Outputs |
|---|---|---|---|---|
| `service-pr.yml` | PR gate for one backend service: ruff + pytest + **SonarCloud** (SAST) + **Snyk** (SCA) + **Trivy** (container scan, not pushed). Hard-fail. | `service-name`, `service-path`, `sonar-organization` | `SONAR_TOKEN`, `SNYK_TOKEN` | — |
| `service-build-push.yml` | Build one service image tagged with the 7-char git SHA, **Trivy gate**, push to ECR via OIDC. Registry host derived from the ECR login (account-portable). No git write-back. | `service-name`, `service-path`, `ecr-repository`, `role-to-assume` | — (OIDC) | `image-tag` |
| `frontend-deploy.yml` | Build the Vite SPA with prod config, `s3 sync`, CloudFront invalidation via OIDC. | `role-to-assume`, `bucket`, `distribution-id`, `vite-*` | — (OIDC) | — |

## Pattern
- **Selective builds:** the caller (`lablumen-app/ci.yml`) uses `dorny/paths-filter` to detect which
  `backend/<svc>/**` changed, and only calls `service-pr` / `service-build-push` for those.
- **Unified CD aggregation:** the write-back is **NOT** in `service-build-push`. The caller has a
  single `cd-dev` job (`needs:` all build callers, `if: always()`) that reads each build's
  `image-tag` output, `yq`-bumps only the services that ran, and does **one** commit to
  `lablumen-k8s` → ArgoCD syncs. No git race.
- **Refs:** callers pin `@main`.
# lablumen-shared
