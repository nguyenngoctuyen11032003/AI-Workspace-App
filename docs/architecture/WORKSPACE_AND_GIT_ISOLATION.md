# Workspace Manager & Git Worktree Isolation

> **Tác giả:** Agent 14 (DevSecOps & Infrastructure) · **Phase:** 0 (documentation only) · **Trạng thái:** Draft v0.1 · **Ngày:** 2026-10-30
>
> Tài liệu liên quan: `docs/SECURITY.md` (A09 – threat model), `docs/DATABASE_DESIGN.md` (A08 – schema cho `tasks`, `workspaces`, `worktrees`), `docs/architecture/INFRASTRUCTURE_DESIGN.md` (A14 – sandbox executor).
>
> Nhãn trạng thái dùng trong tài liệu: **[Đã xác minh]** (đã chạy lệnh trên máy dev), **[Đề xuất]** (thiết kế do A14 đề xuất), **[Giả định]** (chưa kiểm chứng), **[Cần người dùng phê duyệt]**.

## 1. Mục đích

Định nghĩa cách AI WORKSPACE cấp phát "chỗ làm việc" cho từng AI coding agent (Claude Code, Codex CLI) sao cho:

1. Nhiều agent chạy **song song** trên cùng một project mà không ghi đè file của nhau.
2. Mọi thay đổi của agent đều được **truy vết** (branch, commit, trailer) và **kiểm tra** trước khi hợp nhất.
3. Agent chỉ được phép sửa đúng tập file được giao (**ownership manifest**), và vi phạm bị chặn ở cả thời điểm chạy (hooks/sandbox) lẫn sau khi chạy (diff validation).
4. Hoạt động ổn định trên **Windows 11** (NTFS, Git for Windows) và vẫn portable sang Linux/macOS khi chạy chế độ self-hosted.

## 2. Phạm vi

**Trong phạm vi:** Workspace Manager, Worktree Manager, Ownership Manifest, enforcement points, merge/integration strategy, conflict handling, commit conventions, cleanup/quota, Windows gotchas, interface sketches (TypeScript).

**Ngoài phạm vi:** threat model tổng thể (A09), schema DB chi tiết (A08), orchestration/task scheduling (agent khác), UI hiển thị diff (frontend).

## 3. Môi trường đã xác minh

| Hạng mục | Kết quả | Trạng thái |
|---|---|---|
| Git | `git version 2.56.0.windows.1`, hỗ trợ `git worktree` đầy đủ | [Đã xác minh] |
| `core.autocrlf` (system gitconfig) | `true` | [Đã xác minh] – xem §11, cần override thành `false` ở repo level |
| `core.symlinks` (system gitconfig) | `false` | [Đã xác minh] – không dùng symlink trong workspace |
| `core.longpaths` | chưa đặt (mặc định `false`) | [Đã xác minh] – cần bật, xem §11 |
| `gh` CLI | `gh version 2.102.0` | [Đã xác minh] |
| Ổ đĩa D: | 121 GB, còn trống ~109 GB | [Đã xác minh] |
| Claude Code CLI | `2.1.312` | [Đã xác minh] |
| Codex CLI | chưa cài | [Đã xác minh] |

## 4. Directory layout

```
<WORKSPACE_ROOT>                      # mặc định D:\aiw  [Cần người dùng phê duyệt]
├── workspaces/
│   └── <org-slug>/
│       └── <project-slug>/
│           ├── repo.git/             # bare repository (nguồn sự thật, không có working tree)
│           ├── project.json          # metadata: default branch, integration branch, stack, quota
│           └── manifests/
│               └── <task-id>.manifest.json   # ownership manifest snapshot cho từng task
├── worktrees/
│   └── <task-id>/                    # 1 worktree = 1 task = 1 branch = 1 agent process
│       ├── .git                      # file trỏ về repo.git/worktrees/<task-id>
│       ├── .ai-workspace/            # thư mục do platform tạo, agent read-only
│       │   ├── manifest.json         # bản copy manifest (để hook đọc nhanh)
│       │   ├── task.md               # task brief render sẵn
│       │   └── hooks/                # script hook (PreToolUse) cho Claude Code
│       ├── .claude/settings.json     # permission + hooks do platform sinh (xem §7)
│       └── ... source files ...
├── artifacts/
│   └── <task-id>/                    # log, test report, diff.patch, sbom (xem INFRASTRUCTURE_DESIGN)
└── tmp/
```

**Quyết định chính [Đề xuất]:**

- `repo.git` là **bare repo**: tránh tình trạng "main worktree" của một clone thông thường bị agent sửa trực tiếp; mọi working tree đều là worktree ngang hàng.
- `worktrees/` nằm **ngoài** `workspaces/` để đường dẫn ngắn (Windows path length) và để quota/cleanup tính độc lập.
- `<task-id>` là ULID; thư mục dùng 8 ký tự cuối (ví dụ `worktrees/4K7QX2MZ`) để path ngắn; mapping sang tên dễ đọc nằm trong DB (A08).
- Một project có thể có remote GitHub (push/PR qua `gh`) hoặc hoàn toàn local; `repo.git` luôn là origin đối với các worktree.

## 5. Mô hình "one task = one worktree = one branch"

| Thuộc tính | Quy ước |
|---|---|
| Branch name | `agent/<agent-slug>/<task-id>` ví dụ `agent/claude-backend/01JD4K7QX2MZ` |
| Base branch | `integration/<project-slug>` (mặc định) hoặc branch của task cha khi có `dependsOn` |
| Worktree path | `<WORKSPACE_ROOT>/worktrees/<task-short-id>` |
| Lệnh tạo | `git -C repo.git worktree add -b agent/<slug>/<task-id> <path> <base-ref>` |
| Lệnh xoá | `git -C repo.git worktree remove --force <path>` rồi `git worktree prune` |
| Giới hạn | Mỗi project tối đa `maxConcurrentWorktrees` (mặc định 6) [Cần người dùng phê duyệt] |

Lý do một branch mỗi task: diff validation, revert, và attribution đều theo đơn vị task; PR/merge có thể tạo trực tiếp từ branch; một agent process chỉ nhìn thấy đúng một worktree.

## 6. Ownership Manifest

Mỗi task mang một manifest (JSON) do PM agent hoặc người dùng tạo, platform lưu kèm task trong DB (A08) và snapshot ra `manifests/<task-id>.manifest.json`.

```json
{
  "schemaVersion": 1,
  "taskId": "01JD4K7QX2MZ",
  "agent": { "slug": "claude-backend", "cli": "claude" },
  "writable": ["apps/api/src/orders/**", "apps/api/test/orders/**"],
  "readOnly": ["apps/api/src/common/**", "packages/shared/**", "docs/**"],
  "forbidden": [".env*", "**/*.pem", "**/*.key", ".git/**", ".ai-workspace/**", ".claude/**", "package-lock.json", "pnpm-lock.yaml"],
  "allowNewFilesUnder": ["apps/api/src/orders/**", "apps/api/test/orders/**"],
  "allowDeletes": false,
  "maxChangedFiles": 60,
  "maxDiffBytes": 2000000,
  "allowedCommands": ["pnpm test --filter api", "pnpm lint --filter api", "pnpm typecheck"],
  "deniedCommandPatterns": ["git push", "git reset --hard", "rm -rf", "curl ", "wget ", "docker "]
}
```

**Quy tắc ưu tiên:** `forbidden` > `readOnly` > `writable`. File không khớp glob nào → mặc định **read-only**. Lock file và cấu hình toolchain mặc định forbidden; nếu task cần thêm dependency thì manifest phải liệt kê rõ (PM phê duyệt). Glob dùng cú pháp `picomatch`/`minimatch` với `dot: true`, path luôn dùng `/`.

## 7. Enforcement points (phòng thủ nhiều lớp)

### 7.1 Lớp 1 – cấu hình CLI của agent (runtime)

> **[CẦN QUYẾT ĐỊNH, CF-014/Q-17/D-004 (OWR-2 2026-10-30)]**: thiết kế dưới đây (`acceptEdits`, `.claude/settings.json` trong worktree, hook `command`, `claude -p` spawn) mâu thuẫn với hướng đã chốt: Claude Agent SDK là adapter chính, mode mặc định `default` + whitelist, `canUseTool` + PreToolUse policy in-process (SECURITY.md §5.1), settings không ghi file trong worktree. Giữ nguyên nội dung gốc làm biến thể fallback/raw CLI cho đến khi tác giả doc viết lại; xem DOC_SYNC_LOG_OWR2_A.md. Không mount thư mục credential hoặc Docker socket vào Agent; fail closed nếu sandbox không hoạt động.

**Claude Code** (platform sinh `<worktree>/.claude/settings.json` trước khi spawn):

```json
{
  "permissions": {
    "allow": ["Read", "Grep", "Glob", "Edit", "Write", "Bash(pnpm test:*)", "Bash(pnpm lint:*)", "Bash(git status:*)", "Bash(git diff:*)", "Bash(git add:*)", "Bash(git commit:*)"],
    "deny": ["Bash(git push:*)", "Bash(git reset:*)", "Bash(rm -rf:*)", "Bash(curl:*)", "Bash(wget:*)", "Bash(docker:*)", "Read(./.env*)", "Read(**/*.pem)", "Write(./.ai-workspace/**)", "Write(./.claude/**)", "WebFetch", "WebSearch"],
    "defaultMode": "acceptEdits"
  },
  "hooks": {
    "PreToolUse": [
      { "matcher": "Edit|Write|MultiEdit|NotebookEdit", "hooks": [{ "type": "command", "command": "node ./.ai-workspace/hooks/check-manifest.mjs", "timeout": 10 }] },
      { "matcher": "Bash", "hooks": [{ "type": "command", "command": "node ./.ai-workspace/hooks/check-command.mjs", "timeout": 10 }] }
    ]
  }
}
```

- `check-manifest.mjs` đọc `tool_input.file_path` từ stdin (JSON), resolve về path tương đối worktree, so với manifest; vi phạm → exit code 2 + thông điệp lý do trên stderr (Claude Code chặn tool call và đưa lý do về cho model). [Giả định: hành vi exit code 2 theo tài liệu hooks hiện hành; cần test lại trên 2.1.312]
- Thứ tự xử lý theo tài liệu SDK: PreToolUse hook → ask rules → deny rules → permission mode → allow rules. Vì từng có bug report deny rule không được áp dụng (issue anthropics/claude-code#6699, phiên bản 1.0.93), **hook là lớp bắt buộc**, deny rules là lớp bổ sung. [Đã xác minh qua tài liệu/issue công khai, chưa tái hiện trên máy]
- Spawn với `claude -p --permission-mode acceptEdits --max-turns N --output-format stream-json`; **không** dùng `--dangerously-skip-permissions` trừ khi agent chạy trong container sandbox (INFRASTRUCTURE_DESIGN §6.6).
- Matcher dùng **tên tool chuẩn** (`Edit`, `Write`, `Bash`); tên sai sẽ không khớp và hook im lặng – test case bắt buộc.

**Codex CLI** (khi cài; **hoãn khỏi MVP, D-005/OWR-2, không cài ở phạm vi hiện tại**): spawn với `codex exec --sandbox workspace-write --ask-for-approval never --cd <worktree>` và `config.toml` do platform sinh (`sandbox_mode = "workspace-write"`, `sandbox_workspace_write.writable_roots` = chỉ worktree, `network_access = false`). Codex không có hook cấp file-level như Claude, nên manifest tinh hơn worktree chỉ được enforce ở Lớp 3. [Đề xuất; chi tiết flag cần kiểm tra lại khi cài Codex – xem ENVIRONMENT_SETUP §7]

### 7.2 Lớp 2 – hệ điều hành / container

- Chế độ MVP (local): tiến trình agent chạy với `cwd = worktree`, biến môi trường bị lọc (chỉ pass `PATH`, `HOME`/`USERPROFILE`, `TEMP`, và biến CLI cần); không có biện pháp OS-level ACL ở MVP. [Đề xuất]
- Chế độ hardened (sau MVP): chạy agent CLI trong container với bind mount chỉ worktree (INFRASTRUCTURE_DESIGN §6.6). Ràng buộc: login của CLI phải có trong container (mount `~/.claude` read-only) – xem DEPLOYMENT.md §4.

### 7.3 Lớp 3 – post-hoc diff validation (bắt buộc trước merge)

Sau khi agent kết thúc, Worktree Manager chạy:

1. `git -C <wt> add -A -n` + `git status --porcelain=v2 --untracked-files=all` để liệt kê file thay đổi (kể cả untracked).
2. Với từng path: phân loại theo manifest (`writable`/`readOnly`/`forbidden`/`unmatched`). Đổi tên/xoá kiểm tra `allowDeletes`.
3. Kiểm tra `maxChangedFiles`, `maxDiffBytes`, binary file lạ, file > 5 MB, tên file cấm trên Windows.
4. Quét secret trên diff (gitleaks `detect --no-git` trên bản patch, xem DEVSECOPS_GUIDE §3).
5. Kết quả: `PASS` → cho phép merge; `FAIL` → task chuyển trạng thái `needs_review`, diff vi phạm được lưu `artifacts/<task-id>/violations.json`, không merge tự động. Người dùng có thể **chấp nhận ngoại lệ** (ghi audit log) hoặc trả task về agent.

Lớp 3 là lớp **duy nhất không phụ thuộc vào hành vi CLI bên thứ ba**, nên là lớp quyết định.

## 8. Integration branch strategy

```
agent/<slug>/<task-id>  ──merge --no-ff──▶  integration/<project>  ──PR / approve──▶  main
```

- `integration/<project>` được platform tạo từ `main` khi project khởi tạo; tất cả task base từ đây (hoặc từ branch task cha khi có dependency chain).
- Merge task → integration là `--no-ff` để giữ ranh giới task trong lịch sử.
- `integration → main` chỉ xảy ra khi người dùng bấm **Approve/Release** (nếu có remote: tạo PR bằng `gh pr create`, merge qua GitHub với branch protection – xem CI_CD.md). Không bao giờ tự động.
- [Q-04, D-009, OWR-2 2026-10-30] Agent được merge task → `integration/*` khi vượt qua các gate (Lớp 3 validation, review diff). Merge vào `main` luôn cần Owner approval riêng, hết hiệu lực sau 15 phút (Q-05); Lead Agent không được thay thế quyền phê duyệt của Owner. Mỗi task một worktree/branch riêng, review diff trước khi merge, không tự động merge vào `main`; Owner là người phê duyệt cuối cùng.
- Sau mỗi merge vào integration, các worktree đang chạy **không** bị rebase tự động (tránh phá tiến trình agent); chúng sẽ được rebase lúc chính chúng chuẩn bị merge.

### 8.1 Thứ tự merge theo dependency

- Task có `dependsOn: [taskA, taskB]` chỉ được merge khi A, B đã ở trạng thái `merged`.
- Merge queue là **FIFO theo topological order** của DAG dependency; trong cùng mức thì theo thời điểm hoàn thành.
- Chỉ **một** merge vào `integration/<project>` tại một thời điểm (mutex theo project, hiện thực bằng BullMQ queue với `concurrency: 1` theo `jobId` prefix project, hoặc Redis lock).

### 8.2 Xử lý conflict

1. Thử `git rebase integration/<project>` trong worktree của task (chỉ khi agent đã dừng).
2. Thành công → chạy lại Lớp 3 validation + test nhanh (lint/typecheck trong sandbox) → merge.
3. Thất bại → `git rebase --abort`, tạo task mới loại `merge-conflict` gán cho **agent chủ sở hữu** task gốc (cùng manifest + thêm quyền write cho các file conflict); nếu conflict chạm file ngoài manifest → gán cho PM agent / người dùng.
4. Task gốc ở trạng thái `blocked_conflict`, không bị xoá worktree cho đến khi task conflict đóng.

## 9. Commit conventions & attribution

- Định dạng: Conventional Commits `type(scope): subject`, scope = module trong manifest.
- Trailer bắt buộc (platform thêm bằng `git commit --trailer` hoặc hook `commit-msg` nếu agent tự commit):

```
feat(orders): add order cancellation endpoint

Task-Id: 01JD4K7QX2MZ
Agent: claude-backend (claude-code 2.1.312)
Co-Authored-By: Claude <noreply@anthropic.com>
```

- `user.name`/`user.email` của commit trong worktree được platform đặt theo cấu hình project (`git config --worktree`, cần `extensions.worktreeConfig=true` trong `repo.git`) – người dùng là author, agent nằm trong trailer. Lý do: giữ quyền tác giả với người dùng, vẫn truy vết được agent. [Cần người dùng phê duyệt]
- Nếu agent không commit, platform tự commit toàn bộ thay đổi hợp lệ với message `chore(task): checkpoint <task-id>` cuối mỗi session.
- `commit.gpgsign` tắt trong worktree (agent không có khoá ký); ký commit (nếu bật) chỉ xảy ra ở bước merge do platform thực hiện.

## 10. Cleanup, quota, concurrency

| Chính sách | Giá trị mặc định [Đề xuất] |
|---|---|
| Xoá worktree sau khi task `merged`/`cancelled` | sau 24h (giữ để người dùng xem diff), hoặc ngay nếu disk trống < 10 GB |
| Giữ worktree task `failed`/`needs_review` | cho đến khi người dùng đóng task |
| Quota mỗi project | 10 GB worktrees + 2 GB artifacts; vượt → không tạo task mới, cảnh báo UI |
| Worktree đồng thời mỗi project | 6; toàn hệ thống: 12 (Windows dev box 1 máy) |
| `node_modules` | không commit; mỗi worktree cài riêng; pnpm content-addressable store chung giảm dung lượng |
| Orphan detection | job định kỳ (BullMQ repeatable, 15 phút): `git worktree list --porcelain` so với DB; `git worktree prune`; xoá thư mục mồ côi |

**Git index lock:** Mỗi worktree có index riêng (`repo.git/worktrees/<id>/index`), nên các agent không khoá nhau khi `add/commit`. Tuy nhiên `repo.git` (refs, objects, packed-refs) dùng chung; các thao tác ghi ref (`worktree add`, merge, `gc`, `pack-refs`) phải đi qua **một hàng đợi theo project** trong Worktree Manager. Đặt `gc.auto=0` trong bare repo và chạy `git gc` bằng job bảo trì riêng khi không có merge đang chạy.

## 11. Windows gotchas

| Vấn đề | Biện pháp |
|---|---|
| Path dài > 260 ký tự (node_modules sâu) | `git config --system core.longpaths true` (cần admin, ghi vào ENVIRONMENT_SETUP), đặt `WORKSPACE_ROOT` ngắn (`D:\aiw`), task-id ngắn. Bật Long Paths trong Windows (`HKLM\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled=1`). [Cần người dùng phê duyệt – cần quyền admin] |
| `core.autocrlf=true` ở system config [Đã xác minh] | Platform đặt `core.autocrlf=false` trong `repo.git` + `.gitattributes` với `* text=auto eol=lf` do project template cung cấp; tránh diff giả toàn file và tránh script `.sh` bị CRLF khi chạy trong container Linux. |
| `core.symlinks=false` [Đã xác minh] | Không dựa vào symlink; pnpm dùng junction trên Windows – OK. Bind mount vào Docker: junction có thể không resolve → sandbox cài `node_modules` riêng trong container (INFRASTRUCTURE_DESIGN §6.4). |
| File lock bởi antivirus/Explorer/VS Code | `worktree remove` retry với backoff (3 lần, 2s/5s/10s); nếu vẫn lỗi đánh dấu `cleanup_pending`. |
| Case-insensitive FS | Reject manifest/branch có hai path chỉ khác hoa/thường; `core.ignorecase=true` nhất quán. |
| Tên file cấm trên Windows (`aux`, `con`, `nul`, ký tự `:` `<` `>` `|`) | Lớp 3 validation reject; `core.protectNTFS=true` (mặc định). |
| Bind mount từ NTFS vào WSL2 chậm với I/O lớn | Chấp nhận ở MVP; tùy chọn sau: lưu `WORKSPACE_ROOT` trong WSL2 distro (INFRASTRUCTURE_DESIGN §12). |
| Git Credential Manager prompt GUI | Agent không được push; platform push bằng `gh` với token của người dùng, `GIT_TERMINAL_PROMPT=0`. |
| Branch name có `/` tạo thư mục trong `refs/heads` | OK trên NTFS; cấm tên có khoảng trắng/ký tự đặc biệt – validate bằng `git check-ref-format`. |

## 12. TypeScript interface sketches

```ts
// packages/workspace/src/types.ts  (sketch – không phải code chạy)
export type TaskId = string;         // ULID, 26 ký tự; path dùng 8 ký tự cuối
export type Glob = string;

export interface OwnershipManifest {
  schemaVersion: 1;
  taskId: TaskId;
  agent: { slug: string; cli: 'claude' | 'codex' };
  writable: Glob[];
  readOnly: Glob[];
  forbidden: Glob[];
  allowNewFilesUnder: Glob[];
  allowDeletes: boolean;
  maxChangedFiles: number;
  maxDiffBytes: number;
  allowedCommands: string[];
  deniedCommandPatterns: string[];
}

export type PathClass = 'writable' | 'readOnly' | 'forbidden' | 'unmatched';

export interface ManifestEvaluator {
  classify(relPath: string, m: OwnershipManifest): PathClass;
  canWrite(relPath: string, m: OwnershipManifest, isNew: boolean): boolean;
  validateDiff(changes: ChangedFile[], m: OwnershipManifest): DiffValidationResult;
}

export interface ChangedFile { path: string; status: 'A' | 'M' | 'D' | 'R'; oldPath?: string; bytes: number; binary: boolean; }
export interface DiffValidationResult { ok: boolean; violations: { path: string; reason: string }[]; stats: { files: number; bytes: number }; }

export interface ProjectRef { orgSlug: string; projectSlug: string; }
export interface ProjectInfo extends ProjectRef { bareRepoPath: string; defaultBranch: string; integrationBranch: string; remoteUrl?: string; }
export interface WorktreeHandle { taskId: TaskId; project: ProjectRef; path: string; branch: string; baseRef: string; createdAt: Date; }

export interface WorkspaceManager {
  createProject(input: { orgSlug: string; projectSlug: string; remoteUrl?: string; defaultBranch?: string }): Promise<ProjectRef>;
  importRepo(project: ProjectRef, source: { kind: 'clone'; url: string } | { kind: 'local'; path: string }): Promise<void>;
  getProject(project: ProjectRef): Promise<ProjectInfo>;
  ensureIntegrationBranch(project: ProjectRef): Promise<string>;
  quota(project: ProjectRef): Promise<{ usedBytes: number; limitBytes: number }>;
  removeProject(project: ProjectRef, opts: { purge: boolean }): Promise<void>;
}

export interface WorktreeManager {
  create(input: { project: ProjectRef; taskId: TaskId; agentSlug: string; baseRef: string; manifest: OwnershipManifest }): Promise<WorktreeHandle>;
  prepareAgentConfig(h: WorktreeHandle, cli: 'claude' | 'codex'): Promise<void>;   // sinh .claude/settings.json, hooks, config.toml
  snapshot(h: WorktreeHandle, message: string): Promise<{ commit: string }>;          // commit checkpoint với trailer
  validate(h: WorktreeHandle): Promise<DiffValidationResult>;                         // Lớp 3
  rebaseOnto(h: WorktreeHandle, ref: string): Promise<{ ok: boolean; conflicts: string[] }>;
  mergeToIntegration(h: WorktreeHandle): Promise<{ mergeCommit: string }>;           // chạy trong mutex theo project
  remove(h: WorktreeHandle, opts: { force: boolean }): Promise<void>;
  list(project: ProjectRef): Promise<WorktreeHandle[]>;
  pruneOrphans(project: ProjectRef): Promise<string[]>;
}

export interface MergeQueue {
  enqueue(taskId: TaskId): Promise<void>;
  next(project: ProjectRef): Promise<TaskId | null>;   // topological + FIFO
}
```

Thực thi git: gọi binary `git` qua `execa` với `GIT_TERMINAL_PROMPT=0`, timeout, `--no-pager`, `-c core.quotepath=off`. Không dùng `isomorphic-git` cho worktree (hỗ trợ worktree hạn chế). [Đề xuất]

## 13. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
|---|---|---|
| Bare repo + worktree thay vì clone riêng mỗi task | Tiết kiệm disk (objects dùng chung), tạo nhanh (<1s), branch/ref tập trung | [Đề xuất] |
| Manifest theo glob, mặc định read-only | Đơn giản cho PM agent sinh ra, dễ hiển thị UI, dễ kiểm tra | [Đề xuất] |
| Hook là lớp bắt buộc, deny rule là bổ sung | Đã có lịch sử bug deny rule không hiệu lực | [Đề xuất] |
| Post-hoc validation là lớp quyết định | Không phụ thuộc hành vi CLI bên thứ ba | [Đề xuất] |
| Người dùng là author, agent ở trailer | Quyền tác giả rõ ràng, vẫn audit được | [Cần người dùng phê duyệt] |
| Không auto-merge vào `main` | Nguyên tắc human-in-the-loop (SECURITY.md của A09) | [Đề xuất] |
| `WORKSPACE_ROOT = D:\aiw` | Ổ D còn 109 GB, path ngắn | [Cần người dùng phê duyệt] |

## 14. Dependencies

- A08: bảng `projects`, `tasks`, `worktrees`, `manifests`, `merge_queue`; trạng thái task (`needs_review`, `blocked_conflict`, `merged`, `cleanup_pending`).
- A09: threat model cho "agent thoát khỏi worktree", "secret leak qua commit", "prompt injection qua nội dung repo".
- Agent Orchestrator (agent khác): gọi `WorktreeManager.create/validate/mergeToIntegration` theo vòng đời task.
- INFRASTRUCTURE_DESIGN.md: sandbox chạy build/test trên worktree.
- DEVSECOPS_GUIDE.md: gitleaks config dùng ở Lớp 3.

## 15. Rủi ro

| Rủi ro | Mức | Giảm thiểu |
|---|---|---|
| Agent chạy lệnh shell vòng qua hook (ví dụ `node -e "fs.writeFileSync(...)"`) | Cao | Lớp 3 validation; `allowedCommands` hẹp; container sandbox sau MVP |
| `core.longpaths` chưa bật → lỗi checkout deep path | Trung bình | Bước bắt buộc trong ENVIRONMENT_SETUP; preflight check lúc khởi động app |
| Antivirus khoá file khiến `worktree remove` thất bại | Trung bình | Retry + `cleanup_pending`; hướng dẫn exclude `WORKSPACE_ROOT` khỏi Defender real-time scan [Cần người dùng phê duyệt] |
| Repo lớn làm quota vượt nhanh | Trung bình | Sparse-checkout theo manifest (`git sparse-checkout set <dirs>`) – sau MVP |
| Rebase tự động làm hỏng tiến trình agent đang chạy | Cao | Chỉ rebase khi agent đã dừng; mutex per project |
| Agent sửa `.claude/settings.json` để tự nới quyền | Cao | `forbidden` + deny `Write(./.claude/**)`; Lớp 3 reject mọi thay đổi `.claude/**`, `.ai-workspace/**`; platform ghi đè lại file trước mỗi lần spawn |

## 16. Vấn đề chưa giải quyết

1. Codex CLI chưa cài nên chưa xác minh được flag `--sandbox`/`config.toml` thực tế trên Windows (sandbox Codex trên Windows native có thể hạn chế – cần kiểm tra khi cài).
2. Có cho phép agent tự `git commit` hay platform luôn commit thay? (Đề xuất: cho phép, platform luôn checkpoint cuối session.)
3. Sparse-checkout cho repo lớn: MVP hay sau?
4. Chính sách khi hai task cùng cần ghi một file (manifest chồng lấn): từ chối lúc tạo task hay cho phép với cảnh báo và tuần tự hoá merge?
5. `WORKSPACE_ROOT` đặt ở đâu, có nằm trong WSL2 để tăng tốc I/O không?
6. Hành vi chính xác của Claude Code 2.1.312 khi hook exit 2 trong chế độ `-p` (non-interactive) cần test thực tế.

## 17. Acceptance Criteria (cho Phase 1 triển khai)

- [ ] Tạo project từ URL GitHub hoặc thư mục local → tồn tại `repo.git` bare + `integration/<project>`.
- [ ] Tạo task → worktree + branch `agent/<slug>/<task-id>` trong < 2s cho repo ≤ 200 MB; `.claude/settings.json` và hooks được sinh đúng manifest.
- [ ] Test tự động: Claude Code trong worktree cố `Write` vào file `readOnly` → bị chặn với thông điệp lý do; cố `Bash(git push)` → bị chặn.
- [ ] Diff validation phát hiện: file ngoài manifest, xoá file khi `allowDeletes=false`, vượt `maxChangedFiles`, secret giả (`AKIA...`) trong diff, thay đổi `.claude/**`.
- [ ] Hai task song song trên cùng project, mỗi task chạy `git commit` đồng thời → không lỗi `index.lock`.
- [ ] Merge queue: task B phụ thuộc A chỉ merge sau A; conflict giả lập sinh task `merge-conflict` gán đúng agent.
- [ ] Cleanup job xoá worktree mồ côi và prune; `worktree remove` retry khi file bị khoá.
- [ ] Preflight khi khởi động app báo rõ nếu `core.longpaths`/LongPathsEnabled chưa bật, `core.autocrlf` không được override.
- [ ] Mọi commit do platform tạo đều có trailer `Task-Id` và `Agent`.


> **Phụ lục sửa đổi (D-021, PROPOSED):** nạp skill Markdown động theo từng agent, quét lúc giao task, xem `docs/architecture/DYNAMIC_SKILL_INJECTION.md`. Phụ lục này bổ sung nguồn skill trên đĩa và rào chắn `.ai/**`; phần còn lại của tài liệu này giữ nguyên cho đến khi Owner duyệt.
