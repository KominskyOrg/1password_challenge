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

Added a `security` job that runs dependency auditing and Electron source code scanning in parallel with the build job.

**Decisions:**

- Combined `npm audit` and `yarn electronegativity` into a single job — both are security checks, both are fast, avoids duplicating checkout/setup
- `npm audit --audit-level=high` gates on dependency CVEs
- `yarn electronegativity` scans source for Electron-specific security misconfigurations (insecure BrowserWindow options, disabled context isolation, etc.)
- Both tools were already available in the project's package.json

**Trade-offs:**

- This project's 2023-era deps have 14 high and 2 critical vulns, so the audit step will correctly fail. That's the gate working as intended.
- `npm audit` can be noisy. A production pipeline would add SBOM generation and policy-based exceptions for accepted risks.

---

## Feature 4 — Package the Application

Added a `package` job that produces a Linux AppImage using `electron-builder`.

**Decisions:**

- Targeted Linux only — avoids code signing and notarization complexity (macOS/Windows)
- AppImage format — portable, no install required, standard for Linux Electron apps
- `needs: build` — packaging runs after the build job succeeds, establishing a clear pipeline order
- Used `--publish never` (already in the project's `package` script) — no auto-publishing to GitHub Releases

**Trade-offs:**

- macOS (.dmg) and Windows (.exe) packaging would need paid runners and signing infrastructure

---

## Feature 5 — Containerized Build Environment

Added `Dockerfile.ci` and converted the package job to build and run inside a container.

**Decisions:**

- Based on `node:18.20-bullseye-slim` — pinned tag for reproducibility, slim to reduce image size
- Installs `fakeroot` and `dpkg` — system deps needed by electron-builder for Linux packaging
- Dockerfile copies `package.json` + `yarn.lock` first for layer caching, then source
- Package job does `docker build` then `docker run`, mounting only the output directory back to the host
- Applied to the package job specifically — it has the most system-level dependencies

**Why two builds instead of one?**

The build job compiles TypeScript on the runner directly. The package job rebuilds inside a container. This means `tsc` runs twice. In a production pipeline, the container image would be pushed to a registry (e.g. ECR) after the build step, then pulled by downstream jobs — one build, many consumers. A simulated registry push step is included to demonstrate where this would happen.

For this project, `tsc` takes seconds, so the simplicity of a self-contained package job outweighs the complexity of wiring up a registry. The Dockerfile and registry simulation show the intent and the path to get there.

**Trade-offs:**

- Adds build time for the `docker build` step (offset by layer caching on repeat runs)
- No actual registry configured — simulated push shows where ECR/GHCR would plug in
- Local developers can use the same Dockerfile to reproduce CI builds exactly

---

## Feature 6 — Future Improvements

Prioritized by impact to a CI platform team supporting multiple engineering teams at scale.

**Secrets management:**

- Scope secrets per job, not per workflow — each job only gets the credentials it needs
- Use GitHub's OIDC provider for cloud auth (AWS, GCP) instead of long-lived credentials — short-lived tokens with no stored secrets

**Build:**

- Push the container image to a registry (simulated in Feature 5) and tag by commit SHA for traceability

**Observability and performance:**

- Instrument step-level timing and surface it in job summaries
- Track cache hit rates to validate caching is actually helping
- Feed CI metrics into a dashboard (Datadog, Grafana) to spot flaky tests, queue times, and runner saturation

**Self-hosted runners and scaling:**

- Move to ARC when job volume or security requirements outgrow GitHub-hosted runners — cost control, network isolation, and custom toolchains justify the operational overhead

**Quality gates:**

- Add linting to CI — the project has an eslint script but is missing `.eslintrc`
- Smoke test the packaged AppImage headlessly
- Split PR and release workflows — PRs run build + security only, releases add packaging and signing

**Developer experience:**

- One-command local build via the CI Dockerfile
- Reusable workflows and composite actions to reduce duplication across repos
