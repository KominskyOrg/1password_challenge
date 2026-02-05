# Submission Notes

Decisions and trade-offs for each feature of the CI/CD pipeline.

---

## Feature 1 — Build and Bundle

Added `.github/workflows/build.yml` — a GitHub Actions workflow that installs dependencies, builds the project, and uploads the output as a CI artifact.

**Decisions:**
- Used `yarn install --frozen-lockfile` for deterministic installs (project already uses yarn)
- Pinned Node 18 (LTS, compatible with Electron 21)
- Archived `dist/` as `.tar.gz` — standard for Linux CI, preserves permissions
- Set `permissions: contents: read` for least-privilege
- 14-day artifact retention as a sensible default

**Multi-OS matrix:** workflow is structured to support ubuntu, macOS, and Windows, but only ubuntu is active. macOS and Windows runners are commented out to avoid cost — GitHub charges 10x and 2x respectively. The matrix is ready to enable when needed.

**Branch strategy:**
- `main` push → full build, 14-day artifact retention
- `dev` push → full build, 7-day retention
- PRs to main → full build, 3-day retention (validation gate)

**Trade-offs:**
- No runtime validation yet — this only proves it compiles cleanly

---

## Feature 2 — Build Notification

Added an `if: always()` step that writes a structured summary to the GitHub Actions job summary page, plus a mock notification payload printed to the build log.

**Decisions:**
- Used `$GITHUB_STEP_SUMMARY` — native, no external credentials, persistent and auditable
- Summary includes status, commit SHA, ref, runner OS, and a direct link to the run
- Mock JSON payload demonstrates where a real Slack/webhook integration would plug in
- Both steps run on success and failure via `if: always()`

**Trade-offs:**
- Not a real alerting system — no Slack, email, or PagerDuty integration
- The mock payload is printed to stdout, not sent anywhere. In production this would be a `curl` to a webhook URL stored in GitHub Secrets

---

## Feature 3 — Vulnerability Detection and Gating

Added a `audit` job that runs `npm audit --audit-level=high` in parallel with the build job.

**Decisions:**
- Used `npm audit` over `yarn audit` — cleaner severity gating (`--audit-level` flag vs yarn's bitmask exit codes)
- Gates on `high` and `critical` — these represent unacceptable risk
- Runs as a separate parallel job — audit failure doesn't block artifact creation, but both must pass for the workflow to be green
- No install step needed — `npm audit` reads directly from the committed `package-lock.json`

**Trade-offs:**
- This project's 2023-era deps have 14 high and 2 critical vulns, so the audit job will correctly fail. That's the gate working as intended.
- `npm audit` can be noisy. A production pipeline would add SBOM generation and policy-based exceptions for accepted risks.
