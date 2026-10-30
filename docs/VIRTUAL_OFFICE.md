# VIRTUAL_OFFICE.md — Virtual Office 2D: engine, nhân vật, layout, hiệu năng, asset

Tác giả: Agent Virtual Office — Phase 0, chỉ tài liệu. Ngày: 2026-10-30. Trạng thái: DRAFT chờ phê duyệt.
Nhãn: **[Đã xác minh]** (có trong research/spec), **[Đề xuất]**, **[Giả định]**, **[Cần người dùng phê duyệt]**.
> **[OWR-2 — cập nhật Phase 0, chưa phải Phase 0 hoàn tất]** D-011: Phaser 3.90 qua abstraction **`OfficeRenderer`**; **benchmark trước khi khóa implementation**; animation và trạng thái nhân vật chỉ phản ánh event thực tế. Q-14: agent idle **không** tự động đi Coffee/Relax mặc định; chỉ là setting tùy chọn (mặc định tắt). Q-15: asset mặc định **chỉ CC0**; font OFL được phép nếu lưu đầy đủ license/NOTICE; CC-BY chỉ dùng khi Owner phê duyệt **từng package**; asset MIT không rõ nguồn gốc (ví dụ furniture/tile Pixel Agents) **không bundle mặc định** (chặt hơn đề xuất cũ "MIT/OFL được phép"). Việc tái dùng **code** MIT/Apache-2.0 không bị ảnh hưởng (D-012). ~~**Kích thước tile 16 vs 32 KHÔNG được OWR-2 quyết định**: giữ cờ chờ Owner~~ **[OWR-3 A.7/X-6: ĐÃ GIẢI QUYẾT]** art **16 px**, hiển thị phóng **2 lần** (hiệu dụng 32 px); benchmark sau (xem §5, §21 mục 3). **[OWR-3 A.9 / Q-15 làm rõ]** asset MIT chỉ dùng nếu xác minh được nguồn gốc, license và ghi attribution vào NOTICE; nguồn gốc không rõ thì không dùng; CC0 vẫn là mặc định.

Nguồn: `DECISION_LOG.md` (D-011, D-012, D-014), `research/PIXEL_AGENTS_ANALYSIS.md` (MIT), `research/MY_VIRTUAL_OFFICE_ANALYSIS.md` (AGPL, chỉ ý tưởng), `research/LICENSE_AND_REUSE.md`, `design/SCREEN_SPECIFICATION.md` §3.3, `design/DESIGN_TOKENS.md`, `product/FUNCTIONAL_REQUIREMENTS.md` FR-151..FR-165, `DATABASE_DESIGN.md` §4.12, `PROVIDER_INTEGRATION.md` (event chuẩn hóa).

## 1. Mục đích

Cho Director "nhìn thấy công ty" bằng một cái liếc: ai đang làm gì, ai đang chờ, ai bị kẹt, rồi vào chi tiết trong một click (SCREEN §3.3). Office là **lớp nhận thức ngoại vi** của dữ liệu thật, không phải trò chơi. Nguyên tắc tối cao: **không có animation hay di chuyển nào mà không có event thật làm căn cứ** (FR-118, FR-160).

## 2. Phạm vi

- Trong: engine và ranh giới tích hợp React, tile/sprite, 14 phòng, máy trạng thái nhân vật (13 animation), desk, pathfinding, bubble, layout editor và lưu trữ, camera, ngân sách hiệu năng cho 18 nhân vật, asset và giấy phép, danh sách ý tưởng clean-room từ My Virtual Office, quy tắc "không animation nếu không có event", fallback danh sách (mobile/lỗi), kế hoạch test gồm phát lại event tất định.
- Ngoài: nhiều tầng (FR-153, P2; chỉ chừa móc mở rộng), chat trong office, pet/thời tiết/trang trí vô nghĩa (cố ý loại, §15), nội dung Agent Panel (đã có ở SCREEN §3.3), schema event gốc (`PROVIDER_INTEGRATION.md` §5, D-014).

## 3. Kiến trúc tổng thể

```
agent_events (Postgres, outbox D-014) ─► Redis Stream aiw:evt:* ─► WS gateway ─► useOfficeFeed (React)
                                                                                    │ chuẩn hóa → OfficeInput
                                         ┌──────────────────────────────────────────▼───────────────┐
   packages/office-core (TS thuần, không Phaser, không DOM): OfficeSimulation = state machine + desk + pathfinding + layout
                                         └───────────────┬──────────────────────────────────────────┘
                                                         │ snapshot/diff mỗi tick (fixed-step 60 Hz)
   packages/office-phaser: OfficeScene / EditorScene (view) ◄──┘     DOM overlay: bubble + label (React)
   apps/web/features/office: <OfficeHost/>, Agent Panel, HUD, Fallback list
```

Tách `office-core` (mô phỏng tất định) khỏi `office-phaser` (hiển thị) là điều kiện để test replay không cần GPU (§17) **[Đề xuất]**.

## 4. Engine: Phaser 3.90 và ranh giới tích hợp (D-011)

**Chọn Phaser 3.90** (MIT), cấu trúc để nâng lên Phaser 4.x khi cần filter/lighting; tilemap, camera, input, atlas có sẵn (LICENSE_AND_REUSE §5.1). Phương án loại: PixiJS 8 (thiếu tilemap/camera/editor sẵn), Canvas 2D thuần (Pixel Agents dùng, phải tự viết hết) **[Đã xác minh theo D-011]**. SCREEN §3.3 ghi "PixiJS/Canvas 2D [Giả định]" — **lỗi thời**, D-011 thắng (xem §21).

**Ranh giới (đường biên cứng)** **[Đề xuất]**:

| Bên | Được | Không được |
|---|---|---|
| React (`apps/web/features/office`) | tạo `Phaser.Game` **một lần** trong `useEffect` (cleanup `destroy`, chịu StrictMode mount đôi), đẩy lệnh qua store/bus, vẽ HUD/bubble/panel, gọi REST/WS | chạm đối tượng Phaser, chạy vòng lặp game |
| `office-phaser` | đọc snapshot từ `officeStore.subscribe`, phát input người dùng qua bus, vẽ | `fetch`, WebSocket, import React, đọc auth/secret, tự sinh event/animation |
| `office-core` | tính state, path, desk từ `OfficeInput` | `Date.now`, `Math.random`, DOM, Phaser |

```ts
// office-core/src/contract.ts
export type OfficeInput =
  | { type:'domain_event'; eventId:string; globalSeq:number; occurredAt:string; name: DomainEventName; agentId?:string; payload: unknown }
  | { type:'provider_event'; eventId:string; globalSeq:number; occurredAt:string; agentId:string; ev: ProviderEvent }
  | { type:'layout_loaded'; layout: OfficeLayoutV1 } | { type:'clock'; nowMs:number } /* đồng hồ do host cấp */;
export interface OfficeSnapshot { tick:number; agents: Record<string, CharacterView>; bubbles: BubbleView[]; layoutRevision:number }
export interface CharacterView { id:string; xTile:number; yTile:number; facing:'down'|'up'|'left'|'right'; anim: AnimationId;
  causedByEventId:string; roomId?:string; deskId?:string; stale:boolean }
// bus Phaser → React (typed): agent.clicked | agent.dblclicked | agent.context | room.clicked | camera.changed | editor.dirty | perf.sample
// bus React → Phaser: focusAgent(id) | setMode('live'|'edit'|'fullscreen') | setZoom(n) | setFollow(id|null) | setFilter(f)
```

Store: Zustand (D-001) với slice `layout`, `agents` (view-model), `selection`, `camera`, `mode`, `filters`. Scene đọc bằng `getState()/subscribe`, **không** giữ state riêng; cập nhật batch theo `requestAnimationFrame`, không re-render React mỗi tick **[Đã xác minh mẫu từ LICENSE_AND_REUSE §5.1 và Pixel Agents `OfficeState` imperative]**.

## 5. Đặc tả tile và sprite

**Quyết định kích thước [OWR-3 A.7 — Owner chấp thuận; benchmark sau]**: art gốc **16×16 px/tile**, hiển thị **phóng 2 lần (zoom nguyên, hiệu dụng 32 px màn hình)**, khớp "32px tiles" trong SCREEN §3.3. Lý do: toàn bộ nguồn tái dùng được (Pixel Agents, MetroCity) là 16 px. DB `office_layouts.tile_size` mặc định **16** kèm trường hiển thị `display_scale` mặc định **2** (**tên/kiểu cột là ĐỀ XUẤT, cờ phán đoán, chờ Architect/Owner**; DATABASE_DESIGN đã đồng bộ). Benchmark hiệu năng sau khi dựng PoC OfficeRenderer có thể điều chỉnh.

| Loại | Spec | Ghi chú |
|---|---|---|
| Tile sàn | 16×16, 8–12 biến thể xám + colorize theo room (`floor_tile`, `wall_color`) | colorize bằng HSB lúc build, cache texture |
| Tường | atlas 64×128 = 4×4 ô 16×32, autotile bitmask 4-bit N=1,E=2,S=4,W=8; cao hơn tile 16 px, sort cùng furniture | định dạng tương thích Pixel Agents (MIT) |
| Cửa | tile walkable trên đường tường; `door_x,door_y` mỗi room | điểm vào của pathfinding |
| Thảm | marching squares 16 case (NW=1,NE=2,SE=4,SW=8) 64×64 | tùy chọn |
| Furniture | PNG 16×16…48×32 (cao tới 16×64), `manifest.json` lồng rotation→state→animation (`2-way`,`3-way-mirror`,`4-way`; `on/off`; `frame`) | **giữ tương thích manifest Pixel Agents** để nhận pack cộng đồng |
| Nhân vật | frame **16×32**, sheet theo hàng hướng: `down, up, right` (trái = lật ngang), pivot đáy giữa | 3 hướng như Pixel Agents; thêm hàng cho animation mới |
| Bubble icon | 11×13 palette, vẽ bằng DOM/CSS sprite (xem §10) | |
| UI cursor/icon | Kenney (CC0) | |

Atlas: gộp theo nhóm (`chars`, `furn_*`, `tiles`) ≤ 2048×2048, `pixelArt:true`, `roundPixels:true`, `antialias:false`, nearest filter; không bao giờ zoom phân số (chỉ 1–6 nguyên).

**13 animation (FR-158) — spec frame [Đề xuất]** (fps là tốc độ chu kỳ khung của trạng thái đã có căn cứ, không phải hành vi mới):

| ID | Hướng | Frame | fps | Loop | Ghi chú |
|---|---|---|---|---|---|
| `idle` | 3 | 2 | 1.5 | ✔ | đứng, thở nhẹ |
| `walk` | 3 | 4 | 6.7 (0.15 s/frame) | ✔ | 3 tile/s |
| `run` | 3 | 4 | 10 | ✔ | 5 tile/s |
| `sit` | 3 | 1 | – | tĩnh | ngồi trên ghế, dịch xuống 6 px như Pixel Agents |
| `work` | 3 | 2 | 2 | ✔ | ngồi, màn hình sáng, không có công cụ cụ thể (terminal/test/build) |
| `typing` | 3 | 2 | 3.3 (0.3 s/frame) | ✔ | công cụ ghi (edit/write) |
| `reading` | 3 | 2 | 1.5 | ✔ | công cụ đọc/tìm (read/search/fetch) |
| `thinking` | 3 | 3 | 1.2 | ✔ | tay chống cằm/dấu "…" |
| `meeting` | 3 | 1 | – | tĩnh | ngồi quanh bàn họp |
| `talking` | 3 | 2 | 4 | ✔ | miệng/cử chỉ, kèm text bubble |
| `celebrating` | 1 (down) | 4 | 6 | **không** (chạy đúng 1 lần 3 s) | |
| `error` | 1 (down) | 2 | 2 | ✔ | đứng, ⛔/⚠ |
| `waiting` | 3 | 2 | 1 | ✔ | ngẩng đợi, bubble ?/✋ |

Tổng frame ≈ 3 hướng × (2+4+4+1+2+2+2+3+1+2) + 1×(4+2) + `waiting` 3×2 = ~114 frame/nhân vật, trong sheet ≤ 112×… (cỡ tương đương 1–2 sheet 16×32). Biến thể ngoại hình: skin/hair/outfit theo department (FR-156), palette gán **phía server** theo seeded hash `agentId` để mọi client giống nhau (học Pixel Agents `paletteAssigner`); hue shift 45–315° khi hết palette **[Đã xác minh tham chiếu]**.

## 6. 14 phòng

Danh sách theo FR-151: **Reception, Management, BA, Architecture, Design Studio, Frontend, Backend, Database, QA, Security, DevOps, Meeting, Relax, Coffee** (14, khớp yêu cầu; spec dùng tên viết tắt FE/BE/DB). Ánh xạ `office_rooms.kind` (enum DB) và department mặc định (FR-163 **[Giả định]**; phòng ban tùy chỉnh gán vào phòng có sẵn hoặc phòng mới ở P2).

| # | Room | `kind` | Department mặc định | Furniture chính | Desk (seed) |
|---|---|---|---|---|---|
| 1 | Reception | lobby | – (điểm xuất hiện agent mới, người dùng) | quầy lễ tân, sofa, bảng thông báo | 0 |
| 2 | Management | department | Management/PM | bàn lớn, whiteboard, review desk | 4 |
| 3 | BA | department | Business Analysis | bàn, whiteboard, tủ tài liệu | 4 |
| 4 | Architecture | department | Architecture | bàn, whiteboard sơ đồ | 4 |
| 5 | Design Studio | department | UX/UI | bàn, màn hình lớn, giá vẽ | 4 |
| 6 | Frontend | department | FE | bàn + PC | 4 |
| 7 | Backend | department | BE | bàn + PC | 4 |
| 8 | Database | department | DB | bàn + PC, server rack nhỏ | 4 |
| 9 | QA | department | QA | bàn + PC, bảng bug | 4 |
| 10 | Security | department | Security | bàn + PC | 4 |
| 11 | DevOps | department | DevOps | bàn + PC, server rack | 4 |
| 12 | Meeting | meeting | (dùng chung) | bàn họp 10–12 ghế, màn chiếu | 0 (ghế họp) |
| 13 | Relax | lounge | (dùng chung) | sofa, kệ sách, cây | 0 |
| 14 | Coffee | lounge | (dùng chung) | máy pha cà phê, bàn nhỏ | 0 |

Tổng 40 desk ≥ 18 agent mặc định (D-016, vai trò là dữ liệu seed, không hardcode). Bản đồ mặc định **64×40 tile (art 16 px)**, bố cục lưới (x,y,w,h theo tile) **[Đề xuất]**:

```
y 0-13  : Reception(0,0,10,14) Management(10,0,10,14) BA(20,0,10,14) Architecture(30,0,10,14) Design(40,0,12,14) Meeting(52,0,12,14)
y 14-17 : hành lang ngang 4 tile (đường đi chính, có cửa vào từng phòng)
y 18-29 : Frontend(0,18,10,12) Backend(10,18,10,12) Database(20,18,10,12) QA(30,18,10,12) Security(40,18,12,12) DevOps(52,18,12,12)
y 30-39 : Relax(0,30,24,10) Coffee(24,30,14,10) | (38,30,26,10) dự phòng mở rộng/tầng sau
```

Quy tắc phòng **[Đề xuất]**: mỗi room có ≥ 1 cửa nối hành lang; mỗi room `department` có ≥ 1 desk với đường đi tới cửa (validator §11); nhãn phòng + màu viền theo department (token màu từ `DESIGN_TOKENS.md`, không thêm màu mới); click phòng → danh sách agent/task trong phòng (FR-165). Meeting **luôn tồn tại** nhưng chỉ có người khi có event họp/handoff thật; SCREEN §3.3 viết "Meeting room (chỉ hiển thị khi có review)" — ta vẫn vẽ phòng (cố định theo FR-151) và **chỉ nhân vật** xuất hiện khi có event (xem §21).

## 7. Máy trạng thái nhân vật (13 animation ↔ event thật)

### 7.1 Cấu trúc
Phân cấp bốn trục độc lập **[Đề xuất]**: `Presence` (`absent|present`) × `Locomotion` (`stationary|walking|running`) × `Posture` (`standing|seated`) × `Activity` (`none|work|typing|reading|thinking|meeting|talking|waiting|error|celebrating`). Animation hiển thị = hàm thuần của bốn trục (`idle` = present+stationary+standing+none; `sit` = present+stationary+seated+none; `walk/run` theo locomotion). Presentation không phải animation: `paused` = `sit` + bubble ⏸; `offline/disabled/stopped` = ghế trống + nameplate xám (SCREEN §3.3), không sprite.

### 7.2 Bảng event → trạng thái (cấu hình được, FR-162)
Cột "event" là tên event chuẩn: domain (FR-117) hoặc `ProviderEvent.kind` (`PROVIDER_INTEGRATION.md` §5). **[OWR-3 A.16/X-4]** Canonical chỉ là dotted lowercase không `_` (ví dụ `AGENT_CREATED` = `agent.created`); các tên UPPER_SNAKE trong bảng này là tên tương thích hiển thị, ánh xạ cơ học theo `EVENT_SYSTEM.md` §0/§5.7. Chỉ 16 tên ở `EVENT_SYSTEM.md` §0 là event gốc của Owner; còn lại là đề xuất ngoài danh sách gốc, chờ Owner.

| Event (nguồn) | Điều kiện | Vị trí đích | Animation | Bubble |
|---|---|---|---|---|
| `AGENT_CREATED` | | xuất hiện ở Reception rồi đi tới desk | `walk` → `sit` | – |
| `TASK_ASSIGNED` | priority thường | desk | `walk` → `sit` | tên task (text, 6 s) |
| `TASK_ASSIGNED` | priority `critical` | desk | `run` → `sit` | tên task |
| `TASK_STARTED` / `session.started` | | desk | `sit` → `work` | – |
| `thinking.started`…`thinking.completed` | | tại desk | `thinking` | "…" |
| `tool.started` kind `edit|write|delete|move` | | tại desk | `typing` | tên file (path rút gọn) |
| `tool.started` kind `read|search|fetch` | | tại desk | `reading` | mục tiêu đọc |
| `tool.started` kind `execute|mcp|other` | | tại desk | `work` | lệnh rút gọn (đã redact) |
| `tool.finished` | | giữ | quay về `work` | – |
| `message.delta` (đang stream) | | giữ | `talking` (kết thúc 2 s sau delta cuối, §7.4) | trích câu cuối |
| `MESSAGE_SENT` giữa 2 agent (task handoff/hỏi đáp) | cả hai đang rảnh | người gửi đi tới desk người nhận | `walk`→`talking` | tóm tắt tin |
| `MEETING_STARTED` / handoff review (`TASK_HANDOVER`, reviewer thật) | | Meeting room, ghế trống gần nhất | `walk` → `meeting` | chủ đề |
| `MEETING_ENDED` | | về desk | `walk` → `sit` | – |
| `approval.requested` / `APPROVAL_REQUESTED` | | giữ desk (hoặc Management "review desk" nếu Owner bật `goToReviewDesk`) | `waiting` | ✋ amber, sticky |
| `approval.resolved` / `HUMAN_APPROVED` | | giữ | về `work` | ✓ 2 s |
| `session.state.changed → waiting_input` | | giữ | `waiting` | ? amber, sticky |
| `TASK_COMPLETED` | | giữ | `celebrating` (3 s, một lần) → `sit` | ✓ xanh |
| `TASK_FAILED`, `turn.failed`, `error` không retry được | | giữ | `error` | ⛔ đỏ, sticky tới khi giải quyết |
| `TASK_BLOCKED` | | giữ | `error` biến thể `blocked` (bubble màu orange `octagon-alert`) | ⛔ orange |
| `AGENT_STATE_CHANGED → paused` | | giữ | `sit` | ⏸ |
| `rate.limit.updated`/`RATE_LIMITED` (pool cooldown, đã có `resetsAt`) | policy `coffeeOnCooldown` bật | Coffee | `walk` → `sit`/`idle` | "chờ quota hh:mm" |
| `AGENT_STATE_CHANGED → idle/available` kéo dài ≥ `relaxAfterMin` (tính từ `occurredAt` của event) | policy `idleToRelax` bật | Relax | `walk` → `idle` | – |
| `AGENT_STATE_CHANGED → idle/available` (mặc định) | | desk | `sit`/`idle` | – |
| `EMERGENCY_STOP` | | giữ | `error` toàn bộ + banner | ⛔ |
| `subagent.started/finished` | | ephemeral helper cạnh parent | `walk` → `work` / biến mất | tên tool |
| `AGENT_STATE_CHANGED → disabled/offline`, `session.ended` | | đi tới Reception rồi biến mất, hoặc rời tại chỗ (Owner chọn) | `walk` rồi despawn | – |

Ưu tiên khi nhiều event đồng thời trên một nhân vật (cao → thấp): `error/blocked` > `waiting` > `meeting` > `typing|reading|work` > `thinking` > `talking` > `idle`. Event ưu tiên thấp không ghi đè trạng thái ưu tiên cao cho tới khi trạng thái đó được giải bằng event tương ứng. Mỗi mục "sticky" kết thúc **chỉ** bằng event giải quyết, không bằng timer.

### 7.3 Chuyển động
Mọi `walk/run` là hệ quả của event **thay đổi đích** (desk/phòng/ghế họp/Coffee/Relax); đích đổi giữa đường → repath từ vị trí hiện tại. Đến đích → posture `seated` nếu đích là ghế. Không có "đi dạo ngẫu nhiên": **loại bỏ** `wanderTimer`/`wanderLimit` của Pixel Agents `characters.ts` (§12, §15).

### 7.4 Timer hậu quả của event (decay) — được phép, có điều kiện
Một timer chỉ được tạo như hệ quả tất định của một event, ghi `causedByEventId` của event cha và chạy theo **đồng hồ mô phỏng** (không `Date.now`): `celebrating` kết thúc sau 3 s; ✓ bubble 2 s; text bubble 6 s; `talking` kết thúc 2 s sau `message.delta`/`message.completed` cuối; hạ `stale=true` khi không có `heartbeat` hoặc event nào của session quá `silence_timeout` (20 phút, RESOURCE §7) — hiển thị nhãn "không có tín hiệu từ hh:mm", **không đổi sang animation khác**. Độ trễ 300 ms giữa `tool.started`→`tool.finished` để tránh chớp (kỹ thuật Pixel Agents) áp dụng ở view, không tạo animation mới.

## 8. Phân bổ desk

Dữ liệu: bảng `desks(room_id,x,y,facing,seat_x,seat_y,assigned_agent_id)` + unique `assigned_agent_id` **[Đã xác minh DB]**. Thuật toán `assignDesk(agent)` **[Đề xuất]**:
1. Nếu đã có `desks.assigned_agent_id` → dùng (ổn định qua phiên).
2. Room = room có `department_id` = `agent.departmentId`; nếu nhiều room, chọn room ít desk bị chiếm nhất.
3. Trong room, chọn desk trống gần cửa nhất (khoảng cách đường đi, tie-break theo `id`).
4. Hết desk: desk trống của room cùng `kind` gần nhất → nếu không còn, desk "hot" ở Management; cuối cùng đứng chờ ở Reception với bubble "chưa có chỗ ngồi" (event-backed: sinh từ `agent.seat.unavailable` do server phát).
5. Gán thủ công: kéo agent → desk trong Edit mode, hoặc Owner đổi trong Agent Detail; ghi DB với kiểm tra unique, phát `agent.seat.assigned`.

Server là nguồn quyết định (một nơi gán), client chỉ hiển thị, để mọi người dùng thấy giống nhau. Sub-agent (`subagent.started`) dùng ghế/điểm đứng ephemeral cạnh parent, tối đa 4 helper/parent, không ghi DB. Facing ngồi: theo `desks.facing`; nếu bàn ở phía trước → `down`/`up`.

## 9. Pathfinding

- Lưới 4 hướng từ `collision_layer` + footprint furniture không walkable + tường; cửa là tile walkable. Ghế của chính nhân vật được "mở khóa" khi tính path tới chỗ ngồi (kỹ thuật `withOwnSeatUnblocked`, Pixel Agents).
- Thuật toán: **A\* qua `easystarjs` (MIT)** hoặc BFS của Pixel Agents (MIT, `layout/tileMap.ts`); đề xuất A\* vì có cost cho hành lang/cửa; map 64×40 = 2560 tile, tính path ≤ 5 ms/lần **[Đề xuất; Pixel Agents dùng BFS trên tối đa 64×64 là đủ — Đã xác minh]**.
- Cache: key `(roomDoorA, roomDoorB)` cho đoạn hành lang; invalidate khi `layoutRevision` đổi.
- Va chạm nhân vật–nhân vật: **không** mô phỏng (cho phép chồng, lệch tâm xác định `hash(agentId)%3` px để khỏi trùng hoàn toàn); hàng chờ máy cà phê chỉ khi event Coffee và có ≥ 2 agent (xếp theo `occurredAt`).
- Đích không tới được: không teleport, giữ vị trí, phát `office.path.failed` (metric + dấu ⚠ trong editor/list) và Agent Panel ghi "không có đường tới desk". Validator layout (§11) ngăn tình huống này từ gốc.
- Tốc độ: walk 3 tile/s, run 5 tile/s (nguồn: 48 px/s ở tile 16 px = 3 tile/s **[Đã xác minh]**). Dt cap 0.1 s để tab ẩn không nhảy.

## 10. Speech bubble

Hai loại **[Đề xuất]** (học cách Pixel Agents vẽ bubble bằng DOM overlay chồng canvas, `projection.ts`):
1. **Status bubble** (icon): `?` waiting_input, `✋` waiting_approval, `⛔` error/failed (đỏ) hoặc blocked (orange), `⏸` paused, `✓` done. Icon lucide + token màu (`--status-waiting` amber, `--status-blocked` orange, `--status-failed` red, `--status-completed` green); dùng **hình + màu** (không chỉ màu, a11y).
2. **Text bubble**: ≤ 80 ký tự, từ **tóm tắt đã redact** của event (tên task, đường dẫn file rút gọn, câu cuối của message), không bao giờ nội dung thô của tool input; SECURITY §4.4 áp dụng trước khi lên màn hình. Render bằng `textContent` (không HTML), tôn trọng i18n.

Quy tắc: tối đa 1 text bubble + 1 status bubble/nhân vật; text TTL 6 s, status sticky tới event giải quyết; hàng đợi FIFO khi nhiều text; bubble xuất hiện **≤ 1 s** sau event (SCREEN AC); agent mờ 40% ngoài project đang lọc; label tên/role luôn hiện khi hover/chọn. Vị trí tính bằng `camera.worldView` → toạ độ màn hình, cập nhật khi pan/zoom (≤ 18 phần tử DOM, `transform` composited). Click bubble = mở Agent Panel đúng task/approval.

## 11. Layout editor và lưu trữ

### 11.1 Chế độ và công cụ (SCREEN §3.3, phím `E`)
Toolbar: Select, Floor, Wall, Door, Furniture, Desk, Room/Area, Label, Eraser; thuộc tính: Room (tên, màu, department, kind, số desk), Furniture (xoay R, walkable); undo/redo 50 bước theo stroke; Save / Save as / Reset to default (confirm); Import/Export JSON. Trong Edit nhân vật **đứng yên và mờ** (không tạo event mới, không đổi state mô phỏng); quay lại Live áp snapshot hiện tại. Viewer không có Edit. Lược đồ phím: R xoay, Esc đa cấp, Ctrl+Z/Y (tương tự Pixel Agents, **[Đã xác minh]** về hành vi tham chiếu).

### 11.2 Bảng DB (có sẵn — `DATABASE_DESIGN.md` §4.12)
| Bảng | Vai trò | Ghi chú |
|---|---|---|
| `office_layouts` | một bản bố cục: kích thước, `tile_size`, `tileset`, `ground_layer`, `collision_layer`, `is_active` (unique theo org) | JSONB mảng tile id (Tiled-compatible) |
| `office_rooms` | vùng chữ nhật: `kind`, `department_id`, x/y/w/h, `floor_tile`, `wall_color` | |
| `office_furniture` | vật dụng gắn vào room: `kind`, `sprite`, x/y, `rotation`, `is_walkable`, `props` | |
| `desks` | ghế/bàn gán agent (unique `assigned_agent_id`) | bảng bổ sung, đã có |
| `character_positions` | vị trí/animation hiện tại, 1 hàng/agent, write-behind Redis 5 s | **không** lưu lịch sử (lịch sử đi qua event) |

**Delta schema đề xuất cho chủ DB** (tôi không sửa file không do mình tạo) **[Đề xuất, Cần người dùng phê duyệt]**:
- `office_layouts`: thêm `revision int not null default 1` (optimistic concurrency), `schema_version smallint`, `floor_index smallint default 0` (móc cho FR-153); đổi mặc định `tile_size` = 16 (art) và thêm `display_scale` = 2 (**đề xuất**, OWR-3 A.7; hiệu dụng 32 px) và `width/height` mặc định 64×40.
- `office_rooms`: thêm `door_x,door_y smallint` (điểm vào cho pathfinding), `kind` mở rộng giá trị `'department'|'meeting'|'lounge'|'lobby'|'server'` (đủ cho 14 phòng, không cần đổi).
- `character_positions.animation`: enum hiện **7 giá trị** (`idle, walk, typing, thinking, talking, sleeping, error`) nhưng FR-158 yêu cầu **13** và không có `sleeping`; chuyển sang 13 giá trị §5 và thêm `caused_by_event_id uuid`, `caused_by_global_seq bigint` để kiểm toán "animation phải có event" (§12).

### 11.3 Lưu và đồng bộ
- Autosave debounce 500 ms → `PUT /api/office/layouts/:id` kèm `revision`; xung đột (409) → hiện diff, người dùng chọn giữ/ghi đè. Một layout `is_active` mỗi org (index unique có sẵn).
- Validator phía server **và** client trước khi lưu: (1) mọi room `department` có ≥ 1 cửa và ≥ 1 desk; (2) mọi desk có đường tới cửa → hành lang → Reception; (3) không furniture chồng lên tường/ghế; (4) không xóa desk đang gán agent nếu chưa chuyển agent (hoặc cảnh báo + gán lại); (5) kích thước ≤ 64×64; (6) mọi `sprite` thuộc asset được phép (CREDITS allowlist).
- Định dạng import/export: JSON **riêng** `aiw-office-layout` v1 (chứa `schema_version`, tham chiếu asset bằng `asset_id`, không nhúng pixel); **từ chối** định dạng `.mvo-layout.json` (AGPL) và không tự nhập layout của Pixel Agents (chỉ tham khảo cấu trúc). Layout chia sẻ không nhúng pixel của pack trả phí; thiếu asset → placeholder (LICENSE_AND_REUSE §6.2).
- Khởi động lại: `character_positions` dùng làm **vị trí đặt ban đầu** (không phục hồi animation); animation suy lại từ trạng thái agent hiện tại và event gần nhất, nếu agent không có event nào kể từ lúc đó thì `idle/sit` kèm `stale` — không dựng lại "đang làm" từ dữ liệu cũ.

## 12. Quy tắc cứng: không animation nếu không có event thật (FR-118, FR-160)

1. **Cổng duy nhất**: `office-core` chỉ nhận `OfficeInput` sinh từ `agent_events`/stream do backend phát; frontend **không** tự sinh domain/provider event. Mọi lệnh animation/di chuyển nội bộ mang `causedByEventId` (bắt buộc ở kiểu TypeScript); thiếu → ném lỗi ở dev/CI, ở production bị loại bỏ và tăng `office_unbacked_transition_total` (alert > 0).
2. **Cấm ngẫu nhiên hành vi**: không `Math.random`, không `setInterval/setTimeout` trực tiếp trong `office-core`/`office-phaser` cho hành vi (ESLint rule tùy biến + test grep). PRNG có seed chỉ dùng cho palette ngoại hình (seed = hash `agentId`).
3. **Ngoại lệ có phạm vi hẹp**: chu kỳ khung hình của animation hiện tại (vd `idle` 2 frame) là cách vẽ trạng thái đã có căn cứ, không phải chuyển trạng thái; timer decay §7.4 chỉ là hệ quả tất định của event cha.
4. **Không "hoạt động giả"**: loại pet, thời tiết, ngày/đêm, TV, ném máy bay giấy, oẳn tù tì, phi tiêu, đi uống cà phê ngẫu nhiên, họp giả lập (§15). Relax/Coffee chỉ dùng theo các event ở §7.2 và **mặc định tắt** (`idleToRelax=false`, `coffeeOnCooldown=false`) cho tới khi Owner bật. **[OWR-2 Q-14 ĐÃ DUYỆT: idle không tự động chuyển Coffee/Relax mặc định; có thể có setting tùy chọn. FR-164 phải đồng bộ (agent khác sở hữu).]**
5. **Mất kết nối**: đóng băng toàn bộ + banner "Mất kết nối — trạng thái tại hh:mm"; khi nối lại replay từ `lastEventId`, nhân vật chuyển thẳng tới trạng thái hiện tại (không phát lại chuỗi di chuyển cũ nếu cách > 60 s — snap có đánh dấu, ghi `reason:'resync'`).
6. **Kiểm toán**: ring buffer `office_transition_log` (500 mục: `tick, agentId, from→to, causedByEventId`) xem được ở chế độ Debug và xuất được; AC của SCREEN §3.3 "mọi animation transition có event id" được kiểm bằng log này.
7. **Harness demo/test không phải sản phẩm**: kiểm 13 animation (AC-096) chạy ở route dev `/__office-harness` và Storybook, bị loại khỏi build production (CI scan chuỗi), nhãn "HARNESS" cố định trên màn hình; không dùng cho trạng thái rỗng (empty = sàn trống + ghế "Create your first agent").

## 13. Camera, zoom, pan

- Phaser `Cameras.Scene2D`: zoom **nguyên 1–6** (mặc định `round(2*devicePixelRatio)` giới hạn 1–6), bước bằng wheel/pinch/`+`/`-`/nút HUD; pan bằng kéo chuột giữa/`Space`+kéo/phím mũi tên/WASD; kéo hai ngón (touch) pan + pinch; **bounds** clamp theo kích thước layout, căn giữa khi map nhỏ hơn viewport.
- Follow (HUD): camera bám agent với lerp 0.1 (tắt khi `prefers-reduced-motion`: nhảy tức thời); click phòng → fit-to-room; double-click nhân vật → `/agents/:id`.
- Minimap (HUD dưới): toạ độ phòng + chấm trạng thái; click để nhảy.
- `ResizeObserver` điều chỉnh canvas theo DPR (không `ctx.scale`, tính backing store = `rect*DPR` như Pixel Agents). Lưu camera (zoom, tâm) theo người dùng trong `localStorage` (try/catch) để mở lại đúng chỗ.
- Fullscreen: ẩn sidebar+topbar, HUD overlay, thoát bằng `Esc`. Bàn phím: mọi thao tác đạt được không cần chuột (WCAG), `Tab` duyệt qua nhân vật theo thứ tự ưu tiên trạng thái (blocked trước).
- Nhiều tầng (FR-153) để P2: camera/scene có tham số `floorIndex` nhưng MVP cố định 0.

## 14. Ngân sách hiệu năng (18 nhân vật)

Máy tham chiếu: laptop i5-11400H, đồ họa tích hợp, 16 GB (RESOURCE §3) **[Đã xác minh cấu hình máy; số đo office chưa có — Giả định]**. Đối tượng: 18 nhân vật chính + ≤ 8 helper sub-agent (tối đa 26 sprite động), map 64×40.

| Hạng mục | Ngân sách | Cách đạt |
|---|---|---|
| FPS mục tiêu | 60 (live); 30 khi tab ẩn (hoặc tạm dừng render, vẫn nhận event) | `visibilitychange`, throttle `game.loop` |
| Thời gian khung | ≤ 16.6 ms; update ≤ 3 ms, render ≤ 6 ms | |
| Draw call | ≤ 120/khung | nền (sàn+tường+thảm) bake vào `RenderTexture` hoặc tilemap layer **tĩnh**, chỉ vẽ lại khi layout đổi (Pixel Agents *không* cache layer tĩnh — điểm cải tiến) |
| Game object | ≤ 450 (furniture ≤ 300, nhân vật+helper ≤ 26, còn lại UI) | gộp furniture tĩnh cùng depth |
| Texture | ≤ 4 atlas 2048², GPU ≤ 64 MB | atlas theo nhóm, không texture rời |
| Heap JS | ≤ 150 MB phần office | không tạo object mỗi khung (object pool cho path/bubble) |
| Pathfinding | ≤ 5 ms/yêu cầu, ≤ 3 yêu cầu/khung, cache hành lang | hàng đợi, không tính trong khung render |
| Xử lý event | ≤ 1 ms/event, đỉnh ≤ 50 event/s | `message.delta` **gộp**: chỉ chuyển "đang stream" + câu cuối mỗi 500 ms cho office |
| DOM overlay | ≤ 18 bubble + 18 label, chỉ `transform/opacity` | |
| Cập nhật store | batch 1 lần/rAF; React không render lại theo tick | scene `subscribe` trực tiếp |
| Khởi động | ≤ 1.5 s tới khi tương tác (asset ≤ 2 MB nén) | preload atlas nhỏ |

Thang suy giảm (tự động khi `perf.sample` dưới ngưỡng liên tục 5 s): (1) hạ 30 FPS → (2) tắt hiệu ứng đệm/lerp camera → (3) tắt label động → (4) chuyển **Fallback list** (§16) và hiển thị lý do. `prefers-reduced-motion`: tắt đi lại (nhân vật dịch chuyển tức thời tới đích có đánh dấu), giữ bubble. Đo bằng `game.loop.actualFps`, `performance.mark`, và bài test replay §17 (CI fail nếu p95 khung > 20 ms trên runner tham chiếu) **[Đề xuất]**. Có sự khác biệt về ngữ cảnh đo: SCREEN §3.3 ghi "chốt với Agent Performance" — số trên chờ xác nhận.

## 15. Asset và giấy phép

### 15.1 Chính sách gói mặc định
Theo yêu cầu của chủ sở hữu: **chỉ asset CC0 được đóng gói trong bản cài/ repo/ docker image/ CDN**. Mỗi thư mục asset có `CREDITS.json` `{author, source, spdx, modified}` + `LICENSE.txt`; CI fail nếu thiếu hoặc SPDX ∉ allowlist (`CC0-1.0`); mục này **chặt hơn** `LICENSE_AND_REUSE.md` §3.2 và D-012 đối với **asset** (cho phép cả MIT/OFL/CC-BY có file license). **[OWR-2 Q-15 ĐÃ DUYỆT]**: mặc định CC0; OFL font cho phép khi lưu license/NOTICE; CC-BY chỉ khi Owner phê duyệt từng package; MIT-asset nguồn gốc không rõ không bundle mặc định. **[OWR-3 A.9 làm rõ Q-15]** asset MIT chỉ được dùng nếu xác minh nguồn gốc + license và ghi attribution vào `NOTICE`; không rõ nguồn gốc thì KHÔNG dùng; CC0 là mặc định; font OFL cho phép kèm license/NOTICE; CC-BY theo từng package khi Owner phê duyệt; furniture/tile Pixel Agents (nguồn gốc không rõ) vẫn bị loại. Allowlist SPDX của CI cần mở rộng có kiểm soát cho `OFL-1.1` (chỉ font, kèm NOTICE) và `CC-BY-4.0` (chỉ package đã được Owner phê duyệt). Font UI Inter/JetBrains Mono (D-015, OFL) là UI ngoài canvas, không thuộc "asset office"; font trong canvas dùng Kenney (CC0) hoặc không dùng font bitmap.

| Nhóm | Nguồn | Giấy phép | Gói mặc định? |
|---|---|---|---|
| Nhân vật chibi | MetroCity Free Top Down Character Pack — JIK-A-4 | **CC0 1.0** (trang itch.io hiển thị, tác giả xác nhận dùng thương mại — LICENSE_AND_REUSE §6.1) | **Có**, nhưng phải chụp lại trang license vào PR (PIXEL_AGENTS_ANALYSIS nêu "chưa xác minh", mâu thuẫn §21); tự vẽ thêm biến thể màu/outfit theo department (derivative CC0, sản phẩm của ta) |
| UI/icon/cursor | Kenney | CC0 | Có |
| Nội thất/sàn/tường cho 14 phòng | **tự vẽ nội bộ (designer)**, phong cách 16×16, palette thống nhất | tài sản công ty (xuất bản CC0 hoặc giữ riêng) | Có — **đây là việc lớn** vì không có pack CC0 office chibi đủ bộ (§22) |
| Pixel Agents furniture/floor/wall/carpet/pet | repo MIT, nguồn gốc "original" chưa xác nhận bằng văn bản | MIT (theo repo) | **Không bundle** dưới chính sách CC0-only cho tới khi (a) tác giả xác nhận, (b) Owner nới lỏng chính sách |
| LimeZu, Donarg | itch.io, EULA riêng, cấm phân phối lại | proprietary | **Không bao giờ bundle**; chỉ BYO |
| BYO pack của người dùng | `~/.ai-workspace/assets/<pack>/` + `manifest.json` | do người dùng sở hữu | Cơ chế hỗ trợ; preset **chỉ toạ độ cắt**, không chứa pixel; nhắc credit theo EULA |

### 15.2 Có thể tái dùng từ Pixel Agents (MIT, kèm attribution) **[Đã xác minh license MIT, Copyright (c) 2026 Pablo De Lucca]**
Header bắt buộc mỗi file: `// Portions derived from pixel-agents (https://github.com/pixel-agents-hq/pixel-agents), Copyright (c) 2026 Pablo De Lucca, MIT. Modified by AI WORKSPACE team: <tóm tắt>.` và mục trong `THIRD_PARTY_NOTICES.md` (+ màn About → Open-source licenses).
- **Code logic (được phép, đã đề xuất trong PIXEL_AGENTS_ANALYSIS §8.1)**: BFS `tileMap.ts`, tính ghế `layoutSerializer.ts`/`seatPlacement.ts`, thứ tự z-sort (`zY`) làm `setDepth`, `colorize.ts`, autotile `wallTiles.ts`/`carpetTiles.ts`, `projection.ts`, logic thuần của editor (`editorActions.ts`, `editorState.ts` — undo theo stroke), `MessageTransport`/`webSocketTransport.ts`, `pngDecoder.ts`/`buildFurnitureCatalog` và **format manifest furniture**. UI React và renderer Canvas không dùng (ta dùng Phaser, design system riêng).
- **Cần sửa khi dùng**: bỏ wander ngẫu nhiên khỏi FSM (§7.3); thêm trạng thái ngoài `idle/walk/type`; tách khỏi `OfficeState` toàn cục.
- **Không được dùng / không bundle**: (1) bất kỳ PNG ở commit **trước `add749a`** (tileset proprietary cũ) — không lấy từ git history; (2) font `FSPixelSansUnicode-Regular.ttf` (giấy phép mâu thuẫn); (3) sprite nhân vật `char_0..5.png` lấy qua repo — phải lấy **từ nguồn CC0 gốc của JIK-A-4**, không qua bản của Pixel Agents; (4) furniture/floor/wall/carpet/pet trong repo chưa được tác giả xác nhận (xem bảng trên); (5) tên, logo "Pixel Agents" (không có trademark grant); (6) `default-layout-1.json`: về mặt MIT được dùng nhưng ta tự thiết kế 14 phòng nên không dùng; (7) hook installer ghi `~/.claude/settings.json` (ta dùng hook in-process, `PROVIDER_INTEGRATION.md` §18).

### 15.3 Clean-room — ý tưởng lấy từ My Virtual Office (AGPL-3.0), **không** lấy mã/asset/prompt/layout
Quy trình: người viết spec đọc MVO ≠ người viết code; người code chỉ đọc tài liệu này; **cấm** mở/clone/symlink repo MVO trong monorepo; PR template có mục "Clean-room attestation"; CI cấm dependency AGPL/GPL.
Ý tưởng (khái niệm, không biểu đạt) được phép:
1. Cờ năng lực (capability flags) quyết định UI hiện/ẩn — đã chuyển thành `ProviderCapabilities` (`PROVIDER_INTEGRATION.md` §6).
2. Chuẩn hóa nhiều tín hiệu nguồn về một bộ trạng thái hiển thị nhỏ có TTL/grace (ở đây là 13 animation + presentation state).
3. Ánh xạ "trạng thái → địa điểm" (làm việc → bàn, họp → bàn họp, chờ → khu vực riêng) thay vì tọa độ tay.
4. Gán desk tự động theo phòng ban; boss/review desk là chỗ ngồi đặc biệt.
5. Pathfinding lưới 4 hướng có tường ở cạnh tile (ý tưởng thuật toán công cộng; tự viết hoặc dùng `easystarjs`).
6. Hàng chờ dùng chung cho vật thể dịch vụ (máy cà phê) — chỉ khi có event thật.
7. Họp phải kết thúc bằng bản tóm tắt (ở ta: `MEETING_ENDED` kèm summary).
8. Giao tiếp agent–agent hiển thị bằng speech bubble; tool call có đường dẫn/URL → preview gắn vào agent.
9. Màu khu vực theo department; panel danh bạ agent + activity log bên phải.
10. Vòng đời "presence" (bắt đầu run, tool, chờ approval, kết thúc) làm từ vựng sự kiện.
Loại bỏ có chủ đích (vi phạm "không hoạt động giả" hoặc rủi ro bản quyền): pet, thời tiết thật, ngày/đêm, TV, trò chơi xã hội (máy bay giấy, oẳn tù tì, phi tiêu, bóng bàn), nhân vật vẽ thủ tục 17k dòng, polling 5 s, `.mvo-layout.json`, `default-office-config.json`, prompt/skill text, CSS/theme, ảnh chụp, favicon, Kasm browser script (`--no-sandbox` + stealth flags).

## 16. Fallback danh sách (mobile và lỗi)

Kích hoạt khi **bất kỳ** điều kiện: viewport < `md` 768 px (DESIGN_TOKENS breakpoint) trừ khi người dùng chọn "Xem bản đồ"; WebGL/Canvas không khởi tạo được; lỗi render (ErrorBoundary); thang suy giảm hạ tới mức 4 (§14); người dùng chọn chế độ Danh sách. Office **không được là điểm lỗi đơn** (SCREEN §3.3).
- Dùng cùng `officeStore` và cùng event, **không** hai nguồn dữ liệu.
- Nhóm theo room/department; mỗi hàng: ô avatar tĩnh (khung `idle` từ atlas qua CSS sprite), tên, role, chip trạng thái (icon + màu token status), task hiện tại, "cập nhật hh:mm", text bubble gần nhất; sắp xếp mặc định: blocked/error → waiting → working → idle → offline.
- Bấm hàng → bottom sheet (mobile) hoặc panel phải = cùng Agent Panel (8 trường, 10 action) — Resolve/Approve/Pause/Stop/Send instruction.
- Live region `aria-live="polite"` thông báo `waiting/blocked` mới; hỗ trợ lọc project/department/state như HUD.
- Mobile không có Edit layout (chỉ xem); bản đồ nếu mở dùng pan/pinch (đã có) nhưng không đảm bảo 60 FPS.

## 17. Test plan

1. **Unit `office-core`** (Vitest, không GPU): từng hàng bảng §7.2 → trạng thái/đích đúng; ưu tiên khi chồng event; sticky chỉ kết thúc bằng event giải quyết; desk assignment (đủ chỗ/hết chỗ/gán tay); pathfinding (đường tới cửa, cửa bị chặn → `path_failed`, repath khi đổi đích); validator layout (6 luật §11.3).
2. **Tính tất định**: `OfficeSimulation` chạy fixed-step `dt=1/60` với đồng hồ ảo; cấm `Date.now/Math.random` (lint + test chạy hai lần so hash). 
3. **Replay event tất định (golden)** — **cốt lõi**:
   - Định dạng fixture: NDJSON các hàng `agent_events` (`globalSeq`, `occurredAt`, `type`, `agentId`, `payload` đã redact) — ghi từ phiên thật hoặc từ `MockProviderAdapter` **chỉ trong CI** (`PROVIDER_INTEGRATION.md` §20.5); đóng băng trong `tests/fixtures/office/*.ndjson`.
   - Bộ chạy `ReplayRunner(fixture, speed)`: đưa event theo `globalSeq` và `occurredAt` (đồng hồ ảo, tốc độ ×1…×60), snapshot mỗi 1 s mô phỏng: `{tick, agents:[{id,xTile,yTile,facing,anim,causedByEventId,roomId,bubble}]}` → hash SHA-256 chuẩn hóa JSON; so với `*.golden.json`. Hai lần chạy → hash giống hệt; đổi thứ tự delivery giữa Redis stream/WS không đổi kết quả (đảo trong cùng `globalSeq` bị loại).
   - Kịch bản bắt buộc: (a) vòng đời một task đầy đủ (assign → start → thinking → read → edit → approval → resolve → complete); (b) 18 agent song song 10 phút; (c) rate-limit → pool cooldown → Coffee (policy bật) / không Coffee (policy tắt); (d) crash giữa tool (`tool.started` không có `finished`) → `error`/`stale`, không kẹt animation; (e) mất kết nối + resync `lastEventId` (snap có `reason:'resync'`); (f) sub-agent spawn/despawn; (g) handoff review → Meeting → `MEETING_ENDED`; (h) agent bị disable giữa chừng.
4. **Chống hoạt động giả (property test)**: mô phỏng 10 phút với luồng event **rỗng** → 0 chuyển trạng thái, 0 thay đổi vị trí; với luồng chỉ `heartbeat` → chỉ `stale` bật sau 20 phút ảo; quét mã: không `Math.random`, `setInterval` trong `office-core/office-phaser`; log kiểm toán không có chuyển trạng thái thiếu `causedByEventId`.
5. **Phủ 13 animation (AC-096)**: route harness dev phát event mô phỏng cho từng animation, kiểm render và `anim` mong đợi; CI xác nhận harness không có trong bundle production (grep `__office-harness`, `HARNESS`).
6. **Hiển thị (Playwright)**: chụp ảnh tại các checkpoint replay với seed cố định, so bằng `pixelmatch` (ngưỡng nhỏ) ở zoom 2×; kiểm bubble ≤ 1 s sau event, legend, mờ 40% ngoài project, dark/light theme, reduced-motion (không đi lại).
7. **Hiệu năng**: replay 18 agent ×10 phút, đo `actualFps`, p95 thời gian khung, heap, số draw call, số object; CI fail khi vượt §14 (runner tham chiếu cố định).
8. **Editor**: round-trip lưu/tải layout, undo/redo 50, conflict 409 theo `revision`, validator, import `aiw-office-layout` v1 và từ chối định dạng lạ/`.mvo-layout.json`, tải layout thiếu asset → placeholder.
9. **Fallback**: ép viewport < 768, ép lỗi WebGL, ép lỗi render → danh sách xuất hiện, dùng được toàn bộ action; dữ liệu bằng bản đồ.
10. **Giấy phép/asset**: CI quét `CREDITS.json` (SPDX allowlist `CC0-1.0`), thiếu file → fail; quét tên "Pixel Agents"/"OpenHands"/"my-virtual-office" trong asset; đối chiếu hash với danh sách cấm (pre-`add749a`).
11. **A11y**: điều hướng bàn phím, tương phản bubble, không truyền đạt trạng thái chỉ bằng màu, live region.

## 18. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| V1 | Phaser 3.90 sau abstraction `OfficeRenderer`, `office-core` tách khỏi view; benchmark trước khi khóa | D-011 (OWR-2); test tất định không cần GPU | Đã chọn (OWR-2 D-011), chưa khóa implementation cho tới khi benchmark |
| V2 | Art 16 px, zoom nguyên (mặc định 2×) | khớp mọi nguồn tái dùng; pixel-perfect | **[OWR-3 A.7: Owner chấp thuận art 16 px, phóng 2x; benchmark sau]** (trước đây: OWR-2 không quyết định) |
| V3 | 14 phòng cố định, map 64×40, 4 desk/phòng ban | FR-151; 40 desk ≥ 18 agent | Đề xuất |
| V4 | Mô phỏng 4 trục → 13 animation | tránh bùng nổ trạng thái, rõ căn cứ | Đề xuất |
| V5 | Bỏ wander ngẫu nhiên; Relax/Coffee chỉ theo event và mặc định tắt | FR-160/164, SCREEN §3.3 | Cần phê duyệt |
| V6 | Bubble bằng DOM overlay, text đã redact | i18n/a11y, an toàn dữ liệu | Đề xuất |
| V7 | Gói mặc định CC0-only; font OFL kèm NOTICE; CC-BY từng package cần Owner duyệt; BYO cho pack trả phí | OWR-2 Q-15; LICENSE_AND_REUSE | Đã duyệt (OWR-2 Q-15; làm rõ OWR-3 A.9: MIT-asset chỉ khi xác minh nguồn/license/attribution trong NOTICE) |
| V8 | Tái dùng logic MIT của Pixel Agents có attribution; MVO clean-room | D-012 | Đề xuất (theo D-012) |
| V9 | Fallback danh sách dùng chung store | Office không là single point of failure | Đề xuất |
| V10 | Layout JSON riêng `aiw-office-layout`, không nhận `.mvo-layout.json` | AGPL | Đề xuất |

## 19. Dependencies

`phaser@3.90.x` (MIT), `easystarjs` (MIT, tùy chọn), `zustand@5`, `mitt`/typed bus, `pngjs` (build-time), Playwright + `pixelmatch`, Vitest; React 19 + Tailwind v4 + shadcn/ui (D-001). Tài liệu/Hệ thống: `PROVIDER_INTEGRATION.md` (event chuẩn hóa, mock CI), `DATABASE_DESIGN.md` §4.12 + delta §11.2, `SCREEN_SPECIFICATION.md` §3.3, `DESIGN_TOKENS.md` (màu trạng thái: agent-state `online`=green, `busy`=violet, `waiting`=amber, `error`=red, `offline`=neutral; status `blocked`=orange), `RESPONSIVE_DESIGN.md`, `SECURITY.md` §4.4, WS gateway + Redis Stream (D-014), asset pipeline và `THIRD_PARTY_NOTICES.md`.

## 20. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Thiếu asset CC0 cho nội thất 14 phòng → phải tự vẽ, trễ tiến độ | lập backlog asset sớm (~60–80 sprite), dùng placeholder hình khối có nhãn khi chưa có |
| Giấy phép sprite MetroCity bị tranh cãi | lưu ảnh chụp trang license vào PR, hỏi tác giả; kế hoạch dự phòng tự vẽ |
| Phaser 4 đổi API | khóa 3.90, adapter mỏng, test hiển thị golden |
| Replay lệch do thứ tự event đa nguồn | sắp theo `globalSeq`, tie-break `id`; đồng hồ ảo |
| Event "đang làm" nhưng provider đã chết (stale) | `heartbeat`/silence 20 phút → `stale` hiển thị, không bịa trạng thái |
| 18 nhân vật + bubble gây giật trên iGPU | bake layer tĩnh, gộp delta, thang suy giảm, fallback list |
| Người dùng mong "văn phòng sống động" (đi lại, cà phê) | giải thích nguyên tắc evidence-driven; policy tùy chọn gắn event |
| Rò dữ liệu nhạy cảm lên bubble | chỉ dùng tóm tắt redact, `textContent`, test fuzz |
| Mâu thuẫn tài liệu (§21) gây sai lệch triển khai | giải quyết trước khi code |

## 21. Vấn đề chưa giải quyết và mâu thuẫn phát hiện

1. **CC0-only vs D-012**: **[ĐÃ GIẢI QUYẾT bởi OWR-2 Q-15]** CC0 mặc định; OFL font cho phép với license/NOTICE; CC-BY từng package cần Owner duyệt; MIT-asset không rõ nguồn gốc không bundle; code MIT/Apache vẫn tái dùng được (D-012). (Còn lại: Pixel Agents asset không bundle trừ khi tác giả xác nhận nguồn gốc.)
2. **Giấy phép sprite JIK-A-4**: PIXEL_AGENTS_ANALYSIS §0/§5.3 ghi "chưa rõ, cần xác minh"; LICENSE_AND_REUSE §1.3/§6.1 ghi CC0 đã xác minh. Cần chụp lại trang license làm bằng chứng.
3. ~~**Kích thước tile**: DB `tile_size` mặc định 32 và SCREEN "32px tiles" vs nguồn tái dùng 16 px (§5).~~ **[OWR-3 A.7/X-6: đã giải quyết — art 16 px, phóng 2x = 32 px hiệu dụng; DB `tile_size`=16 + `display_scale`=2 (đề xuất)]** Kèm kích thước map mặc định DB 40×24 vs thiết kế 64×40.
4. **Số animation**: DB `character_positions.animation` 7 giá trị (có `sleeping`) vs FR-158 13 giá trị (không có `sleeping`).
5. **Relax/Coffee**: SCREEN §3.3 "idle không đi uống cà phê, xem TV; không họp giả lập; Meeting chỉ hiển thị khi có review" vs FR-151 (14 phòng cố định) và FR-164 (Relax/Coffee khi event Idle). Hướng giải: phòng luôn có, nhân vật chỉ vào theo event + policy tắt mặc định.
6. **Engine trong SCREEN §3.3** ghi "PixiJS/Canvas 2D [Giả định]" vs D-011 Phaser.
7. **Màu trạng thái**: SCREEN §3.3 ⛔ bubble "đỏ" cho `blocked/failed` vs DESIGN_TOKENS `blocked`=orange, `failed`=red; DESIGN_TOKENS agent-state `busy`=violet nhưng status `running`=blue. Đề xuất: blocked=orange, failed/error=red; `busy` (avatar ring) violet, chip trạng thái dùng token status.
8. **Agent đang `waiting_approval`** nên đứng tại desk hay đi tới "Review desk" (SCREEN §3.3 nói đi tới Review desk): đề xuất mặc định đứng desk (ít chuyển động giả), có tùy chọn `goToReviewDesk`.
9. Tên `TASK_BLOCKED`/`AGENT_STATE_CHANGED` (FR-117) và tên event ở DB/WS spec (`agent.state.changed`, `agent.tool.event`, `agent.seat.assigned`, `task.handoff`) chưa được thống nhất một bảng ánh xạ.
10. Nguồn asset nội thất CC0 đủ bộ chưa tồn tại; có thuê/ủy quyền designer?
11. Nhiều tầng (FR-153) và chat/click-to-talk trong office để P2: chỉ chừa móc.
12. Chưa có số đo hiệu năng thật của Office trên máy tham chiếu; ngưỡng §14 là đề xuất.
13. Chưa xác minh cách gom `message.delta` của Codex/OpenHands để "talking" không nhấp nháy.

## 22. Acceptance Criteria

- [ ] 14 phòng đúng danh sách FR-151 tồn tại trong layout mặc định; mọi phòng department có cửa, ≥ 1 desk và đường tới Reception (validator xanh); click phòng liệt kê agent/task.
- [ ] Mỗi trong 13 animation có ít nhất một event kích hoạt trong bảng §7.2 và được test render (AC-096) chỉ qua route harness không có trong build production.
- [ ] Mọi chuyển trạng thái trong `office_transition_log` có `causedByEventId`; `office_unbacked_transition_total` = 0 trong test E2E; luồng event rỗng 10 phút → 0 chuyển động.
- [ ] Replay golden: cùng fixture → cùng hash snapshot qua 3 lần chạy và qua tốc độ ×1/×10/×60; 8 kịch bản §17.3 đều có golden.
- [ ] Bubble xuất hiện ≤ 1 s sau `approval.requested`/`TASK_BLOCKED`/`waiting_*`; bubble không chứa chuỗi khớp secret mẫu (fuzz redaction).
- [ ] 18 agent + 8 helper: p95 thời gian khung ≤ 16.6 ms trên máy tham chiếu (hoặc ghi nhận ngoại lệ có phê duyệt), draw call ≤ 120, heap office ≤ 150 MB; tab ẩn ≤ 30 FPS.
- [ ] Edit mode lưu layout, tải lại giữ nguyên (kể cả `revision`), Reset to default có confirm, xung đột 409 xử lý, từ chối định dạng lạ; nhân vật đứng yên và mờ khi Edit.
- [ ] Camera: zoom nguyên 1–6, pan chuột/phím/touch, bounds, Follow; reduced-motion tắt đi lại và lerp.
- [ ] Viewport < 768 px hoặc lỗi WebGL/render → Fallback list đầy đủ action, nhận cùng dữ liệu, sort blocked trước; Office lỗi không làm hỏng app.
- [ ] Mất kết nối → đóng băng + banner; nối lại resync từ `lastEventId` có đánh dấu.
- [ ] Gói mặc định chỉ chứa asset CC0 có `CREDITS.json`; CI fail với SPDX khác/thiếu; không có asset hoặc code lấy từ `my-virtual-office`; mọi file dẫn xuất Pixel Agents có header MIT + mục trong `THIRD_PARTY_NOTICES.md`; không có font FS Pixel Sans, không có PNG tiền-`add749a`.
- [ ] Mock adapter chỉ xuất hiện trong test/CI (kiểm bundle); sản phẩm không có pet/thời tiết/họp giả/đi dạo ngẫu nhiên.
