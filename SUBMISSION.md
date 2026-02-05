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
