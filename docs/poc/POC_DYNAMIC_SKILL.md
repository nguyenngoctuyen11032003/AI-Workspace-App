# PoC Dynamic Skill / Resume (SK-01..SK-05)

Ngày: 2026-10-30. Căn cứ: OWR-2 mục 1 Q-18 và 5.5; `docs/architecture/DYNAMIC_SKILL_INJECTION.md` mục 4.5 và 4.7. Người chạy: Validation Engineer. Claude CLI 2.1.312.
Thư mục scratch (ngoài repo, không commit): `C:\Users\TUYEN\AppData\Local\Temp\claude\d--Code-Tuyen-Agent-management\9dde48b7-c365-4f17-9f11-5331b00699a6\scratchpad\poc-skill`. Tệp repo duy nhất được ghi: báo cáo này.

**Trạng thái D-021: KHÔNG tuyên bố VERIFIED.** Báo cáo chỉ nêu tiêu chí OWN-AC nào đã được chứng minh trong PoC và tiêu chí nào chưa.

## 1. Kết quả tóm tắt

| Test | Kết quả | Ghi chú |
|---|---|---|
| SK-01 sửa skill, task mới nhận nội dung mới | PASS | marker 1234 rồi 5678 (task mới), marker cũ không xuất hiện |
| SK-02 không restart/recompile | PASS | không có bước build; quét + render 103 ms (cả tiến trình node) |
| SK-03 resume sau khi overlay đổi | PASS (cơ chế đã chứng minh), có điều kiện | (a) giữ snapshot: trả marker CŨ (stale). (b) `--system-prompt-snapshot off`: trả marker MỚI. (c) hash-check + session mới: trả marker MỚI, có audit |
| SK-04 sửa overlay trong lúc task chạy | SIMULATED (guard); agent thật BỊ CHẶN bởi permission | 2 lần thử với agent thật đều bị chặn quyền, tệp không đổi; phần ghi tệp được mô phỏng bằng script |
| SK-05 xác nhận thì mới chạy tiếp | PASS (mức guard/CLI) | không confirm: task mới bị từ chối; có confirm: baseline + audit cập nhật, task mới được nhận |

Số lần gọi model: **8 / 14** (tổng `total_cost_usd` = 0,006283 USD).

## 2. Cấu hình chạy (giảm quota, cô lập)

Mọi lệnh chạy với cwd = scratch, qua `run.sh`:

```bash
claude -p "$PROMPT" --model haiku --max-turns ${MT:-2} --strict-mcp-config --mcp-config mcp-empty.json \
  --setting-sources project --safe-mode --output-format json "$@" < /dev/null > out.json
# test thuần ngữ cảnh: --tools ""   (tắt toàn bộ tool)
```
`mcp-empty.json` = `{"mcpServers":{}}`. `--safe-mode` tắt CLAUDE.md, skill, hook, MCP, plugin. Không dùng `--dangerously-skip-permissions`. Không đọc tệp credential. Model báo trong `modelUsage`: `claude-haiku-5-5` (alias `haiku` hoạt động).

## 3. Mã nguồn (trích)

`overlay.mjs` (41 dòng, không build, chạy lúc giao task). Trích phần lõi:

```js
const MAX_FILE = 32 * 1024, MAX_FILES = 50, MAX_TOTAL = 128 * 1024;
const sha = (s) => crypto.createHash('sha256').update(s).digest('hex');
export function scan(root) {
  ...
  if (rel.split('/').includes('..') || path.isAbsolute(rel)) { warnings.push({ rel, reason: 'bad-path' }); continue; }
  if (e.isSymbolicLink() || fs.lstatSync(abs).isSymbolicLink()) { warnings.push({ rel, reason: 'symlink-rejected' }); continue; }
  if (!fs.realpathSync(abs).startsWith(rootReal)) { warnings.push({ rel, reason: 'escapes-root' }); continue; }
  let t = fs.readFileSync(abs, 'utf8'); if (t.charCodeAt(0) === 0xFEFF) t = t.slice(1);   // strip BOM
  t = t.replace(/\r\n/g, '\n');                                                           // CRLF -> LF
  if (Buffer.byteLength(t) > MAX_FILE) { warnings.push({ rel, reason: 'too-large' }); continue; }
  files.push({ rel, sha256: sha(t), text: t });
  ...
  const manifestHash = sha(files.map((f) => `${f.rel}\0${f.sha256}`).join('\n'));
  return { files, warnings, manifestHash };
}
// render(): ghi out/system-fragment.md (dòng đầu "AIW overlay manifest <hash>") và out/manifest.json
```

`guard.mjs` (55 dòng). Trích:

```js
const NOTE = 'Same Windows OS user may be Owner or agent: guard proves what/when/which-execution only, NOT who changed the file.';
// start <execId>: từ chối nếu có run PAUSED; quét; idle-edit được chấp nhận (OWN-AC-1) + audit; render; ghi run RUNNING
// check|finish <execId>: so hash từng tệp với manifest lúc start
for (const d of diffs) audit({ event: 'OVERLAY_MODIFIED_DURING_RUN', executionId: id, ...d,
                               detectedBy: `guard.${cmd}`, severity: 'HIGH', suspicious: true });
// sao chép nội dung đổi vào state/quarantine/<execId>/ ; run.state = 'PAUSED_PENDING_CONFIRMATION'; exit 3
// confirm <execId> --by owner: baseline = hiện tại; run.state = CONFIRMED; audit OVERLAY_CHANGE_CONFIRMED
// resume-check <sessionId>: so manifestHash ghi cho session với hiện tại; lệch -> audit RESUME_CONTEXT_STALE
//    action NEW_SESSION_WITH_NEW_FRAGMENT, exit 4
```

## 4. SK-01 và SK-02

Tạo `.ai/skills/backend-dev/10-rules.md` với `Rule BD-1: The project marker for backend-dev is AIW-POC-MARKER-1234.`

```
$ node guard.mjs start exec-sk01a
STARTED exec-sk01a manifestHash=a4dd5a1f33dbb65c05b92fb08ec8b0b1facad8457eecb4973d0f7fa6f37058a3
$ ./run.sh out/call1.json "Per your system prompt rules BD-1, what is the project marker? Reply with only the marker string." --tools "" --append-system-prompt-file out/system-fragment.md
{"result":"AIW-POC-MARKER-1234","session":"cfb65a1a-e761-42ce-81f1-dd6411fa15b9", ...}
```
Sửa marker bằng `sed` (đổi thành `AIW-POC-MARKER-5678`), task MỚI:
```
$ time node guard.mjs start exec-sk01b
STARTED exec-sk01b manifestHash=07d02852378e7a8da70a224d55663bd07f4b905f0214792bc465ba693c3a4b75
real 0m0.103s        # quét + hash + render, không build/restart
$ ./run.sh out/call3_new.json "...same prompt..." --tools "" --append-system-prompt-file out/system-fragment.md
{"result":"AIW-POC-MARKER-5678","session":"9262080d-1c81-4efd-b863-9128e72d6d2a", ...}
```
SK-01: PASS (marker mới được trả, marker cũ không xuất hiện). SK-02: PASS (không có bước compile; 0,103 s tổng cho quét + render; đồng hồ bao quanh: 15:14:27.479 -> 15:14:27.617). Hạn chế: "không restart ứng dụng" ở đây chỉ chứng minh trên script Node ngắn, không phải trên ứng dụng AI Workspace thật.

## 5. SK-03 resume

S1 được tạo với marker A (`1234`) bằng prompt "Reply with exactly the word READY." để lịch sử hội thoại KHÔNG chứa marker (tránh nhiễu): session `f38c1827-068f-4d28-b784-a8d40f5882a5`, hash `a4dd5a1f...`. Sau đó overlay đổi sang B (`5678`, hash `07d02852...`). Câu hỏi: "Per your CURRENT system prompt rules BD-1, what is the project marker?"

```
(a) --resume S1 --append-system-prompt-file fragment-B.md                         -> "AIW-POC-MARKER-1234"  (STALE, snapshot được dùng lại)
(b) --resume S1 --system-prompt-snapshot off --append-system-prompt-file fragment-B.md -> "AIW-POC-MARKER-5678"  (làm mới)
(c) node guard.mjs resume-check S1 -> STALE recorded=a4dd5a1f... current=07d02852... (exit 4)
    audit: RESUME_CONTEXT_STALE ... action NEW_SESSION_WITH_NEW_FRAGMENT
    phiên mới 8ddf4895-6ce0-47c9-9471-663c0be6b083 với fragment B                  -> "AIW-POC-MARKER-5678"
```
Lưu ý (b) chạy SAU (a) trên cùng S1, nên lịch sử đã chứa câu trả lời `1234` của (a); dù vậy mô hình vẫn trả `5678`, nên kết quả (b) có sức nặng hơn.

Kết luận [đã đo]: `--system-prompt-snapshot` mặc định làm marker cũ tồn tại khi resume (đúng như `claude --help` mô tả). Hai cơ chế làm mới đã được chứng minh hoạt động: (b) tắt snapshot, và (c) phát hiện bằng hash rồi tạo phiên mới. Về thực tế, cơ chế khuyến nghị là (c) làm cơ chế chính (phiên mới + audit; không phụ thuộc hành vi cờ của CLI, chi phí thấp vì lịch sử ngắn), (b) khi cần giữ lịch sử hội thoại. Phương án "chèn thông điệp cập nhật" ở 4.5(b) cũ của tài liệu thiết kế KHÔNG được test trong PoC này.

Cache/chi phí (haiku, tiện so sánh gần đúng; mỗi mẫu chỉ 1 lần đo nên không kết luận thống kê):

| Biến thể | input | cache_creation | cache_read | cost USD |
|---|---|---|---|---|
| (a) resume, snapshot on | 2 | 919 | 2482 | 0,000576 |
| (b) resume, snapshot off | 2 | 1191 | 2280 | 0,000845 |
| (c) phiên mới | 2 | 854 | 2493 | 0,000204 |

Quan sát: (b) tạo cache nhiều hơn và đọc cache ít hơn (prefix system prompt đổi nên cache bị vô hiệu một phần), chi phí cao hơn (a) khoảng 47% trong mẫu này. (c) rẻ nhất trong mẫu nhưng lịch sử ngắn; với lịch sử dài (c) mất lịch sử. Chưa đo với hội thoại dài.

## 6. SK-04 sửa overlay khi task đang chạy

Thử với agent thật (haiku, `--tools "Bash"`, cwd scratch), hai lần, đều bị hệ thống quyền chặn (`permission_denials: 1`), tệp không đổi, agent không tìm đường vòng:
```
--allowedTools "Bash(echo:*)"                                      -> bị chặn (redirect ">>" cần phê duyệt)
--allowedTools "Bash(echo * >> .ai/skills/backend-dev/10-rules.md)" -> bị chặn
```
Tôi không nới quyền thêm (không `acceptEdits`, không bypass) nên **không có bằng chứng agent thật ghi được tệp**. Phần ghi tệp được MÔ PHỎNG bằng script (`echo 'Rule BD-9: AIW-POC-SIMULATED-AGENT-WRITE' >> ...`) trong lúc run `exec-sk04` ở trạng thái RUNNING:

```
$ node guard.mjs check exec-sk04
SUSPICIOUS: 1 change(s); run exec-sk04 -> PAUSED_PENDING_CONFIRMATION; content quarantined; next task uses confirmed baseline
Same Windows OS user may be Owner or agent: guard proves what/when/which-execution only, NOT who changed the file.   (exit 3)
```
Dòng audit (JSONL, rút gọn trường `note` giống dòng trên):
```
{"time":"2026-10-30T08:15:23.499Z","event":"OVERLAY_MODIFIED_DURING_RUN","executionId":"exec-sk04","relPath":"10-rules.md","oldHash":"22893669b922e8c624ef43b69e8e996a0a9e7a4e39ccb0779a249eafcf9f96d8","newHash":"47c62f6a2a39c44d9391064e6ecbfe946831b6d1f8b3e62d240d30479c5d956c","detectedBy":"guard.check","severity":"HIGH","suspicious":true,"note":"Same Windows OS user ... NOT who changed the file."}
```
Kết quả: SIMULATED (guard hoạt động; chưa có PASS-với-agent-thật). Phát hiện phụ: lớp quyền của CLI đã chặn redirect ghi vào `.ai/**` ngay cả với allow rule hẹp, nhưng đây là hành vi của CLI trong lần chạy này, không phải kiểm soát ta có thể dựa vào (xem `SECURITY.md` R1).

Hạn chế đã biết của guard PoC:
- Phát hiện theo điểm kiểm tra (`check` cuối task hoặc checkpoint), không theo dõi thời gian thực (không có watcher); thay đổi rồi hoàn tác trong cùng khoảng giữa hai lần kiểm tra sẽ không thấy.
- Cách ly (quarantine) mới chỉ sao chép nội dung nghi vấn vào `state/quarantine/<execId>/` và chặn task mới; **chưa** cài đặt render fragment từ baseline đã xác nhận trong lúc chờ, và `confirm` chỉ có hướng "chấp nhận" (chưa có "hoàn tác/reject" khôi phục tệp từ baseline). Cần làm trong Phase 1.
- Không xác định được ai sửa: audit ghi rõ giới hạn này. Giá trị `--by owner` do người gõ khai, KHÔNG được xác thực (audit ghi "asserted by CLI arg, not authenticated").
- Mới mô phỏng một run; chưa kiểm thử nhiều agent song song.

## 7. SK-05 xác nhận

```
$ node guard.mjs start exec-sk05-neg          # CHƯA confirm
REFUSED: paused exec-sk04; confirm required   (exit 2)   -> vẫn PAUSED_PENDING_CONFIRMATION
$ node guard.mjs confirm exec-sk04            # thiếu --by
--by required                                  (exit 1)
$ node guard.mjs confirm exec-sk04 --by owner
CONFIRMED exec-sk04 by owner; baseline=8dc4799a76cb109b91d52dc8d4ce7e7db9521a3bd62928898f36274383575972
$ node guard.mjs start exec-sk05
STARTED exec-sk05 manifestHash=8dc4799a...    (exit 0)
```
Audit có `TASK_REJECTED_PAUSED` (ca âm tính) và `OVERLAY_CHANGE_CONFIRMED` (kèm diff). Kết quả: PASS ở mức guard/CLI. Không gọi model cho bước "task mới được nhận" (task mới do `start` chấp nhận; việc tiêm fragment đã được chứng minh ở SK-01). Hạn chế: "agent không tự tiếp tục" được thực thi bởi cổng `start` của guard, không phải tiến trình Claude đang chạy bị tạm dừng giữa chừng; trong sản phẩm cần cơ chế `WAITING_APPROVAL` của orchestrator.

## 8. Bảng chi phí từng lần gọi (8 lần)

Cột rate_limit: định dạng `--output-format json` không phát `rate_limit_event`; không gặp lỗi auth hoặc rate limit.

| # | Mục đích | Model (init) | cost USD | in | cache_create | cache_read | out |
|---|---|---|---|---|---|---|---|
| 1 | SK-01 marker A | claude-haiku-5-5 | 0,000739 | 2 | 2699 | 658 | 129 |
| 2 | S1 (READY, marker A) | claude-haiku-5-5 | 0,000359 | 2 | 1047 | 2288 | 4 |
| 3 | SK-01 marker B (task mới) | claude-haiku-5-5 | 0,000552 | 2 | 2688 | 658 | 16 |
| 4 | SK-03 (a) resume snapshot on | claude-haiku-5-5 | 0,000576 | 2 | 919 | 2482 | 16 |
| 5 | SK-03 (b) resume snapshot off | claude-haiku-5-5 | 0,000845 | 2 | 1191 | 2280 | 16 |
| 6 | SK-03 (c) phiên mới | claude-haiku-5-5 | 0,000204 | 2 | 854 | 2493 | 16 |
| 7 | SK-04 agent thật, thử 1 (bị chặn) | claude-haiku-5-5 | 0,001925 | 4 | 6217 | 5467 | 740 |
| 8 | SK-04 agent thật, thử 2 (bị chặn) | claude-haiku-5-5 | 0,001083 | 4 | 1790 | 10077 | 977 |
| | Tổng | | 0,006283 | | | | |

## 9. Agent SDK so với CLI

Owner quyết định SDK là adapter chính; PoC chỉ test CLI (được phép). Tương ứng SDK, **NOT TESTED**:
- `systemPrompt: { type: 'preset', preset: 'claude_code', append: <fragment> }` tương ứng `--append-system-prompt[-file]`.
- `additionalDirectories` tương ứng `--add-dir`.
- Tùy chọn SDK tương ứng `--system-prompt-snapshot` và `--resume`: chưa xác minh có tồn tại và có cùng hành vi. Cần kiểm chứng riêng trước khi dựa vào kết luận SK-03 cho SDK. Phương án (c) (phiên mới khi hash lệch) không phụ thuộc cờ này nên là lựa chọn an toàn hơn cho SDK.
Không cài SDK trong scratch.

## 10. Đối chiếu OWN-AC

| Tiêu chí | Đã chứng minh | Chưa chứng minh |
|---|---|---|
| OWN-AC-1 | Sửa Markdown thì task mới nhận nội dung mới, không restart/build (script + CLI) | Trên ứng dụng AI Workspace thật; với SDK |
| OWN-AC-2 | Guard phát hiện đổi hash trong lúc RUNNING, audit JSONL, đánh dấu nghi vấn, chuyển PAUSED_PENDING_CONFIRMATION, chặn task mới, ghi rõ không xác định người sửa | Agent thật ghi được tệp qua Bash (bị chặn quyền, chỉ SIMULATED); phát hiện thời gian thực; reject/khôi phục; render từ baseline khi chờ |
| OWN-AC-3 | Snapshot làm resume dùng marker cũ (đo được); hash-check phát hiện lệch; hai cơ chế làm mới (phiên mới, snapshot off) hoạt động; có audit | Hash của System Prompt chung (mới hash overlay); chi phí với hội thoại dài; phía SDK; chưa so sánh phương án "chèn thông điệp cập nhật" |

Cần Owner quyết định tiếp: chọn (c) hay (b) làm mặc định cho resume; có cho phép lần thử SK-04 với agent thật bằng cấu hình quyền rộng hơn trong scratch hay không (hiện không làm vì ngoài giới hạn "allow hẹp").
