# SKILL SYSTEM — Skill Library, Teach Agent & Agent Memory

> Phase 0 · Agent 15 (AI Agent Architect — Skill & Knowledge System) · Phiên bản 0.1 · Trạng thái: Draft chờ owner phê duyệt
> Quy ước đánh dấu: **[Đã xác minh]** (kiểm tra từ tài liệu chính thức / source code local ngày 2026-10-30) · **[Đề xuất]** (quyết định của Architect, có lý do) · **[Giả định]** (chưa có dữ liệu) · **[Cần người dùng phê duyệt]** (owner phải chốt trước Phase 1)

## 1. Mục đích

Tài liệu này định nghĩa cách AI WORKSPACE lưu trữ, phân phối và kiểm chứng **tri thức có thể tái sử dụng** cho AI employee:

1. **Skill Library** — gói kiến thức theo chuẩn mở Agent Skills (`SKILL.md` + `references/`, `examples/`, `scripts/`), có category, version, rule, checklist, evaluation test.
2. **Provider projection** — cách một skill được "vật chất hóa" thành file mà Claude Code CLI, Codex CLI (và OpenHands trong tương lai) thực sự đọc được tại thời điểm bắt đầu session.
3. **Relevance selection** — chỉ nạp skill liên quan vào task để giữ context budget.
4. **Teach Agent** — biến một câu chỉ dẫn tự nhiên thành rule có cấu trúc, gắn vào skill, gán cho agent, xác minh agent đã nhận.
5. **Agent Memory** — mô hình bộ nhớ (episodic / semantic / procedural) tách biệt với skill, luôn qua phê duyệt, không bao giờ tự lưu từ hội thoại.

Nguyên tắc nền **[Đã xác minh — brief owner]**: *Skill không phải fine-tuning*. Ở v1, skill = instructions + references + workflows + tool definitions + rules. Mọi thứ đều là file và metadata; model không thay đổi.

## 2. Phạm vi

- **Trong phạm vi**: định dạng gói skill, schema frontmatter + extension của ta, storage (DB + disk), projection layer cho 3 provider, thuật toán chọn skill, versioning/pinning, import/export, lint, "Test Skill" harness, Teach Agent pipeline, Agent Memory model, permission, security, seed skill list, API/event sketch, acceptance criteria nhóm D.
- **Ngoài phạm vi**: nội dung đầy đủ của từng seed skill (Phase 1), UI chi tiết màn Skills (xem `docs/design/UI_UX_GUIDELINES.md` §2, khu vực "Skills"), lựa chọn vector DB cụ thể (Agent Database), cơ chế spawn/worktree của Provider Adapter (`PROVIDER_ADAPTER.md`), schema SQL đầy đủ (`DATABASE_DESIGN.md`).
- **Liên quan**: `DATABASE_DESIGN.md` (bảng `skills`, `skill_versions`, `skill_documents`, `agent_skills`, `agent_memories`), `PROVIDER_ADAPTER.md` (session bootstrap), `EVENT_SYSTEM.md` (event bus), `SECURITY.md` (sandbox, RBAC).

## 3. Kết quả nghiên cứu (tóm tắt những gì đã kiểm chứng)

### 3.1 Chuẩn mở Agent Skills (agentskills.io) **[Đã xác minh]**

Nguồn: `https://agentskills.io/specification` (fetch 2026-10-30).

- Một skill là một thư mục chứa tối thiểu `SKILL.md`; thư mục tùy chọn theo convention: `scripts/`, `references/`, `assets/`. Cho phép "any additional files or directories".
- Frontmatter YAML: `name` (bắt buộc, 1–64 ký tự, chỉ `a-z0-9-`, không bắt đầu/kết thúc bằng `-`, không `--`, **phải trùng tên thư mục cha**), `description` (bắt buộc, 1–1024 ký tự, mô tả *làm gì* và *khi nào dùng*), `license`, `compatibility` (≤500 ký tự, free text về yêu cầu môi trường), `metadata` (map string→string, client tự định nghĩa key), `allowed-tools` (chuỗi cách nhau bởi space, *experimental*).
- Progressive disclosure 3 tầng: (1) metadata ~100 token nạp lúc khởi động cho mọi skill; (2) body `SKILL.md` (<5000 token khuyến nghị, <500 dòng) nạp khi kích hoạt; (3) `references/`, `scripts/`, `assets/` nạp theo nhu cầu. Tham chiếu file chỉ nên sâu 1 cấp.
- Có thư viện tham chiếu `skills-ref validate ./my-skill` để lint.
- Ngoài Claude Code, chuẩn chỉ chấp nhận 6 field trên; field lạ gây **hard error** khi upload lên claude.ai. ⇒ Mọi extension của ta phải nằm trong `metadata` hoặc file phụ.

### 3.2 Claude Code CLI **[Đã xác minh]**

Nguồn: `https://code.claude.com/docs/en/skills`, `https://code.claude.com/docs/en/cli-reference`.

- Vị trí discover skill: enterprise (`.claude/skills/` trong managed settings dir) › personal `~/.claude/skills/<name>/SKILL.md` › project `.claude/skills/<name>/SKILL.md` › nested `<subdir>/.claude/skills/` › **thư mục truyền qua `--add-dir` (`.claude/skills/` trong đó, chỉ cho session đó)** › plugin `<plugin>/skills/<name>/` (namespace `/plugin:skill`). Trùng tên: enterprise > personal > project. Monorepo: nạp từ start dir lên tới repo root.
- `--add-dir` cấp quyền file nhưng **không** discover hầu hết cấu hình `.claude/` khác — riêng skills và `.claude/agents/` thì có. `--bare` bỏ qua auto-discovery nhưng **skills trong `--add-dir` vẫn được nạp**.
- Frontmatter Claude Code hỗ trợ thêm: `when_to_use`, `argument-hint`, `arguments`, `disable-model-invocation`, `user-invocable`, `allowed-tools`, `disallowed-tools`, `model`, `effort`, `context: fork`, `agent`, `background`, `hooks`, `paths` (glob giới hạn auto-activation), `shell`.
- Giới hạn: `description + when_to_use` bị cắt ở **1 536 ký tự** trong listing; sau compaction giữ 5 000 token/skill, tổng 25 000 token cho skills.
- Subagent: `.claude/agents/*.md`; flag `--agents '<json>'` định nghĩa subagent động (với `-p` có thể là đường dẫn file JSON, ≥ v2.1.281).
- Non-interactive: `claude -p`, `--output-format json|stream-json`, `--json-schema '<schema>'` (structured output, print mode), `--append-system-prompt-file`, `--system-prompt-file`, `--allowedTools`, `--tools`, `--max-turns`, `--max-budget-usd`, `--settings <file|json>`, `--setting-sources user,project,local`, `--restricted` (dành cho evaluation harness trên máy dùng chung), `--bare`.

### 3.3 Codex CLI **[Đã xác minh]**

Nguồn: `https://learn.chatgpt.com/docs/build-skills`, `.../agent-configuration/agents-md`, `.../non-interactive-mode` (`developers.openai.com/codex/*` redirect 308 sang đây).

- Skills: Codex đọc `$CWD/.agents/skills` → các thư mục cha lên đến repo root → `$REPO_ROOT/.agents/skills` → `$HOME/.agents/skills` → `/etc/codex/skills` → built-in. Trùng `name` thì **không merge**. Hỗ trợ symlink. Tùy chọn `agents/openai.yaml` trong thư mục skill (`policy.allow_implicit_invocation`, `dependencies.tools` khai báo MCP). Bật/tắt qua `[[skills.config]]` trong `~/.codex/config.toml`; `skills.max_context_tokens` (≤10 000) cho danh sách skill ban đầu.
- Tài liệu bên thứ ba còn nhắc `~/.codex/skills` (`$CODEX_HOME/skills`, đường dẫn cũ) — **[Giả định]** vẫn hoạt động nhưng ta không dựa vào; `.agents/skills` là đường dẫn chính thức hiện tại.
- AGENTS.md: global `~/.codex/AGENTS.override.md` hoặc `AGENTS.md` → project root đi xuống cwd, mỗi thư mục tối đa 1 file (`AGENTS.override.md` > `AGENTS.md` > `project_doc_fallback_filenames`). Nối từ root xuống, file gần cwd nằm sau nên override. **Tổng giới hạn `project_doc_max_bytes` mặc định 32 KiB**, vượt là bị cắt.
- Non-interactive: `codex exec "<task>"` (stdout chỉ in final message), `--json` (JSONL event: `thread.started`, `turn.completed`, `item.*`), `-o/--output-last-message <file>`, `--output-schema <schema.json>` (structured output), `--sandbox read-only|workspace-write|danger-full-access`, `--approve-for-me` (thay `--full-auto` đã bỏ), `--skip-git-repo-check`, `-c key=value` override config.

### 3.4 OpenHands **[Đã xác minh]**

Nguồn: `https://docs.openhands.dev/usage/prompting/microagents-overview` + repo local `D:\Code\Tuyen\Agent management\OpenHands` (Agent Canvas, xem `docs/research/OPENHANDS_ANALYSIS.md`).

- OpenHands đã chuyển sang **Agent Skills spec**: khuyến nghị `.agents/skills/<name>/SKILL.md` (project) và `~/.agents/skills/` (user); `.openhands/skills/` và `.openhands/microagents/` là legacy còn hỗ trợ. Repository context always-on nằm trong `AGENTS.md` (cũng đọc `CLAUDE.md`, `GEMINI.md`).
- Extension OpenHands: `triggers` (keyword kích hoạt), `paths` (path-triggered rule, ưu tiên hơn `triggers`).
- Repo local: `src/utils/skill-scope.ts` phân scope `project | personal | public` theo marker đường dẫn `/.agents/skills/`, `/.openhands/skills/`, `/.openhands/microagents/`; `src/api/cloud/skills-service.api.ts` gọi `/api/v1/skills/search` và `/api/v1/app-conversations/{id}/skills` (danh sách skill thực sự nạp vào một conversation) — mẫu tốt cho API "verify loaded" của ta. `AGENTS.md` của repo dùng bảng "Skill | Use for" trỏ sang `.agents/skills/*/SKILL.md` — progressive disclosure thủ công.

### 3.5 ChatDev 2.0 **[Đã xác minh]**

Repo local `D:\Code\Tuyen\Agent management\ChatDev`:

- `.agents/skills/<name>/SKILL.md` + `references/examples.md` (vd `rest-api-caller`, `python-scratchpad`), frontmatter `name/description/allowed-tools`.
- `runtime/node/agent/skills/manager.py`: `AgentSkillManager` discover từ thư mục cố định, validate `name == dirname`, parse `allowed-tools`, `compatibility`; **bỏ qua skill yêu cầu tool không bind trên node**; sinh block `<available_skills>` (name/description/location/allowed_tools) đưa vào prompt; cung cấp 2 tool nội bộ `activate_skill(skill_name)` và `read_skill_file(skill_name, relative_path)` (chỉ đọc sau khi activate, chặn path traversal, giới hạn 128 KiB/file). Nếu không có skill tương thích, agent được chỉ dẫn rõ "không được claim đã dùng skill".
- `entity/configs/node/skills.py`: config node `skills: {enabled, allow: [{name}]}` — tức **allowlist per-agent**.

### 3.6 Pixel Agents **[Đã xác minh]**

`D:\Code\Tuyen\Agent management\pixel-agents` không có khái niệm skill (chỉ visualize session Claude Code). Không dùng làm reference cho tài liệu này.

## 4. Nội dung chi tiết

### 4.1 Định dạng gói skill (Skill Package)

**[Đề xuất]** Layout chuẩn, tương thích 100% với Agent Skills spec; mọi phần mở rộng đặt ở file phụ `skill.yaml` và `metadata` để `SKILL.md` luôn upload/chia sẻ được nguyên trạng:

```
skills/<slug>/
├── SKILL.md              # Bắt buộc. Frontmatter chuẩn + body instructions (<500 dòng)
├── skill.yaml            # Extension của AI WORKSPACE (category, rules, checklists, tests, compat…)
├── references/           # Tài liệu tham chiếu .md (PDF/DOCX upload được convert sang .md, bản gốc giữ ở assets/)
│   ├── REFERENCE.md
│   └── <topic>.md
├── examples/             # Code example, mỗi file có header 3 dòng mô tả ngữ cảnh
├── checklists/           # Markdown checklist (1 file / checklist), tham chiếu từ skill.yaml
├── scripts/              # Script hỗ trợ (bash/python/node) — xem §4.13 Security
├── assets/               # Template, file gốc upload (pdf, docx), schema JSON
├── tests/                # Evaluation probes (probe-*.yaml) — xem §4.8
└── CHANGELOG.md          # Sinh tự động theo version
```

Khác biệt với spec: ta thêm `examples/`, `checklists/`, `tests/`, `skill.yaml`, `CHANGELOG.md`. Spec cho phép "any additional files" nên vẫn hợp lệ.

#### 4.1.1 `SKILL.md` frontmatter

```yaml
---
name: nestjs-modular-api            # = slug = tên thư mục (spec)
description: >-                     # ≤1024 ký tự; ưu tiên 1 536 ký tự đầu cho Claude Code listing
  Build NestJS REST APIs with modular architecture, DTO validation, exception
  filters, structured logging and Swagger. Use when creating or refactoring
  NestJS controllers, modules, DTOs or API docs.
license: Proprietary
compatibility: Designed for Claude Code, Codex CLI and OpenHands. Requires Node 20+.
allowed-tools: Read Grep Glob Bash(npm:*) Bash(npx nest:*)   # experimental (spec)
metadata:                            # chỉ string→string (spec)
  aiw-category: development/nestjs
  aiw-version: "1.2.0"
  aiw-skill-id: "01J…ULID"
  aiw-always-on: "false"
---
```

Chỉ các key `aiw-*` trong `metadata` là của ta; Claude Code/Codex/OpenHands bỏ qua mà không lỗi **[Đã xác minh — spec: metadata là map tự do]**.

#### 4.1.2 `skill.yaml` — schema mở rộng **[Đề xuất]**

```yaml
schema: aiw.skill/v1
slug: nestjs-modular-api
category: development          # development|architecture|database|testing|security|devops|custom
subcategory: nestjs
version: 1.2.0                 # semver; trùng metadata.aiw-version
status: published              # draft|in_review|published|deprecated
tags: [nestjs, rest, dto, swagger, backend]
departments: [engineering]     # dùng cho relevance (§4.5)
activation:
  mode: on-demand              # always-on|on-demand
  triggers: [nestjs, "@nestjs", nest g, controller, dto, swagger]   # keyword (map sang OpenHands triggers)
  paths: ["**/*.module.ts", "**/*.controller.ts", "**/*.dto.ts"]      # map sang Claude Code paths / OpenHands paths
  estimated_tokens: 1800       # lint tính lại khi publish
required_tools:                # tool bắt buộc; thiếu ⇒ skill bị bỏ (theo cách ChatDev)
  - Bash
  - Read
rules:                         # rule có cấu trúc — nguồn chính của Teach Agent
  - id: R1
    severity: must             # must|should|may
    text: "Every feature lives in its own Nest module; no logic in AppModule."
    check: "No providers/controllers registered directly in app.module.ts"
  - id: R2
    severity: must
    text: "All request bodies are DTO classes validated with class-validator + global ValidationPipe({whitelist:true})."
  - id: R3
    severity: must
    text: "Use a global HttpExceptionFilter; never leak stack traces."
  - id: R4
    severity: should
    text: "Use the Nest Logger (or pino) with request-id correlation."
  - id: R5
    severity: must
    text: "Every controller method has @ApiOperation/@ApiResponse Swagger decorators."
checklists:
  - id: pre-commit
    file: checklists/pre-commit.md
workflows:
  - id: new-endpoint
    steps: ["Create module", "Define DTOs", "Write controller + service", "Add Swagger", "Write unit test"]
references:
  - file: references/REFERENCE.md
    when: "Need decorator signatures or ValidationPipe options"
examples:
  - file: examples/users.module.ts
tests:                         # xem §4.8
  - file: tests/probe-new-endpoint.yaml
compatibility:
  claude-code: { supported: true, projection: skills-dir }
  codex:       { supported: true, projection: skills-dir+agents-md }
  openhands:   { supported: true, projection: skills-dir }
provenance:
  created_by: user:<id>
  source: teach|manual|import
  taught_from_instruction_id: 01J…   # nếu sinh từ Teach Agent
```

**Lý do tách `skill.yaml`**: (a) spec cấm field lạ ở frontmatter khi upload claude.ai; (b) rule/checklist/test cần cấu trúc máy đọc được để làm conflict detection và evaluation, Markdown body không đủ; (c) body `SKILL.md` vẫn là nơi provider đọc — lint sẽ **render** section `## Mandatory rules` từ `rules[]` vào body để agent thực sự nhìn thấy rule (xem §4.7).

### 4.2 Storage model

**[Đề xuất]** Hybrid: metadata trong PostgreSQL, nội dung file trên disk (immutable theo version), tham chiếu bảng trong `DATABASE_DESIGN.md`:

| Bảng | Vai trò | Cột chính (tham chiếu) |
|---|---|---|
| `skills` | Identity, slug, category, trạng thái, owner, `latest_version_id`, `visibility` (org/project/private) | `id ULID, slug UNIQUE, name, category, subcategory, status, org_id, project_id?, created_by` |
| `skill_versions` | Một bản immutable mỗi lần publish | `id, skill_id, version semver, frontmatter JSONB, skill_yaml JSONB, body_md TEXT, content_hash sha256, storage_path, estimated_tokens, lint_report JSONB, published_by, published_at, changelog` |
| `skill_documents` | Từng file trong gói (references/examples/scripts/assets/tests) | `id, skill_version_id, relative_path, kind, mime, size, sha256, original_upload_id?, text_extracted BOOL` |
| `agent_skills` | Gán skill → agent, có pin version | `agent_id, skill_id, pinned_version_id NULL=latest, mode (always-on/on-demand/inherit), assigned_by, assigned_at, verified_at, last_loaded_at` |
| `skill_assignments_scope` (tùy chọn) | Gán theo department/role thay vì từng agent | `scope_type, scope_id, skill_id` |
| `skill_evaluations` | Kết quả Test Skill | `id, skill_version_id, agent_id?, provider, probe_id, score, rubric JSONB, transcript_ref, ran_at` |
| `agent_memories` | Xem §4.11 | |

Disk layout **[Đề xuất]**:

```
data/skills/<slug>/<version>/          # immutable sau publish (read-only)
data/skills/<slug>/draft/              # bản nháp đang edit (1 draft / skill)
data/skills/_index/<slug>.json         # cache metadata để build listing nhanh
data/skills/_cache/<provider>/<content_hash>/   # bản projection đã render (§4.3), key theo hash
```

Quy tắc:
- `storage_path = data/skills/<slug>/<version>/`; `content_hash` = sha256 của tar canonical toàn bộ thư mục → dùng để verify agent nạp đúng bản (§4.9) và dedupe projection cache.
- Upload PDF/DOCX: giữ bản gốc trong `assets/originals/`, trích text sang `references/<name>.md` (pdf → `pdf-parse`; docx → `mammoth`) **[Đề xuất]**; nếu trích thất bại đánh dấu `text_extracted=false` và lint cảnh báo "agent sẽ không đọc được file này".
- Giới hạn: 1 gói ≤ 20 MB, 1 file text ≤ 128 KiB (theo ChatDev), tổng text ≤ 1 MB **[Cần người dùng phê duyệt]**.

### 4.3 Provider projection layer

Skill trong DB vô nghĩa nếu CLI không đọc được. Projection chạy trong bước *session bootstrap* của Provider Adapter, **trước** khi spawn process.

#### 4.3.1 Nguyên tắc chung **[Đề xuất]**

1. **Hai vị trí vật lý**: (a) *shared read-only skills dir* trên host: `data/runtime/skills-share/<agent_id>/<session_id>/` chứa toàn bộ skill được chọn; (b) *fragment trong worktree* chỉ chứa file chỉ dẫn nhỏ (`CLAUDE.md` fragment / `AGENTS.md`) trỏ sang (a). Lý do: không làm bẩn git worktree bằng hàng trăm file skill (sẽ lọt vào commit), và cho phép mount read-only.
2. Mọi file sinh ra có header `<!-- generated by AI WORKSPACE · skill=<slug>@<version> · hash=<sha256[:12]> · do not edit -->` để hệ thống (và agent) nhận ra.
3. Fragment được bao bởi marker `<!-- AIW:BEGIN skills -->` … `<!-- AIW:END skills -->` để cập nhật idempotent khi repo đã có `CLAUDE.md`/`AGENTS.md` của người dùng (chỉ chèn/thay thế vùng marker, không ghi đè file).
4. Worktree phải exclude các file sinh ra (`.claude/skills/aiw-*`, `.agents/skills/*`, vùng marker) — ghi vào `.git/info/exclude` của worktree thay vì sửa `.gitignore` của repo **[Đề xuất]**.

#### 4.3.2 Claude Code **[Đã xác minh cơ chế discover, Đề xuất cách dùng]**

Layout:

```
<share>/claude/                                   # thư mục truyền qua --add-dir
├── .claude/
│   ├── skills/
│   │   ├── nestjs-modular-api/SKILL.md           # nguyên bản (frontmatter chuẩn + `paths:` nếu có)
│   │   ├── nestjs-modular-api/references/…
│   │   └── owasp-top-10/SKILL.md
│   └── agents/                                   # (tùy chọn) subagent definition sinh từ role của agent
│       └── reviewer.md
├── system-fragment.md                            # always-on rules + memory (§4.11)
├── settings.json                                 # permissions.allow sinh từ allowed-tools ∩ policy
└── AIW_SKILLS_INDEX.md                           # bảng name | when to use | path (progressive disclosure)
<worktree>/CLAUDE.md                              # vùng marker: always-on rules + link sang AIW_SKILLS_INDEX.md
```

Lệnh spawn (sketch):

```
claude -p --output-format stream-json --verbose \
  --add-dir <share>/claude \
  --append-system-prompt-file <share>/claude/system-fragment.md \
  --allowedTools "<union allowed-tools của skill ∩ policy agent>" \
  --agents <share>/claude/agents.json \
  --settings <share>/claude/settings.json \
  --max-budget-usd <task budget> "<task prompt>"
```

- Skills trong `--add-dir` được discover kể cả `--bare` **[Đã xác minh]** ⇒ ta ưu tiên `--add-dir` hơn ghi vào `<worktree>/.claude/skills/` (tránh lọt vào git). Fallback nếu owner muốn skill nằm trong repo: ghi `<worktree>/.claude/skills/aiw-<slug>/` + exclude.
- **Always-on rules** (mode `always-on`): render vào `system-fragment.md` (qua `--append-system-prompt-file`) và lặp lại trong `CLAUDE.md` marker; **on-demand**: chỉ metadata (name/description) đi vào listing tự động của Claude Code, body nạp khi Claude chọn kích hoạt.
- Description phải nằm trong 1 536 ký tự đầu của `description + when_to_use` **[Đã xác minh]** → lint enforce.
- `allowed-tools` của skill chỉ **pre-approve** tool cho turn kích hoạt; permission thật vẫn do `--allowedTools`/settings của Provider Adapter quyết định — skill không thể mở rộng quyền vượt policy agent.

#### 4.3.3 Codex CLI **[Đã xác minh cơ chế discover, Đề xuất cách dùng]**

Codex không có `--add-dir`; discover theo `$CWD/.agents/skills` và các thư mục cha đến repo root, cộng `$HOME/.agents/skills`. Hai lựa chọn:

- **(A) Junction/symlink trong worktree** *(mặc định)*: `<worktree>/.agents/skills/<slug>` → junction tới `<share>/codex/skills/<slug>` (Codex hỗ trợ symlink **[Đã xác minh]**; trên Windows dùng directory junction `mklink /J`, không cần quyền admin). Exclude qua `.git/info/exclude`.
- **(B) Parent-dir trick**: đặt worktree tại `<runs>/<session>/repo/` và skills ở `<runs>/<session>/.agents/skills/` — Codex đi lên thư mục cha sẽ thấy, nhưng dừng ở repo root nên chỉ hoạt động khi không có `.git` ở trên. **[Giả định]** cần thử nghiệm Phase 1; giữ (A) làm chuẩn.

Layout:

```
<worktree>/.agents/skills/<slug>/          # junction → share
<worktree>/AGENTS.md                       # vùng marker: always-on rules + bảng skill (như OpenHands AGENTS.md)
<share>/codex/skills/<slug>/SKILL.md       # bản chuẩn; thêm agents/openai.yaml nếu skill khai báo MCP dependency
<share>/codex/config.toml                  # [[skills.config]] enable đúng path; skills.max_context_tokens
```

Lệnh spawn (sketch): `codex exec --json -C <worktree> --sandbox workspace-write -c 'skills.max_context_tokens=8000' --output-schema <schema> "<task prompt>"`.

Ràng buộc: tổng AGENTS.md chain ≤ 32 KiB (`project_doc_max_bytes`) **[Đã xác minh]** ⇒ always-on fragment cho Codex giới hạn **≤ 8 KiB**, phần còn lại phải là on-demand skill; lint cảnh báo nếu vượt. Global `~/.codex/AGENTS.md` của host **không** được dùng (nhiều agent chia sẻ một máy) — mọi thứ per-worktree.

#### 4.3.4 OpenHands **[Đã xác minh cơ chế, Đề xuất — chưa nằm trong MVP]**

```
<worktree>/.agents/skills/<slug>/SKILL.md   # thêm `triggers:` và `paths:` từ skill.yaml.activation
<worktree>/AGENTS.md                        # repository context (OpenHands đọc cả CLAUDE.md)
```

Vì OpenHands đã theo Agent Skills spec, projection trùng Codex gần như hoàn toàn; chỉ khác là frontmatter được phép có `triggers`/`paths`. Legacy `.openhands/microagents/` **không** sinh nữa.

#### 4.3.5 Bảng tóm tắt projection

| Thành phần skill | Claude Code | Codex | OpenHands |
|---|---|---|---|
| `SKILL.md` body + references | `--add-dir <share>/claude` (`.claude/skills/`) | junction `<worktree>/.agents/skills/` | `<worktree>/.agents/skills/` |
| Always-on rules | `--append-system-prompt-file` + `CLAUDE.md` marker | `AGENTS.md` marker (≤8 KiB) | `AGENTS.md` marker |
| On-demand listing | tự động (name+description) | tự động (`skills.max_context_tokens`) | tự động + `triggers` |
| `allowed-tools` | frontmatter + `--allowedTools` giao với policy | `agents/openai.yaml` `dependencies.tools` (MCP) | bỏ qua |
| Subagent role | `.claude/agents/*.md` / `--agents` | không có tương đương | không có |
| Verify loaded | hook `SessionStart` / stream-json + probe | parse `thread.started` + probe | `/skills` API của Agent Server |

### 4.4 Skill versioning & pinning

**[Đề xuất]**

- Semver: `MAJOR` khi đổi/bỏ rule `must` hoặc đổi tên; `MINOR` khi thêm rule/reference; `PATCH` khi sửa câu chữ.
- Mỗi `skill_versions` row immutable; `skills.latest_version_id` trỏ bản published mới nhất. Draft chỉ có 1 và không bao giờ được projection.
- `agent_skills.pinned_version_id`: NULL = theo latest (mặc định); set = ghim. Khi publish MAJOR mới, hệ thống **không** tự nâng agent đang ghim, và tạo notification "Skill X có bản 2.0, 4 agent đang ở 1.x" cho owner.
- Rollback = publish lại bản cũ thành version mới (`1.3.1` từ nội dung `1.2.0`), không xóa lịch sử.
- `deprecated` version: vẫn projection được cho agent đang ghim, nhưng không thể gán mới.
- Session đang chạy **không** hot-reload: phiên bản được chốt lúc bootstrap và ghi vào `task_runs.skill_snapshot` (danh sách `slug@version#hash`).

### 4.5 Relevance selection (chọn skill nào cho task)

Đầu vào: `agent` (department, role, skills được gán, pin), `task` (title, description, tags, repo path, file touched dự kiến), `provider`, `budget_tokens`.

**[Đề xuất]** Thuật toán 5 bước, deterministic trước, embedding sau:

```
1. Candidate = agent_skills(agent) ∪ scope assignments(department/role/org)   # explicit assignment là điều kiện cần
2. Loại skill: compat[provider].supported=false, required_tools ⊄ tools(agent), status≠published
3. Score mỗi skill (0–1):
     0.40 * explicit_assignment      (1 nếu gán trực tiếp cho agent, 0.6 nếu qua department/role)
   + 0.25 * keyword_match(task.text, triggers ∪ tags)      # BM25 đơn giản, normalize
   + 0.20 * embedding_cosine(task.embedding, skill.description_embedding)   # tùy chọn, bật khi có vector store
   + 0.10 * path_match(task.files, activation.paths)
   + 0.05 * recency_success(skill_evaluations trên agent này)
4. always-on skills luôn vào trước (budget riêng ALWAYS_ON_CAP, mặc định 3 000 token); phần còn lại sort theo score,
   greedy thêm cho đến khi Σ estimated_tokens(body) ≤ budget (mặc định 12 000 token on-demand bodies);
   chỉ metadata ~100 token/skill thực sự bị nạp lúc đầu, nên cap thứ hai: ≤ 20 skill trong listing.
5. Ghi quyết định: task_runs.skill_selection = [{slug, version, score, reason[]}] để UI "Why this skill?" và audit.
```

Ngưỡng: skill có score < 0.15 và không phải gán trực tiếp ⇒ loại. Người dùng có thể override thủ công khi tạo task ("Force include / exclude skill"). Khi chưa có embedding, trọng số 0.20 dồn sang keyword. Embedding model chọn tại `DATABASE_DESIGN.md` (pgvector) — **[Giả định]** sẽ dùng local embedding để không gửi nội dung skill ra ngoài.

### 4.6 Import / Export

**[Đề xuất]**

- **Export**: `GET /skills/:slug/versions/:v/export` → `.zip` chứa đúng layout §4.1 (kể cả `skill.yaml`). Vì `SKILL.md` chuẩn spec, file có thể thả thẳng vào `~/.claude/skills/` hoặc `.agents/skills/` của bất kỳ tool nào.
- **Import**: nguồn = zip upload, đường dẫn thư mục local, git URL (subpath), hoặc URL marketplace (`npx skills add` style). Pipeline: giải nén vào thư mục cách ly → lint (§4.7) → quét script (§4.13) → tạo draft → người dùng review → publish. Nếu thiếu `skill.yaml`, hệ thống suy ra `category=custom`, `rules=[]`, hỏi người dùng gán category.
- Import từ `.claude/skills` hoặc `.agents/skills` của repo mà người dùng kết nối: quét và đề xuất "Found 4 skills in repo — import?" (không tự import).

### 4.7 Validation / Lint

Chạy ở mọi lần save draft và bắt buộc pass trước publish. **[Đề xuất]** Rule set (`error` chặn publish, `warn` cho qua):

| ID | Mức | Kiểm tra |
|---|---|---|
| L01 | error | `name` khớp regex spec, == tên thư mục == `slug` |
| L02 | error | `description` 1–1024 ký tự; warn nếu > 1 536 cộng `when_to_use` (Claude Code cắt) |
| L03 | error | Frontmatter chỉ chứa 6 field spec (`paths`/`when_to_use` cho Claude Code được tự chèn khi projection, không lưu trong source) |
| L04 | error | `SKILL.md` ≤ 500 dòng; body ≤ 5 000 token ước tính |
| L05 | error | Mọi link tương đối trong `SKILL.md`/`skill.yaml` trỏ tới file tồn tại trong gói; không `..` |
| L06 | warn | Tham chiếu sâu > 1 cấp (file trong references lại link file khác) |
| L07 | error | `rules[].id` duy nhất; mỗi rule `must` xuất hiện trong body (lint tự render section `## Mandatory rules`) |
| L08 | error | `required_tools` ⊆ catalog tool của hệ thống |
| L09 | error | Không có secret pattern (AWS key, JWT, `sk-…`, private key) trong bất kỳ file nào |
| L10 | error | `scripts/` chỉ chứa phần mở rộng allowlist (`.sh .py .js .ts .ps1`), có shebang, không > 64 KiB |
| L11 | warn | Reference file > 32 KiB (khuyến nghị tách) |
| L12 | warn | Description không nêu "when to use" (heuristic: thiếu `use when`/`khi`) |
| L13 | error | Tổng always-on text của agent (sau khi gán) > 8 KiB với Codex |
| L14 | warn | Reference chứa pattern prompt-injection phổ biến ("ignore previous instructions", "you are now…") |
| L15 | info | Chạy `skills-ref validate` (nếu cài) và đính kết quả |

Lint output lưu vào `skill_versions.lint_report` JSONB và hiển thị trong editor.

### 4.8 "Test Skill" harness

Mục tiêu: trả lời "agent có *tuân thủ* skill này không" bằng probe thật qua provider, không chỉ kiểm tra file tồn tại.

**[Đề xuất]** Định dạng `tests/probe-*.yaml`:

```yaml
id: new-endpoint
prompt: "Add a GET /users/:id endpoint that returns a user profile to this NestJS project."
fixture: fixtures/nest-minimal        # repo mẫu nhỏ (zip trong assets/) hoặc tham chiếu template chung
provider: [claude-code, codex]
max_turns: 15
budget_usd: 0.50
rubric:
  - rule: R1   # tham chiếu rules[] — tự động sinh câu hỏi grader
  - rule: R2
  - rule: R5
  - custom: "Response mentions Swagger decorators"
    weight: 0.5
asserts:                              # kiểm tra cứng trên output/worktree
  - file_exists: src/users/users.module.ts
  - grep: { file: "src/users/users.controller.ts", pattern: "@ApiOperation" }
  - no_grep: { file: "src/app.module.ts", pattern: "UsersController" }
```

Luồng chạy:

1. Tạo worktree tạm từ fixture; projection skill **đang test** (đúng version) cho provider.
2. Spawn provider ở chế độ non-interactive, sandbox (`--restricted`/`--sandbox workspace-write` trên bản copy fixture), `--max-turns`, `--max-budget-usd`.
3. Thu transcript (stream-json / JSONL) + diff worktree.
4. **Grader** = provider (`claude -p --json-schema`) hoặc Codex `--output-schema` nhận (rule text, transcript, diff) → JSON `{rule_id, pass, evidence, confidence}`; cộng với `asserts` cứng.
5. Score = Σ(weight·pass)/Σweight; lưu `skill_evaluations`; hiển thị bảng rule × pass/fail + evidence. Ngưỡng "pass" mặc định 0.8 **[Cần người dùng phê duyệt]**.
6. **Baseline mode** (tùy chọn): chạy cùng probe *không có* skill để đo delta — chứng minh skill có tác dụng.

Chi phí: mỗi probe ≈ 1 task nhỏ; UI phải hiển thị ước tính trước khi chạy và yêu cầu xác nhận.

### 4.9 Verify "agent đã nhận skill" (MVP D5)

Hai mức **[Đề xuất]**:

- **Mức 1 — Static verify (ngay khi gán, không tốn token)**: Skill Service chạy projection *dry-run* cho provider hiện tại của agent vào thư mục tạm, kiểm tra file tồn tại, hash khớp, lint pass, always-on size trong giới hạn → `agent_skills.verified_at`. Với Claude Code có thể thêm `claude --bare -p --add-dir <tmp> --max-turns 1 --output-format json "List the skills available to you by name"` (chi phí ~1 request) và so sánh danh sách — **[Giả định]** cần xác nhận listing skills có xuất hiện trong `init` event của stream-json; nếu không, dùng prompt liệt kê.
- **Mức 2 — Runtime verify (mỗi task)**: khi session bắt đầu, Provider Adapter phát `SKILL_LOADED` cho từng skill có trong snapshot; với Claude Code dùng hook `SessionStart` (ghi danh sách skill đã discover vào share dir) hoặc đọc `init` event; với Codex parse `thread.started` + 1 probe nhỏ nếu `verify_strict=true`. Kết quả hiển thị badge "Loaded 5/5 skills" trên task.

### 4.10 Teach Agent pipeline

Yêu cầu owner **[Đã xác minh]**: instruction → parse rules → suggest skill (new/existing) → save → assign → verify → optional evaluation. Không bao giờ tự lưu.

```
[1] Input          Người dùng nhập instruction tự nhiên (+ chọn agent đích, tùy chọn ngữ cảnh: repo, department)
[2] Extract        Gọi provider ở -p mode với JSON schema (xem dưới) → {rules[], suggested_category, suggested_slug, triggers[], tools[]}
[3] Match          Tìm skill hiện có: slug/alias trùng, cosine(description), overlap rule text ≥ 0.7 → top-3 gợi ý "Thêm vào skill X" hoặc "Tạo skill mới"
[4] Conflict       So mỗi rule mới với rules[] của (a) skill đích, (b) mọi skill always-on của agent đích:
                   - duplicate (sim ≥ 0.9)  → gợi ý bỏ
                   - contradiction (LLM judge trả `contradicts: R3`) → bắt buộc người dùng chọn giữ cái nào / set severity
                   - override cùng chủ đề khác severity → cảnh báo
[5] Review UI      Bảng rule (editable: text, severity, check), category, slug, "render preview" của SKILL.md section sẽ được thêm
[6] Save           Tạo draft (skill mới hoặc version mới của skill cũ) → lint → publish (hoặc gửi approve nếu người dùng không có quyền)
[7] Assign         Gán cho agent đích (và hỏi "gán cho cả department?"), mode always-on/on-demand do người dùng chọn
                   (mặc định: rule `must` ⇒ always-on fragment, phần còn lại on-demand)
[8] Verify         §4.9 mức 1 ngay lập tức → `AGENT_TAUGHT` event
[9] Evaluate (opt) Chạy probe tự sinh từ instruction (hệ thống đề xuất 1 prompt probe, người dùng sửa) → §4.8
```

JSON schema cho bước [2] (dùng với `claude -p --json-schema` hoặc `codex exec --output-schema`) **[Đề xuất]**:

```json
{
  "type": "object",
  "required": ["rules", "suggested_slug", "suggested_category", "summary"],
  "properties": {
    "summary": { "type": "string", "maxLength": 300 },
    "suggested_slug": { "type": "string", "pattern": "^[a-z0-9]+(-[a-z0-9]+)*$" },
    "suggested_category": { "enum": ["development","architecture","database","testing","security","devops","custom"] },
    "triggers": { "type": "array", "items": { "type": "string" } },
    "required_tools": { "type": "array", "items": { "type": "string" } },
    "rules": {
      "type": "array", "minItems": 1,
      "items": {
        "type": "object", "required": ["text", "severity"],
        "properties": {
          "text": { "type": "string" },
          "severity": { "enum": ["must", "should", "may"] },
          "check": { "type": "string", "description": "How a reviewer could verify compliance" },
          "scope_hint": { "type": "string", "description": "e.g. file glob or layer" }
        }
      }
    },
    "ambiguities": { "type": "array", "items": { "type": "string" } }
  }
}
```

Ví dụ owner: *"When building NestJS APIs always use modular architecture, DTO validation, exception filters, logging and Swagger"* → 5 rule R1–R5 như §4.1.2, slug `nestjs-modular-api`, category `development`, triggers `[nestjs, api, controller]`, match seed skill `nestjs-modular-api` (nếu có) → gợi ý "append rules → v1.2.0" hoặc "create new".

Bước [2] chạy với `--bare --tools "" --max-turns 1` (không tool) để rẻ và an toàn; prompt hệ thống có few-shot song ngữ Việt/Anh. Nếu provider lỗi/timeout → fallback heuristic (tách câu theo `and`/`,`/`always`/`never`) và đánh dấu "cần kiểm tra".

### 4.11 Agent Memory model (tách biệt với skill)

Skill = tri thức *được thiết kế*, dùng chung. Memory = tri thức *phát sinh từ vận hành*, gắn với agent/project. **[Đã xác minh — owner]**: không tự persist từ hội thoại; phải được quản lý/duyệt/sửa/xóa.

**[Đề xuất]** Bảng `agent_memories` (chi tiết trong `DATABASE_DESIGN.md`):

| Trường | Ý nghĩa |
|---|---|
| `type` | `episodic` (ghi chú task: "Task #231: repo X dùng pnpm, test chạy 4 phút"), `semantic` (sự thật về dự án/org: "Service auth deploy ở cluster eu-1"), `procedural` (quy tắc hành vi: "Luôn chạy `pnpm lint` trước khi báo xong") |
| `scope_type / scope_id` | `agent` / `project` / `org` |
| `status` | `proposed` → `approved` → `active` → `archived`; `rejected` |
| `source` | `agent_proposal` (agent đề xuất cuối task qua structured output), `user` (người dùng viết), `teach` (từ Teach Agent khi người dùng chọn "memory" thay vì "skill") |
| `content_md`, `tags[]`, `confidence`, `expires_at`, `pinned` | |
| `evidence_ref` | task_run id / message id tạo ra đề xuất |

Workflow:
1. Cuối mỗi task, Provider Adapter có thể yêu cầu agent trả thêm block `memory_proposals[]` (structured output) — tối đa 5, mỗi cái ≤ 300 ký tự. Lưu với `status=proposed`, **không** inject vào task kế tiếp.
2. Approval Center hiển thị đề xuất; người dùng approve/edit/reject. Procedural memory ở scope `org`/`project` cần quyền `skills:approve` (như publish skill) vì ảnh hưởng nhiều agent.
3. Retention: `episodic` mặc định hết hạn sau 30 ngày (configurable) và giữ tối đa 50/agent; `semantic` không hết hạn nhưng có "last confirmed" và nhắc re-confirm sau 90 ngày; `procedural` không hết hạn. **[Cần người dùng phê duyệt]**
4. Injection: memory active được render thành file riêng `AIW_MEMORY.md` trong share dir và một section trong fragment (`<!-- AIW:BEGIN memory -->`), giới hạn 2 000 token, chọn theo scope ⊇ task + tag match + recency; **không trộn** vào skill để người dùng luôn phân biệt "kiến thức thiết kế" và "kinh nghiệm tích lũy". Procedural memory được hiển thị trong Teach Agent như nguồn gợi ý "nâng lên thành skill".
5. Xóa agent ⇒ memory scope agent bị archive (không xóa cứng trong 30 ngày).

### 4.12 Permissions

**[Đề xuất]** Permission string (map vào RBAC của `SECURITY.md`):

| Permission | Owner | Manager | Member | Viewer |
|---|---|---|---|---|
| `skills:read` | ✓ | ✓ | ✓ | ✓ |
| `skills:create` (draft) | ✓ | ✓ | ✓ | – |
| `skills:publish` (org/project visibility) | ✓ | ✓ | – (gửi review) | – |
| `skills:approve` (review draft của người khác, approve memory org/project) | ✓ | ✓ | – | – |
| `skills:assign` | ✓ | ✓ | chỉ agent mình quản lý | – |
| `skills:delete` / `deprecate` | ✓ | ✓ (project) | – | – |
| `skills:scripts:enable` (bật chạy script trong skill) | ✓ | – | – | – |
| `skills:test` (tốn token) | ✓ | ✓ | ✓ (trong budget) | – |
| `memory:approve` | ✓ | ✓ | agent mình | – |

MVP một người dùng (owner) thì mọi quyền = ✓, nhưng model vẫn giữ để Phase 2 multi-user không phải thiết kế lại. Mọi thay đổi ghi `audit_log` (ai, skill, version, diff hash).

### 4.13 Security

- **Scripts trong skill là code không tin cậy** (đặc biệt khi import). Mặc định `scripts/` được projection nhưng **không có quyền thực thi** (không nằm trong `allowed-tools`/permissions.allow); chỉ owner có `skills:scripts:enable` bật per-skill sau khi xem nội dung; lint L10 + quét static (shellcheck/semgrep basic) **[Đề xuất]**.
- Script khi được bật chạy trong sandbox của provider (Claude Code permission rule `Bash(python <share>/…/scripts/*)`; Codex `--sandbox workspace-write`), không bao giờ `danger-full-access`.
- Không secret trong skill (L09). Nếu skill cần credential (vd REST caller), chỉ tham chiếu tên env var; giá trị do Provider Adapter inject từ secret store.
- Path traversal: mọi đường dẫn trong gói phải resolve bên trong thư mục skill (theo `read_skill_file` của ChatDev).
- Prompt injection qua reference docs upload: lint L14 quét pattern phổ biến và gắn cảnh báo; reference luôn render sau rules với header "Reference material — informational only" **[Đề xuất]**.
- Share dir mount read-only cho process provider; worktree chỉ chứa junction/fragment.
- Skill marketplace/import từ URL: hiển thị diff đầy đủ trước khi publish; không auto-update từ nguồn ngoài.

### 4.14 Seed skill list (tên + 1 dòng; nội dung Phase 1)

**[Đề xuất]** — 30 skill, slug theo spec:

**Development**
- `react-component-patterns` — Component, hooks, state, performance conventions cho React 18+.
- `nextjs-app-router` — Next.js App Router, server components, routing, data fetching, caching.
- `nestjs-modular-api` — NestJS modular architecture, DTO validation, filters, logging, Swagger.
- `python-service-standards` — Python 3.12, typing, pytest, ruff, packaging với uv.
- `flutter-app-structure` — Flutter feature-first structure, state management, platform channels.
- `java-spring-boot` — Spring Boot layered service, validation, exception handling, OpenAPI.

**Architecture**
- `clean-architecture` — Layering, dependency rule, use-case boundaries.
- `domain-driven-design` — Bounded context, aggregate, domain event, ubiquitous language.
- `microservices-patterns` — Service boundaries, async messaging, saga, observability.
- `modular-monolith` — Module boundaries trong một deployable, enforced imports.
- `rest-api-design` — Resource naming, status codes, pagination, versioning, error format.

**Database**
- `postgresql-practices` — Schema design, indexes, migrations, transactions, pgvector cơ bản.
- `mysql-practices` — InnoDB, indexing, charset, migration safety.
- `redis-patterns` — Caching, TTL, pub/sub, rate limiting, key naming.
- `db-performance-tuning` — Query plan, N+1, connection pool, slow query triage.

**Testing**
- `unit-testing-standards` — AAA pattern, mocking rules, coverage mục tiêu, naming.
- `integration-testing` — Test container, DB fixture, contract test.
- `playwright-e2e` — Page object, locator strategy, trace, flaky test policy.
- `security-testing` — SAST/DAST checklist, dependency audit, auth test cases.

**Security**
- `owasp-top-10` — Checklist OWASP 2021 áp dụng khi review/implement.
- `authentication-patterns` — Password, session, JWT, OAuth2/OIDC, MFA.
- `authorization-patterns` — RBAC/ABAC, policy enforcement point, least privilege.
- `secure-coding` — Input validation, output encoding, secret handling, logging an toàn.

**DevOps**
- `docker-best-practices` — Multi-stage build, non-root, image size, healthcheck.
- `kubernetes-deployment` — Manifest/Helm conventions, probes, resources, secrets.
- `github-actions-ci` — Workflow structure, caching, matrix, permissions, reusable workflows.
- `linux-ops-basics` — Shell safety, systemd, permissions, log inspection.
- `nginx-configuration` — Reverse proxy, TLS, headers, rate limit.

**Cross-cutting (always-on candidates)**
- `aiw-task-protocol` — Cách agent báo cáo tiến độ, trả kết quả có cấu trúc, đề xuất memory (nội bộ hệ thống).
- `code-review-checklist` — Checklist review chung trước khi mark task done.

### 4.15 API surface (sketch)

Prefix `/api/v1`. Auth + RBAC theo `SECURITY.md`. **[Đề xuất]**

```
# Library
GET    /skills?category=&q=&status=&visibility=         # listing + search (BM25/embedding)
POST   /skills                                          # tạo draft {slug, name, category, description}
GET    /skills/:slug                                    # metadata + latest + versions summary
PATCH  /skills/:slug                                    # đổi name/category/tags/visibility
DELETE /skills/:slug                                    # soft delete (deprecate all)
GET    /skills/:slug/draft                              # nội dung draft (SKILL.md, skill.yaml, file tree)
PUT    /skills/:slug/draft/files/*path                  # ghi/sửa file trong draft (SKILL.md, references/…)
DELETE /skills/:slug/draft/files/*path
POST   /skills/:slug/draft/uploads                      # multipart: md/pdf/docx → references/ (+ assets/originals)
POST   /skills/:slug/draft/lint                         # trả lint report
POST   /skills/:slug/draft/publish                      # {bump: major|minor|patch, changelog} → skill_version
GET    /skills/:slug/versions
GET    /skills/:slug/versions/:v                        # nội dung immutable
GET    /skills/:slug/versions/:v/export                 # zip
POST   /skills/import                                   # {source: zip|path|git|url} → draft

# Assignment
GET    /agents/:id/skills                               # effective skills (direct + scope), pin, verified_at, last_loaded_at
POST   /agents/:id/skills                               # {skill_id, pinned_version_id?, mode}
PATCH  /agents/:id/skills/:skill_id                     # pin/unpin, mode
DELETE /agents/:id/skills/:skill_id
POST   /agents/:id/skills/verify                        # static verify (§4.9 mức 1) → report
GET    /agents/:id/skills/selection-preview?task_id=    # chạy §4.5 không spawn, để UI "Why this skill?"
POST   /departments/:id/skills                          # scope assignment

# Test & evaluation
POST   /skills/:slug/versions/:v/tests                  # chạy probe(s) {probe_ids?, provider, agent_id?} → evaluation job
GET    /skill-evaluations/:id                           # kết quả + transcript ref

# Teach Agent
POST   /teach/extract                                   # {instruction, agent_id?} → {rules, suggestions, conflicts}
POST   /teach/commit                                    # {extracted (đã sửa), target: {new|skill_id}, assign_to[], mode, run_eval?}

# Memory
GET    /memories?scope=&type=&status=
POST   /memories                                        # user-authored
PATCH  /memories/:id                                    # edit / approve / reject / archive
DELETE /memories/:id
GET    /agents/:id/memories/effective                   # những gì sẽ được inject
```

### 4.16 Events

Phát qua event bus (`EVENT_SYSTEM.md`), payload luôn có `actor`, `org_id`, `at`. **[Đề xuất]**

| Event | Payload chính | Phát khi |
|---|---|---|
| `SKILL_CREATED` | skill_id, slug, category | tạo draft |
| `SKILL_UPDATED` | skill_id, changed_fields | sửa metadata/draft |
| `SKILL_LINTED` | skill_id, errors, warnings | lint xong |
| `SKILL_PUBLISHED` | skill_id, version, content_hash, bump | publish |
| `SKILL_DEPRECATED` | skill_id, version | deprecate |
| `SKILL_IMPORTED` | skill_id, source | import thành draft |
| `SKILL_ASSIGNED` / `SKILL_UNASSIGNED` | agent_id or scope, skill_id, pinned_version, mode | gán/bỏ |
| `SKILL_VERIFIED` | agent_id, skill_id, level (static/runtime), ok, report_ref | §4.9 |
| `SKILL_SELECTED` | task_run_id, agent_id, selection[] | sau §4.5 |
| `SKILL_LOADED` | task_run_id, agent_id, slug, version, hash, provider | runtime verify |
| `SKILL_LOAD_FAILED` | task_run_id, slug, reason | projection/verify lỗi |
| `SKILL_TEST_STARTED` / `SKILL_TEST_COMPLETED` | evaluation_id, score, per_rule[] | §4.8 |
| `AGENT_TEACH_EXTRACTED` | instruction_id, agent_id, rule_count, conflicts | Teach bước [2–4] |
| `AGENT_TAUGHT` | instruction_id, agent_id, skill_id, version, verified | Teach bước [8] |
| `MEMORY_PROPOSED` / `MEMORY_APPROVED` / `MEMORY_REJECTED` / `MEMORY_ARCHIVED` | memory_id, type, scope | §4.11 |

Virtual Office có thể lắng nghe `AGENT_TAUGHT` để hiện animation "đang học"; Analytics đếm `SKILL_LOADED` để tính skill usage.

## 5. Quyết định & lý do

| # | Quyết định | Lý do | Trạng thái |
|---|---|---|---|
| D1 | Theo Agent Skills spec nguyên vẹn cho `SKILL.md`; mọi extension vào `skill.yaml` + `metadata.aiw-*` | Spec cấm field lạ; đảm bảo export dùng được ở Claude Code/Codex/OpenHands/claude.ai mà không chuyển đổi | Đề xuất |
| D2 | Rule có cấu trúc (`rules[]`) là đơn vị nguyên tử, body `SKILL.md` render section rule từ đó | Teach Agent, conflict detection, rubric đều cần máy đọc được | Đề xuất |
| D3 | Projection qua share dir + `--add-dir` (Claude) / junction `.agents/skills` (Codex) thay vì ghi vào repo | Không làm bẩn git, mount read-only, cache theo hash | Đề xuất — cần test junction trên Windows |
| D4 | Always-on ≤ 3 000 token (Claude) / ≤ 8 KiB (Codex), on-demand ≤ 20 skill, body budget 12 000 token | Giới hạn thực tế: Claude cắt description 1 536 ký tự, Codex `project_doc_max_bytes` 32 KiB, `skills.max_context_tokens` ≤ 10 000 | Đề xuất — [Cần người dùng phê duyệt] |
| D5 | Memory tách khỏi skill, luôn `proposed` → approve | Yêu cầu owner: không tự persist | Đã xác minh (yêu cầu) / Đề xuất (cơ chế) |
| D6 | Teach extraction dùng chính provider ở `-p` + JSON schema, không gọi API model riêng | Không thêm API key/chi phí ngoài CLI đã có; structured output hỗ trợ ở cả hai CLI | Đề xuất |
| D7 | Scripts trong skill mặc định tắt, owner bật per-skill | Import từ ngoài = code không tin cậy | Đề xuất |
| D8 | Version immutable + pin per agent, không hot-reload trong session | Reproducibility, audit | Đề xuất |
| D9 | Không sinh legacy `.openhands/microagents/`; OpenHands ngoài MVP | OpenHands đã chuyển sang `.agents/skills` | Đã xác minh |
| D10 | 6 category cố định + `custom`; subcategory tự do | Theo brief owner; tránh taxonomy phình | Đã xác minh (danh sách) / Đề xuất (`custom`) |

## 6. Dependencies

- `DATABASE_DESIGN.md`: bảng `skills`, `skill_versions`, `skill_documents`, `agent_skills`, `skill_evaluations`, `agent_memories`, `audit_log`; pgvector cho embedding (tùy chọn).
- `PROVIDER_ADAPTER.md`: hook bootstrap session (nhận `skill_snapshot`, share dir, lệnh spawn), parser stream-json/JSONL để phát `SKILL_LOADED`, structured-output helper (`--json-schema` / `--output-schema`), junction/exclude trong worktree.
- `EVENT_SYSTEM.md`: event names §4.16.
- `SECURITY.md`: RBAC permission strings §4.12, sandbox policy, secret store.
- `UI_UX_GUIDELINES.md`: khu vực Skills, Approval Center (memory), Agent detail (skills tab).
- Thư viện: `yaml`, `gray-matter` (frontmatter), `pdf-parse`/`mammoth` (upload), bộ ước lượng token, tùy chọn `skills-ref` CLI.
- Windows: `mklink /J` cho junction; Provider Adapter cần fallback copy vật lý nếu junction fail.

## 7. Rủi ro

| Rủi ro | Ảnh hưởng | Giảm thiểu |
|---|---|---|
| Codex không thấy skill qua junction trên Windows hoặc dừng tìm ở repo root | Skill không nạp cho Codex | Test sớm Phase 1; fallback copy vật lý + exclude |
| Claude Code đổi hành vi `--add-dir` với skills | Mất kênh projection chính | Giữ fallback ghi `<worktree>/.claude/skills/`; pin version CLI và test theo release |
| Agent "claim" đã dùng skill nhưng không đọc body | Verify giả | Runtime verify + evaluation probe; thêm rule "không claim nếu chưa activate" (học từ ChatDev) |
| Rule mâu thuẫn giữa nhiều skill always-on | Hành vi agent không ổn định | Conflict detection khi gán (không chỉ khi teach); report "effective rules" per agent |
| Context phình khi người dùng gán 30 skill | Chi phí, chất lượng giảm | Budget cap §4.5, cảnh báo UI, report skill unused (kiểu `/skill-doctor`) |
| PDF upload chứa prompt injection | Agent hành xử lệch | Lint L14, render reference sau rules, cảnh báo |
| Chi phí Test Skill | Người dùng bất ngờ hóa đơn | Ước tính trước, `--max-budget-usd`, giới hạn probe/ngày |
| Teach extraction sai nghĩa (tiếng Việt lẫn Anh) | Rule sai | Review UI bắt buộc, few-shot song ngữ, `ambiguities[]` hiển thị |
| Trùng tên skill giữa project và personal của người dùng trên máy host | Codex "không merge", Claude ưu tiên personal | Prefix không cần (spec cấm namespace), nhưng lint cảnh báo khi slug trùng skill phổ biến; projection chạy với HOME riêng cho agent **[Đề xuất]** |

## 8. Vấn đề chưa giải quyết

1. Claude Code có phát danh sách skill đã discover trong `init` event của stream-json hay không — chưa xác minh; ảnh hưởng cách verify mức 1 không tốn prompt. **[Giả định]**
2. Codex: đường dẫn skill cũ `~/.codex/skills` còn được đọc không; `agents/openai.yaml` có được Codex CLI (không phải ChatGPT desktop) dùng cho `dependencies.tools` không. **[Giả định]**
3. Embedding local nào (model, kích thước) — thuộc `DATABASE_DESIGN.md`; §4.5 phải chạy được *không* có embedding.
4. Có cho phép skill *per-project* (visibility project) ngay MVP hay chỉ org-wide? Ảnh hưởng UI filter và cột `skills.project_id`. **[Cần người dùng phê duyệt]**
5. Có cần "skill pack" (gán một nhóm skill theo role: "Backend Engineer pack") ở MVP? Đề xuất Phase 2; schema `skill_assignments_scope` đã dự phòng.
6. Chính sách retention memory (30/90 ngày, 50 episodic/agent) — con số cần owner chốt.
7. Subagent definitions (`.claude/agents/`) sinh từ role agent hay từ skill? Đề xuất: từ role (thuộc Agent Lifecycle doc), skill chỉ có thể *khuyến nghị* subagent qua `context: fork`.
8. Giới hạn kích thước gói/text (§4.2) và ngưỡng pass evaluation 0.8 (§4.8). **[Cần người dùng phê duyệt]**

## 9. Acceptance Criteria (map MVP nhóm D)

| ID | Tiêu chí | Nhóm D |
|---|---|---|
| AC-D1-1 | Tạo skill mới qua UI với name/description/category/instructions; lint pass; publish tạo `skill_versions` 1.0.0 và thư mục `data/skills/<slug>/1.0.0/` đúng layout §4.1; export zip được `skills-ref validate` chấp nhận | D1 Create |
| AC-D1-2 | Upload `.md`, `.pdf`, `.docx` vào references; PDF được trích text; file gốc giữ trong `assets/originals/` | D1 Create |
| AC-D1-3 | Thêm ≥1 rule `must`, 1 checklist, 1 code example, `allowed-tools`; body `SKILL.md` chứa section `## Mandatory rules` sinh từ rules | D1 Create |
| AC-D2-1 | Sửa skill đã publish tạo draft, publish lại bump version; bản cũ vẫn đọc được; CHANGELOG cập nhật | D2 Edit |
| AC-D2-2 | Lint chặn publish khi vi phạm L01–L05, L07–L10, L13; hiển thị lỗi theo dòng | D2 Edit |
| AC-D3-1 | Gán skill cho agent (direct) và cho department; `GET /agents/:id/skills` trả effective list đúng; pin version hoạt động | D3 Assign |
| AC-D3-2 | Một skill gán cho ≥2 agent, một agent có ≥5 skill; tạo task cho agent đó → `SKILL_SELECTED` chỉ chứa skill liên quan trong budget, kèm `reason[]` | D3 Assign |
| AC-D3-3 | Session Claude Code: share dir chứa `.claude/skills/<slug>/SKILL.md` đúng hash; `CLAUDE.md` worktree có vùng marker; file sinh ra không xuất hiện trong `git status` | D3 Assign |
| AC-D3-4 | Session Codex: `<worktree>/.agents/skills/<slug>` tồn tại (junction), `AGENTS.md` marker ≤ 8 KiB; `codex exec` liệt kê được skill khi hỏi | D3 Assign |
| AC-D4-1 | Teach: nhập câu ví dụ của owner → trả ≥4 rule đúng ý, gợi ý slug/category, hiển thị review UI, không có gì được lưu trước khi bấm Save | D4 Teach |
| AC-D4-2 | Teach với rule mâu thuẫn rule hiện có → UI bắt buộc giải quyết conflict trước khi Save | D4 Teach |
| AC-D4-3 | Save → skill (mới/version mới) + assign + `AGENT_TAUGHT` event; tùy chọn chạy evaluation trả score | D4 Teach |
| AC-D5-1 | Sau assign, static verify trả `ok` và `verified_at` được set; khi task chạy, `SKILL_LOADED` phát cho từng skill trong snapshot và UI task hiển thị "Loaded n/n" | D5 Verify |
| AC-D5-2 | Test Skill: chạy 1 probe trên Claude Code và Codex, trả bảng rule × pass/fail với evidence; chi phí ≤ `budget_usd` | D5 Verify |
| AC-M-1 | Memory: cuối task có đề xuất `proposed`; không được inject cho đến khi approve; approve → xuất hiện trong `AIW_MEMORY.md` của session kế tiếp; reject/delete hoạt động | Memory |
| AC-S-1 | Skill import có `scripts/` → script không thực thi được cho đến khi owner bật; lint L09 chặn file chứa secret giả | Security |
