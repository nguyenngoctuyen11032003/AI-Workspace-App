# LICENSE & REUSE — Phân tích license repo tham chiếu và đánh giá stack frontend/desktop

> Tác giả: Agent 10 (License & Reuse Compliance + Frontend Stack Evaluation)
> Ngày: 2026-10-30
> Phạm vi: 4 repo tham chiếu tại `D:\Code\Tuyen\Agent management\` (ChatDev, pixel-agents, my-virtual-office, OpenHands)
> Giả định sản phẩm: **AI WORKSPACE là sản phẩm proprietary / closed-source** (trừ khi có quyết định khác). Mọi verdict bên dưới dựa trên giả định này; nếu đổi sang MIT thì verdict chỉ nới lỏng hơn, không chặt hơn.
> Lưu ý: tài liệu này là phân tích kỹ thuật, không phải tư vấn pháp lý. Với quyết định thương mại lớn (đặc biệt liên quan AGPL) nên có luật sư xác nhận.

---

## 1. Tổng quan license từng repo

### 1.1. Bảng tóm tắt

| Repo | File license thực tế | License | Copyright holder | NOTICE / CREDITS riêng? | Kết luận nhanh |
|---|---|---|---|---|---|
| `ChatDev` | `ChatDev/LICENSE` | **Apache-2.0** | `Copyright 2025 OpenBMB` (appendix cuối file) | Không có NOTICE file | Copy được (kèm attribution + ghi chú thay đổi) |
| `pixel-agents` | `pixel-agents/LICENSE`, `package.json: "license": "MIT"` | **MIT** | `Copyright (c) 2026 Pablo De Lucca` | Không có CREDITS file; credit asset nằm trong `README.md` dòng 48 | Code copy được; **asset cần kiểm tra riêng** |
| `my-virtual-office` | `my-virtual-office/LICENSE`, `README.md` mục "License" | **AGPL-3.0-or-later** (+ commercial license bán riêng; có `app/license.py` kích hoạt Lemon Squeezy) | Không ghi tên chủ sở hữu trong LICENSE (text chuẩn FSF); README nói "Commercial licensing is available separately" | Không | **KHÔNG copy code**. Chỉ tham khảo ý tưởng |
| `OpenHands` (thực tế checkout là `@openhands/agent-canvas` v1.34.0 — frontend + Electron shell) | `OpenHands/LICENSE`, `package.json: "license": "MIT"` | **MIT** | `Copyright © 2025 OpenHands contributors` | Không | Copy được; chú ý logo/trademark và font Google Fonts |

Lệnh đã chạy để xác minh (không có file nào khác ngoài các file kể trên):

```
find <repo> -not -path '*/node_modules/*' \( -iname 'LICENSE*' -o -iname 'NOTICE*' -o -iname '*CREDIT*' -o -iname '*ATTRIBUTION*' -o -iname 'COPYING*' \)
# ChatDev           -> ChatDev/LICENSE
# pixel-agents      -> pixel-agents/LICENSE
# my-virtual-office -> my-virtual-office/LICENSE, my-virtual-office/app/license.py (không phải license file, là module kích hoạt key)
# OpenHands         -> OpenHands/LICENSE
```

### 1.2. ChatDev (Apache-2.0)

- `ChatDev/LICENSE` là Apache License 2.0 full text, có appendix `Copyright 2025 OpenBMB`.
- Repo **không có file NOTICE** → nghĩa vụ "preserve NOTICE" (mục 4(d) của Apache-2.0) không phát sinh nội dung gì thêm ngoài copyright line.
- Không có file CREDITS/ATTRIBUTION cho asset; thư mục `assets/` chủ yếu là ảnh README/demo (không dùng cho sản phẩm).
- README có mục "Citation" (bibtex các paper ChatDev, Co-Learning, MacNet, iAgents, Puppeteer) — đây là yêu cầu học thuật, **không phải** nghĩa vụ license, nhưng nên ghi credit paper nếu ta dùng ý tưởng "chat chain / role-play phases".
- Các phần có giá trị tái sử dụng: `workflow/`, `yaml_template/`, `yaml_instance/`, `schema_registry/`, `runtime/`, `entity/` (Python). Ta viết lại bằng TypeScript nên phần lớn là **port ý tưởng**, không copy nguyên văn; nhưng nếu copy prompt template / YAML schema thì vẫn phải attribution.

### 1.3. pixel-agents (MIT)

- `pixel-agents/LICENSE`: MIT, `Copyright (c) 2026 Pablo De Lucca`. `CONTRIBUTING.md` dòng 5 xác nhận mọi contribution cũng MIT, không CLA.
- **Code**: `core/`, `adapters/`, `server/`, `webview-ui/` (React + Tailwind v4, canvas renderer thuần, không dùng Phaser) — copy/modify tự do, chỉ cần giữ copyright + permission notice.
- **Asset** (quan trọng — phân tích kỹ):
  - `CHANGELOG.md` v1.1.0 / commit `add749a` (#117): *"Replaces bundled proprietary tileset with open-source assets... replace all previously privately-licensed assets with fully open-source ones"*. Tức là **trước v1.1.0 repo bundle tileset có license riêng (nhiều khả năng LimeZu Modern Office/Interiors); từ v1.1.0 đã thay bằng 25+ furniture "original", 9 floor, 1 wall set**. Commit message gọi furniture là "original" — hiểu là tác giả tự vẽ và phát hành theo MIT của repo. **Không có file license riêng cho asset, và `manifest.json` không có trường `author`/`license`** → về mặt pháp lý chúng nằm dưới MIT của repo, nhưng không có bằng chứng độc lập về nguồn gốc. Mức rủi ro: thấp-trung bình.
  - **Characters** (`webview-ui/public/assets/characters/char_0..5.png`): README dòng 48: *"based on the amazing work of JIK-A-4, Metro City"*. Trang itch.io của pack ghi license **Creative Commons Zero v1.0 Universal (CC0)**, tác giả nói "You can use it as you wish… Credits are not necessary but would be appreciated… you can also use it in a commercial project". → **An toàn để bundle**, nên credit tự nguyện.
  - **Pets** (`assets/pets/`), **carpets**, **walls**, **floors**: thêm sau #117 (commit `928ccd4`, `7a1643f`), không có credit riêng → coi như MIT của repo (cùng mức rủi ro như furniture).
  - **Font** `webview-ui/public/fonts/FSPixelSansUnicode-Regular.ttf` (FS Pixel Sans Unicode, tác giả NZW Studios/FontStruct): nguồn web **mâu thuẫn** (1001fonts: SIL OFL; cufonfonts: "personal use only"; trang FontStruct gốc đã tắt download). Repo không kèm file OFL.txt. → **Không bundle font này** vào sản phẩm thương mại cho đến khi xác minh được license; thay bằng font pixel có license rõ ràng (ví dụ "Press Start 2P" – OFL trên Google Fonts, "Pixelify Sans" – OFL, "Silkscreen" – OFL, hoặc Kenney Fonts – CC0).
  - `docs/external-assets.md` dòng 181 gợi ý người dùng tự mua "Office Interior Tileset (16x16)" của Donarg trên itch.io và import qua cơ chế external asset directory — đây chính là mô hình **"user-imported packs"** mà ta muốn áp dụng.
  - `scripts/` có pipeline 7 bước import tileset bên thứ ba (`0-import-tileset.ts`…): code MIT, copy được; dùng cho tính năng import pack của người dùng.

### 1.4. my-virtual-office (AGPL-3.0-or-later)

- `my-virtual-office/LICENSE` là AGPL v3 full text. `README.md` dòng 306-317 xác nhận `AGPL-3.0-or-later` và nói rõ: *"If you modify the app and make it available to users over a network, you must offer those users the corresponding source code"*; *"Commercial licensing is available separately for organizations that need terms outside the AGPL"*.
- `app/license.py`: hệ thống kích hoạt key qua Lemon Squeezy (Early Bird $9.99 / Full $35.99, demo giới hạn 3 agent, watermark). Đây là mô hình **open-core + license key**, chủ sở hữu rõ ràng có ý định thương mại hóa và bảo vệ bằng AGPL.
- Không có asset sprite riêng trong repo (chỉ `favicon.png`, `icon-192/512.png`, `screenshot.png`); sprite render bằng code trong `app/` (Python/JS).
- Kiến trúc: Python server (`app/server.py`), docker-compose, kasm browser, integrations. Mọi thứ đều AGPL.

### 1.5. OpenHands / agent-canvas (MIT)

- `OpenHands/LICENSE`: MIT, `Copyright © 2025 OpenHands contributors`. `package.json`: `@openhands/agent-canvas` 1.34.0, `"license": "MIT"`.
- **Không có** sub-folder với license khác, không có font file bundle, không có THIRD_PARTY file.
- Font: `src/index.css` import **Google Fonts "Outfit" và "IBM Plex Mono"** qua URL (cả hai là SIL OFL 1.1) — ta có thể dùng lại, nếu self-host thì kèm OFL.txt.
- Icon: dùng npm `lucide-react` 1.25.0 (ISC) và `react-icons` 5.6.0 (MIT; các icon set bên trong có license riêng — Font Awesome CC-BY-4.0, Material Apache-2.0, v.v. — `package-lock.json` của pixel-agents cũng cho thấy các dòng `"license": "CC-BY-4.0"`, `"CC-BY-3.0"`, `"CC0-1.0"` kiểu này).
- Logo OpenHands (`assets.openhands.dev/logo-*.png`) chỉ link từ README — **không được dùng logo/tên OpenHands** trong sản phẩm (trademark không nằm trong MIT grant).
- Docker: `docker/Dockerfile` base `node:24-slim` (MIT) và `ghcr.io/openhands/agent-server` (MIT upstream) — chỉ liên quan nếu ta ship image, hiện không.
- Electron shell (`electron/main.mjs`, `electron-builder.config.mjs`): bundle Node runtime + uv vào `extraResources`, `asar: false` để spawn child process, dùng `safeStorage` cho secrets, NSIS cho Windows. Đây là **tham chiếu rất tốt** cho phần Desktop (mục 5.3).

---

## 2. Nghĩa vụ (obligations) theo từng license

### 2.1. Apache-2.0 (ChatDev)

Khi phân phối sản phẩm (binary hoặc source) có chứa code/derivative của ChatDev:

1. **Kèm bản sao license** Apache-2.0 (mục 4(a)).
2. **Ghi rõ file đã sửa** ("You must cause any modified files to carry prominent notices stating that You changed the files" — mục 4(b)). Thực tế: thêm header `// Modified from ChatDev (OpenBMB), Apache-2.0. Changes: ...` ở đầu file được port.
3. **Giữ nguyên copyright/patent/trademark/attribution notices** trong source (mục 4(c)).
4. **NOTICE**: ChatDev không có NOTICE file → không có nghĩa vụ chép NOTICE, chỉ cần dòng copyright trong THIRD_PARTY_NOTICES.
5. **Patent grant**: ta được cấp patent license; nếu ta kiện OpenBMB về patent liên quan thì grant chấm dứt (mục 3).
6. **Không dùng tên "ChatDev"/"OpenBMB"** để quảng bá sản phẩm (mục 6).
7. Apache-2.0 **không copyleft** → sản phẩm proprietary được phép, không cần công bố source của ta.

### 2.2. MIT (pixel-agents, OpenHands)

1. **Giữ copyright notice + permission notice** trong mọi bản sao hoặc "substantial portions" → chép nguyên khối MIT text kèm tên tác giả vào `THIRD_PARTY_NOTICES.md` (và/hoặc màn hình About/Licenses trong app).
2. Không có nghĩa vụ ghi chú thay đổi, không copyleft, không patent clause.
3. Không có trademark grant → không dùng tên/logo "Pixel Agents", "OpenHands".
4. Nếu chỉ port **ý tưởng** (không copy code) thì về mặt pháp lý không bắt buộc attribution, nhưng khuyến nghị ghi credit vì thiện chí cộng đồng.

### 2.3. AGPL-3.0-or-later (my-virtual-office)

1. **Copyleft mạnh**: mọi "covered work" (bao gồm derivative, và theo cách hiểu phổ biến cả chương trình link/tích hợp chặt) phải phát hành **toàn bộ source** dưới AGPL-3.0 khi phân phối (mục 5, 6).
2. **Network use clause (mục 13)**: khác GPL, nếu ta **chỉ chạy phiên bản sửa đổi trên server cho người dùng truy cập qua mạng** (SaaS, không phân phối binary) thì **vẫn** phải cung cấp source cho các user đó. Với AI WORKSPACE (có backend NestJS phục vụ qua HTTP/WS) đây là điểm chết người: chỉ cần 1 module copy từ my-virtual-office là toàn bộ backend có nguy cơ bị kéo vào AGPL.
3. **Không tương thích với proprietary**: không có cách nào bundle code AGPL vào sản phẩm closed-source mà không vi phạm, trừ khi mua **commercial license** từ tác giả (README nói có).
4. Phạm vi "an toàn": đọc để hiểu **ý tưởng, kiến trúc, UX flow, behavior** (idle activities, pet, speech bubble, gender-aware sprite...) rồi **tự viết lại từ đầu** bằng TypeScript. Khuyến nghị "clean-room": người đọc code AGPL viết spec/mô tả bằng lời; người khác implement từ spec, không mở file gốc.
5. Cũng **không copy** prompt, text UI, layout JSON, sprite data generated từ repo này (đều là "work" dưới AGPL).

### 2.4. Asset license (tóm tắt, chi tiết ở mục 6)

| License | Bundle vào sản phẩm closed-source? | Attribution | Redistribute (ta là bên phân phối lại asset trong installer) |
|---|---|---|---|
| CC0 1.0 | Có | Không bắt buộc (nên ghi) | Có |
| CC-BY 4.0 | Có | **Bắt buộc** (tên, link, license, ghi chú thay đổi) | Có |
| CC-BY-SA 4.0 | Rủi ro (ShareAlike cho asset derivative; không "lây" sang code nhưng sprite sửa đổi phải SA) | Bắt buộc | Có, nhưng derivative phải SA |
| SIL OFL 1.1 (font) | Có | Giữ copyright + OFL text; không bán font riêng lẻ | Có |
| LimeZu (custom EULA itch.io) | **Không** — "You CAN'T resell or distribute the asset to others" | Bắt buộc ("Credits required") | **Không** |
| Donarg / các itch.io pack trả phí khác | Thường tương tự LimeZu — mỗi pack phải đọc EULA | Tùy | Thường không |

---

## 3. Verdict tái sử dụng

### 3.1. Bảng verdict

| Repo | Verdict | Được copy | Chỉ tham khảo (reference-only) |
|---|---|---|---|
| **ChatDev** (Apache-2.0) | **ALLOW** với attribution + ghi chú thay đổi | Ý tưởng chat-chain, phase/role YAML schema, prompt templates (có attribution), thuật toán điều phối (port sang TS) | Không có phần cấm; chỉ tránh dùng tên/logo ChatDev |
| **pixel-agents** (MIT) | **ALLOW** code; asset: **ALLOW có điều kiện** chỉ characters CC0, furniture/floor/wall/pet/carpet **KHÔNG dùng** (OWR-2 Q-15 + OWR-3 A.9: nguồn gốc không rõ) | `core/`, `server/`, `webview-ui/` (renderer canvas, layout editor, pathfinding, manifest loader, import-tileset scripts); characters CC0 (JIK-A-4); furniture/floor/wall/pet/carpet post-#117 (~~MIT theo repo, ghi credit~~ **nguồn gốc không rõ, loại khỏi bundle; chỉ xem xét lại nếu xác minh nguồn + license + ghi attribution trong NOTICE và Owner duyệt**) | **Font FS Pixel Sans** (license chưa rõ → không bundle); **bất kỳ asset nào ở commit trước `add749a`** (tileset proprietary cũ) — tuyệt đối không lấy từ git history |
| **my-virtual-office** (AGPL-3.0) | **DENY** copy code/asset/prompt/data | Không gì cả | Toàn bộ: chỉ đọc để học ý tưởng (behavior system, pet, idle events, license-key UX, docker layout). Áp dụng clean-room |
| **OpenHands/agent-canvas** (MIT) | **ALLOW** | Pattern Electron shell (`electron/main.mjs`, `electron-builder.config.mjs`), UI component, hooks, MSW mocks, Playwright config | Logo/tên OpenHands; không copy ảnh `public/*.png` (icon brand) |

### 3.2. Quy tắc thực thi trong repo AI-Workspace-App

1. Mọi file có code copy/port từ ChatDev/pixel-agents/OpenHands phải có header:
   ```ts
   // Portions derived from <repo> (<URL>), Copyright (c) <year> <holder>, licensed under <MIT|Apache-2.0>.
   // Modified by AI WORKSPACE team: <tóm tắt thay đổi>.   // bắt buộc với Apache-2.0
   ```
2. Thêm `THIRD_PARTY_NOTICES.md` ở root + màn hình "About → Open-source licenses" trong app (Tauri/web) render nội dung này.
3. Thêm CI check (ví dụ `license-checker` / `pnpm licenses list`) chặn dependency có license `AGPL*`, `GPL*`, `SSPL`, `BUSL`, `UNKNOWN`.
4. Không clone `my-virtual-office` vào trong monorepo; giữ ở thư mục tham chiếu riêng, không symlink, không import.
5. Asset bundle mặc định: **chỉ CC0** (OWR-2 Q-15, thay thế "CC0 hoặc MIT/CC-BY"). Font **OFL** được phép nếu lưu đầy đủ license/NOTICE. **CC-BY** chỉ dùng khi Owner phê duyệt từng package. Asset MIT có nguồn gốc không rõ (ví dụ furniture/floor/wall/carpet/pet của Pixel Agents) **không bundle mặc định**. Mỗi asset có `LICENSE.txt` + `CREDITS.json` cạnh asset. Việc tái dùng **code** MIT/Apache-2.0 không đổi (D-012; xác minh license và attribution). **[OWR-3 A.9 làm rõ Q-15]** Asset MIT chỉ được dùng nếu xác minh được nguồn gốc VÀ license VÀ ghi attribution vào `NOTICE`; nguồn gốc không rõ thì KHÔNG dùng; CC0 vẫn là mặc định; font OFL cho phép kèm license/NOTICE; CC-BY theo từng package khi Owner phê duyệt. Furniture/tile Pixel Agents (nguồn gốc không rõ) tiếp tục bị loại.

---

## 4. THIRD_PARTY_NOTICES template

Nội dung mẫu cho file `THIRD_PARTY_NOTICES.md` (cập nhật khi thêm dependency/asset):

```markdown
# Third-Party Notices — AI WORKSPACE

AI WORKSPACE incorporates material from the projects listed below. Each is
provided under its own license; the full license text follows each entry.
Nothing in this file grants any right to use the names, logos or trademarks
of the projects or their authors.

---------------------------------------------------------------------------
## ChatDev
Source:    https://github.com/OpenBMB/ChatDev
Copyright: Copyright 2025 OpenBMB
License:   Apache License 2.0
Used for:  Multi-agent "chat chain" orchestration concepts, phase/role
           configuration schema and prompt templates (ported to TypeScript).
Changes:   Files derived from ChatDev have been modified by the AI WORKSPACE
           team; modified files carry a change notice in their header.

Licensed under the Apache License, Version 2.0 (the "License"); you may not
use this file except in compliance with the License. You may obtain a copy
of the License at http://www.apache.org/licenses/LICENSE-2.0
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
License for the specific language governing permissions and limitations
under the License.
[Full text: see licenses/Apache-2.0.txt]

---------------------------------------------------------------------------
## Pixel Agents
Source:    https://github.com/pixel-agents-hq/pixel-agents
Copyright: Copyright (c) 2026 Pablo De Lucca
License:   MIT License
Used for:  Office canvas renderer, layout editor, asset manifest format,
           tileset import pipeline, bundled furniture/floor/wall sprites.

MIT License

Copyright (c) 2026 Pablo De Lucca

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

---------------------------------------------------------------------------
## OpenHands Agent Canvas
Source:    https://github.com/OpenHands/OpenHands
Copyright: Copyright © 2025 OpenHands contributors
License:   MIT License
Used for:  Desktop shell (Electron bootstrap patterns), UI components/hooks.
[MIT text as above, with the OpenHands copyright line]

---------------------------------------------------------------------------
## MetroCity — Free Top Down Character Pack (sprites)
Author:    JIK-A-4 — https://jik-a-4.itch.io/metrocity-free-topdown-character-pack
License:   Creative Commons Zero v1.0 Universal (CC0 1.0)
Note:      Attribution not required; included with thanks. Character sprites
           in AI WORKSPACE are derived from this pack.

---------------------------------------------------------------------------
## <Asset pack name> (CC-BY 4.0 example)
Author:    <name> — <URL>
License:   CC BY 4.0 — https://creativecommons.org/licenses/by/4.0/
Changes:   Recolored / resized / sliced into per-item sprites by AI WORKSPACE.

---------------------------------------------------------------------------
## Fonts
- Press Start 2P — Copyright 2012 The Press Start 2P Project Authors
  (CodeMan38), SIL Open Font License 1.1. [licenses/OFL.txt]
- IBM Plex Mono — Copyright © 2017 IBM Corp., SIL Open Font License 1.1.

---------------------------------------------------------------------------
## npm / crates dependencies
Generated automatically at build time — see `THIRD_PARTY_NOTICES.deps.txt`
(produced by `pnpm licenses list --json` and `cargo about`).
```

Ghi chú:
- **Không** liệt kê my-virtual-office trong NOTICES (vì không được chứa bất kỳ phần nào của nó). Nếu team muốn ghi "inspired by", để ở README/credits dạng tự nguyện, không phải NOTICE pháp lý.
- Với Apache-2.0, nếu sau này ChatDev thêm file `NOTICE`, phải chép nội dung đó vào mục ChatDev.

---

## 5. Đánh giá stack frontend/desktop (cập nhật tháng 10/2026)

### 5.1. Game engine cho virtual office 2D: Phaser vs PixiJS vs plain Canvas

**Hiện trạng version (từ web search 10/2026):**
- **Phaser**: v3.90.0 "Tsugumi" (05/2025) là bản 3.x cuối. **Phaser 4.0.0 "Caladan" phát hành 10/04/2026**, v4.1.0 (30/04/2026), jsDelivr đã có 4.2.0. v4 thay toàn bộ WebGL renderer, thêm filters/lighting mới; API Scene/GameObject/Tilemap/Input/Camera vẫn tương thích phần lớn với v3. License **MIT** (Richard Davey / Phaser Studio Inc.).
- **PixiJS**: v8.x (8.19.0 vào 07/2026), WebGPU + WebGL, license MIT. `@pixi/react` v8 chính thức (hỗ trợ React 19, JSX `<pixiContainer>`, `<pixiSprite>`), fork VisualElectric đã archived 03/2026.
- **Plain Canvas 2D**: không thư viện. pixel-agents hiện dùng cách này (renderer trong `webview-ui/src/`), chứng minh đủ cho ~10-20 nhân vật.

**So sánh cho yêu cầu của ta** (10-50 nhân vật, tilemap, pathfinding, zoom/pan, layout editor):

| Tiêu chí | Phaser 3.90 / 4.x | PixiJS 8 | Plain Canvas |
|---|---|---|---|
| Tilemap (Tiled JSON, layer, collision) | **Built-in** (`Tilemaps`, hỗ trợ Tiled) | Không có; dùng `@pixi/tilemap` (plugin riêng) hoặc tự viết | Tự viết |
| Camera zoom/pan, bounds, follow | **Built-in** (`Cameras.Scene2D`) | Tự viết (pixi-viewport – MIT, cộng đồng) | Tự viết |
| Animation spritesheet, atlas | Built-in | Built-in (`AnimatedSprite`, Assets) | Tự viết |
| Pathfinding | Không built-in; dùng `easystarjs` (MIT) hoặc A* tự viết trên grid | Tương tự | Tương tự |
| Input (drag, pointer over sprite, hit test) | Built-in | Built-in (EventSystem) | Tự viết |
| Pixel-art (nearest filter, `roundPixels`, zoom integer) | `pixelArt: true` config | `TextureSource.scaleMode = 'nearest'` | `imageSmoothingEnabled=false` |
| Hiệu năng 50 nhân vật + tilemap 100x100 | Dư sức (WebGL batching) | Dư sức, nhỉnh hơn về raw render | Đủ nếu culling tốt, nhưng Canvas 2D không batch |
| Bundle size (gzip) | ~1.2 MB (v3) / tương tự v4, tree-shake kém | ~150-300 KB tùy module | 0 |
| React integration | Game chạy trong `<div ref>`, giao tiếp qua **EventEmitter / Zustand store** (không phải React tree); có `phaserjs/template-react` chính thức (Vite + React) | `@pixi/react` v8: render cảnh ngay trong JSX, hooks (`useTick`, `useApplication`) → tự nhiên với React | Hook `useEffect` + `requestAnimationFrame`, hoàn toàn tự quản |
| Layout editor (drag furniture, snap grid, rotate) | Dễ: Scene riêng "EditorScene", `Input.setDraggable` | Dễ, nhưng tự viết snap/grid | Tự viết hết |
| Độ chín & docs | Rất lớn, nhiều ví dụ pixel-art/top-down | Lớn, nhưng thiên về render, ít "game framework" | — |
| Rủi ro | v4 còn mới (6 tháng); v3.90 ổn định nhưng không còn feature mới | Phụ thuộc plugin cộng đồng cho tilemap/viewport | Tốn thời gian engineering, dễ tái phát minh |

**Khuyến nghị: Phaser** — bắt đầu với **Phaser 3.90** cho MVP (ổn định, docs/ví dụ nhiều, LLM sinh code tốt), cấu trúc code để chuyển lên **Phaser 4.x** khi tính năng filter/lighting cần thiết (API tương thích cao). Lý do chính: tilemap + camera + input đã có sẵn, cắt giảm ~60% code hạ tầng so với PixiJS/Canvas; yêu cầu "layout editor + pathfinding + zoom/pan" là đúng "sweet spot" của game framework hơn là render library.

Pattern tích hợp React ↔ Phaser khuyến nghị:
```
<OfficeCanvas/> (React)  ──ref──>  new Phaser.Game({ parent, scene: [BootScene, OfficeScene, EditorScene], pixelArt: true })
Zustand store (agents, layout, selection)  <──subscribe──  OfficeScene (đọc state, không giữ state riêng)
OfficeScene.events / EventBus (mitt)  ──>  React (click agent → mở side panel)
TanStack Query → cập nhật store → Phaser re-render diff (không re-create Game khi React re-render)
```
- Tuyệt đối không tạo `Phaser.Game` trong render body; tạo một lần trong `useEffect`, destroy ở cleanup (lưu ý React 19 StrictMode mount 2 lần ở dev).
- Dữ liệu layout/sprite manifest tái dùng format `manifest.json` của pixel-agents (MIT) để tương thích pack cộng đồng.

Fallback: nếu bundle size là ưu tiên tuyệt đối (web embed), PixiJS 8 + `pixi-viewport` + `easystarjs` là lựa chọn số 2.

### 5.2. Workflow builder: React Flow (`@xyflow/react` v12)

- Package `@xyflow/react` v12.x (12.10.x hiện hành), **MIT**. "React Flow Pro" ($169–289/tháng) chỉ là gói hỗ trợ + quyền **gỡ attribution badge** ("React Flow" link góc canvas) + Pro examples; **không có feature nào bị khóa** trong thư viện.
- Với sản phẩm closed-source: giữ badge (miễn phí) hoặc mua Pro để gỡ. Theo trang attribution, với sản phẩm thương mại họ *yêu cầu* mua Pro nếu gỡ badge — đây là ràng buộc đạo đức/điều khoản dịch vụ, không phải MIT; quyết định business.
- Phù hợp cho workflow builder: custom node (`nodeTypes`), custom edge, handles có `isValidConnection`, `onConnect` validation, `useReactFlow()` để thêm node programmatic, minimap/controls/background sẵn, sub-flow (parent nodes), `NodeResizer`, `NodeToolbar`. Validation phức tạp (DAG cycle check, schema) tự viết trên `nodes/edges` state → nên đặt vào Zustand store (React Flow khuyến nghị pattern này) hoặc dùng `useNodesState`.
- Thay thế đã xem: Rete.js (MIT, nặng hơn), Drawflow (MIT, không React-first), tldraw (license riêng có watermark/ không phải MIT thuần) → **chọn React Flow v12**.

### 5.3. Desktop shell: Tauri 2 vs Electron

**Yêu cầu**: spawn CLI local (`claude`, `codex`), quản lý backend NestJS (Node) sidecar hoặc embed, OS keychain, Windows installer.

| Tiêu chí | Tauri 2 (2.11–2.12, Rust + WebView2) | Electron (v44, Chromium + Node) |
|---|---|---|
| Spawn CLI process | `tauri-plugin-shell` (`Command.create`, sidecar) với capability allowlist; stream stdout/stderr qua event | `child_process.spawn` trực tiếp từ main process — đơn giản nhất |
| Chạy backend Node (NestJS) | Phải bundle **sidecar** (xem bên dưới) hoặc yêu cầu Node cài sẵn | Node đã có sẵn trong runtime → `fork()` NestJS ngay trong app; OpenHands/agent-canvas còn bundle thêm Node runtime riêng vào `extraResources` và tắt `asar` để spawn |
| Secrets / keychain | `tauri-plugin-stronghold` (vault file mã hóa, không phải OS keychain) hoặc crate `keyring` qua `tauri-plugin-keyring-store` (Windows Credential Manager, cộng đồng) | `safeStorage` (DPAPI trên Windows) + lưu file; hoặc `keytar` (deprecated) |
| Windows installer | NSIS + MSI (WiX) built-in, code signing, updater plugin | NSIS/MSI qua electron-builder, auto-update (electron-updater) |
| Kích thước / RAM | ~8–15 MB installer, 30–50 MB RAM idle (chưa tính sidecar Node ~40-80 MB) | 80–240 MB installer, 200–300 MB RAM |
| Render engine | WebView2 (Edge Chromium) trên Windows — ổn; macOS WebKit và Linux WebKitGTK lệch → bug cross-platform | Chromium đồng nhất mọi OS |
| Ngôn ngữ native | Rust (team cần học; ~2-3 tháng để thành thạo theo kinh nghiệm cộng đồng) | JS/TS toàn bộ |
| Bảo mật mặc định | Capability/permission per-window, CSP; tốt hơn | Phải tự cấu hình contextIsolation, sandbox |
| License | MIT/Apache-2.0 | MIT |

**Bundle NestJS làm sidecar với Tauri** (theo `tauri.app/learn/sidecar-nodejs`):
1. Build NestJS → single-file JS bằng `esbuild`/`ncc` (CommonJS, `platform: node`, external native modules).
2. Đóng gói thành executable:
   - **Node SEA** (Single Executable Applications): Node 24 vẫn cần `--experimental-sea-config` + `postject` (stability 1.1 "Active development"); Node 25.5+ có `node --build-sea` một bước và hỗ trợ ESM entry. Native addon (`better-sqlite3`, `node-pty`) **không** nhúng được → phải để ngoài dưới dạng resources và `require` theo đường dẫn tuyệt đối.
   - **pkg** (Vercel, đã deprecated nhưng vẫn chạy) / `@yao-pkg/pkg` (fork đang maintain) / `nexe`: dễ hơn SEA, hỗ trợ asset snapshot, nhưng phụ thuộc prebuilt Node binary.
   - Hoặc **ship Node runtime** (node.exe ~70 MB) + JS bundle trong `resources/` và spawn `node main.js` — cách OpenHands/agent-canvas làm với Electron; với Tauri cũng khả thi (`bundle.resources`).
3. Đổi tên binary theo target triple (`backend-x86_64-pc-windows-msvc.exe`) vào `src-tauri/binaries/`, khai báo `bundle.externalBin`, cấp `shell:allow-execute`/`shell:allow-spawn` trong capabilities.
4. Rust `setup()` spawn sidecar, chọn port rảnh, truyền token qua env, đợi `GET /health` 200, rồi mở window; `on_window_event(Destroyed)` kill sidecar. Tauri docs ghi rõ guide sidecar nhắm tới process ngắn hạn; backend dài hạn nên giao tiếp qua **localhost HTTP/WS** (đúng mô hình của ta) thay vì stdin/stdout.
5. Trade-off vs. chạy backend như dev process riêng: dev chạy `nest start --watch` + `vite` + `tauri dev` (3 process, `beforeDevCommand`), không bundle sidecar; chỉ production mới bundle. Điểm đau: native module ABI (node-pty, sqlite) phải khớp Node version của sidecar; code signing 2 binary; update backend = update cả app.

**Khuyến nghị cho MVP**:
1. **Phase 1 (MVP)**: **Web app chạy trong browser + NestJS backend chạy local như process riêng** (`pnpm dev` / `npx ai-workspace start` mở browser). Lý do: (a) không chặn bởi packaging/sidecar/code-signing; (b) spawn `claude`/`codex` CLI làm **từ NestJS** (Node `child_process`/`node-pty`) nên desktop shell không cần quyền gì đặc biệt; (c) hot-reload nhanh, debug dễ; (d) pixel-agents và my-virtual-office đều chứng minh mô hình browser + local server dùng tốt; (e) secrets lưu qua `@napi-rs/keyring` (Rust keyring binding cho Node, MIT) ngay trong NestJS → về sau Tauri/Electron chỉ là vỏ.
2. **Phase 2**: **Tauri 2 shell** bọc web app, backend NestJS bundle sidecar (ưu tiên `@yao-pkg/pkg` hoặc Node SEA khi Node 26 LTS ổn định), installer NSIS, `tauri-plugin-updater`. Chọn Tauri thay Electron vì: installer/RAM nhỏ, bảo mật capability tốt hơn, WebView2 có sẵn trên Windows 11 (platform mục tiêu chính), và mọi logic native thực ra đã nằm trong NestJS nên phần Rust rất mỏng (spawn sidecar + keychain + tray).
3. **Khi nào chọn Electron thay thế**: nếu phải hỗ trợ Linux rộng (WebKitGTK yếu), cần `node-pty` terminal nhúng trực tiếp trong renderer, hoặc team không muốn Rust. Có thể tái dùng pattern `electron/main.mjs` của OpenHands/agent-canvas (MIT).

### 5.4. Frontend toolchain & thư viện

**Version khuyến nghị (10/2026)**: Vite 7.x; React 19.2.x; TypeScript 5.9+; Tailwind CSS **4.1.x** (`@tailwindcss/vite`, cấu hình CSS-first `@theme`, không còn `tailwind.config.js`); shadcn/ui CLI mới (`npx shadcn@latest init`, đã hỗ trợ Tailwind v4 + React 19; dùng `tw-animate-css` thay `tailwindcss-animate`); lucide-react (ISC) cho icon. pixel-agents đã migrate webview-ui sang Tailwind v4 (commit `dfb5b30`) → tham chiếu cấu hình được.

**State: Zustand vs Redux Toolkit**
- Zustand 5.0.x: ~0.5 KB gzip, API hook đơn giản, không boilerplate, middleware `persist`/`devtools`/`immer`, hoạt động ngoài React (quan trọng: **Phaser scene đọc store trực tiếp qua `store.getState()`/`subscribe`**). v5 chuyển `useShallow` sang `zustand/react/shallow`.
- Redux Toolkit 2.12: ~13 KB gzip, time-travel devtools, RTK Query, phù hợp team lớn đã quen Redux.
- **Chọn Zustand** (client state: selection, UI, editor mode, agent runtime cache) + **TanStack Query v5** (server state: REST từ NestJS; `useQuery`/`useMutation`, cache invalidation theo WS event) — đây là cặp phổ biến nhất 2026. Chỉ cân nhắc RTK nếu cần audit log/undo-redo phức tạp toàn cục (undo trong layout editor có thể dùng `zundo`).

**Event streaming NestJS → React: socket.io vs native ws**
- NestJS hỗ trợ hai adapter: `@nestjs/platform-socket.io` (mặc định) và `@nestjs/platform-ws`.
- **socket.io v4**: rooms/namespaces, auto-reconnect, ack, binary, fallback long-polling; client `socket.io-client` (~10 KB gzip); không nói chuyện với native `WebSocket`; scale nhiều instance cần redis-adapter + sticky session. Tauri WebView2 và browser dùng được bình thường.
- **ws**: nhẹ, native WebSocket client (browser/Tauri/CLI khác đều kết nối được), nhanh hơn, không rooms/namespace (tự mô phỏng), không auto-reconnect (tự viết hoặc `reconnecting-websocket`).
- **Khuyến nghị**: **socket.io** cho MVP — ta cần rooms theo `project/agent/run`, reconnect tin cậy khi backend restart, và ack cho lệnh điều khiển agent; backend local single-instance nên không gặp vấn đề scale. Thiết kế payload là JSON event `{type, runId, ts, data}` tách khỏi transport để sau này đổi sang `ws`/SSE nếu cần (ví dụ cho CLI/third-party client). Dùng SSE (`@Sse()` của NestJS) cho stream token LLM một chiều nếu muốn tận dụng HTTP cache/proxy đơn giản.

### 5.5. Tóm tắt stack đề xuất

| Lớp | Chọn | License |
|---|---|---|
| Build | Vite 7 + TypeScript 5.9 | MIT |
| UI | React 19.2 + Tailwind 4.1 + shadcn/ui + lucide-react | MIT / MIT / MIT / ISC |
| Office 2D | Phaser 3.90 → 4.x (+ easystarjs cho A*) | MIT |
| Workflow | @xyflow/react 12 | MIT (badge attribution hoặc Pro) |
| State / API | Zustand 5 + TanStack Query 5 | MIT |
| Realtime | socket.io 4 (NestJS gateway) | MIT |
| Desktop | Phase 1 browser + local NestJS; Phase 2 Tauri 2 + sidecar | MIT/Apache-2.0 |

---

## 6. Đề xuất asset strategy

### 6.1. Kết quả khảo sát asset pack

| Pack | Nguồn | License (đã xác minh) | Chibi/office? | Bundle mặc định? |
|---|---|---|---|---|
| **MetroCity – Free Top Down Character Pack** (JIK-A-4) | itch.io | **CC0 1.0 [chưa lưu bằng chứng: cần ảnh chụp trang itch.io trong PR]**, tác giả xác nhận commercial OK | Chibi top-down, hiện đại, 16x16-ish; pixel-agents đã dùng | **Có** |
| **MetroCity – Environment / Cars** (JIK-A-4) | itch.io | CC0 1.0 | Môi trường ngoài trời (bổ sung) | Có |
| **pixel-agents furniture/floors/walls/pets/carpets** (post-#117) | GitHub (MIT repo) | MIT theo repo; commit nói "original" | Office nội thất 16x16 | **Có** (ghi credit Pablo De Lucca) |
| **LimeZu Modern Office** | itch.io ($5) | EULA riêng: commercial OK, **credits required**, **"CAN'T resell or distribute the asset to others"**, "CAN'T edit and resell" | Rất đẹp, chuẩn chibi office, là "chuẩn ngành" | **Không** — chỉ cho phép user tự mua và import |
| **LimeZu Modern Interiors / Modern Exteriors** | itch.io | Comment cộng đồng nói CC-BY nhưng trang pack nêu điều khoản tương tự Modern Office; license từng đổi giữa phiên bản | Chibi | **Không** (treat như Modern Office) |
| **Donarg Office Interior Tileset 16x16** | itch.io | Trả phí, EULA riêng (pixel-agents docs gợi ý user tự mua) | Office | Không — user import |
| **Kenney** (kenney.nl) | kenney.nl / itch.io / OGA | **CC0** toàn bộ (fonts, UI, pixel packs) | Không có office chibi chuyên dụng; dùng cho UI, icons, cursor, font | Có (UI/icons) |
| **OpenGameArt** (lọc CC0 / OGA-BY) | opengameart.org | Per-asset; phải đọc từng trang | LPC (CC-BY-SA/GPL — tránh), một số CC0 | Chỉ khi CC0 |
| **0x72 / Pixel Frog** | itch.io | CC0 | Không phải office (dungeon/platformer) | Không cần |
| **Fonts**: Press Start 2P, Pixelify Sans, Silkscreen, VT323 | Google Fonts | SIL OFL 1.1 | Pixel UI font | Có |
| **Font** FS Pixel Sans Unicode (trong pixel-agents) | FontStruct | Mâu thuẫn (OFL vs personal-use) | — | **Không** cho đến khi xác minh |

### 6.2. Chiến lược

1. **Bundled defaults = CC0 (OWR-2 Q-15; OFL font kèm license/NOTICE; CC-BY chỉ từng package được Owner duyệt; mục "MIT/OFL" cũ bị thay thế)**:
   - Nhân vật: MetroCity (CC0) → tự vẽ thêm biến thể màu da/tóc/trang phục (derivative CC0 → ta sở hữu phần sửa, có thể giữ proprietary hoặc release lại CC0).
   - Nội thất/floor/wall: ~~bộ pixel-agents (MIT) làm nền~~ **[OWR-2 Q-15: KHÔNG bundle mặc định vì nguồn gốc art chưa xác nhận; chỉ dùng nếu tác giả xác nhận bằng văn bản và Owner duyệt]**; dùng asset CC0/tự vẽ; bổ sung do designer nội bộ vẽ (style 16x16, palette thống nhất) → tài sản riêng của công ty.
   - UI/icon/cursor/font: Kenney (CC0), Press Start 2P / Pixelify Sans (OFL).
   - Mỗi thư mục asset có `CREDITS.json` `{ "author", "source", "license", "modified": true/false }` và build script sinh phần "Assets" trong `THIRD_PARTY_NOTICES.md`.
   - **Không bao giờ** đưa CC-BY-SA, GPL (LPC), hoặc asset itch.io có "no redistribution" vào installer / repo / docker image / CDN.
2. **User-imported packs (BYO assets)**:
   - Tái dùng cơ chế của pixel-agents: thư mục ngoài (`~/.ai-workspace/assets/<pack>/`), `manifest.json` per item, hot-reload vào palette; port `scripts/import-tileset` thành wizard trong app (cắt tileset → PNG per item + manifest).
   - Cung cấp **preset manifest cho LimeZu Modern Office/Interiors** (chỉ toạ độ cắt, **không chứa pixel**) để user mua pack ($5) rồi trỏ vào file zip → app tự slice. Đây là cách hợp pháp vì ta không phân phối asset, user là licensee. Hiển thị nhắc "Credit: LimeZu" theo yêu cầu "Credits required".
   - Layout/office share giữa user chỉ gồm **tham chiếu asset id**, không nhúng pixel của pack trả phí; nếu người nhận không có pack → fallback placeholder.
3. **Asset "marketplace" tương lai**: chỉ nhận pack có license machine-readable (SPDX id trong manifest: `CC0-1.0`, `CC-BY-4.0`, `MIT`, `OFL-1.1`; **OWR-2 Q-15: mặc định chỉ `CC0-1.0`, `OFL-1.1` cho font kèm NOTICE, `CC-BY-4.0` cần Owner duyệt từng package, `MIT` asset không rõ nguồn không bundle mặc định**); từ chối `CC-BY-NC*`, `CC-BY-SA*` cho pack mặc định (NC cấm thương mại, SA lây sang derivative).
4. **Kiểm tra lịch sử git**: khi port code/asset từ pixel-agents, chỉ checkout từ commit `add749a` trở về sau; không lấy bất kỳ PNG nào trong history trước đó (tileset proprietary cũ).
5. **Quy trình review asset**: PR thêm asset phải đính kèm link nguồn + screenshot trang license; CI fail nếu `CREDITS.json` thiếu hoặc license không thuộc allowlist.

### 6.3. Việc cần làm ngay (action items)

- [ ] Tạo `THIRD_PARTY_NOTICES.md` + `licenses/` (Apache-2.0.txt, MIT.txt, OFL.txt, CC0.txt) theo template mục 4.
- [ ] Thêm `license-checker`/`pnpm licenses` vào CI với deny-list AGPL/GPL/SSPL/BUSL/UNKNOWN.
- [ ] Thay font FS Pixel Sans bằng Press Start 2P hoặc Pixelify Sans (OFL).
- [ ] Viết ADR ghi rõ: my-virtual-office = reference-only, clean-room; ChatDev = port có attribution + change notice.
- [ ] Hỏi tác giả pixel-agents (issue/discussion) để xác nhận bằng văn bản nguồn gốc bộ furniture "original" post-#117 — giảm rủi ro còn lại.
- [ ] Quyết định business: giữ badge React Flow hay mua React Flow Pro.

---

## Nguồn tham khảo

- Phaser releases: https://phaser.io/download/phaser4 , https://github.com/phaserjs/phaser/releases , license: https://phaser.io/download/license
- PixiJS React v8: https://pixijs.com/blog/pixi-react-v8-live ; VisualElectric fork archived: https://github.com/VisualElectric/pixi-react/releases
- React Flow: https://www.npmjs.com/package/@xyflow/react , https://reactflow.dev/attribution
- Tauri sidecar Node.js: https://tauri.app/learn/sidecar-nodejs ; Stronghold: https://rust-digger.code-maven.com/crates/tauri-plugin-stronghold ; keyring store: https://docs.rs/tauri-plugin-keyring-store
- Node SEA: https://github.com/nodejs/single-executable , https://docs.sheetjs.com/docs/demos/cli/nodesea
- Tauri vs Electron 2026: https://moklabs-research.pages.dev/reports/technology/2026-03-20-tauri-vs-electron-desktop-framework-decision , https://dev.to/johalputt/retrospective-6-months-of-using-tauri-20-for-desktop-apps-60-smaller-installers-20-more-13m4
- Tailwind v4 + shadcn: https://v3.shadcn.com/docs/tailwind-v4 , https://itsourcecode.com/web-dev/tailwind-css-v4-complete-guide-2026/
- Zustand vs RTK: https://www.pkgpulse.com/guides/zustand-vs-redux-toolkit-2026
- NestJS WS adapters: https://docs.nestjs.com/websockets/gateways , https://docs.nestjs.cn/websockets/adapter
- LimeZu Modern Office license: https://limezu.itch.io/modernoffice
- JIK-A-4 MetroCity (CC0): https://jik-a-4.itch.io/metrocity-free-topdown-character-pack
- Kenney CC0: https://forum.godotengine.org/t/kenneys-assets-free-and-creative-commons-cc0/36658
- FS Pixel Sans license (mâu thuẫn): https://www.1001fonts.com/fs-pixel-sans-unicode-regular-font.html , https://www.cufonfonts.com/font/fs-pixel-sans-unicode
