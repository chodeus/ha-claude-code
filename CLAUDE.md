# Claude Code Instructions

This file contains instructions for Claude Code when working on this repository.

## Before Every Commit

**IMPORTANT:** Update `claudecode/CHANGELOG.md` with the changes being committed before making any commit. Follow the existing format:

```markdown
## [VERSION] - YYYY-MM-DD

### Added/Changed/Fixed
- Description of change
```

## Project Structure

- `repository.yaml` - Add-on repository metadata
- `renovate.json` - Dependency automation (regex manager reads `build.yaml` `build_from`)
- `.github/workflows/builder.yaml` - Lints + publishes GHCR images. Its three jobs
  (`Lint add-on`, `Build amd64`, `Build aarch64`) are the required status checks on `main`,
  so the `pull_request` trigger must stay **unfiltered** — a path-filtered workflow never
  reports a required context and leaves such PRs blocked forever. The `push` trigger keeps
  its path filter to save CI minutes; `skip-existing` is what stops a re-push. Don't rename
  these jobs without re-registering the branch-protection contexts.
- `.github/workflows/image-refresh.yml` - Weekly: chodeus-ops' image-refresh check on
  `claudecode-amd64:latest`; dispatches `refresh-release.yml` when Alpine packages are pending
- `.github/workflows/refresh-release.yml` - Bumps the patch version + CHANGELOG and pushes to `main`
  with the `RELEASE_TOKEN` secret, which starts the Builder workflow
- `claudecode/` - Claude Code add-on
  - `config.yaml` - Add-on configuration (bump version here)
  - `Dockerfile` - Container build instructions
  - `build.yaml` - Multi-architecture build settings
  - `run.sh` - Startup script (copied to `/usr/local/bin/start-addon.sh`)
  - `rootfs/` - Files COPY'd into the image (dotfiles, ttyd page snippets)
  - `translations/` - Config UI strings; en/es/fr/pt-BR must stay key-identical
  - `README.md` - User documentation (rendered as the add-on's Documentation tab)
  - `CHANGELOG.md` - Version history (**update before commits**)
  - `apparmor.txt` - Security profile
- `playwright-browser/` - Optional headless Chromium add-on (own `config.yaml`/`CHANGELOG.md`;
  not built by CI — users build it locally)

## Version Bumping

When making changes that require a new release:
1. Update version in `claudecode/config.yaml`
2. Add entry to `claudecode/CHANGELOG.md`
3. Commit and push

## Home Assistant Add-on Notes

- Rebuild button only rebuilds from cached config
- To pick up config.yaml changes: uninstall/reinstall or bump version and update
- Base images use s6-overlay v3 - be careful with init configuration
- `init: true` uses Docker's tini, `init: false` uses s6-overlay's /init
- **`build_from` must be a two-segment image path.** The Supervisor validates it against
  `^([a-zA-Z\-\.:\d{}]+/)*?([\-\w{}]+)/([\-\w{}]+)(:[\.\-\w{}]+)?$`, so `ghcr.io/home-assistant/base`
  passes but a single-segment repo like `mcr.microsoft.com/playwright` does **not**. On a rejected
  value the Supervisor logs one warning and silently substitutes `ghcr.io/home-assistant/base:latest`
  — the build then fails much later on a distro mismatch (e.g. `apt-get: not found` on Alpine).
  Hardcode `FROM` in the Dockerfile for such images instead of using `build.yaml`.
- **CI builds with `home-assistant/builder/actions/build-image`**, HA's replacement for the deprecated
  `home-assistant/builder` action. It reads nothing from `config.yaml` or `build.yaml`, so the
  `Lint add-on` job passes the version, the `io.hass.*` labels and each arch's `build_from`
  (`BUILD_ARCH` and `BUILD_VERSION` it sets itself). It builds natively, so `aarch64` runs on
  `ubuntu-24.04-arm`. `skip-existing` keeps an already-published version from being re-pushed, so
  a change that should reach users needs a version bump: `Lint add-on` fails a PR (Renovate's
  excepted) that changes image files under an already-published version. The workflow's
  `concurrency` uses `queue: max`, which actionlint doesn't know (`.github/actionlint.yaml`
  ignores it). Every action is SHA-pinned; keep it so.
- **Refresh releases need the `RELEASE_TOKEN` secret**: a fine-grained PAT for this repo with
  Contents: Read and write. The Supervisor only updates an add-on on a new version, so a rebuild
  under the same version never reaches users — `refresh-release.yml` bumps the patch version instead.
