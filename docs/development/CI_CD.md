# CI/CD – GitHub Actions Pipeline for the Monorepo

> **Tác giả:** Agent 14 (DevSecOps & Infrastructure) · **Phase:** 0 · **Trạng thái:** Draft v0.1 · **Ngày:** 2026-10-30
>
> Liên quan: `docs/development/ENVIRONMENT_SETUP.md`, `docs/development/DEPLOYMENT.md`, `docs/operations/DEVSECOPS_GUIDE.md`, `docs/architecture/INFRASTRUCTURE_DESIGN.md`, `docs/SECURITY.md` (A09).
>
> Nhãn: **[Đã xác minh]** / **[Đề xuất]** / **[Giả định]** / **[Cần người dùng phê duyệt]**.

## 1. Mục đích

Thiết kế pipeline GitHub Actions cho repo AI WORKSPACE (monorepo pnpm workspaces, chưa dùng Turborepo; D-001 APPROVED) để mọi PR được lint, typecheck, test (unit/integration/e2e), build, quét bảo mật trước khi merge; và release có phiên bản, artifact, image đã scan. Áp dụng cho **mã nguồn của chính platform**, không phải cho project do agent sinh ra (những project đó được build/test trong Sandbox Executor – INFRASTRUCTURE_DESIGN §6).

## 2. Phạm vi

Workflow files, job graph, services, caching, matrix, security scans, branch protection, required checks, release/versioning, kế hoạch Tauri build. Không bao gồm deploy lên server (DEPLOYMENT.md) và chính sách triage CVE (DEVSECOPS_GUIDE).

## 3. Giả định về repo

- GitHub repo `<org>/ai-workspace-app`, default branch `main`, nhánh phát hành `release/*` (tuỳ chọn). `gh` CLI 2.102.0 trên máy dev [Đã xác minh].
- Monorepo: `apps/api`, `apps/web`, `apps/desktop` (sau), `packages/*`, `infra/*`. pnpm 10 workspaces thuần, chưa dùng Turborepo (D-001 APPROVED, 2026-10-30; thêm Turborepo khi > ~5 package hoặc CI vượt ngưỡng). Các lệnh `pnpm turbo ...` bên dưới là minh họa, Phase 1 thay bằng `pnpm -r`/`pnpm --filter`.
- Node 24 (`.nvmrc`), `packageManager` field trong `package.json` để `corepack` chọn pnpm đúng.
- Self-hosted runner: không dùng ở MVP; GitHub-hosted `ubuntu-24.04`, `windows-2025` cho Tauri. [Đề xuất]

## 4. Workflow tổng quan

```
.github/workflows/
├── ci.yml            # PR + push main: lint, typecheck, unit, integration, e2e, build, scans
├── images.yml        # build + trivy scan sandbox images (infra/images/**), push GHCR khi main
├── release.yml       # tag v*: build artifacts, SBOM, GitHub Release, (Tauri sau)
├── codeql.yml        # CodeQL JS/TS hàng tuần + PR (tuỳ chọn)
└── dependency-review.yml  # PR: GitHub dependency review action
```

### 4.1 `ci.yml` – job graph

```
changes ─┬─▶ lint ─────────┐
         ├─▶ typecheck ────┤
         ├─▶ unit ─────────┼─▶ build ─▶ e2e (playwright) ─▶ ci-ok (gate)
         ├─▶ integration ──┤
         └─▶ security (gitleaks, npm audit, dependency-review) ┘
```

- `changes`: `dorny/paths-filter` xác định package bị ảnh hưởng (`api`, `web`, `packages`, `infra`, `docs-only`). Docs-only PR chỉ chạy `lint-docs` (markdownlint) và `ci-ok`.
- Turborepo `--filter=...[origin/main]` để chỉ chạy task cho package thay đổi; vẫn chạy full trên `main`.

### 4.2 Rút gọn `ci.yml` [Đề xuất]

```yaml
name: ci
on:
  pull_request:
  push: { branches: [main] }
concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }
permissions: { contents: read }

env:
  TURBO_TELEMETRY_DISABLED: 1
  DO_NOT_TRACK: 1

jobs:
  setup:
    runs-on: ubuntu-24.04
    outputs: { filter: ${{ steps.f.outputs.filter }} }
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - id: f
        run: echo "filter=--filter=...[origin/main]" >> "$GITHUB_OUTPUT"

  lint:
    needs: setup
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-node-pnpm      # composite: corepack, pnpm cache, pnpm install --frozen-lockfile
      - run: pnpm turbo lint typecheck ${{ needs.setup.outputs.filter }}

  unit:
    needs: setup
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-node-pnpm
      - run: pnpm turbo test:unit ${{ needs.setup.outputs.filter }} -- --coverage
      - uses: actions/upload-artifact@v4
        with: { name: coverage-unit, path: "**/coverage/lcov.info", retention-days: 7 }

  integration:
    needs: setup
    runs-on: ubuntu-24.04
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_USER: aiw, POSTGRES_PASSWORD: aiw, POSTGRES_DB: aiw_test }
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U aiw" --health-interval 5s --health-timeout 3s --health-retries 20
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
        options: --health-cmd "redis-cli ping" --health-interval 5s --health-timeout 3s --health-retries 20
    env:
      DATABASE_URL: postgresql://aiw:aiw@127.0.0.1:5432/aiw_test
      REDIS_URL: redis://127.0.0.1:6379
      WORKSPACE_ROOT: ${{ runner.temp }}/aiw
      SANDBOX_ENABLED: "true"         # runner ubuntu có Docker daemon -> test sandbox thật
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-node-pnpm
      - run: pnpm --filter api db:migrate
      - run: pnpm turbo test:integration ${{ needs.setup.outputs.filter }}

  security:
    needs: setup
    runs-on: ubuntu-24.04
    permissions: { contents: read, security-events: write }
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2
        env: { GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
      - uses: ./.github/actions/setup-node-pnpm
      - run: pnpm audit --audit-level=high --prod      # fail khi high/critical trong prod deps
      - uses: aquasecurity/trivy-action@0.28.0          # fs scan: lockfile + misconfig (Dockerfile, compose)
        with: { scan-type: fs, scanners: "vuln,misconfig,secret", severity: "HIGH,CRITICAL", exit-code: "1", ignore-unfixed: true }

  build:
    needs: [lint, unit, integration]
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-node-pnpm
      - run: pnpm turbo build
      - uses: actions/upload-artifact@v4
        with: { name: web-dist, path: apps/web/dist, retention-days: 3 }
      - uses: actions/upload-artifact@v4
        with: { name: api-dist, path: apps/api/dist, retention-days: 3 }

  e2e:
    needs: build
    runs-on: ubuntu-24.04
    timeout-minutes: 30
    services: { postgres: {...}, redis: {...} }        # như integration
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-node-pnpm
      - uses: actions/download-artifact@v4
      - run: pnpm --filter web exec playwright install --with-deps chromium
      - run: pnpm --filter api db:migrate && pnpm turbo test:e2e
        env: { AGENT_MODE: mock }                         # không gọi claude/codex thật trong CI
      - uses: actions/upload-artifact@v4
        if: failure()
        with: { name: playwright-report, path: apps/web/playwright-report, retention-days: 7 }

  ci-ok:
    if: always()
    needs: [lint, unit, integration, security, build, e2e]
    runs-on: ubuntu-24.04
    steps:
      - run: |
          echo '${{ toJSON(needs) }}' | jq -e 'to_entries | all(.value.result == "success" or .value.result == "skipped")'
```

Composite action `setup-node-pnpm`: `actions/setup-node@v4` (`node-version-file: .nvmrc`), `corepack enable`, `actions/cache` cho pnpm store (`key: pnpm-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}`), `pnpm install --frozen-lockfile`. Turborepo cache: `actions/cache` cho `.turbo/` (key theo `github.sha`, restore-keys theo branch) – đủ cho MVP, Vercel remote cache tuỳ chọn sau.

Agent CLI trong CI: **không** chạy `claude`/`codex` thật (không có login, tốn tiền). `AGENT_MODE=mock` dùng adapter giả lập stream-json. Test tích hợp thật với CLI chạy thủ công trên máy dev (`pnpm test:agent-live`, cần phê duyệt).

### 4.3 `images.yml` – sandbox images

- Trigger: `infra/images/**` thay đổi (PR: build + scan, không push), push `main` (push `ghcr.io/<org>/aiw-sandbox-<stack>:<sha>` và `:latest`), schedule hàng tuần (rebuild để lấy patch OS).
- Steps: `docker/setup-buildx-action`, `docker/build-push-action` với cache `type=gha`, `aquasecurity/trivy-action` (`image-ref`, `severity: HIGH,CRITICAL`, `exit-code: 1`), `anchore/sbom-action` sinh SBOM (SPDX JSON) upload artifact + attach vào image (`cosign attach sbom`, tuỳ chọn), `cosign sign` keyless với OIDC [Đề xuất].
- `permissions: packages: write, id-token: write, contents: read`.

### 4.4 `release.yml`

- Trigger: tag `v*.*.*` (tạo bởi Changesets/`release-please` – xem §7).
- Jobs: `build` (ubuntu) → `sbom` (CycloneDX cho npm workspace: `pnpm dlx @cyclonedx/cyclonedx-npm`) → `docker` (image `aiw-api`, `aiw-web`, `aiw-worker` cho self-hosted mode, scan + sign) → `github-release` (changelog, artifacts: `api-dist.tgz`, `web-dist.tgz`, `sbom.json`, `compose-bundle.tgz`) → `tauri` (matrix, sau MVP).
- `tauri` matrix [Đề xuất, kích hoạt khi `apps/desktop` tồn tại]:

```yaml
tauri:
  strategy:
    fail-fast: false
    matrix:
      include:
        - { os: windows-2025, target: x86_64-pc-windows-msvc }
        - { os: ubuntu-24.04, target: x86_64-unknown-linux-gnu }
        - { os: macos-15, target: aarch64-apple-darwin }
  runs-on: ${{ matrix.os }}
  steps:
    - uses: actions/checkout@v4
    - uses: ./.github/actions/setup-node-pnpm
    - uses: dtolnay/rust-toolchain@stable
      with: { targets: ${{ matrix.target }} }
    - uses: Swatinem/rust-cache@v2
      with: { workspaces: apps/desktop/src-tauri }
    - if: runner.os == 'Linux'
      run: sudo apt-get update && sudo apt-get install -y libwebkit2gtk-4.1-dev libappindicator3-dev librsvg2-dev patchelf
    - uses: tauri-apps/tauri-action@v0
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        TAURI_SIGNING_PRIVATE_KEY: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY }}
        TAURI_SIGNING_PRIVATE_KEY_PASSWORD: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY_PASSWORD }}
      with: { tagName: ${{ github.ref_name }}, releaseDraft: true, args: --target ${{ matrix.target }} }
```

Windows code signing (Authenticode) và macOS notarization: cần chứng chỉ – [Cần người dùng phê duyệt], MVP phát hành bản chưa ký với cảnh báo SmartScreen.

## 5. Branch protection & required checks [Đề xuất]

Áp dụng qua GitHub Rulesets cho `main`:

| Rule | Giá trị |
|---|---|
| Require PR before merge | bật; ≥ 1 approval (solo dev: cho phép self-approve bằng CODEOWNERS trống; nâng lên 1 reviewer khác khi team ≥ 2) |
| Required status checks | `ci-ok`, `security`, `dependency-review`, `images / scan` (khi thay đổi `infra/images`) |
| Require branches up to date | bật (merge queue của GitHub tuỳ chọn khi PR nhiều) |
| Require signed commits | tuỳ chọn [Cần người dùng phê duyệt] |
| Linear history | bật (squash merge mặc định) |
| Block force push / delete | bật |
| Restrict who can push | chỉ qua PR; bot release-please được phép tạo PR |

Branch `integration/*` và `agent/*` trong **project sinh ra** (không phải repo platform) tuân theo WORKSPACE doc §8, và nếu project có remote GitHub, người dùng tự cấu hình ruleset tương tự; platform chỉ tạo PR qua `gh`.

## 6. Caching & tốc độ

- pnpm store cache (actions/cache, ~1–2 phút tiết kiệm), Turborepo local cache `.turbo/`, Playwright browsers cache (`~/.cache/ms-playwright`, key theo version Playwright), Docker buildx `type=gha`, Rust cache cho Tauri.
- Mục tiêu: PR thông thường ≤ 10 phút end-to-end; docs-only ≤ 1 phút.
- `concurrency.cancel-in-progress` theo ref để huỷ run cũ khi push mới.

## 7. Release versioning [Đề xuất]

- **SemVer** cho toàn sản phẩm (một version chung cho api/web/desktop; packages nội bộ `private: true`, không publish npm).
- Công cụ: `release-please` (googleapis/release-please-action) đọc Conventional Commits → tự mở "release PR" cập nhật `CHANGELOG.md` và version; merge PR → tạo tag `vX.Y.Z` → `release.yml`. Thay thế: Changesets (tốt hơn khi publish nhiều package) – không cần ở MVP.
- Pre-release: `v0.x` trong suốt MVP; `v1.0.0` khi desktop shell ổn định.
- Image tag: `vX.Y.Z`, `vX.Y`, `sha-<short>`; `latest` chỉ cho stable.
- Migration DB đi kèm release notes "Requires migration: yes/no" (DEPLOYMENT.md §6).

## 8. Bảo mật pipeline

- `permissions` tối thiểu ở mức workflow, nâng theo job; `GITHUB_TOKEN` read-only mặc định.
- Pin actions theo major tag là tối thiểu; pin SHA cho actions bên thứ ba ít tin cậy (`trivy-action`, `gitleaks-action`) – Dependabot `github-actions` ecosystem cập nhật hàng tuần.
- Không dùng `pull_request_target` với checkout code PR. Fork PR không được truy cập secrets.
- Secrets: chỉ `TAURI_SIGNING_*`, (sau) `COSIGN_*`, token GHCR dùng `GITHUB_TOKEN`. Không có API key Anthropic/OpenAI trong CI (AGENT_MODE=mock).
- OIDC cho đăng ký GHCR/cosign thay vì long-lived token.
- Dependabot: `npm` (weekly, grouped minor/patch), `docker`, `github-actions`. Dependency review action chặn license cấm (GPL-3.0 trong prod deps) và CVE high.

## 9. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
|---|---|---|
| GitHub Actions, GitHub-hosted runners | Repo trên GitHub, `gh` sẵn; không có hạ tầng runner riêng | [Đề xuất] |
| Turborepo filter theo thay đổi | Giảm thời gian PR trong monorepo | Hoãn: D-001 APPROVED chọn pnpm thuần, chưa dùng Turborepo |
| Integration test với service containers + sandbox thật trên ubuntu runner | Runner có Docker daemon, test được executor thật | [Đề xuất] |
| Agent CLI mock trong CI | Không có login, chi phí, tính không xác định | [Đề xuất] |
| release-please + SemVer chung | Đơn giản cho sản phẩm đơn | [Cần người dùng phê duyệt] |
| Trivy cho image + fs, gitleaks cho secret, `pnpm audit` cho deps | Công cụ OSS phổ biến, có action chính thức | [Đề xuất] |

## 10. Dependencies

- ENVIRONMENT_SETUP.md: `packageManager`, `.nvmrc`, script `lint/typecheck/test:*` thống nhất tên giữa các `package.json` (không dùng `turbo.json`, D-001).
- A08: lệnh migration để chạy trước integration/e2e.
- Frontend team: Playwright config, `AGENT_MODE=mock` adapter.
- DEVSECOPS_GUIDE.md: ngưỡng CVE, quy trình triage khi `security` job fail.
- Tauri: chỉ khi `apps/desktop` xuất hiện.

## 11. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| `pnpm audit` fail vì CVE không có bản vá → chặn mọi PR | `ignore-unfixed`, file `audit-ci` allowlist có hạn dùng (DEVSECOPS_GUIDE §6) |
| E2E flaky kéo dài PR | Retry 1 lần trong Playwright, tách e2e ra `required` chỉ trên `main` nếu quá flaky |
| GitHub-hosted Windows runner chậm (Tauri build 15–25 phút) | Chỉ chạy khi tag; Rust cache |
| Sandbox test cần Docker trong runner – không có trên macOS/Windows runner | Chỉ chạy integration trên ubuntu |
| Secret lọt vào log workflow | `::add-mask::`, pino redaction, gitleaks trên artifact log (tuỳ chọn) |

## 12. Vấn đề chưa giải quyết

1. ĐÃ CHỐT (D-001): pnpm workspaces thuần, không có `turbo.json` ở Phase 1.
2. Có bật GitHub merge queue không (hữu ích khi > 3 PR/ngày).
3. Có yêu cầu signed commits không.
4. Code signing certificate cho Windows/macOS (chi phí, chủ sở hữu).
5. Remote cache Turborepo (Vercel) – tuỳ chọn, cần token.
6. CodeQL: bật ngay hay sau khi có code.

## 13. Acceptance Criteria

- [ ] PR docs-only hoàn thành `ci-ok` < 2 phút; PR code thường < 10 phút.
- [ ] Integration job khởi động Postgres/Redis service, chạy migration, test sandbox executor thật trên ubuntu runner.
- [ ] `security` job fail khi commit chứa secret giả (test bằng PR thử) và khi thêm dependency có CVE critical đã biết.
- [ ] `images.yml` build 3 image sandbox, Trivy không có HIGH/CRITICAL chưa vá, push GHCR với SBOM.
- [ ] Ruleset `main` yêu cầu `ci-ok` + `security`; force push bị chặn.
- [ ] Tag `v0.1.0` tạo GitHub Release với artifacts + SBOM + changelog tự động.
- [ ] (Khi có desktop) Tauri matrix sinh installer Windows `.msi/.exe` dưới dạng draft release.
