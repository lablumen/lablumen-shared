# lablumen-shared

Centralized GitHub Actions workflows shared across all LabLumen service repositories. CI/CD logic lives here once — each service repo calls these workflows as thin wrappers. Updating a workflow here propagates to all services on their next run.

This repository must be accessible to all service repos (public or granted org-level Actions access).

---

## Workflows

All three workflows use `on: workflow_call` and cannot be triggered directly — they are called by individual service repos.

---

### `service-pr.yml` — Pull Request Gate

Runs on every pull request. All four jobs must pass before a PR can be merged.

| Job | What It Runs |
|---|---|
| `lint-and-test` | Python: `ruff check` + `pytest` / Node: `npm ci` + `npm run build` |
| `sast` | SonarCloud static analysis — code security vulnerabilities, quality gate |
| `sca` | Snyk software composition analysis — known CVEs in `requirements.txt` or `package-lock.json` |
| `container-scan` | Builds the Docker image (not pushed), Trivy scans all layers, fails on CRITICAL/HIGH |

`sast`, `sca`, and `container-scan` run in parallel after `lint-and-test` succeeds.

**Inputs:** `service-name`, `service-path`, `runtime` (`python` or `node`), `sonar-organization`, `sonar-sources`  
**Secrets required:** `SONAR_TOKEN`, `SNYK_TOKEN`

---

### `service-build-push.yml` — Dev Deploy

Runs when a PR is merged to `main`.

1. Builds the Docker image tagged with the 7-character git SHA (e.g., `abc1234`).
2. Runs a Trivy gate — the image is **not pushed** if CRITICAL/HIGH vulnerabilities are found.
3. Pushes the image to ECR using GitHub OIDC (no static AWS credentials).
4. Checks out `lablumen-k8s` and writes the new SHA tag into `services/<name>/values-dev.yaml`.
5. Uses a `git pull --rebase` retry loop (up to 5 attempts) to handle concurrent service builds writing to the same repo.
6. ArgoCD detects the Git change and rolls out the new image to the `lablumen-dev` namespace.

**Inputs:** `service-name`, `service-path`, `ecr-repository`, `role-to-assume`, `aws-region`, `k8s-repo`  
**Secrets required:** `K8S_REPO_PAT` (PAT with write access to `lablumen-k8s`)  
**Outputs:** `image-tag` (the 7-character SHA)

---

### `service-release.yml` — Production Promotion

Runs when a GitHub Release is published on a service repo.

1. Retrieves the ECR image manifest for the SHA tag that was deployed to dev.
2. Creates a new ECR tag with the release semver (e.g., `v1.2.0`) pointing to the same image layers — **no rebuild**.
3. Writes the semver tag into `services/<name>/values-prod.yaml` in `lablumen-k8s`.
4. ArgoCD detects the change and rolls out to the `lablumen` (production) namespace.

**Inputs:** `service-name`, `ecr-repository`, `role-to-assume`, `release-tag`, `aws-region`, `k8s-repo`  
**Secrets required:** `K8S_REPO_PAT`

---

## How Service Repos Call These Workflows

Each service has a single `.github/workflows/ci.yml` that delegates to the right workflow based on the event:

```yaml
jobs:
  pr:
    if: github.event_name == 'pull_request'
    uses: lablumen/lablumen-shared/.github/workflows/service-pr.yml@main
    with:
      service-name: appointment-service
      service-path: .
      sonar-organization: ${{ vars.SONAR_ORGANIZATION }}
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  build:
    if: github.event_name == 'push'
    uses: lablumen/lablumen-shared/.github/workflows/service-build-push.yml@main
    with:
      service-name: appointment-service
      ecr-repository: lablumen/appointment-service
      role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/lablumen-app-ci-ecr
    secrets:
      K8S_REPO_PAT: ${{ secrets.K8S_REPO_PAT }}

  release:
    if: github.event_name == 'release'
    uses: lablumen/lablumen-shared/.github/workflows/service-release.yml@main
    with:
      service-name: appointment-service
      ecr-repository: lablumen/appointment-service
      role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/lablumen-app-ci-ecr
      release-tag: ${{ github.event.release.tag_name }}
    secrets:
      K8S_REPO_PAT: ${{ secrets.K8S_REPO_PAT }}
```

The `if:` conditions ensure only one job runs per event trigger.

---

## Security Design

**No static AWS credentials.** All AWS access uses GitHub OIDC — each workflow job exchanges a GitHub-signed JWT for temporary IAM credentials that expire after 1 hour.

**Least privilege IAM roles.** Separate roles exist for different purposes:
- `lablumen-app-ci-ecr` — ECR push for backend services
- `lablumen-frontend-build` — ECR push for the frontend only
- `lablumen-ai-lambda-deploy` — SAM deploy for the Lambda

**Build-once, promote.** The same Docker image tested in dev is promoted to production by retagging its ECR manifest — not by rebuilding. What was tested is exactly what runs in production.

---

## Required GitHub Configuration

| Setting | Where | Purpose |
|---|---|---|
| Variable `AWS_ACCOUNT_ID` | Org or repo | Used to construct IAM role ARNs |
| Variable `SONAR_ORGANIZATION` | Org or repo | SonarCloud organization key |
| Secret `K8S_REPO_PAT` | Org or repo | PAT with write access to `lablumen-k8s` |
| Secret `SONAR_TOKEN` | Org or repo | SonarCloud API token |
| Secret `SNYK_TOKEN` | Org or repo | Snyk API token |
