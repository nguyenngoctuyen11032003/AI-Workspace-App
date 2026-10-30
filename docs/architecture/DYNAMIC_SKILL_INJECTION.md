# DYNAMIC_SKILL_INJECTION.md — Nạp skill dạng Markdown động theo từng agent

Ngày: 2026-10-30. Trạng thái: **PROPOSED** (quyết định D-021, chờ Owner duyệt). Tài liệu bổ sung cho `docs/architecture/SKILL_SYSTEM.md`, `docs/AGENT_ARCHITECTURE.md`, `docs/PROVIDER_INTEGRATION.md`, `docs/architecture/WORKSPACE_AND_GIT_ISOLATION.md`. Không có mã nào được viết.

Nhãn: **[ĐX]** đã xác minh bằng lệnh hoặc tài liệu nguồn; **[ĐXuất]** đề xuất; **[GĐ]** giả định; **[PD]** cần Owner phê duyệt.

## 1. Mục đích
Cho phép Owner thêm, sửa, xóa file Markdown quy chuẩn riêng cho từng AI agent **sau khi hệ thống đã chạy hoặc đã làm xong một sản phẩm**, và các agent áp dụng ngay ở task kế tiếp mà không phải biên dịch lại mã, cấu hình lại agent hay khởi động lại dịch vụ.

## 2. Phạm vi
Trong phạm vi: thư mục skill trên đĩa theo từng agent, bộ quét và nạp lúc giao task, tiêm vào ngữ cảnh của Claude Code (đường chính) và Codex (đường phụ), cấu hình agent nhận diện thư mục, an toàn, audit, giao diện quản lý, kế hoạch hiện thực hóa.
Ngoài phạm vi: huấn luyện lại mô hình; nạp lại giữa chừng một lượt suy luận đang chạy; đồng bộ nhiều máy.

## 3. Yêu cầu của Owner (nguyên văn rút gọn) và cách đáp ứng
| # | Yêu cầu | Cách đáp ứng |
|---|---|---|
| R1 | Agent không bị đóng băng hành vi hay prompt. Sau khi xong, Owner vào `.ai/skills/[agent-name]/` thêm, sửa, xóa file `.md` | Mục 4.1, 4.2: Skill Overlay Directory, nguồn sự thật là file trên đĩa |
| R2 | Khi agent nhận task mới, lớp điều phối hoặc wrapper của Claude Code tự quét, đọc các file `.md` mới cập nhật và đưa vào ngữ cảnh thực thi ngay, không biên dịch lại, không cấu hình lại | Mục 4.3 đến 4.5: quét lúc giao task, snapshot theo task, tiêm vào system prompt |
| R3 | Cập nhật tài liệu và cơ chế cấu hình agent nhận diện thư mục skill | Mục 4.9, 6, và các phụ lục sửa đổi ở mục 8 |

## 4. Thiết kế

### 4.1 Skill Overlay Directory [ĐXuất]
Mỗi agent có hai phạm vi thư mục, đều **nằm ngoài mọi git worktree** của task để agent không thể tự sửa:

```
<project_root>/.ai/skills/<agent-slug>/       # phạm vi project: chỉ áp dụng cho dự án này
<data_root>/agent-skills/<agent-slug>/        # phạm vi agent toàn cục: áp dụng mọi dự án
<project_root>/.ai/skills/_shared/            # tùy chọn: dùng chung cho mọi agent của project
```
- `<agent-slug>` là slug ổn định của agent (không đổi khi đổi tên hiển thị). Ví dụ `backend-dev`, `frontend-dev`, `designer`.
- `<project_root>` là thư mục dự án do Workspace Manager quản lý, **không phải** worktree của task. Thư mục này có thể được Owner đưa vào git của dự án (khuyến nghị) để có lịch sử thay đổi; hệ thống không bắt buộc.
- **Thứ tự ưu tiên khi trùng tên file hoặc trùng rule** (cao đến thấp): project theo agent, project dùng chung, agent toàn cục, skill gán từ Skill Library. Ưu tiên chỉ quyết định thứ tự xuất hiện và chỗ đứng khi ngữ cảnh bị cắt. Nó **không** cho overlay ghi đè quyền hạn (xem 4.7).
- Khác với `<worktree>/.ai-workspace/` (thư mục do nền tảng sinh trong worktree, agent chỉ đọc, đã có trong `WORKSPACE_AND_GIT_ISOLATION.md`). Hai thư mục này khác nhau và không được nhầm: `.ai/` là nội dung của Owner, `.ai-workspace/` là cấu hình do máy sinh.

### 4.2 Định dạng file [ĐXuất]
- Mỗi file `*.md` hoặc `*.markdown` ở cấp 1 và các thư mục con tối đa 2 cấp. Hỗ trợ `references/`, `examples/` kèm theo giống chuẩn Agent Skills.
- Frontmatter YAML là **tùy chọn**. Khóa được chấp nhận: `name`, `description`, `mode` (`always-on` hoặc `on-demand`), `priority` (số nguyên, mặc định 100), `applies_to` (danh sách glob đường dẫn hoặc từ khóa), `enabled` (mặc định true). Khóa lạ bị bỏ qua kèm cảnh báo.
- Không có frontmatter: file là `on-demand`, tên lấy từ tên file, mô tả lấy từ dòng tiêu đề đầu tiên.
- File tên bắt đầu bằng `_` hoặc `.` bị bỏ qua. File `00-` đến `99-` sắp xếp theo tiền tố khi cùng `priority`.
- File `AGENT.md` ở gốc thư mục, nếu có, là **always-on** theo mặc định (quy chuẩn tổng quát của agent đó).
- **Frontmatter không bao giờ được cấp quyền.** Mọi khóa như `allowed-tools`, `permissions`, `hooks`, `mcp`, `env` trong overlay bị loại bỏ và ghi cảnh báo (mục 4.7).

Giới hạn mặc định [PD]: mỗi file tối đa 64 KiB, tối đa 100 file mỗi agent, tổng nội dung nạp always-on tối đa 3.000 token, tổng on-demand tối đa 12.000 token (đồng bộ `SKILL_SYSTEM.md` mục 4.5).

### 4.3 Bộ quét và manifest [ĐXuất]
Thành phần mới `SkillOverlayScanner` trong module Skills của backend. **Nguồn sự thật là hệ thống file**, không phải bộ theo dõi sự kiện: quét luôn diễn ra lúc giao task.

```ts
interface OverlayFileEntry {
  relPath: string;            // chuẩn hóa dấu '/', phân biệt hoa thường theo chuẩn Linux
  scope: 'project-agent' | 'project-shared' | 'agent-global';
  size: number;
  sha256: string;             // băm sau khi chuẩn hóa (bỏ BOM, đổi CRLF thành LF)
  mode: 'always-on' | 'on-demand';
  priority: number;
  estimatedTokens: number;
  lint: { ok: boolean; warnings: string[]; dropped: string[] };
}
interface OverlayManifest {
  agentId: string;
  scannedAt: string;
  files: OverlayFileEntry[];
  manifestHash: string;       // băm danh sách (relPath, sha256, mode, priority) đã sắp xếp
  totals: { alwaysOnTokens: number; onDemandTokens: number; fileCount: number };
}
interface SkillOverlayScanner {
  scan(agent: AgentRef, project: ProjectRef): Promise<OverlayManifest>;
  materialize(manifest: OverlayManifest, target: Provider, taskRunId: string): Promise<OverlayInjection>;
}
```
Thuật toán `scan`:
1. Xác định ba thư mục theo agent (4.1). Thư mục không tồn tại là bình thường, kết quả rỗng.
2. Duyệt đệ quy có giới hạn độ sâu. **Từ chối** liên kết tượng trưng, junction và đường dẫn thoát ra ngoài gốc (kiểm tra bằng `realpath` rồi so tiền tố). Bỏ qua file không phải văn bản UTF-8.
3. Đọc từng file, **kiểm tra ổn định**: đọc kích thước và băm hai lần cách nhau ngắn (khoảng 150 ms). Nếu khác nhau, thử lại tối đa 3 lần rồi dùng bản đã ổn định gần nhất và ghi cảnh báo `PARTIAL_WRITE_SUSPECTED`. Điều này xử lý trình soạn thảo ghi file tạm rồi đổi tên.
4. Phân tích frontmatter, chạy lint (4.7), quét bí mật, ước lượng token.
5. Dùng bộ nhớ đệm theo `(relPath, size, mtimeMs)` để không băm lại file không đổi. `mtime` chỉ để tăng tốc, **không bao giờ** là căn cứ quyết định đã đổi hay chưa. Căn cứ là `sha256`.
6. Trả `OverlayManifest` và `manifestHash`.

**Bộ theo dõi tệp (tùy chọn, chỉ cho giao diện).** `chokidar` hoặc tương đương phát sự kiện `SKILL_OVERLAY_CHANGED` để giao diện báo "có thay đổi chưa được nạp, sẽ áp dụng ở task kế tiếp". Nếu bộ theo dõi lỗi hoặc bỏ sót, chức năng vẫn đúng vì lúc giao task luôn quét lại.

### 4.4 Chọn nội dung cần nạp [ĐXuất]
Dùng lại thuật toán 5 bước ở `SKILL_SYSTEM.md` mục 4.5, thêm quy tắc cho overlay:
- File `always-on` luôn vào, trong ngân sách riêng (mặc định 3.000 token). Vượt ngân sách: cắt theo `priority` giảm dần, **ghi rõ phần bị cắt vào snapshot và cảnh báo trên giao diện**, không cắt im lặng.
- File `on-demand` chỉ đưa tên và mô tả vào danh mục; nội dung mở khi agent cần (qua đường `--add-dir` của Claude). `applies_to` khớp với file task dự kiến chạm thì tăng điểm.
- Mỗi lần chọn ghi `task_runs.skill_snapshot` (danh sách `scope/relPath@sha256`) và `skill_selection` với lý do, để trả lời "tại sao agent áp dụng quy chuẩn này".

### 4.5 Tiêm vào ngữ cảnh Claude Code (đường chính) [ĐXuất, cần PoC]
Wrapper là `ClaudeCodeAdapter` (D-004). Lúc bắt đầu một **task run**:
1. Orchestrator gọi `scan` rồi `materialize`.
2. `materialize` sinh trong thư mục chạy riêng của phiên (`data/runtime/skills-share/<agent_id>/<task_run_id>/claude/`):
   - `system-fragment.md`: quy chuẩn `always-on` đã gộp, có tiêu đề nguồn từng đoạn, có dòng đầu `AIW overlay manifest <manifestHash>`.
   - `.claude/skills/<slug>/SKILL.md` cho từng file `on-demand` (chuyển đổi nhẹ: thêm frontmatter chuẩn `name`, `description` nếu thiếu; giữ nguyên nội dung), cùng `AIW_SKILLS_INDEX.md`.
3. Spawn phiên với:
   - **SDK**: `systemPrompt: { type: 'preset', preset: 'claude_code', append: <system-fragment> }` và `additionalDirectories: [<share>/claude]` [ĐX nêu trong `CLAUDE_CODE_INTEGRATION.md` mục SDK; hành vi thật chưa được kiểm thử trong dự án].
   - **CLI dự phòng**: `--append-system-prompt-file <share>/claude/system-fragment.md` và `--add-dir <share>/claude`. [ĐX: `claude --help` bản 2.1.312 liệt kê `--append-system-prompt[-file]` và `--add-dir`, kiểm tra ngày 2026-10-30.]
4. Không ghi gì vào worktree của dự án và không sửa `CLAUDE.md` của repo.

**Phát hiện then chốt cho resume [ĐX].** `claude --help` mô tả `--system-prompt-snapshot`: mặc định **bật**, nghĩa là system prompt, kể cả phần `--append-system-prompt`, được ghi lại **một lần** ở request đầu của cuộc hội thoại và **dùng lại nguyên văn mỗi lần request và resume**, dù lần chạy sau truyền nội dung khác, cho đến khi cuộc hội thoại được nén (compact). Hệ quả:
- **Task mới = hội thoại mới = snapshot mới**: cơ chế quét và tiêm ở trên áp dụng đúng. Đây là trường hợp yêu cầu R2 nhắm tới.
- **Resume một phiên cũ** (ví dụ sau khi agent bị gián đoạn, hoặc Owner trả lời yêu cầu duyệt): nếu chỉ truyền lại `--append-system-prompt`, **thay đổi overlay sẽ không có hiệu lực**. Có hai cách, cần PoC để chọn [PD]:
  - (a) Thêm `--system-prompt-snapshot off` cho phiên của chúng ta để system prompt render lại mỗi request. Đổi lại: có thể tăng chi phí cache và độ trễ. Cần đo.
  - (b) Giữ snapshot, nhưng khi resume mà `manifestHash` đổi thì chèn một thông điệp người dùng ở đầu lượt: "Quy chuẩn của bạn đã được cập nhật: <phần khác biệt>". Ưu điểm: giữ cache. Nhược điểm: không phải system prompt đúng nghĩa và agent có thể ưu tiên thấp hơn.
  - Mặc định đề xuất: task mới luôn là hội thoại mới; resume dùng (b), cho phép bật (a) theo cấu hình agent. Chưa kiểm chứng.
- **Phiên SDK sống lâu** (streaming input, nhiều lượt): thay đổi overlay chỉ có hiệu lực ở **ranh giới task**. Giữa task, giao diện hiển thị "có cập nhật, áp dụng ở task kế tiếp". Không hot-reload giữa chừng, khớp với quyết định D8 trong `SKILL_SYSTEM.md`.
- Việc có thể truyền tương đương cờ này qua SDK hay không **chưa xác minh** [GĐ].

### 4.6 Codex và provider khác [ĐXuất]
- **Codex**: sinh `AGENTS.md` cho phiên (vùng đánh dấu `<!-- AIW:BEGIN -->`) hoặc junction vào `.agents/skills` theo `SKILL_SYSTEM.md` mục 4.3.3 và D-013, với nội dung lấy từ cùng `OverlayManifest`. Mỗi `thread/start` của `codex app-server` truyền `developerInstructions`. Chưa test vì Codex CLI chưa cài.
- **OpenHands / Generic CLI**: dùng cơ chế microagent hoặc tệp chỉ dẫn của provider; ngoài MVP.
- Mọi provider dùng chung `OverlayManifest`, nên hành vi quét và nhật ký giống nhau.

### 4.7 An toàn và ràng buộc [ĐXuất]
Nội dung overlay do Owner viết, nên là nguồn đáng tin hơn nội dung tải về, nhưng vẫn là **văn bản được tiêm vào system prompt**, nên cần rào chắn:
1. **Chống agent tự sửa overlay**: thư mục `.ai/**` và `<data_root>/agent-skills/**` nằm ngoài worktree; thêm `.ai/**` vào danh sách `forbidden` mặc định của manifest sở hữu file và thêm deny `Write(.ai/**)`, `Edit(.ai/**)` trong cấu hình CLI sinh ra. Lớp kiểm diff sau cùng từ chối mọi thay đổi vào `.ai/**`. Nếu không, agent có thể tự nới quy tắc cho chính mình.
   **Giới hạn thật của biện pháp này (QA lần hai, F-03):** thư mục overlay nằm ngoài worktree nên lớp kiểm diff theo worktree **không nhìn thấy** nó. Ở Tier 1 CLI chạy trên host cùng quyền người dùng, nên một lệnh Bash lọt qua hook có thể ghi vào `.ai/**` và deny rule của CLI không bao phủ mọi cách ghi (xem `SECURITY.md` R1). Nếu `.ai/` được commit vào repo dự án, agent cũng có thể đổi nó qua merge. Vì vậy 1 là **lớp phòng thủ nông**, và cần thêm mục 10.
2. **Overlay không cấp quyền**: các khóa `allowed-tools`, `permissions`, `hooks`, `mcp`, `env` bị loại; quyền thật vẫn do chính sách agent và Approval quyết định (đồng nhất với `SKILL_SYSTEM.md` mục 4.3.2). Overlay chỉ có thể **thắt chặt** hành vi, không thể nới lỏng chính sách.
3. **Quét bí mật** (mẫu `sk-`, `Bearer`, khóa riêng, `.env` thật) trước khi tiêm. Phát hiện: chặn file đó, báo cho Owner, không tiêm. Overlay có thể bị ghi vào log và event, nên áp dụng lọc che (redaction) đã có trong `CREDENTIAL_MANAGEMENT.md`.
4. **Chống đường dẫn độc hại**: từ chối symlink/junction, `..`, đường dẫn tuyệt đối, tên thiết bị Windows (`CON`, `NUL`...). Kiểm tra bằng `realpath`.
5. **Giới hạn kích thước và số lượng** (4.2) để tránh làm nghẽn ngữ cảnh hoặc chi phí.
6. **Lint nội dung**: cảnh báo (không chặn) câu có dạng "bỏ qua mọi hướng dẫn trước đó", "tắt phê duyệt", "đọc file ngoài workspace". Đây là heuristic, không thay thế kiểm soát quyền. Owner là người chịu trách nhiệm cuối.
7. **Audit**: mỗi lần nạp ghi `SKILL_OVERLAY_LOADED` với `manifestHash`, danh sách `relPath@sha256`, phần bị cắt, cảnh báo. Mỗi lần phát hiện đổi ghi `SKILL_OVERLAY_CHANGED` kèm người sửa nếu biết (không biết thì để trống, không bịa).
8. **Cách ly giữa agent**: agent A không đọc được thư mục overlay của agent B (đường dẫn ngoài `additionalDirectories` của A). `_shared` là ngoại lệ có chủ đích.
9. Bản quét lúc giao task là bản **bất biến** cho task đó. Sửa file trong lúc task chạy không ảnh hưởng task đang chạy.
10. **Toàn vẹn và nguồn gốc thay đổi (provenance)** [ĐXuất, cần Owner chọn mức]:
    - **Baseline đã xác nhận**: hệ thống giữ `overlay.lock` (băm từng file) **ngoài** thư mục overlay, trong dữ liệu của app, cập nhật khi Owner xác nhận thay đổi qua giao diện hoặc lệnh `aiw skills accept`.
    - **Phát hiện sửa trong lúc agent chạy**: nếu băm của bất kỳ file overlay nào đổi trong khoảng thời gian có task của **bất kỳ agent** đang `RUNNING` trên project đó, ghi `OVERLAY_MODIFIED_DURING_RUN` và đánh dấu thay đổi là **nghi vấn**, vì không phân biệt được Owner sửa hay agent sửa (cùng một người dùng hệ điều hành). Hệ thống **không tự biết ai sửa**; chỉ biết thời điểm.
    - **Hai chế độ cấu hình** (`skills.overlay.trust`): `owner-edits` (mặc định, áp dụng ngay thay đổi, chỉ cảnh báo khi nghi vấn) và `review-required` (thay đổi chưa xác nhận bị cách ly, task kế tiếp dùng baseline đã xác nhận cho đến khi Owner xem diff và chấp nhận). Dự án có agent chạy lệnh shell trên host nên dùng `review-required`.
    - **Chặn đường merge**: kiểm diff sau cùng của nhánh task từ chối mọi thay đổi vào `.ai/**` nếu `.ai/` nằm trong repo, và `.ai/**` thuộc `forbidden` trong manifest.
    - **Hạn chế khi chạy Tier 2** (container): thư mục overlay **không** được mount vào container của agent, nên agent không ghi được.
    - **Rủi ro còn lại, không giải quyết được ở Tier 1**: tiến trình cùng người dùng có thể sửa tệp bất kỳ. Bảo vệ duy nhất là phát hiện sau sự việc và xem xét của Owner. Owner phải thừa nhận rủi ro này cùng R1 trong `SECURITY.md`.

### 4.8 Giao diện và quan sát [ĐXuất]
- Agent Detail, thẻ **Skills & Knowledge** có mục "Local files": đường dẫn thư mục (nút mở trong trình quản lý file), danh sách file, trạng thái từng file (`đang áp dụng`, `chờ áp dụng ở task kế tiếp`, `bị chặn vì bí mật`, `bị cắt vì vượt ngân sách`), số token, băm rút gọn.
- Nút **Rescan**, **Open folder**, **Preview injected context** (hiển thị đúng nội dung sẽ được tiêm cho task kế tiếp), **Promote to Library** (nhập file thành skill có phiên bản trong Skill Library) và **Export to folder** (xuất skill từ Library ra overlay).
- Trang task run có nút **Why these rules** hiển thị `skill_selection` và `manifestHash`.
- Mọi hiển thị lấy từ `OverlayManifest` thật. Không số liệu giả.

### 4.9 Cấu hình agent nhận diện thư mục skill [ĐXuất]
Thêm khối cấu hình vào `agents` (cột hoặc JSONB có kiểm soát, xem `DATABASE_DESIGN.md`):
```jsonc
{
  "skills": {
    "overlay": {
      "enabled": true,                          // tắt thì chỉ dùng skill từ Library
      "dir": null,                              // null: dùng đường dẫn mặc định theo slug (4.1)
      "includeShared": true,
      "includeGlobal": true,
      "applyOnResume": "message",               // "message" | "snapshot-off" | "none"
      "limits": { "maxFiles": 100, "maxFileBytes": 65536, "alwaysOnTokens": 3000, "onDemandTokens": 12000 }
    }
  }
}
```
- `dir` nếu đặt phải nằm dưới `project_root` hoặc `data_root` đã đăng ký. Giá trị ngoài hai gốc này bị từ chối khi lưu.
- Thư mục mặc định được tạo khi agent được tạo (kèm file mẫu `AGENT.md` rỗng có chú thích), nhưng việc tạo không được ghi đè nếu đã tồn tại.
- Đổi `slug` agent không làm mất nội dung: hệ thống đổi tên thư mục kèm xác nhận, hoặc giữ `dir` cố định.
- Dữ liệu bổ sung ở DB: cột `agents.skills_config JSONB`, bổ sung `task_runs.skill_snapshot` để chứa mục overlay `{scope, relPath, sha256}`, và nếu cần bảng `agent_skill_overlay_state (agent_id, manifest_hash, last_scanned_at, warnings JSONB)` làm bộ nhớ đệm cho giao diện. Đây chỉ là đề xuất; không thay đổi schema khi chưa được duyệt.

### 4.10 Quan hệ với Skill Library và Teach Agent [ĐXuất]
- Overlay là **nguồn bổ sung** song song với Library, không thay thế nó. Library cho phiên bản bất biến, ghim, đánh giá và phê duyệt. Overlay cho chỉnh nhanh bằng file.
- Cơ chế "Teach Agent" vẫn đi qua luồng duyệt (FR-061: không rule nào được lưu tự động). Overlay không tự sinh từ hội thoại. Việc Owner tự tay sửa file là hành động chủ ý của Owner.
- Xung đột nội dung giữa overlay và skill Library: hiển thị cảnh báo "hai nguồn nói khác nhau" nếu lint phát hiện cùng chủ đề (heuristic), nhưng thứ tự ưu tiên ở 4.1 quyết định chỗ đứng. Hệ thống không cố tự hòa giải ngữ nghĩa.
- **Promote to Library** tạo `skill_versions` mới từ file overlay sau lint, rồi gán cho agent, để có lịch sử và đánh giá.

## 5. Mâu thuẫn và điểm cần sửa trong tài liệu hiện có (kiểm tra ngày 2026-10-30)
| # | Tài liệu | Nội dung hiện tại | Vấn đề so với yêu cầu mới | Xử lý đề xuất |
|---|---|---|---|---|
| S1 | `SKILL_SYSTEM.md` mục 4.4 và D8 | "Session đang chạy không hot-reload, chốt lúc bootstrap, ghi `skill_snapshot`" | Không mâu thuẫn nếu hiểu là ranh giới task; nhưng chưa nói tới nguồn file trên đĩa | Giữ nguyên, thêm mục tham chiếu tới tài liệu này |
| S2 | `SKILL_SYSTEM.md` mục 4.2 | Skill chỉ ở `data/skills/<slug>/<version>/` (bất biến) và `draft/`; không có thư mục theo agent | Thiếu vị trí `.ai/skills/<agent>/` | Thêm khái niệm overlay (tài liệu này) |
| S3 | `SKILL_SYSTEM.md` mục 4.3.1 | File sinh ra có header "do not edit" | Người đọc có thể nhầm là không được sửa thư mục skill | Ghi rõ: chỉ file projection sinh ra thì không sửa; overlay là file của Owner |
| S4 | `AGENT_ARCHITECTURE.md` 3.5.1 | Tập skill cuối = template + `extraSkills`/`removedSkills`, hash vào `task_context_snapshot` | Không có nguồn file overlay | Thêm nguồn overlay vào công thức tập skill |
| S5 | `AGENT_ARCHITECTURE.md` kiểm tra "instance pin `templateVersion`" | Đổi template giữa session không đổi prompt đã snapshot | Tương thích với hành vi ở ranh giới task | Giữ, thêm kiểm tra tương tự cho overlay |
| S6 | `AGENT_ORGANIZATION.md` | Template giữ "system prompt skeleton" cố định trong DB | Prompt gốc vẫn tĩnh, bổ sung động qua overlay là đủ | Làm rõ: prompt gốc tĩnh, overlay là lớp bổ sung có kiểm soát |
| S7 | `WORKSPACE_AND_GIT_ISOLATION.md` mục manifest | `forbidden` có `.ai-workspace/**`, `.claude/**`, chưa có `.ai/**` | Agent có thể sửa overlay nếu chạy trong project root (không phải worktree) hoặc nếu `.ai/` bị đưa vào worktree | Thêm `.ai/**` vào `forbidden` mặc định và deny `Write(.ai/**)` |
| S8 | `PROVIDER_INTEGRATION.md` | Spawn Claude với `systemPrompt.append` cố định (`systemPromptAppend`) | Chưa mô tả chuyện snapshot system prompt khi resume | Thêm mục snapshot và cách xử lý resume (4.5) |
| S9 | `DATABASE_DESIGN.md` | `agents.system_prompt` và `working_directory` là chuỗi tĩnh; không có khối cấu hình skill | Thiếu chỗ lưu cấu hình overlay | Thêm cột `skills_config` (đề xuất, chờ duyệt) |
| S10 | `product/FUNCTIONAL_REQUIREMENTS.md` FR-051..075 | Skill quản lý qua Library, không có yêu cầu chỉnh bằng file | Thiếu FR | Đề xuất FR-293..FR-298 (mục 7), chờ Owner duyệt trước khi sửa file FR |
| S11 | `SECURITY.md` và `security/PERMISSION_MODEL.md` | Chưa có mối đe dọa "agent tự sửa quy chuẩn của chính mình" cho thư mục overlay | Lỗ hổng mới | Thêm vào mô hình mối đe dọa khi duyệt |
| S12 | `AGENT_ARCHITECTURE.md` mục 3.9 bước 5 | Skill/Context Builder materialize skill từ Library | Cần bước quét overlay | Thêm bước quét trước materialize |

Không phát hiện tài liệu nào quy định rõ rằng cấu hình agent là bất biến vĩnh viễn; mâu thuẫn thực sự chỉ là thiếu nguồn file và thiếu rào chắn `.ai/**`.

## 6. Các bước hiện thực hóa với Claude Code
Không được bắt đầu khi Owner chưa duyệt D-021 và Phase 1 chưa mở. Thứ tự đề xuất:

| Bước | Nội dung | Phụ thuộc | Bằng chứng nghiệm thu |
|---|---|---|---|
| P0 | **PoC kiểm chứng giả định cốt lõi** (Owner cho phép chạy; tốn ít quota): (1) hai lần `claude -p` mới, giữa hai lần sửa file `system-fragment.md`, lần hai phải phản ánh thay đổi; (2) `--resume` có và không có `--system-prompt-snapshot off` để xem thay đổi có hiệu lực không; (3) `--add-dir` nạp `.claude/skills/*` từ thư mục ngoài worktree; (4) cờ tương đương trong SDK | Claude CLI đã đăng nhập | Báo cáo PoC kèm log, `session_id`, và kết luận chọn (a) hay (b) ở mục 4.5 |
| 1 | `SkillOverlayScanner`: duyệt, chuẩn hóa, băm, ổn định, lint, quét bí mật, manifest. Thuần TypeScript, kiểm thử bằng thư mục tạm | Không | Unit test mục 7 |
| 2 | Cấu hình `agents.skills_config` và tạo thư mục mặc định, kiểm tra đường dẫn | DB schema được duyệt | Migration và test |
| 3 | `materialize` cho Claude: sinh `system-fragment.md`, `.claude/skills`, chỉ mục | Bước 1 | Test so khớp nội dung |
| 4 | Tích hợp vào `ClaudeCodeAdapter` bước bootstrap: quét, tiêm, ghi `skill_snapshot`, phát `SKILL_OVERLAY_LOADED` | Adapter nền, event system | E2E với CLI thật: sửa file giữa hai task, task sau áp dụng |
| 5 | Rào chắn: `.ai/**` forbidden, deny Write/Edit, kiểm diff, audit | Workspace và hook | Test an ninh (agent cố sửa overlay bị chặn) |
| 6 | Giao diện "Local files", Preview injected context, Rescan, Promote/Export | Frontend nền | E2E Playwright |
| 7 | Bộ theo dõi tệp cho thông báo giao diện (tùy chọn) | Bước 4 | Test ổn định trên Windows |
| 8 | Codex và provider khác dùng chung manifest | Codex CLI cài, app-server | Test live Codex |

## 7. Tiêu chí nghiệm thu và yêu cầu đề xuất (PROPOSED, chưa vào FR/AC chính thức)
Yêu cầu chức năng đề xuất, mã tạm theo dải mới để không trùng: FR-293 thư mục overlay theo agent; FR-294 quét và nạp lúc giao task; FR-295 cấu hình agent nhận diện thư mục; FR-296 rào chắn và audit; FR-297 xem trước ngữ cảnh sẽ tiêm; FR-298 chuyển file sang Library và ngược lại. Priority đề xuất: M cho FR-293 đến FR-296, S cho FR-297, C cho FR-298 [PD].

| AC (tạm) | Given / When / Then | Cách kiểm thử |
|---|---|---|
| AC-501 | Given agent B có thư mục `.ai/skills/backend-dev/` trống, When Owner thêm `10-api.md` và giao task mới, Then manifest của task chứa `10-api.md` với `sha256` đúng và nội dung có trong system prompt của phiên | Integration, CLI thật, đọc `skill_snapshot` và nội dung tiêm |
| AC-502 | Given file đã nạp, When Owner sửa file rồi giao task mới, Then task mới dùng `sha256` mới, task cũ đã kết thúc giữ `sha256` cũ | Integration |
| AC-503 | Given file bị xóa, When giao task mới, Then manifest không còn file đó và agent không áp dụng quy chuẩn đã xóa | Integration |
| AC-504 | Given Owner sửa file trong lúc task đang chạy, Then task đang chạy không đổi và giao diện báo "áp dụng ở task kế tiếp" | E2E |
| AC-505 | Given file chứa khóa `allowed-tools`, Then khóa bị loại, có cảnh báo, và quyền thực tế của agent không đổi | Unit và integration |
| AC-506 | Given file chứa chuỗi giống khóa API, Then file bị chặn, không xuất hiện trong prompt, log hay event, và Owner được báo | Unit và security test |
| AC-507 | Given symlink hoặc đường dẫn `..` trong thư mục overlay, Then bị từ chối | Security test trên Windows |
| AC-508 | Given agent cố ghi vào `.ai/skills/**`, Then hook từ chối, kiểm diff từ chối và có bản ghi audit | Security test |
| AC-509 | Given tổng always-on vượt ngân sách, Then phần bị cắt được ghi rõ trong snapshot và giao diện, không cắt im lặng | Unit |
| AC-510 | Given trình soạn thảo ghi file tạm rồi đổi tên, Then bộ quét không nạp nửa file | Unit với mô phỏng ghi |
| AC-511 | Given `--resume` và overlay đã đổi, Then hành vi đúng theo cách (a) hoặc (b) đã chọn ở PoC, có bằng chứng | E2E |
| AC-512 | Given agent A, Then không đọc được overlay của agent B | Security test |

## 8. Phụ lục sửa đổi và hành động tiếp theo
Tài liệu này là phụ lục sửa đổi. Con trỏ đã thêm vào `SKILL_SYSTEM.md`, `AGENT_ARCHITECTURE.md`, `PROVIDER_INTEGRATION.md`, `WORKSPACE_AND_GIT_ISOLATION.md`. Quyết định D-021 đã ghi vào `DECISION_LOG.md` ở trạng thái chờ duyệt. Các thay đổi FR, AC, DB và danh sách `forbidden` **chưa** được áp dụng vào tài liệu chính cho đến khi Owner duyệt.

## 9. Rủi ro
| Rủi ro | Mức | Giảm thiểu |
|---|---|---|
| Resume không nhận thay đổi do snapshot system prompt | Cao nếu bỏ qua | PoC và hai phương án (a), (b); task mới luôn là hội thoại mới |
| Agent tự sửa quy chuẩn của mình | Cao | `.ai/**` forbidden, deny, kiểm diff, audit |
| Overlay nới lỏng chính sách | Cao | Loại khóa quyền; quyền chỉ do policy |
| Bí mật lọt vào prompt, log | Trung bình | Quét và chặn, lọc che |
| Ngữ cảnh phình, tốn token | Trung bình | Ngân sách, cắt có ghi chú, preview |
| Ghi file dở dang trên Windows | Trung bình | Kiểm tra ổn định hai lần |
| Prompt injection do Owner dán văn bản không tin cậy vào overlay | Trung bình | Lint heuristic, cảnh báo, Owner chịu trách nhiệm |

## 10. Vấn đề chưa giải quyết (cần Owner)
1. Thư mục mặc định đặt trong repo của dự án (`.ai/skills/`) để có lịch sử git, hay chỉ ở `data_root` ngoài repo? Đề xuất: project root, có thể commit.
2. Khi resume: dùng cách (a) `--system-prompt-snapshot off` hay cách (b) thông điệp cập nhật? Chờ PoC.
3. Overlay có được phép thêm file ở `references/` và `scripts/` không? Đề xuất: `references/` và `examples/` có, `scripts/` không trong MVP.
4. Giới hạn kích thước và ngân sách token mặc định.
5. Có duyệt FR-293..FR-298 như đề xuất ở mục 7 không.

## 11. Bổ sung sau QA lần hai
| AC (tạm) | Given / When / Then | Cách kiểm thử |
|---|---|---|
| AC-513 | Given chế độ `review-required` và một file overlay đổi băm trong lúc có task `RUNNING`, When giao task kế tiếp, Then task dùng baseline đã xác nhận, giao diện hiển thị diff chờ duyệt và ghi `OVERLAY_MODIFIED_DURING_RUN` | Integration |
| AC-514 | Given nhánh task có thay đổi vào `.ai/**` trong repo, When kiểm diff trước merge, Then merge bị từ chối và có bản ghi audit | Security test |
| AC-515 | Given agent chạy trong Tier 2, Then thư mục overlay không có trong danh sách mount của container (kiểm tra bằng `docker inspect`) | Security test, cần Docker |

Sửa đổi này nằm trong phạm vi đề xuất D-021 và chưa được Owner duyệt.
