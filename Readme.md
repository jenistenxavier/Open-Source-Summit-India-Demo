# Open-Source-Summit-India-Demo

Secure-by-default **reusable GitHub Actions platform**. App repos inherit a full
CI → release chain with one `uses:` line each. Every security gate is a
**required, fail-hard** job — security is the workflow, not an optional step.

> All reusable workflows and composite actions are referenced as
> `jenistenxavier/Open-Source-Summit-India-Demo/...@main`.

## Pipeline at a glance

```
PR / push ─► reusable-ci ─► (on main, green) ─► reusable-release
            secret/dep/SAST/        semantic-release →
            SBOM/tests +            build → Trivy → push → cosign
            DevOps agent            (image published to GHCR)
```

| Workflow | Purpose | Trigger |
|----------|---------|---------|
| `reusable-ci.yml` | Security + quality gate chain | every PR / push |
| `reusable-release.yml` | Version, build, **scan**, sign, push to GHCR | after CI green on `main` |
| `reusable-devops-agent.yml` | AI version-bump / debug / Sonar self-heal | standalone or from CI |

## reusable-ci.yml — gate chain

| Gate | Tool | Fails on |
|------|------|----------|
| 1. Secret scan | GitLeaks | any committed secret |
| 2. Dependency CVEs | **OSV-Scanner** (PR-diff) | **only vulns the PR introduces** (full scan on push) |
| 3. SAST + quality | SonarQube CE | quality-gate breach |
| 4. SBOM | CycloneDX (cdxgen) → Dependency-Track | — (publishes SBOM) |
| Tests | npm / gradle / php | unit test or coverage failure |
| Quality gate | aggregate | any gate above red |
| DevOps agent | routed by which gate failed | — (auto-heal) |

**Dependency scanning = OSV-Scanner only.** OWASP Dependency-Check was removed —
OSV's PR-diff gating (new-vulns-only, pre-existing debt reported not blocked) is
strictly better for PR ergonomics. Requires the calling checkout to use
`fetch-depth: 0` so the base ref is reachable for the diff.

**No container scan here — by design.** CI never builds the deployable image, so
scanning in CI would scan a throwaway artifact ≠ what ships → false confidence.
Container image scanning lives in `reusable-release.yml` (below).

### Usage

```yaml
jobs:
  ci:
    uses: jenistenxavier/Open-Source-Summit-India-Demo/.github/workflows/reusable-ci.yml@main
    with:
      language: node            # node | java | php
      project-key: inventory-service
    secrets: inherit            # SONAR_TOKEN, DEPENDENCY_TRACK_API_KEY, ANTHROPIC_API_KEY
```

## reusable-release.yml — build, scan, ship

semantic-release (Conventional Commits) → **build → Trivy gate → push** to GHCR →
cosign sign. The released, signed image lands in GHCR — deployment is out of scope
for this demo.

**Container scan is here, fail-closed:**

```
build (load: true, no push) ─► Trivy (HIGH/CRITICAL, ignore-unfixed)
                                  │ pass            │ fail
                                  ▼                 ▼
                             push to GHCR      job aborts — nothing pushed
```

The image is built into the local daemon first, Trivy scans the **exact bytes**
that will ship, and the push only runs if the scan passes. A vulnerable image
never reaches GHCR. This is the "scan before publish" gate.
Trivy here covers OS packages + base-image layers; lockfile CVEs are covered by
CI's OSV gate.

```yaml
jobs:
  release:
    needs: ci
    uses: jenistenxavier/Open-Source-Summit-India-Demo/.github/workflows/reusable-release.yml@main
    with: { image-name: inventory-service }
    secrets: inherit
```

## reusable-devops-agent.yml — AI DevOps agent

One job, task routed by failure: `sonar-heal` (SAST red) / `debug` (tests red) /
`version-bump` (all green on `main`). Agent code lives in
`jenistenxavier/DevOps-Agent-demo`, checked out at runtime; it edits, commits, and
pushes to the triggering branch with `GITHUB_TOKEN` (which does not retrigger CI →
no loop). Embedded in `reusable-ci.yml` and also callable standalone.

## Composite actions

- `.github/actions/osv-scan` — OSV-Scanner with PR-diff gating (new vulns only on
  PRs, full scan on push, skip on PR close). Diff mode needs `fetch-depth: 0`.
- `.github/actions/sonar-scan` — SonarQube CE scan + quality gate.

## Required secrets

| Secret | Used by | Required |
|--------|---------|----------|
| `SONAR_TOKEN` | CI (SAST), agent | when SAST/heal runs |
| `DEPENDENCY_TRACK_API_KEY` | CI (SBOM upload) | optional |
| `ANTHROPIC_API_KEY` | DevOps agent | when agent runs |
| `GITLEAKS_LICENSE` | CI (secret scan) | optional (OSS = none) |
