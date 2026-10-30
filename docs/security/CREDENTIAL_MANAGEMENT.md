# CREDENTIAL MANAGEMENT

> Security Architect · Phase 0 (documentation only) · 2026-10-30 · Trạng thái: PROPOSAL v1
> Mở rộng [`../SECURITY.md`](../SECURITY.md) §4 và `docs/operations/DEVSECOPS_GUIDE.md` §3, §8 (không lặp quy trình incident response — chỉ liên kết). Ràng buộc gốc: D-008 (credential do CLI tự quản; secret khác ở OS keychain; không plaintext trong Postgres/frontend).
> Nhãn: **[Đã xác minh]** · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**.

## 1. Mục đích

Định nghĩa: ai sở hữu từng credential, app được và không được chạm vào đâu, cách dùng keychain, danh sách regex redaction đầy đủ kèm quy tắc phụ trợ, vòng đời (rotate/logout/reconnect/expire), scrubbing log/event, cách cô lập env theo từng agent, và cách xử lý secret của project sinh ra.

## 2. Phạm vi

- Trong: Claude Code OAuth, Codex login, API key provider khác, GitHub token, secret hạ tầng của app (DB, hook/UI token, HMAC), secret của project sinh ra, mọi đường log/event/transcript.
- Ngoài: SSO/OAuth của người dùng cuối project sinh ra, quản lý khoá ký release (CI), Vault/KMS server mode (mở ở §14).

## 3. Kiểm kê credential

| ID | Credential | Provider | Chủ sở hữu | Nơi lưu | App được làm | App **không** được làm | Trạng thái |
| --- | --- | --- | --- | --- | --- | --- | --- |
| C-01 | OAuth token Claude (`.credentials.json`; file người dùng đọc được bởi tiến trình cùng user, kiểm soát bằng ACL) | Anthropic | **CLI Claude Code** (user đăng nhập) | `%USERPROFILE%\.claude\` [Đã xác minh] | Gọi `claude auth status` (exit code + JSON) | Đọc/copy/parse/backup/log; mount vào container | Đã xác minh tồn tại |
| C-02 | `~/.claude.json` (account info), `history.jsonl`, `projects/*.jsonl` | Anthropic | CLI | `%USERPROFILE%\.claude*` | Không | Đọc; khuyến nghị user đặt `cleanupPeriodDays` nhỏ (R9) | Đã xác minh |
| C-03 | Codex login `auth.json` | OpenAI | **CLI Codex** | `%USERPROFILE%\.codex\` | `codex login status` | Như C-01 | Codex chưa cài [Đã xác minh]; thiết kế = Giả định |
| C-04 | `ANTHROPIC_API_KEY` riêng cho Tier 2 | Anthropic | **User** (nhập vào app) | Credential Manager `aiw/provider.anthropic.t2` | Giải mã trong backend, inject qua file tmpfs | Đặt trong env Tier 1 (override OAuth); ghi DB | Đề xuất |
| C-05 | `OPENAI_API_KEY` (Codex/OpenHands API mode) | OpenAI | User | Credential Manager | Như C-04 | Như C-04 | Đề xuất |
| C-06 | Key provider khác (Gemini…) | — | User | Credential Manager | Như C-04 | — | Đề xuất |
| C-07 | GitHub token cho `gh`/push | GitHub | User (Git Credential Manager hoặc PAT trong keychain) | Git Credential Manager / keychain | Orchestrator push bằng helper; agent **không** push | Đặt token trong URL remote; truyền argv | Đề xuất |
| C-08 | `DATABASE_URL`/mật khẩu Postgres, Redis | Local | App | `.env` ACL user-only (dev) hoặc keychain | Dùng trong backend | Truyền cho CLI/container | Đề xuất |
| C-09 | UI bearer token (`AIW_API_TOKEN`) | App | Backend | memory + `%LOCALAPPDATA%\aiw\run\server.json` (ACL user) | Cấp cho Tauri qua `invoke` | Query string, log, env của CLI | Đề xuất |
| C-10 | Hook token (`AIW_HOOK_TOKEN`) per session | App | Backend | memory; hook header trong `--settings` | Verify ở hook listener | Cho phép approve (chỉ nhận quyết định) | Đề xuất |
| C-11 | HMAC key ký audit/export | App | Backend | Credential Manager `aiw/audit.hmac` | Ký bundle, chain checkpoint | Xuất ra UI | Đề xuất |
| C-12 | DEK (data encryption key) cho ciphertext dự phòng | App | Backend | bọc DPAPI scope CurrentUser | Giải mã trong backend | Dùng cho headless nếu có keychain | Đề xuất |
| C-13 | Secret của project sinh ra (`.env`, API key của app khách) | Bên thứ ba | **User/Project** | Ngoài phạm vi platform | Chỉ phát hiện & chặn | Đọc, lưu, đưa vào prompt | Đề xuất |
| C-14 | MCP server credential (Redmine, Slack…) do user cấu hình | Bên thứ ba | User | Connector của host (claude.ai connector) hoặc keychain | Không đọc | Đưa vào env agent | Giả định |

Nguyên tắc sở hữu: **một credential, một chủ**. Không credential nào được dùng chung giữa hai agent (luật bất biến #3 trong `USER_ROLES_AND_PERMISSIONS.md` §4.5). Với OAuth Claude, thực tế mọi agent Tier 1 dùng chung login của user (một tài khoản) — đây là hệ quả của thiết kế CLI, ghi nhận là R2/R4, **không** phải chia sẻ credential do app cấp.

## 4. Danh sách "không bao giờ đọc" (never-read list)

Áp dụng cho backend (code app), agent (qua deny rules + hook), container (không mount), và log pipeline.

| Nhóm | Đường dẫn / mẫu (Windows, đã resolve) | Ai bị cấm | Cơ chế enforce |
| --- | --- | --- | --- |
| CLI creds | `%USERPROFILE%\.claude\.credentials.json`, `%USERPROFILE%\.claude.json`, `%USERPROFILE%\.claude\projects\**`, `history.jsonl`, `%USERPROFILE%\.codex\auth.json`, `%USERPROFILE%\.codex\**` | backend + agent | ESLint rule cấm `fs.*` tới path này (`no-restricted-syntax` + test), Read deny, denylist §3.1, không mount |
| SSH/cloud | `.ssh`, `.aws`, `.azure`, `.config\gcloud`, `.gnupg`, `.kube`, `.docker\config.json`, `.npmrc`, `.pypirc`, `.netrc`, `.git-credentials` | agent | Read deny + denylist |
| Windows secrets | `AppData\Local\Microsoft\Credentials`, `AppData\Roaming\Microsoft\Protect`, `HKLM\SAM`, `SYSTEM`, `cmdkey /list`, `vaultcmd` | agent | denylist + deny `Read(AppData/**)` |
| Env | `.env`, `.env.*` (trừ `.env.example`), `*.pem`, `*.key`, `*.pfx`, `id_rsa*` | agent | manifest `forbidden`, Read deny, Lớp 3 |
| Chính app | `%LOCALAPPDATA%\aiw\run\**`, `%LOCALAPPDATA%\aiw\audit\spool\**`, `.aiw\**`, policy dir | agent | Read/Edit deny; ACL user-only |
| Quy tắc cho backend | Backend **không** có code đọc nhóm "CLI creds"; CI chạy `grep -R "\.credentials\|auth\.json"` và fail nếu match ngoài `docs/`, `test/fixtures/` | dev | SAST rule Semgrep custom |

Phát hiện đăng nhập: chỉ `claude auth status` / `codex login status`, parse JSON `{loggedIn, method, email_masked}`, không bao giờ lưu email đầy đủ (hash + 3 ký tự đầu).

## 5. Dùng OS keychain

| Hạng mục | Thiết kế |
| --- | --- |
| Thư viện | `@napi-rs/keyring` (Rust prebuilt, không node-gyp; `keytar` đã ngừng bảo trì) [Đề xuất; cần thử trên Node 24 + Tauri sidecar — DEVSECOPS §14.1] |
| Service | `aiw` ; account = `<scope>.<secretId>` (ví dụ `provider.anthropic.t2`, `audit.hmac`) |
| Backend Windows | Windows Credential Manager (generic credential, persistence `LOCAL_MACHINE`→ đề xuất `ENTERPRISE`/`SESSION` không dùng; dùng mặc định theo user) |
| Một bên ghi | **Chỉ backend NestJS ghi/đọc**; Tauri không dùng `tauri-plugin-keyring` ghi trực tiếp (tránh hai nguồn sự thật) — SECURITY §4.1 mục 4 |
| Postgres | Không có cột value. DATABASE_DESIGN dùng `credential_ref text` trong `provider_configs`/`git_repositories` (dòng 317, 874) [Đã xác minh]; SECURITY.md đề xuất bảng `secrets(...keychain_ref)` — **chưa có trong schema** (xem §12 inconsistency) |
| Ciphertext dự phòng (headless) | AES-256-GCM, DEK bọc DPAPI (`CryptProtectData`, CurrentUser, entropy thêm = hằng số app); ghi ADR "yếu hơn keychain" [Đề xuất] |
| Truy cập | Hàm duy nhất `SecretVault.use(secretId, purpose, fn)`: giải mã, gọi `fn(value)`, zeroize buffer (`Buffer.fill(0)`), ghi audit `secret_access{secretId,purpose,process}` — không trả value ra ngoài scope hàm |
| Inject vào tiến trình | stdin pipe hoặc file `%LOCALAPPDATA%\aiw\run\<rand>.sec` (ACL chỉ user, `FILE_ATTRIBUTE_TEMPORARY`), xoá sau ≤ 5 s hoặc khi tiến trình đọc xong; không argv, không env chung |
| UI | Form POST một chiều; GET trả `{id, name, maskedPreview:"sk-…4f2a", lastUsedAt}` |
| Export/backup | Không export secret; backup Postgres không chứa secret; HMAC key được user tự sao lưu thủ công (Cần người dùng phê duyệt) |

Giới hạn trung thực: Credential Manager chỉ bảo vệ khỏi người dùng khác, **không** khỏi tiến trình cùng user (R1/R2). Lợi ích chính: không plaintext trên đĩa, không trong Postgres/backups.

## 6. Cô lập env var theo agent

### 6.1 Env whitelist khi spawn CLI (Tier 1)

Backend **không** dùng `process.env` làm nền. Hàm `buildAgentEnv(agent, task)` bắt đầu từ `{}` và thêm:

| Biến | Giá trị | Lý do |
| --- | --- | --- |
| `PATH` | PATH tối thiểu: thư mục Node/Git/CLI (đã canonicalize), **không** thư mục user tuỳ ý | giảm hijack |
| `SYSTEMROOT`, `WINDIR`, `COMSPEC` | giá trị hệ thống | Windows cần |
| `USERPROFILE`, `HOME`, `APPDATA`, `LOCALAPPDATA` | profile thật | CLI cần đọc login của chính nó [Đã xác minh: không dùng profile giả được với OAuth] |
| `TEMP`, `TMP` | `%LOCALAPPDATA%\aiw\tmp\<agentId>` (tạo riêng per agent, ACL user) | tránh dùng chung temp |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | `1` | giảm telemetry |
| `DISABLE_AUTOUPDATER` | `1` | ổn định version |
| `BASH_MAX_TIMEOUT_MS` | `600000` | SECURITY §3.5 |
| `GIT_TERMINAL_PROMPT` | `0` | không prompt |
| `AIW_HOOK_TOKEN` | token session | hook (chỉ nhận quyết định) |
| `AIW_SESSION_ID` | uuid | định danh phía server |

Cấm tuyệt đối trong env agent: `ANTHROPIC_API_KEY` (override OAuth), `OPENAI_API_KEY`, `DATABASE_URL`, `AIW_API_TOKEN`, `GITHUB_TOKEN`/`GH_TOKEN`, `AWS_*`, `NODE_OPTIONS` (inject code), `NODE_EXTRA_CA_CERTS` (trừ khi policy cho phép), `HTTP(S)_PROXY` tuỳ ý. Test: đặt `FAKE_SECRET=sk-ant-…` trong env backend ⇒ agent `process.env` không có (DEVSECOPS §15).

### 6.2 Per-agent

| Hạng mục | Quy tắc |
| --- | --- |
| Hook token | Một token per **session**, hết hạn khi session kết thúc; token A không hợp lệ cho session B |
| TEMP/cache | Thư mục riêng per agent; xoá khi task kết thúc |
| Container | Mỗi project một container; env container rỗng (`Env: []`); biến từng lệnh qua `exec` allowlist; **không** inherit từ backend |
| Tier 2 | Secret riêng per container (file tmpfs `/run/secrets/*`), không qua `Env` (vì lộ trong `docker inspect`) |
| Không chia sẻ | Hai agent không bao giờ nhận cùng một secret do app cấp (trừ OAuth CLI, xem §3) |
| Kiểm soát argv | Mọi `spawn(file, args[])`; không `shell:true`; Semgrep cấm; mật khẩu không bao giờ qua argv (hiện trong `Get-Process`/Event ID 4688) |

## 7. Secret cho container (Tier 1 và Tier 2)

| Tier | Secret trong container | Cách |
| --- | --- | --- |
| 1 | **Không có** | `.env.test` chỉ giá trị giả/sandbox; registry auth (private npm) qua proxy sidecar inject header, container không thấy token [Đề xuất] |
| 2 | API key LLM | Docker secret/tmpfs file mode 0400 do `aiw` tạo lúc start, xoá khi stop; CLI đọc qua `apiKeyHelper` script đọc file (Claude Code hỗ trợ `apiKeyHelper` [Giả định: kiểm tra trên 2.1.312]) |
| 2 | Hook token | env một lần cho process CLI, không cho shell con (wrapper `env -i`) |

## 8. Redaction

### 8.1 Điểm áp dụng (5 điểm, SECURITY §4.4)

(a) ingest stream-json (tool_use input, tool_result, assistant text) · (b) logger transport (pino `redact` + serializer) · (c) WS/SSE broadcast · (d) audit writer · (e) error reporter/exception filter. Pipeline duy nhất `Redactor.scrub(text|obj, ctx)`; không điểm nào tự cài regex riêng.

### 8.2 Regex list (bản đầy đủ, mở rộng SECURITY §4.4)

Cờ mặc định `gi` trừ khi ghi khác. Thay bằng `[REDACTED:<type>:…<last4>]` (giữ 4 ký tự cuối chỉ với type có `keepLast4=true`).

| Type | Regex | keepLast4 |
| --- | --- | --- |
| `anthropic_key` | `sk-ant-[A-Za-z0-9_\-]{20,}` | có |
| `openai_key` | `sk-(?:proj-\|live_\|test_\|svcacct-)?[A-Za-z0-9_\-]{20,}` | có |
| `bearer` | `bearer\s+[A-Za-z0-9\-._~+/]{20,}=*` | không |
| `basic_auth_url` | `[a-z][a-z0-9+.\-]*://[^\s:/@]+:[^\s@/]+@` | không |
| `jwt` | `eyJ[A-Za-z0-9_\-]{10,}\.[A-Za-z0-9_\-]{10,}\.[A-Za-z0-9_\-]{10,}` | không |
| `github_token` | `gh[pousr]_[A-Za-z0-9]{36,}` | có |
| `github_pat` | `github_pat_[A-Za-z0-9_]{60,}` | có |
| `gitlab_pat` | `glpat-[A-Za-z0-9_\-]{20,}` | có |
| `aws_access_key` | `\b(?:AKIA\|ASIA)[0-9A-Z]{16}\b` | có |
| `aws_secret` | `aws_secret_access_key\s*[=:]\s*\S{30,}` | không |
| `slack_token` | `xox[baprs]-[A-Za-z0-9\-]{10,}` | có |
| `google_api_key` | `AIza[0-9A-Za-z\-_]{35}` | có |
| `private_key` | `-----BEGIN (?:RSA \|EC \|OPENSSH \|DSA \|PGP \|ENCRYPTED )?PRIVATE KEY(?: BLOCK)?-----[\s\S]+?-----END[^-]*-----` | không |
| `npm_token` | `npm_[A-Za-z0-9]{36}` | có |
| `stripe_key` | `\b[rs]k_(?:live\|test)_[A-Za-z0-9]{20,}` | có |
| `pg_url` | `postgres(?:ql)?://[^:\s]+:[^@\s]+@` | không |
| `generic_kv` | `(?:api[_-]?key\|secret\|token\|passw(?:or)?d\|pwd\|client_secret)["']?\s*[=:]\s*["']?[^\s"',;]{8,}` | không |
| `oauth_fields` | `(?:sessionKey\|refresh_token\|access_token\|id_token)"?\s*[:=]\s*"?[A-Za-z0-9\-._]{20,}` | không |
| `hook_token` | `AIW_(?:HOOK\|API)_TOKEN\s*[=:]\s*\S+` + giá trị token đang sống (known-secret) | không |
| `windows_cred` | `(?:cmdkey\|vaultcmd)[^\n]*(?:/pass:\|/password)[^\s]*` | không |
| `email_pw_pair` | `[\w.+\-]+@[\w\-]+\.[\w.\-]+\s*[:/]\s*\S{8,}` (cảnh báo, độ chính xác thấp) | không |

### 8.3 Quy tắc phụ trợ

1. **Entropy:** token ≥ 32 ký tự không khoảng trắng và Shannon > 4.0 bit/ký tự ⇒ `[REDACTED:high_entropy]`; loại trừ SHA-256 hex trong trường `*_hash`, UUID, đường dẫn, và nội dung trong khối code được allowlist qua cờ `knownBenign`.
2. **Known-secret match:** mọi secret đang nạp trong bộ nhớ backend (C-04..C-12) được so khớp exact + base64 + base64url + URL-encoded + hex + JSON-escaped (`\u00xx`); cập nhật khi rotate.
3. **Chunk boundary:** buffer overlap 2 KB giữa các chunk stream; secret bị chia đôi qua hai chunk vẫn bị bắt; giữ `pending` tối đa 2 KB cho tới khi chunk kế tới hoặc flush 200 ms.
4. **Canonicalize trước match:** giải JSON-escape, Unicode NFKC, bỏ zero-width (U+200B–U+200D, U+FEFF), gộp `r""m`-style nối chuỗi trong giá trị khi dạng nối trùng known-secret.
5. **Cấu trúc:** với object, scrub theo key (`password`, `token`, `authorization`, `cookie`, `set-cookie`, `x-api-key`, `credential_ref` value) trước regex trên value.
6. Redaction **không đảo ngược**; bản raw không lưu. UI "show raw" (SECURITY §5.3) chỉ hiển thị raw lấy từ memory của approval đang pending, không từ DB.
7. **Hiệu năng:** < 1 ms/KB; regex biên dịch một lần, kiểm tra ReDoS (`safe-regex2`) trong CI; giới hạn input 1 MB/lần (cắt, đánh dấu `truncated`).
8. **False positive:** tỷ lệ chấp nhận được < 1 % trên corpus code thật; allowlist theo (rule, đường dẫn) có TTL; không allowlist theo giá trị.

### 8.4 Prompt đầu vào

UI quét prompt/task description người dùng bằng cùng regex trước khi gửi; phát hiện ⇒ cảnh báo chặn mềm ("Prompt chứa thứ giống secret — gửi vẫn sẽ bị redact khỏi log, nhưng provider vẫn nhận").

## 9. Rotation / logout / reconnect / expiry

| Sự kiện | Phát hiện | Hành động tự động | Hành động người dùng | Audit |
| --- | --- | --- | --- | --- |
| Claude OAuth hết hạn/bị revoke | `claude auth status` exit ≠ 0 hoặc stream-json báo auth error 401 | Pause mọi agent Claude (không retry vô hạn), đánh dấu provider `needs_login`, **không** tự mở trình duyệt | `claude auth login` thủ công trong terminal của user | `config_change` provider.status |
| User `claude logout` ngoài app | Poll `auth status` mỗi 5 phút + trước mỗi spawn | Như trên | Login lại | như trên |
| Reconnect (login xong) | Poll | Resume task `blocked`, nhưng cần user bấm Resume nếu pause > 30 phút | — | `provider.reconnected` |
| API key Tier 2 xoay | User nhập key mới | Ghi keychain version mới → restart container Tier 2 → xoá file secret cũ → cập nhật known-secret; key cũ coi là "retired" vẫn nằm trong known-secret redaction 30 ngày | Revoke key cũ tại console | `secret_rotate` |
| API key nghi lộ | gitleaks/redaction hit, billing bất thường | Emergency stop (nếu bật), revoke cached approvals, vô hiệu key trong app; hướng dẫn theo DEVSECOPS §8 | Revoke tại provider | `secret_compromise` severity critical |
| GitHub token hết hạn | `gh auth status` fail / 401 khi push | Task push → `blocked`, ask user | Đăng nhập lại GCM | `auth` |
| Token UI/hook | Restart app | Sinh mới mỗi launch; session hook mới mỗi spawn | — | `auth` |
| DB password | Lịch 12 tháng | Script `aiw secrets rotate db` (ALTER ROLE + keychain + restart) | Xác nhận | `secret_rotate` |
| HMAC key audit | 12 tháng hoặc nghi lộ | Chain checkpoint cuối ký bằng key cũ, bắt đầu epoch mới (AUDIT_LOGGING §7) | Lưu bản sao key cũ để verify | `secret_rotate` |
| Hết hạn secret có `expiresAt` | Job hàng ngày | Cảnh báo trước 14/3/1 ngày; hết hạn ⇒ vô hiệu | Nhập mới | `secret_expiry` |

Quy tắc: không bao giờ **tự động** đăng nhập thay user, không lưu mật khẩu/MFA, không điền form login. Luật bất biến #1 (không hỏi/nhận mật khẩu).

## 10. Scrubbing log và event

| Kênh | Quy tắc |
| --- | --- |
| pino | `redact.paths`: `req.headers.authorization`, `req.headers.cookie`, `*.apiKey`, `*.token`, `*.password`, `env.*`, `args`, `tool_input.command`(qua Redactor); serializer chung gọi `Redactor` |
| Exception | Global filter bỏ `stack` có chứa path home; message qua Redactor; không bao giờ log `err.config` của axios/undici (chứa header) |
| `agent_events` / stream-json ingest | Redactor trước khi insert; payload lớn > 64 KB lưu object theo hash đã redact |
| WS/SSE | Scrub trước broadcast; không phát lại raw từ cache |
| Audit | `target`, `before/after` qua Redactor; **không** ghi giá trị `credential_ref`, `system_prompt` (hash + length) |
| Crash dump / core | Tắt (`--ulimit core=0` container; WER LocalDumps tắt cho node.exe app) [Đề xuất] |
| Clipboard/UI | Nút copy chỉ copy bản đã mask |
| Test | Corpus 200 secret × {raw, JSON-escaped, base64, URL-encoded, split chunk, Unicode zero-width} ⇒ 0 leak (RD-corpus) |
| CLI transcript (C-02) | Ngoài tầm app (R9); khuyến nghị giảm retention; app không đọc |

## 11. Secret của project sinh ra

| Vấn đề | Chính sách |
| --- | --- |
| Agent cần `.env` để chạy test | Dùng `.env.test` chứa giá trị giả; sinh bởi orchestrator từ `.env.example`; không giá trị thật |
| Phát hiện secret trong diff | gitleaks Lớp 3 + rule tuỳ chỉnh; `needs_review`, không merge; thông báo user rotate nếu thật |
| Secret thật cần cho tích hợp | User tự cấu hình ngoài platform (MVP); sau MVP: "Project Secret" lưu keychain `aiw/project.<id>.<name>`, inject **chỉ** vào container preview theo yêu cầu có approval, không bao giờ vào prompt/worktree |
| `.gitignore` mặc định | `.env*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `secrets/` |
| Agent sinh code chứa key | Prompt hệ thống yêu cầu dùng biến môi trường; gitleaks bắt hard-code; test mẫu |
| Tài liệu sinh ra (README) | Quét redaction trước commit |

## 12. Quyết định & lý do; inconsistency

| Quyết định | Lý do | Trạng thái |
| --- | --- | --- |
| Không đọc credential CLI; chỉ `auth status` | Giảm bề mặt, tôn trọng D-008 | D-008 (OWR-2 2026-10-30): ứng dụng không tự ý đọc `.credentials.json`/`auth.json`; không lưu plaintext token trong DB/frontend; secret khác dùng OS keychain; **không mount thư mục credential của Owner vào Agent sandbox** |
| `SecretVault.use()` scoped + zeroize | Hạn chế secret tồn tại trong memory/log | Đề xuất |
| Redactor duy nhất cho 5 điểm | Tránh regex lệch nhau | Đề xuất |
| Key retired vẫn nằm trong known-secret 30 ngày | Bắt secret cũ còn xuất hiện trong output | Đề xuất |
| Không tự động login | Luật bất biến #1 | Đề xuất |
| Tier 2 dùng API key riêng | Không đưa OAuth subscription vào container | Cần người dùng phê duyệt |
| Bảng `secrets` hay chỉ `credential_ref` | Hiện lệch giữa SECURITY.md §4.1 và DATABASE_DESIGN | Cần chốt (xem §14) |

## 13. Dependencies

- [`AUDIT_LOGGING.md`](AUDIT_LOGGING.md) (category `secret_access`, scrubbing), [`SANDBOX_DESIGN.md`](SANDBOX_DESIGN.md) §5 (Tier 2), [`PERMISSION_MODEL.md`](PERMISSION_MODEL.md) (path deny), [`THREAT_MODEL.md`](THREAT_MODEL.md) (TM-I-*).
- `DATABASE_DESIGN.md` (`provider_configs.credential_ref`, `git_repositories.credential_ref`), DEVSECOPS §3, §8.
- `@napi-rs/keyring`, `safe-regex2`, gitleaks.

## 14. Rủi ro và vấn đề chưa giải quyết

**Rủi ro**

| Rủi ro | Mức | Giảm thiểu |
| --- | --- | --- |
| R2: OAuth file đọc được bởi tiến trình cùng user | Cao | Read deny, không mount; chấp nhận |
| Credential Manager không bảo vệ khỏi cùng user | Trung bình | Không lưu lâu; scope hàm; chấp nhận |
| Redaction miss với dạng secret mới | Trung bình | entropy + known-secret; corpus mở rộng |
| ReDoS trong regex | Thấp | `safe-regex2`, giới hạn input |
| Known-secret match tốn RAM/CPU | Thấp | Aho-Corasick trên biến thể |

**Vấn đề chưa giải quyết**

1. Bảng `secrets` hay chỉ `credential_ref`? (xem inconsistency: DATABASE_DESIGN không có bảng `secrets`).
2. `@napi-rs/keyring` vs `keytar` trên Node 24 + Tauri sidecar — chưa thử [Giả định].
3. `claude setup-token`/`apiKeyHelper` có phù hợp Tier 2 không (chưa kiểm chứng trên 2.1.312).
4. Server mode: Vault/SOPS/age (DEVSECOPS §14.2).
5. Có mã hoá `artifacts/` (transcript) ở rest không (DEVSECOPS §14.3).
6. Chính sách với MCP credential do connector quản lý (C-14).
7. Điều khoản dịch vụ khi nhiều agent dùng chung một OAuth login.

## 15. Acceptance Criteria

- [ ] SAST/CI: không có code backend đọc đường dẫn nhóm "CLI creds"; test fail khi thêm.
- [ ] `grep` schema: không cột chứa secret value; `credential_ref` chỉ là khoá keychain.
- [ ] Test env: biến secret ở backend không xuất hiện trong `process.env` của agent và của `docker inspect`.
- [ ] Redaction: corpus ≥ 200 mẫu × 6 encoding ⇒ 0 leak; benchmark < 1 ms/KB; `safe-regex2` pass.
- [ ] `SecretVault.use` ghi audit `secret_access` mỗi lần, không ghi value (kiểm tra bằng scan DB).
- [ ] Mô phỏng: logout Claude ⇒ mọi agent Claude pause trong ≤ 5 phút, UI hiện `needs_login`, không retry vô hạn.
- [ ] Rotate API key Tier 2: container mới dùng key mới, file cũ bị xoá, key cũ bị redact trong 30 ngày.
- [ ] Diễn tập incident lộ khoá theo DEVSECOPS §8 hoàn thành ≤ 1 giờ.
- [ ] gitleaks chặn `.env`/key giả trong diff agent; `.env.test` sinh tự động không chứa giá trị thật.
