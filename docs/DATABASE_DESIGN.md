# AI WORKSPACE — Database & Infrastructure Design

> Agent 08 — Database & Infrastructure Engineer
> Stack: NestJS + TypeScript · PostgreSQL 16 · Redis 7 · BullMQ · Docker · Windows 11 dev
> Trạng thái: DRAFT v1 — chờ review của team (các điểm cần duyệt đánh dấu **[CẦN DUYỆT]**)

---

## 0. Tóm tắt quyết định

| Hạng mục | Quyết định |
|---|---|
| ORM | **Drizzle ORM** (`drizzle-orm` + `drizzle-kit`) trên driver `postgres` (postgres.js) |
| Primary key | `uuid` v7 sinh ở application layer (`uuidv7` package), fallback `gen_random_uuid()` ở DB |
| Thời gian | `timestamptz` cho mọi cột thời gian, lưu UTC |
| Tiền tệ | `numeric(14,6)` USD cho cost; token là `integer` |
| JSONB | Chỉ cho config/payload có schema rõ (validate bằng Zod ở app); không JSON blob cho dữ liệu unbounded |
| Soft delete | `deleted_at` trên các bảng "master data"; append-only cho event/usage/audit |
| Multi-tenancy | `organization_id` trên mọi bảng business; desktop mode = 1 org mặc định `local` |
| Event store | Bảng `agent_events` append-only, partition theo tháng, sequence per aggregate + global cursor |
| Số bảng | 44 bảng, 19 enum |

---

## 1. ORM recommendation: Drizzle ORM

### 1.1 Bảng so sánh

| Tiêu chí | Prisma (v6/v7) | Drizzle (stable 0.45.x; 1.0 còn beta, không dùng nhãn "v1.x") | Kết luận |
|---|---|---|---|
| Migrations | `prisma migrate` tốt, nhưng mọi object không mô tả được trong `schema.prisma` (partial index, partitioned table, trigger, `CHECK`) phải viết tay bằng `--create-only`; migrate diff có thể **drop** index lạ trên bảng nó quản lý | `drizzle-kit generate` sinh SQL thuần, editable; hỗ trợ custom migration `drizzle-kit generate --custom`; không xoá object nó không biết | Drizzle |
| JSONB | `Json` type, không typed; filter JSON path giới hạn | `jsonb().$type<T>()` typed, query JSON bằng `sql` operator tự do | Drizzle |
| Type safety | Rất tốt, generated client | Rất tốt, infer từ schema (`InferSelectModel`), không cần codegen step | Hoà |
| Transaction | `$transaction` interactive, timeout mặc định 5s, không nested savepoint | `db.transaction(async tx => ...)` nested = savepoint, không timeout ẩn; phù hợp event-store append + outbox | Drizzle |
| Performance | Prisma 7 bỏ Rust engine (TS query compiler), đã cải thiện; vẫn có overhead abstraction | Gần raw SQL, prepared statements, `postgres.js` pipeline | Drizzle |
| NestJS integration | `nestjs-prisma` chín muồi, `PrismaService` pattern phổ biến | Không có package chính thức nhưng chỉ cần 1 `DrizzleModule` provider (~30 dòng), đơn giản | Hoà |
| Windows dev | Prisma 6 cần engine binary (đôi khi lỗi antivirus/path có dấu cách như `Agent management`); Prisma 7 đã bỏ | Pure TS, không binary, không vấn đề path | Drizzle |
| Enum | Native `enum` → `pg enum`; đổi giá trị enum = migration phức tạp | `pgEnum` native; thêm value sinh `ALTER TYPE ... ADD VALUE` | Hoà (Drizzle nhỉnh) |
| Partial index | **Không** hỗ trợ trong schema | `.where(sql\`deleted_at is null\`)` ngay trong schema | Drizzle |
| Partitioned table | Không nhận biết, migrate diff conflict | Không declarative nhưng không conflict; khai báo table bình thường + custom SQL tạo partition | Drizzle |
| Raw SQL khi cần | `$queryRaw` tagged template | `sql` tagged template, compose được với schema columns | Drizzle |
| Studio / tooling | Prisma Studio tốt | Drizzle Studio tốt | Hoà |

### 1.2 Lý do chọn Drizzle

1. **Event store và usage ledger cần partition + partial index + BRIN index** — Drizzle mô tả được hầu hết ngay trong schema, phần còn lại (partition) viết SQL migration mà không bị tool "sửa lại".
2. **Soft delete với unique constraint** (`unique(org_id, slug) where deleted_at is null`) là partial unique index, Prisma không mô tả được → dễ drift.
3. **Transaction semantics** quan trọng cho `appendEvent + updateAggregate + outbox` trong một transaction; Drizzle không áp timeout ẩn và hỗ trợ savepoint.
4. **Windows + đường dẫn có khoảng trắng**: Drizzle không có binary engine, không có bước `prisma generate` phải chạy lại sau mỗi `npm install`.
5. Team đã quen SQL (thiết kế này SQL-first); Drizzle = "SQL có type", learning curve thấp.

Rủi ro / mitigation: Drizzle relational query API (`db.query.x.findMany({ with })`) kém trưởng thành hơn Prisma include → dùng explicit join cho query phức tạp; viết repository layer mỏng trong NestJS để cô lập.

### 1.3 Cấu trúc thư mục đề xuất

```
apps/api/src/database/
  schema/
    enums.ts
    identity.ts        # users, organizations, organization_members
    org.ts             # departments
    agents.ts          # agents, agent_profiles, agent_provider_connections, provider_configs
    skills.ts
    memories.ts
    projects.ts
    workflows.ts
    tasks.ts
    runtime.ts         # agent_sessions, agent_events, agent_messages
    approvals.ts
    artifacts.ts       # artifacts, git_repositories, git_worktrees
    usage.ts           # usage_records, usage_daily_rollups
    audit.ts
    office.ts
    system.ts          # notifications, system_settings, resource_limits
    index.ts
  migrations/          # drizzle-kit output (SQL)
  seeds/
  drizzle.module.ts
drizzle.config.ts
```

---

## 2. ERD (Mermaid)

```mermaid
erDiagram
  organizations ||--o{ organization_members : has
  users ||--o{ organization_members : joins
  organizations ||--o{ departments : owns
  departments ||--o{ departments : parent
  departments ||--o{ agents : contains
  agents ||--o| departments : heads
  agents ||--o{ agents : reports_to
  agents ||--|| agent_profiles : has
  agents ||--o{ agent_provider_connections : uses
  provider_configs ||--o{ agent_provider_connections : via
  provider_configs ||--o{ agents : default_provider
  organizations ||--o{ skills : owns
  skills ||--o{ skill_versions : versions
  skill_versions ||--o{ skill_documents : files
  agents ||--o{ agent_skills : learns
  skills ||--o{ agent_skills : assigned
  agents ||--o{ agent_memories : remembers
  organizations ||--o{ projects : owns
  projects ||--o{ project_members : staffed
  agents ||--o{ project_members : member
  projects ||--o{ project_revisions : snapshots
  projects ||--o{ git_repositories : repo
  git_repositories ||--o{ git_worktrees : worktree
  organizations ||--o{ workflows : owns
  workflows ||--o{ workflow_versions : versions
  workflow_versions ||--o{ workflow_nodes : nodes
  workflow_versions ||--o{ workflow_edges : edges
  workflow_versions ||--o{ workflow_runs : executes
  projects ||--o{ workflow_runs : for
  workflow_runs ||--o{ node_runs : steps
  workflow_nodes ||--o{ node_runs : instance
  projects ||--o{ tasks : contains
  agents ||--o{ tasks : assigned
  tasks ||--o{ task_dependencies : depends
  tasks ||--o{ task_assignments : history
  node_runs ||--o{ tasks : spawns
  agents ||--o{ agent_sessions : runs
  tasks ||--o{ agent_sessions : for
  git_worktrees ||--o{ agent_sessions : cwd
  agent_sessions ||--o{ agent_events : emits
  agents ||--o{ agent_events : actor
  tasks ||--o{ agent_events : about
  workflow_runs ||--o{ agent_events : within
  agents ||--o{ agent_messages : sends
  agents ||--o{ agent_messages : receives
  tasks ||--o{ agent_messages : context
  approval_policies ||--o{ approvals : governs
  tasks ||--o{ approvals : requires
  users ||--o{ approvals : decides
  tasks ||--o{ artifacts : produces
  project_revisions ||--o{ artifacts : snapshot
  agent_sessions ||--o{ usage_records : consumes
  agents ||--o{ usage_records : by
  tasks ||--o{ usage_records : for
  organizations ||--o{ usage_daily_rollups : aggregates
  organizations ||--o{ audit_logs : logs
  users ||--o{ audit_logs : actor
  organizations ||--o{ office_layouts : owns
  office_layouts ||--o{ office_rooms : rooms
  office_rooms ||--o{ office_furniture : furniture
  office_rooms ||--o{ desks : desks
  departments ||--o{ office_rooms : maps
  desks ||--o| character_positions : seats
  agents ||--|| character_positions : avatar
  users ||--o{ notifications : receives
  organizations ||--o{ resource_limits : caps
  organizations ||--o{ system_settings : config
```

---

## 3. Enum definitions

```ts
// schema/enums.ts
import { pgEnum } from 'drizzle-orm/pg-core';

export const orgRole = pgEnum('org_role', ['owner', 'admin', 'member', 'viewer']);
export const agentStatus = pgEnum('agent_status', [
  'created', 'idle', 'available', 'assigned', 'running', 'waiting',
  'blocked', 'reviewing', 'completed', 'failed', 'paused', 'disabled',
]);
// OWR-2 Q-03: KHÔNG có giá trị 'full' (đã xóa). L0..L5 ánh xạ qua cột permission_profile (§4, agents).
export const permissionLevel = pgEnum('permission_level', ['read_only', 'standard', 'elevated']);
export const providerKind = pgEnum('provider_kind', [
  // Sửa sau QA-001 (P0-2): thêm codex, openhands, generic_cli, mock theo PROVIDER_INTEGRATION.md mục adapter.
  // 'mock' chỉ dùng trong CI/test, bị chặn ở môi trường sản phẩm. Các kind API (anthropic_api...) giữ cho phương án API key.
  'claude_code', 'codex', 'openhands', 'generic_cli', 'mock',
  'anthropic_api', 'openai', 'gemini', 'ollama', 'openrouter', 'custom',
]);
export const connectionStatus = pgEnum('connection_status', ['unverified', 'ok', 'error', 'revoked']);
export const skillCategory = pgEnum('skill_category', [
  'engineering', 'review', 'testing', 'devops', 'product', 'design', 'research', 'writing', 'ops', 'other',
]);
export const skillDocKind = pgEnum('skill_doc_kind', ['skill_md', 'reference', 'example', 'script', 'asset']);
export const memoryType = pgEnum('memory_type', ['fact', 'preference', 'procedure', 'episode', 'summary']);
export const memoryScope = pgEnum('memory_scope', ['agent', 'project', 'department', 'organization']);
export const memorySource = pgEnum('memory_source', ['agent', 'user', 'system', 'imported']);
export const projectStatus = pgEnum('project_status', ['draft', 'active', 'paused', 'completed', 'archived']);
export const taskStatus = pgEnum('task_status', [
  'BACKLOG', 'READY', 'ASSIGNED', 'RUNNING', 'BLOCKED', 'WAITING_APPROVAL',
  'REVIEWING', 'FAILED', 'COMPLETED', 'CANCELLED',
]);
export const taskPriority = pgEnum('task_priority', ['critical', 'high', 'medium', 'low']);
export const runStatus = pgEnum('run_status', [
  'pending', 'running', 'waiting', 'paused', 'succeeded', 'failed', 'cancelled',
]);
export const sessionStatus = pgEnum('session_status', [
  'starting', 'active', 'suspended', 'resumable', 'exited', 'crashed',
]);
export const approvalStatus = pgEnum('approval_status', ['pending', 'approved', 'rejected', 'expired', 'cancelled']);
export const messageType = pgEnum('message_type', ['request', 'response', 'handoff', 'notice', 'review', 'escalation']);
export const artifactType = pgEnum('artifact_type', ['file', 'diff', 'report', 'test_result', 'log', 'binary', 'document']);
export const notificationLevel = pgEnum('notification_level', ['info', 'success', 'warning', 'error']);
export const auditSeverity = pgEnum('audit_severity', ['info', 'notice', 'warning', 'critical']);
```

Lưu ý: `task_status` giữ UPPERCASE; các enum khác lowercase snake. **[OWR-2 Q-13: ĐÃ DUYỆT]** (không còn cần duyệt casing). `permission_level` không chứa `full` (OWR-2 Q-03). Enum `provider_kind` giữ `codex`/`openhands` chỉ để mở rộng qua `AgentAdapter`; OWR-2 D-005/D-006 hoãn hai provider này khỏi MVP (không seed/không dùng trong MVP).

---

## 4. Table definitions (Drizzle schema)

Quy ước chung: mọi bảng có `id uuid PK`, `created_at`, `updated_at`; bảng business có `organization_id`; bảng master data có `deleted_at`. Helper:

```ts
// schema/_common.ts
import { sql } from 'drizzle-orm';
import { timestamp, uuid } from 'drizzle-orm/pg-core';
import { organizations } from './identity';

export const id = () => uuid('id').primaryKey().default(sql`gen_random_uuid()`);
export const timestamps = {
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
};
export const softDelete = {
  deletedAt: timestamp('deleted_at', { withTimezone: true }),
};
export const orgId = () => uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' });
```

Import dùng chung cho các file schema bên dưới (lược bỏ ở từng block cho gọn):

```ts
import { sql } from 'drizzle-orm';
import {
  pgTable, uuid, text, boolean, integer, smallint, bigint, bigserial, real, numeric, date, inet,
  jsonb, timestamp, index, uniqueIndex, primaryKey, foreignKey, check,
} from 'drizzle-orm/pg-core';
import { vector } from 'drizzle-orm/pg-core'; // pgvector, optional
import * as e from './enums';
import { id, timestamps, softDelete, orgId } from './_common';
```

### 4.1 Identity & tenancy

```ts
// schema/identity.ts
export const users = pgTable('users', {
  id: id(),
  email: text('email').notNull(),
  displayName: text('display_name').notNull(),
  avatarUrl: text('avatar_url'),
  passwordHash: text('password_hash'),            // null khi desktop/local mode
  isLocal: boolean('is_local').notNull().default(false),
  locale: text('locale').notNull().default('vi'),
  preferences: jsonb('preferences').$type<UserPreferences>().notNull().default({}), // {timezone, theme, notifications}
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  ...timestamps, ...softDelete,
}, (t) => [
  uniqueIndex('users_email_uq').on(sql`lower(${t.email})`).where(sql`${t.deletedAt} is null`),
]);

export const organizations = pgTable('organizations', {
  id: id(),
  slug: text('slug').notNull(),
  name: text('name').notNull(),
  logoUrl: text('logo_url'),
  isDefault: boolean('is_default').notNull().default(false),   // desktop mode org
  settings: jsonb('settings').$type<OrgSettings>().notNull().default({}),
  ...timestamps, ...softDelete,
}, (t) => [
  uniqueIndex('organizations_slug_uq').on(t.slug).where(sql`${t.deletedAt} is null`),
  uniqueIndex('organizations_default_uq').on(t.isDefault).where(sql`${t.isDefault} = true`),
]);

export const organizationMembers = pgTable('organization_members', {
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  role: e.orgRole('role').notNull().default('member'),
  invitedBy: uuid('invited_by').references(() => users.id, { onDelete: 'set null' }),
  joinedAt: timestamp('joined_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  primaryKey({ columns: [t.organizationId, t.userId] }),
  index('org_members_user_idx').on(t.userId),
]);
```

### 4.2 Departments & agents

```ts
// schema/org.ts
export const departments = pgTable('departments', {
  id: id(),
  organizationId: orgId(),
  parentDepartmentId: uuid('parent_department_id'),          // self FK khai báo ở dưới
  headAgentId: uuid('head_agent_id'),                        // FK → agents (thêm bằng migration tay để tránh vòng)
  name: text('name').notNull(),
  slug: text('slug').notNull(),
  description: text('description'),
  color: text('color'),
  icon: text('icon'),
  budgetMonthlyUsd: numeric('budget_monthly_usd', { precision: 14, scale: 2 }),
  budgetMonthlyTokens: bigint('budget_monthly_tokens', { mode: 'number' }),
  sortOrder: integer('sort_order').notNull().default(0),
  ...timestamps, ...softDelete,
}, (t) => [
  foreignKey({ columns: [t.parentDepartmentId], foreignColumns: [t.id], name: 'departments_parent_fk' }).onDelete('set null'),
  uniqueIndex('departments_org_slug_uq').on(t.organizationId, t.slug).where(sql`${t.deletedAt} is null`),
  index('departments_parent_idx').on(t.parentDepartmentId),
  check('departments_no_self_parent', sql`${t.parentDepartmentId} is distinct from ${t.id}`),
]);
```

```ts
// schema/agents.ts
export const providerConfigs = pgTable('provider_configs', {
  id: id(),
  organizationId: orgId(),
  kind: e.providerKind('kind').notNull(),
  name: text('name').notNull(),                          // "Claude Code (local CLI)"
  baseUrl: text('base_url'),
  credentialRef: text('credential_ref'),                 // key trong OS credential store (DPAPI/keyring), KHÔNG lưu secret
  defaultModel: text('default_model'),
  availableModels: jsonb('available_models').$type<string[]>().notNull().default([]),
  pricing: jsonb('pricing').$type<ProviderPricing>().notNull().default({}), // {model:{inputPerM,outputPerM,cacheReadPerM,cacheWritePerM}}
  maxConcurrent: integer('max_concurrent').notNull().default(5), // D-010/Q-07: mặc định 5 (không phải xác nhận subscription); seed Claude 5; Codex hoãn (D-005)
  rateLimitRpm: integer('rate_limit_rpm'),
  isEnabled: boolean('is_enabled').notNull().default(true),
  ...timestamps, ...softDelete,
}, (t) => [
  uniqueIndex('provider_configs_org_name_uq').on(t.organizationId, t.name).where(sql`${t.deletedAt} is null`),
]);

export const agents = pgTable('agents', {
  id: id(),
  organizationId: orgId(),
  departmentId: uuid('department_id').references(() => departments.id, { onDelete: 'set null' }),
  reportingManagerAgentId: uuid('reporting_manager_agent_id'),
  name: text('name').notNull(),
  slug: text('slug').notNull(),
  avatar: text('avatar'),                                // sprite key hoặc URL
  jobTitle: text('job_title').notNull(),
  description: text('description'),
  responsibilities: text('responsibilities').array().notNull().default(sql`'{}'::text[]`),
  systemPrompt: text('system_prompt').notNull(),
  providerId: uuid('provider_id').references(() => providerConfigs.id, { onDelete: 'set null' }),
  model: text('model'),
  permissionLevel: e.permissionLevel('permission_level').notNull().default('standard'),
  // [OWR-2 Q-03, đề xuất cột] permission_profile = hồ sơ quyền chi tiết ánh xạ L0..L5 (ví dụ 'L0'..'L5'); KHÔNG hỗ trợ quyền 'full'.
  // CHECK (permission_profile IN ('L0','L1','L2','L3','L4','L5')) và CHECK ràng buộc nhất quán permission_level <-> permission_profile (bảng ánh xạ chốt ở USER_ROLES_AND_PERMISSIONS / AGENT_ORGANIZATION; Owner xác nhận).
  permissionProfile: text('permission_profile').notNull().default('L1'),
  workingDirectory: text('working_directory'),
  maxTokenBudget: integer('max_token_budget').notNull().default(500_000),
  maxExecutionTimeSec: integer('max_execution_time_sec').notNull().default(1800),
  maxRetry: integer('max_retry').notNull().default(2),
  status: e.agentStatus('status').notNull().default('created'),
  statusChangedAt: timestamp('status_changed_at', { withTimezone: true }).notNull().defaultNow(),
  currentTaskId: uuid('current_task_id'),                // FK → tasks thêm bằng migration tay (vòng)
  isTemplate: boolean('is_template').notNull().default(false),
  ...timestamps, ...softDelete,
}, (t) => [
  foreignKey({ columns: [t.reportingManagerAgentId], foreignColumns: [t.id], name: 'agents_manager_fk' }).onDelete('set null'),
  uniqueIndex('agents_org_slug_uq').on(t.organizationId, t.slug).where(sql`${t.deletedAt} is null`),
  index('agents_org_status_idx').on(t.organizationId, t.status).where(sql`${t.deletedAt} is null`),
  index('agents_department_idx').on(t.departmentId),
  index('agents_manager_idx').on(t.reportingManagerAgentId),
  check('agents_budget_positive', sql`${t.maxTokenBudget} > 0 and ${t.maxExecutionTimeSec} > 0 and ${t.maxRetry} >= 0`),
]);

export const agentProfiles = pgTable('agent_profiles', {
  agentId: uuid('agent_id').primaryKey().references(() => agents.id, { onDelete: 'cascade' }),
  personality: text('personality'),
  communicationStyle: text('communication_style'),
  strengths: text('strengths').array().notNull().default(sql`'{}'::text[]`),
  weaknesses: text('weaknesses').array().notNull().default(sql`'{}'::text[]`),
  preferredTools: text('preferred_tools').array().notNull().default(sql`'{}'::text[]`),
  bio: text('bio'),
  spriteConfig: jsonb('sprite_config').$type<SpriteConfig>().notNull().default({}),   // {sheet, skin, hair, outfit}
  stats: jsonb('stats').$type<AgentStats>().notNull().default({}),                     // cache: tasksDone, successRate (rollup)
  ...timestamps,
});

export const agentProviderConnections = pgTable('agent_provider_connections', {
  id: id(),
  agentId: uuid('agent_id').notNull().references(() => agents.id, { onDelete: 'cascade' }),
  providerId: uuid('provider_id').notNull().references(() => providerConfigs.id, { onDelete: 'cascade' }),
  model: text('model'),
  priority: integer('priority').notNull().default(0),    // 0 = primary, 1.. = fallback
  overrides: jsonb('overrides').$type<ProviderOverrides>().notNull().default({}),  // {temperature, maxTokens, tools[]}
  status: e.connectionStatus('status').notNull().default('unverified'),
  lastVerifiedAt: timestamp('last_verified_at', { withTimezone: true }),
  lastError: text('last_error'),
  ...timestamps,
}, (t) => [
  uniqueIndex('agent_provider_uq').on(t.agentId, t.providerId),
  uniqueIndex('agent_provider_priority_uq').on(t.agentId, t.priority),
]);
```

### 4.3 Skills

```ts
// schema/skills.ts
export const skills = pgTable('skills', {
  id: id(),
  organizationId: orgId(),
  slug: text('slug').notNull(),
  name: text('name').notNull(),
  category: e.skillCategory('category').notNull().default('other'),
  tags: text('tags').array().notNull().default(sql`'{}'::text[]`),
  description: text('description'),
  isBuiltin: boolean('is_builtin').notNull().default(false),
  currentVersionId: uuid('current_version_id'),          // FK → skill_versions (migration tay)
  ...timestamps, ...softDelete,
}, (t) => [
  uniqueIndex('skills_org_slug_uq').on(t.organizationId, t.slug).where(sql`${t.deletedAt} is null`),
  index('skills_org_category_idx').on(t.organizationId, t.category),
  index('skills_tags_gin').using('gin', t.tags),
]);

export const skillVersions = pgTable('skill_versions', {
  id: id(),
  skillId: uuid('skill_id').notNull().references(() => skills.id, { onDelete: 'cascade' }),
  version: text('version').notNull(),                    // semver "1.2.0"
  changelog: text('changelog'),
  frontmatter: jsonb('frontmatter').$type<SkillFrontmatter>().notNull().default({}), // parse từ SKILL.md
  checksum: text('checksum').notNull(),                  // sha256 toàn bộ documents
  publishedBy: uuid('published_by').references(() => users.id, { onDelete: 'set null' }),
  publishedAt: timestamp('published_at', { withTimezone: true }).notNull().defaultNow(),
  isDeprecated: boolean('is_deprecated').notNull().default(false),
}, (t) => [
  uniqueIndex('skill_versions_uq').on(t.skillId, t.version),
]);

export const skillDocuments = pgTable('skill_documents', {
  id: id(),
  skillVersionId: uuid('skill_version_id').notNull().references(() => skillVersions.id, { onDelete: 'cascade' }),
  kind: e.skillDocKind('kind').notNull(),
  path: text('path').notNull(),                          // "SKILL.md", "references/api.md", "scripts/run.ps1"
  mimeType: text('mime_type').notNull().default('text/markdown'),
  content: text('content'),                              // text files; binary → storage_ref
  storageRef: text('storage_ref'),                       // path trên disk/object storage cho asset
  sizeBytes: integer('size_bytes').notNull().default(0),
  checksum: text('checksum').notNull(),
  sortOrder: integer('sort_order').notNull().default(0),
}, (t) => [
  uniqueIndex('skill_documents_path_uq').on(t.skillVersionId, t.path),
  uniqueIndex('skill_documents_one_skill_md').on(t.skillVersionId).where(sql`${t.kind} = 'skill_md'`),
]);

export const agentSkills = pgTable('agent_skills', {
  agentId: uuid('agent_id').notNull().references(() => agents.id, { onDelete: 'cascade' }),
  skillId: uuid('skill_id').notNull().references(() => skills.id, { onDelete: 'cascade' }),
  pinnedVersionId: uuid('pinned_version_id').references(() => skillVersions.id, { onDelete: 'set null' }), // null = current
  isEnabled: boolean('is_enabled').notNull().default(true),
  proficiency: integer('proficiency').notNull().default(3), // 1..5 hiển thị UI
  assignedAt: timestamp('assigned_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  primaryKey({ columns: [t.agentId, t.skillId] }),
  index('agent_skills_skill_idx').on(t.skillId),
  check('agent_skills_proficiency_range', sql`${t.proficiency} between 1 and 5`),
]);
```

### 4.4 Memories

```ts
// schema/memories.ts
export const agentMemories = pgTable('agent_memories', {
  id: id(),
  organizationId: orgId(),
  agentId: uuid('agent_id').references(() => agents.id, { onDelete: 'cascade' }),   // null khi scope != agent
  projectId: uuid('project_id').references(() => projects.id, { onDelete: 'cascade' }),
  departmentId: uuid('department_id').references(() => departments.id, { onDelete: 'cascade' }),
  type: e.memoryType('type').notNull(),
  scope: e.memoryScope('scope').notNull().default('agent'),
  source: e.memorySource('source').notNull().default('agent'),
  title: text('title'),
  content: text('content').notNull(),
  importance: smallint('importance').notNull().default(3),           // 1..5
  isApproved: boolean('is_approved').notNull().default(false),
  approvedBy: uuid('approved_by').references(() => users.id, { onDelete: 'set null' }),
  approvedAt: timestamp('approved_at', { withTimezone: true }),
  sourceTaskId: uuid('source_task_id'),                              // FK → tasks (migration tay)
  sourceEventId: uuid('source_event_id'),                            // không FK (partition), chỉ tham chiếu
  embedding: vector('embedding', { dimensions: 1536 }),              // pgvector, optional [CẦN DUYỆT]
  expiresAt: timestamp('expires_at', { withTimezone: true }),
  lastAccessedAt: timestamp('last_accessed_at', { withTimezone: true }),
  accessCount: integer('access_count').notNull().default(0),
  ...timestamps, ...softDelete,
}, (t) => [
  index('memories_agent_idx').on(t.agentId, t.scope).where(sql`${t.deletedAt} is null`),
  index('memories_project_idx').on(t.projectId).where(sql`${t.deletedAt} is null`),
  index('memories_approved_idx').on(t.organizationId, t.isApproved),
  index('memories_content_fts').using('gin', sql`to_tsvector('simple', coalesce(${t.title},'') || ' ' || ${t.content})`),
  check('memories_scope_target', sql`
    (${t.scope} = 'agent' and ${t.agentId} is not null) or
    (${t.scope} = 'project' and ${t.projectId} is not null) or
    (${t.scope} = 'department' and ${t.departmentId} is not null) or
    (${t.scope} = 'organization')`),
]);
```

### 4.5 Projects

```ts
// schema/projects.ts
export const projects = pgTable('projects', {
  id: id(),
  organizationId: orgId(),
  key: text('key').notNull(),                            // "AIW" → task_id "AIW-42"
  name: text('name').notNull(),
  description: text('description'),
  status: e.projectStatus('status').notNull().default('draft'),
  rootPath: text('root_path'),                           // đường dẫn workspace trên máy
  defaultBranch: text('default_branch').notNull().default('main'),
  ownerUserId: uuid('owner_user_id').references(() => users.id, { onDelete: 'set null' }),
  leadAgentId: uuid('lead_agent_id').references(() => agents.id, { onDelete: 'set null' }),
  taskCounter: integer('task_counter').notNull().default(0),   // tăng atomically khi tạo task
  settings: jsonb('settings').$type<ProjectSettings>().notNull().default({}),
  startedAt: timestamp('started_at', { withTimezone: true }),
  completedAt: timestamp('completed_at', { withTimezone: true }),
  ...timestamps, ...softDelete,
}, (t) => [
  uniqueIndex('projects_org_key_uq').on(t.organizationId, t.key).where(sql`${t.deletedAt} is null`),
  index('projects_org_status_idx').on(t.organizationId, t.status),
]);

export const projectMembers = pgTable('project_members', {
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  agentId: uuid('agent_id').notNull().references(() => agents.id, { onDelete: 'cascade' }),
  roleInProject: text('role_in_project').notNull().default('contributor'),  // lead/reviewer/contributor
  addedAt: timestamp('added_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [primaryKey({ columns: [t.projectId, t.agentId] })]);

export const projectRevisions = pgTable('project_revisions', {
  id: id(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  number: integer('number').notNull(),
  label: text('label'),
  commitSha: text('commit_sha'),
  summary: text('summary'),
  createdByAgentId: uuid('created_by_agent_id').references(() => agents.id, { onDelete: 'set null' }),
  createdByUserId: uuid('created_by_user_id').references(() => users.id, { onDelete: 'set null' }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [uniqueIndex('project_revisions_uq').on(t.projectId, t.number)]);
```

### 4.6 Workflows (React Flow compatible)

```ts
// schema/workflows.ts
export const workflows = pgTable('workflows', {
  id: id(),
  organizationId: orgId(),
  slug: text('slug').notNull(),
  name: text('name').notNull(),
  description: text('description'),
  isTemplate: boolean('is_template').notNull().default(false),
  isBuiltin: boolean('is_builtin').notNull().default(false),
  currentVersionId: uuid('current_version_id'),          // FK → workflow_versions (migration tay)
  ...timestamps, ...softDelete,
}, (t) => [
  uniqueIndex('workflows_org_slug_uq').on(t.organizationId, t.slug).where(sql`${t.deletedAt} is null`),
]);

export const workflowVersions = pgTable('workflow_versions', {
  id: id(),
  workflowId: uuid('workflow_id').notNull().references(() => workflows.id, { onDelete: 'cascade' }),
  version: integer('version').notNull(),
  viewport: jsonb('viewport').$type<{ x: number; y: number; zoom: number }>().notNull().default({ x: 0, y: 0, zoom: 1 }),
  variablesSchema: jsonb('variables_schema').$type<JsonSchema>().notNull().default({}),  // input của run
  isPublished: boolean('is_published').notNull().default(false),
  publishedAt: timestamp('published_at', { withTimezone: true }),
  createdBy: uuid('created_by').references(() => users.id, { onDelete: 'set null' }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [uniqueIndex('workflow_versions_uq').on(t.workflowId, t.version)]);

// 1 row = 1 React Flow node: { id, type, position, data }
export const workflowNodes = pgTable('workflow_nodes', {
  id: id(),
  workflowVersionId: uuid('workflow_version_id').notNull().references(() => workflowVersions.id, { onDelete: 'cascade' }),
  nodeKey: text('node_key').notNull(),                   // React Flow node.id, ổn định giữa các version
  type: text('type').notNull(),                          // 'start' | 'agent_task' | 'approval' | 'condition' | 'parallel' | 'join' | 'end'
  label: text('label').notNull(),
  positionX: real('position_x').notNull().default(0),
  positionY: real('position_y').notNull().default(0),
  width: real('width'),
  height: real('height'),
  parentNodeKey: text('parent_node_key'),                // group/subflow
  data: jsonb('data').$type<WorkflowNodeData>().notNull().default({}),
  // data = { agentId?, departmentId?, roleSelector?, taskTemplate:{name,description,acceptanceCriteria}, approvalPolicyId?, condition?, timeoutSec?, retry? }
  sortOrder: integer('sort_order').notNull().default(0),
}, (t) => [
  uniqueIndex('workflow_nodes_key_uq').on(t.workflowVersionId, t.nodeKey),
]);

export const workflowEdges = pgTable('workflow_edges', {
  id: id(),
  workflowVersionId: uuid('workflow_version_id').notNull().references(() => workflowVersions.id, { onDelete: 'cascade' }),
  edgeKey: text('edge_key').notNull(),                   // React Flow edge.id
  sourceNodeKey: text('source_node_key').notNull(),
  targetNodeKey: text('target_node_key').notNull(),
  sourceHandle: text('source_handle'),                   // 'approved' | 'rejected' | 'true' | 'false'
  targetHandle: text('target_handle'),
  type: text('type').notNull().default('default'),
  label: text('label'),
  condition: jsonb('condition').$type<EdgeCondition>(),  // {expr: "output.testsPassed == true"}
  data: jsonb('data').$type<Record<string, unknown>>().notNull().default({}),
}, (t) => [
  uniqueIndex('workflow_edges_key_uq').on(t.workflowVersionId, t.edgeKey),
  foreignKey({ columns: [t.workflowVersionId, t.sourceNodeKey], foreignColumns: [workflowNodes.workflowVersionId, workflowNodes.nodeKey], name: 'edges_source_fk' }).onDelete('cascade'),
  foreignKey({ columns: [t.workflowVersionId, t.targetNodeKey], foreignColumns: [workflowNodes.workflowVersionId, workflowNodes.nodeKey], name: 'edges_target_fk' }).onDelete('cascade'),
  index('workflow_edges_source_idx').on(t.workflowVersionId, t.sourceNodeKey),
]);

export const workflowRuns = pgTable('workflow_runs', {
  id: id(),
  organizationId: orgId(),
  workflowVersionId: uuid('workflow_version_id').notNull().references(() => workflowVersions.id, { onDelete: 'restrict' }),
  projectId: uuid('project_id').references(() => projects.id, { onDelete: 'set null' }),
  triggeredByUserId: uuid('triggered_by_user_id').references(() => users.id, { onDelete: 'set null' }),
  status: e.runStatus('status').notNull().default('pending'),
  input: jsonb('input').$type<Record<string, unknown>>().notNull().default({}),
  output: jsonb('output').$type<Record<string, unknown>>(),
  error: text('error'),
  currentNodeKeys: text('current_node_keys').array().notNull().default(sql`'{}'::text[]`),
  startedAt: timestamp('started_at', { withTimezone: true }),
  finishedAt: timestamp('finished_at', { withTimezone: true }),
  ...timestamps,
}, (t) => [
  index('workflow_runs_project_idx').on(t.projectId, t.status),
  index('workflow_runs_org_created_idx').on(t.organizationId, t.createdAt),
]);

export const nodeRuns = pgTable('node_runs', {
  id: id(),
  workflowRunId: uuid('workflow_run_id').notNull().references(() => workflowRuns.id, { onDelete: 'cascade' }),
  workflowNodeId: uuid('workflow_node_id').notNull().references(() => workflowNodes.id, { onDelete: 'restrict' }),
  nodeKey: text('node_key').notNull(),
  attempt: integer('attempt').notNull().default(1),
  status: e.runStatus('status').notNull().default('pending'),
  assignedAgentId: uuid('assigned_agent_id').references(() => agents.id, { onDelete: 'set null' }),
  taskId: uuid('task_id'),                               // FK → tasks (migration tay)
  input: jsonb('input').$type<Record<string, unknown>>().notNull().default({}),
  output: jsonb('output').$type<Record<string, unknown>>(),
  checkpoint: jsonb('checkpoint').$type<NodeCheckpoint>(),   // {step, cursorEventSeq, partialOutput, resumeToken}
  error: text('error'),
  startedAt: timestamp('started_at', { withTimezone: true }),
  finishedAt: timestamp('finished_at', { withTimezone: true }),
  ...timestamps,
}, (t) => [
  uniqueIndex('node_runs_attempt_uq').on(t.workflowRunId, t.nodeKey, t.attempt),
  index('node_runs_status_idx').on(t.workflowRunId, t.status),
]);
```

### 4.7 Tasks

```ts
// schema/tasks.ts
export const tasks = pgTable('tasks', {
  id: id(),
  organizationId: orgId(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  taskId: text('task_id').notNull(),                     // "AIW-42" (human readable)
  number: integer('number').notNull(),
  parentTaskId: uuid('parent_task_id'),
  nodeRunId: uuid('node_run_id').references(() => nodeRuns.id, { onDelete: 'set null' }),
  name: text('name').notNull(),
  description: text('description'),
  assignedAgentId: uuid('assigned_agent_id').references(() => agents.id, { onDelete: 'set null' }),
  createdByUserId: uuid('created_by_user_id').references(() => users.id, { onDelete: 'set null' }),
  createdByAgentId: uuid('created_by_agent_id').references(() => agents.id, { onDelete: 'set null' }),
  priority: e.taskPriority('priority').notNull().default('medium'),
  status: e.taskStatus('status').notNull().default('BACKLOG'),
  statusChangedAt: timestamp('status_changed_at', { withTimezone: true }).notNull().defaultNow(),
  input: jsonb('input').$type<TaskInput>().notNull().default({}),           // {files[], context, variables}
  expectedOutput: text('expected_output'),
  acceptanceCriteria: text('acceptance_criteria').array().notNull().default(sql`'{}'::text[]`),
  result: jsonb('result').$type<TaskResult>(),                               // {summary, artifactIds[], metrics}
  workspacePath: text('workspace_path'),
  branch: text('branch'),
  worktreeId: uuid('worktree_id'),                       // FK → git_worktrees (migration tay)
  retryCount: integer('retry_count').notNull().default(0),
  maxRetry: integer('max_retry').notNull().default(2),
  dueAt: timestamp('due_at', { withTimezone: true }),
  startedAt: timestamp('started_at', { withTimezone: true }),
  completedAt: timestamp('completed_at', { withTimezone: true }),
  blockedReason: text('blocked_reason'),
  ...timestamps, ...softDelete,
}, (t) => [
  uniqueIndex('tasks_project_number_uq').on(t.projectId, t.number),
  uniqueIndex('tasks_task_id_uq').on(t.organizationId, t.taskId),
  foreignKey({ columns: [t.parentTaskId], foreignColumns: [t.id], name: 'tasks_parent_fk' }).onDelete('set null'),
  index('tasks_project_status_idx').on(t.projectId, t.status).where(sql`${t.deletedAt} is null`),
  index('tasks_agent_active_idx').on(t.assignedAgentId).where(sql`${t.status} in ('ASSIGNED','RUNNING','BLOCKED','WAITING_APPROVAL','REVIEWING')`),
  index('tasks_ready_queue_idx').on(t.organizationId, t.priority, t.createdAt).where(sql`${t.status} = 'READY'`),
  index('tasks_parent_idx').on(t.parentTaskId),
]);

export const taskDependencies = pgTable('task_dependencies', {
  taskId: uuid('task_id').notNull().references(() => tasks.id, { onDelete: 'cascade' }),
  dependsOnTaskId: uuid('depends_on_task_id').notNull().references(() => tasks.id, { onDelete: 'cascade' }),
  kind: text('kind').notNull().default('finish_to_start'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  primaryKey({ columns: [t.taskId, t.dependsOnTaskId] }),
  index('task_deps_reverse_idx').on(t.dependsOnTaskId),
  check('task_deps_no_self', sql`${t.taskId} <> ${t.dependsOnTaskId}`),
]);

// Lịch sử giao việc — append-only
export const taskAssignments = pgTable('task_assignments', {
  id: id(),
  taskId: uuid('task_id').notNull().references(() => tasks.id, { onDelete: 'cascade' }),
  agentId: uuid('agent_id').references(() => agents.id, { onDelete: 'set null' }),
  previousAgentId: uuid('previous_agent_id').references(() => agents.id, { onDelete: 'set null' }),
  assignedByUserId: uuid('assigned_by_user_id').references(() => users.id, { onDelete: 'set null' }),
  assignedByAgentId: uuid('assigned_by_agent_id').references(() => agents.id, { onDelete: 'set null' }),
  reason: text('reason'),                                // 'manual' | 'workflow' | 'reassign_failed' | 'escalation'
  assignedAt: timestamp('assigned_at', { withTimezone: true }).notNull().defaultNow(),
  releasedAt: timestamp('released_at', { withTimezone: true }),
  outcome: text('outcome'),                              // 'completed' | 'failed' | 'reassigned' | null (đang giữ)
}, (t) => [
  index('task_assignments_task_idx').on(t.taskId, t.assignedAt),
  index('task_assignments_agent_idx').on(t.agentId, t.assignedAt),
]);
```

### 4.8 Runtime: sessions, events, messages

```ts
// schema/runtime.ts
export const agentSessions = pgTable('agent_sessions', {
  id: id(),
  organizationId: orgId(),
  agentId: uuid('agent_id').notNull().references(() => agents.id, { onDelete: 'cascade' }),
  taskId: uuid('task_id').references(() => tasks.id, { onDelete: 'set null' }),
  workflowRunId: uuid('workflow_run_id').references(() => workflowRuns.id, { onDelete: 'set null' }),
  nodeRunId: uuid('node_run_id').references(() => nodeRuns.id, { onDelete: 'set null' }),
  providerId: uuid('provider_id').references(() => providerConfigs.id, { onDelete: 'set null' }),
  providerSessionId: text('provider_session_id'),        // Claude Code session id / OpenAI thread id
  resumeToken: text('resume_token'),                     // opaque, có thể encrypted
  cwd: text('cwd'),
  worktreeId: uuid('worktree_id'),                       // FK → git_worktrees (migration tay)
  pid: integer('pid'),
  hostname: text('hostname'),
  status: e.sessionStatus('status').notNull().default('starting'),
  model: text('model'),
  lastEventSeq: bigint('last_event_seq', { mode: 'number' }).notNull().default(0),  // sequence cuối của aggregate session
  lastHeartbeatAt: timestamp('last_heartbeat_at', { withTimezone: true }),
  exitCode: integer('exit_code'),
  exitReason: text('exit_reason'),
  startedAt: timestamp('started_at', { withTimezone: true }).notNull().defaultNow(),
  endedAt: timestamp('ended_at', { withTimezone: true }),
  ...timestamps,
}, (t) => [
  index('sessions_agent_active_idx').on(t.agentId).where(sql`${t.status} in ('starting','active','suspended')`),
  index('sessions_task_idx').on(t.taskId),
  index('sessions_provider_sid_idx').on(t.providerId, t.providerSessionId),
]);

// APPEND-ONLY, PARTITIONED BY RANGE (occurred_at) — xem mục 5
export const agentEvents = pgTable('agent_events', {
  id: uuid('id').notNull().default(sql`gen_random_uuid()`),
  globalSeq: bigserial('global_seq', { mode: 'number' }).notNull(),      // cursor toàn cục cho UI/replay
  organizationId: uuid('organization_id').notNull(),                      // không FK để partition nhẹ (validate ở app)
  aggregateType: text('aggregate_type').notNull(),                        // 'session' | 'task' | 'run' | 'agent'
  aggregateId: uuid('aggregate_id').notNull(),
  sequence: bigint('sequence', { mode: 'number' }).notNull(),             // per aggregate, bắt đầu 1
  type: text('type').notNull(),                                           // 'session.started' | 'tool.called' | 'task.status_changed' ...
  payload: jsonb('payload').$type<Record<string, unknown>>().notNull().default({}),
  agentId: uuid('agent_id'),
  taskId: uuid('task_id'),
  runId: uuid('run_id'),
  sessionId: uuid('session_id'),
  nodeRunId: uuid('node_run_id'),
  correlationId: uuid('correlation_id'),                                  // trace một request xuyên nhiều aggregate
  causationId: uuid('causation_id'),                                      // event gây ra event này
  actorUserId: uuid('actor_user_id'),
  schemaVersion: smallint('schema_version').notNull().default(1),
  occurredAt: timestamp('occurred_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  primaryKey({ columns: [t.id, t.occurredAt], name: 'agent_events_pk' }),   // partition key phải nằm trong PK
  uniqueIndex('agent_events_aggregate_seq_uq').on(t.aggregateType, t.aggregateId, t.sequence, t.occurredAt),
  index('agent_events_global_seq_idx').on(t.globalSeq),
  index('agent_events_session_idx').on(t.sessionId, t.sequence),
  index('agent_events_task_idx').on(t.taskId, t.occurredAt),
  index('agent_events_run_idx').on(t.runId, t.occurredAt),
  index('agent_events_agent_time_idx').on(t.agentId, t.occurredAt),
  index('agent_events_org_type_time_idx').on(t.organizationId, t.type, t.occurredAt),
  index('agent_events_occurred_brin').using('brin', t.occurredAt),
]);

export const agentMessages = pgTable('agent_messages', {
  id: id(),
  organizationId: orgId(),
  fromAgentId: uuid('from_agent_id').references(() => agents.id, { onDelete: 'set null' }),
  fromUserId: uuid('from_user_id').references(() => users.id, { onDelete: 'set null' }),
  toAgentId: uuid('to_agent_id').references(() => agents.id, { onDelete: 'cascade' }),   // null = broadcast tới department
  toDepartmentId: uuid('to_department_id').references(() => departments.id, { onDelete: 'cascade' }),
  taskId: uuid('task_id').references(() => tasks.id, { onDelete: 'set null' }),
  threadId: uuid('thread_id'),                           // = id của message gốc
  replyToId: uuid('reply_to_id'),
  type: e.messageType('type').notNull().default('notice'),
  subject: text('subject'),
  content: text('content').notNull(),
  attachments: jsonb('attachments').$type<MessageAttachment[]>().notNull().default([]), // [{artifactId}|{path}]
  isRead: boolean('is_read').notNull().default(false),
  readAt: timestamp('read_at', { withTimezone: true }),
  deliveredAt: timestamp('delivered_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  foreignKey({ columns: [t.replyToId], foreignColumns: [t.id], name: 'agent_messages_reply_fk' }).onDelete('set null'),
  index('agent_messages_inbox_idx').on(t.toAgentId, t.isRead, t.createdAt),
  index('agent_messages_thread_idx').on(t.threadId, t.createdAt),
  index('agent_messages_task_idx').on(t.taskId),
  check('agent_messages_has_sender', sql`${t.fromAgentId} is not null or ${t.fromUserId} is not null`),
  check('agent_messages_has_recipient', sql`${t.toAgentId} is not null or ${t.toDepartmentId} is not null`),
]);
```

### 4.9 Approvals

```ts
// schema/approvals.ts
export const approvalPolicies = pgTable('approval_policies', {
  id: id(),
  organizationId: orgId(),
  slug: text('slug').notNull(),
  name: text('name').notNull(),
  description: text('description'),
  appliesTo: text('applies_to').notNull(),               // 'tool_call' | 'git_push' | 'deploy' | 'spend' | 'task_completion' | 'memory_write'
  rules: jsonb('rules').$type<ApprovalRules>().notNull().default({}),
  // rules = { autoApprove?: {maxCostUsd?, allowedTools?[], pathGlobs?[]}, requireRoles?: ['owner','admin'], minApprovers?: 1 }
  timeoutSec: integer('timeout_sec').notNull().default(3600),
  onTimeout: text('on_timeout').notNull().default('reject'),   // 'reject' | 'approve' | 'escalate'
  isEnabled: boolean('is_enabled').notNull().default(true),
  priority: integer('priority').notNull().default(100),
  ...timestamps, ...softDelete,
}, (t) => [
  uniqueIndex('approval_policies_org_slug_uq').on(t.organizationId, t.slug).where(sql`${t.deletedAt} is null`),
]);

export const approvals = pgTable('approvals', {
  id: id(),
  organizationId: orgId(),
  policyId: uuid('policy_id').references(() => approvalPolicies.id, { onDelete: 'set null' }),
  type: text('type').notNull(),                          // = policy.applies_to tại thời điểm tạo
  status: e.approvalStatus('status').notNull().default('pending'),
  requestedByAgentId: uuid('requested_by_agent_id').references(() => agents.id, { onDelete: 'set null' }),
  taskId: uuid('task_id').references(() => tasks.id, { onDelete: 'cascade' }),
  sessionId: uuid('session_id').references(() => agentSessions.id, { onDelete: 'set null' }),
  nodeRunId: uuid('node_run_id').references(() => nodeRuns.id, { onDelete: 'set null' }),
  title: text('title').notNull(),
  payload: jsonb('payload').$type<ApprovalPayload>().notNull().default({}),   // {tool, args, diff, estimatedCostUsd, command}
  decidedByUserId: uuid('decided_by_user_id').references(() => users.id, { onDelete: 'set null' }),
  decision: text('decision'),                            // 'approve' | 'reject' | 'approve_once' | 'approve_always'
  decisionNote: text('decision_note'),
  decidedAt: timestamp('decided_at', { withTimezone: true }),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
  ...timestamps,
}, (t) => [
  index('approvals_pending_idx').on(t.organizationId, t.expiresAt).where(sql`${t.status} = 'pending'`),
  index('approvals_task_idx').on(t.taskId),
]);
```

### 4.10 Artifacts & git

```ts
// schema/artifacts.ts
export const gitRepositories = pgTable('git_repositories', {
  id: id(),
  organizationId: orgId(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  name: text('name').notNull(),
  localPath: text('local_path').notNull(),               // C:/work/repo (normalized forward slashes)
  remoteUrl: text('remote_url'),
  defaultBranch: text('default_branch').notNull().default('main'),
  credentialRef: text('credential_ref'),                 // OS credential store key
  lastFetchedAt: timestamp('last_fetched_at', { withTimezone: true }),
  ...timestamps, ...softDelete,
}, (t) => [
  uniqueIndex('git_repositories_path_uq').on(t.organizationId, t.localPath).where(sql`${t.deletedAt} is null`),
]);

export const gitWorktrees = pgTable('git_worktrees', {
  id: id(),
  repositoryId: uuid('repository_id').notNull().references(() => gitRepositories.id, { onDelete: 'cascade' }),
  taskId: uuid('task_id').references(() => tasks.id, { onDelete: 'set null' }),
  agentId: uuid('agent_id').references(() => agents.id, { onDelete: 'set null' }),
  path: text('path').notNull(),
  branch: text('branch').notNull(),
  baseCommitSha: text('base_commit_sha'),
  headCommitSha: text('head_commit_sha'),
  status: text('status').notNull().default('active'),    // 'active' | 'merged' | 'abandoned' | 'removed'
  lockedBySessionId: uuid('locked_by_session_id').references(() => agentSessions.id, { onDelete: 'set null' }),
  removedAt: timestamp('removed_at', { withTimezone: true }),
  ...timestamps,
}, (t) => [
  uniqueIndex('git_worktrees_path_uq').on(t.repositoryId, t.path).where(sql`${t.removedAt} is null`),
  index('git_worktrees_task_idx').on(t.taskId),
]);

export const artifacts = pgTable('artifacts', {
  id: id(),
  organizationId: orgId(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  taskId: uuid('task_id').references(() => tasks.id, { onDelete: 'set null' }),
  sessionId: uuid('session_id').references(() => agentSessions.id, { onDelete: 'set null' }),
  revisionId: uuid('revision_id').references(() => projectRevisions.id, { onDelete: 'set null' }),
  producedByAgentId: uuid('produced_by_agent_id').references(() => agents.id, { onDelete: 'set null' }),
  type: e.artifactType('type').notNull(),
  path: text('path').notNull(),                          // relative to project root hoặc storage key
  name: text('name').notNull(),
  mimeType: text('mime_type'),
  sizeBytes: bigint('size_bytes', { mode: 'number' }).notNull().default(0),
  checksum: text('checksum').notNull(),                  // sha256
  revision: integer('revision').notNull().default(1),    // cùng path tăng dần
  storageRef: text('storage_ref'),                       // null = đọc trực tiếp từ workspace
  metadata: jsonb('metadata').$type<Record<string, unknown>>().notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  uniqueIndex('artifacts_path_rev_uq').on(t.projectId, t.path, t.revision),
  index('artifacts_task_idx').on(t.taskId, t.createdAt),
  index('artifacts_checksum_idx').on(t.checksum),
]);
```

### 4.11 Usage & audit

```ts
// schema/usage.ts  — APPEND-ONLY, partition theo tháng (giống agent_events)
export const usageRecords = pgTable('usage_records', {
  id: uuid('id').notNull().default(sql`gen_random_uuid()`),
  organizationId: uuid('organization_id').notNull(),
  providerId: uuid('provider_id'),
  providerKind: text('provider_kind').notNull(),
  model: text('model').notNull(),
  agentId: uuid('agent_id'),
  departmentId: uuid('department_id'),                   // denormalized tại thời điểm ghi
  projectId: uuid('project_id'),
  taskId: uuid('task_id'),
  sessionId: uuid('session_id'),
  workflowRunId: uuid('workflow_run_id'),
  requestId: text('request_id'),                         // id từ provider, idempotency
  inputTokens: integer('input_tokens').notNull().default(0),
  outputTokens: integer('output_tokens').notNull().default(0),
  cacheReadTokens: integer('cache_read_tokens').notNull().default(0),
  cacheWriteTokens: integer('cache_write_tokens').notNull().default(0),
  costUsd: numeric('cost_usd', { precision: 14, scale: 6 }).notNull().default('0'),
  pricingSnapshot: jsonb('pricing_snapshot').$type<ModelPricing>(),   // giá áp dụng lúc ghi, để audit lại
  latencyMs: integer('latency_ms'),
  occurredAt: timestamp('occurred_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  primaryKey({ columns: [t.id, t.occurredAt] }),
  uniqueIndex('usage_records_request_uq').on(t.providerId, t.requestId, t.occurredAt).where(sql`${t.requestId} is not null`),
  index('usage_records_org_time_idx').on(t.organizationId, t.occurredAt),
  index('usage_records_agent_time_idx').on(t.agentId, t.occurredAt),
  index('usage_records_task_idx').on(t.taskId),
  index('usage_records_session_idx').on(t.sessionId),
  index('usage_records_occurred_brin').using('brin', t.occurredAt),
]);

const NIL = '00000000-0000-0000-0000-000000000000';
export const usageDailyRollups = pgTable('usage_daily_rollups', {
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  day: date('day').notNull(),
  agentId: uuid('agent_id'),            // nullable → dimension "all"
  departmentId: uuid('department_id'),
  projectId: uuid('project_id'),
  providerId: uuid('provider_id'),
  model: text('model').notNull().default('*'),
  requests: integer('requests').notNull().default(0),
  inputTokens: bigint('input_tokens', { mode: 'number' }).notNull().default(0),
  outputTokens: bigint('output_tokens', { mode: 'number' }).notNull().default(0),
  cacheReadTokens: bigint('cache_read_tokens', { mode: 'number' }).notNull().default(0),
  cacheWriteTokens: bigint('cache_write_tokens', { mode: 'number' }).notNull().default(0),
  costUsd: numeric('cost_usd', { precision: 14, scale: 6 }).notNull().default('0'),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  uniqueIndex('usage_daily_rollups_uq').on(
    t.organizationId, t.day,
    sql`coalesce(${t.agentId}, '${sql.raw(NIL)}')`,
    sql`coalesce(${t.departmentId}, '${sql.raw(NIL)}')`,
    sql`coalesce(${t.projectId}, '${sql.raw(NIL)}')`,
    sql`coalesce(${t.providerId}, '${sql.raw(NIL)}')`,
    t.model),
  index('usage_daily_rollups_org_day_idx').on(t.organizationId, t.day),
]);
```

```ts
// schema/audit.ts — APPEND-ONLY
export const auditLogs = pgTable('audit_logs', {
  id: id(),
  organizationId: uuid('organization_id').notNull(),
  actorUserId: uuid('actor_user_id'),
  actorAgentId: uuid('actor_agent_id'),
  action: text('action').notNull(),                      // 'agent.create' | 'task.status_change' | 'approval.decide' | 'settings.update'
  entityType: text('entity_type').notNull(),
  entityId: uuid('entity_id'),
  severity: e.auditSeverity('severity').notNull().default('info'),
  before: jsonb('before').$type<Record<string, unknown>>(),
  after: jsonb('after').$type<Record<string, unknown>>(),
  ipAddress: inet('ip_address'),
  userAgent: text('user_agent'),
  requestId: text('request_id'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  index('audit_logs_org_time_idx').on(t.organizationId, t.createdAt),
  index('audit_logs_entity_idx').on(t.entityType, t.entityId),
  index('audit_logs_actor_idx').on(t.actorUserId, t.createdAt),
]);
```

### 4.12 Office (pixel-office view)

```ts
// schema/office.ts
export const officeLayouts = pgTable('office_layouts', {
  id: id(),
  organizationId: orgId(),
  name: text('name').notNull(),
  isActive: boolean('is_active').notNull().default(false),
  widthTiles: integer('width_tiles').notNull().default(40),
  heightTiles: integer('height_tiles').notNull().default(24),
  tileSize: integer('tile_size').notNull().default(32),
  tileset: text('tileset').notNull().default('default'),
  groundLayer: jsonb('ground_layer').$type<number[][]>().notNull().default([]),   // tile ids (Tiled-compatible)
  collisionLayer: jsonb('collision_layer').$type<number[][]>().notNull().default([]),
  ...timestamps, ...softDelete,
}, (t) => [
  uniqueIndex('office_layouts_active_uq').on(t.organizationId).where(sql`${t.isActive} = true and ${t.deletedAt} is null`),
]);

export const officeRooms = pgTable('office_rooms', {
  id: id(),
  layoutId: uuid('layout_id').notNull().references(() => officeLayouts.id, { onDelete: 'cascade' }),
  departmentId: uuid('department_id').references(() => departments.id, { onDelete: 'set null' }),
  name: text('name').notNull(),
  kind: text('kind').notNull().default('department'),    // 'department' | 'meeting' | 'lounge' | 'lobby' | 'server'
  x: integer('x').notNull(), y: integer('y').notNull(),
  width: integer('width').notNull(), height: integer('height').notNull(),
  floorTile: integer('floor_tile'),
  wallColor: text('wall_color'),
}, (t) => [index('office_rooms_layout_idx').on(t.layoutId)]);

export const officeFurniture = pgTable('office_furniture', {
  id: id(),
  roomId: uuid('room_id').notNull().references(() => officeRooms.id, { onDelete: 'cascade' }),
  kind: text('kind').notNull(),                          // 'plant' | 'whiteboard' | 'sofa' | 'printer' ...
  sprite: text('sprite').notNull(),
  x: integer('x').notNull(), y: integer('y').notNull(),
  rotation: smallint('rotation').notNull().default(0),
  isWalkable: boolean('is_walkable').notNull().default(false),
  props: jsonb('props').$type<Record<string, unknown>>().notNull().default({}),
});

export const desks = pgTable('desks', {
  id: id(),
  roomId: uuid('room_id').notNull().references(() => officeRooms.id, { onDelete: 'cascade' }),
  label: text('label'),
  x: integer('x').notNull(), y: integer('y').notNull(),
  facing: text('facing').notNull().default('down'),      // 'up' | 'down' | 'left' | 'right'
  seatX: integer('seat_x').notNull(), seatY: integer('seat_y').notNull(),
  assignedAgentId: uuid('assigned_agent_id').references(() => agents.id, { onDelete: 'set null' }),
}, (t) => [
  uniqueIndex('desks_agent_uq').on(t.assignedAgentId).where(sql`${t.assignedAgentId} is not null`),
  uniqueIndex('desks_room_pos_uq').on(t.roomId, t.x, t.y),
]);

// Trạng thái hiện tại của nhân vật — hot row, 1 row/agent; không lưu lịch sử (lịch sử đi qua Redis/events)
export const characterPositions = pgTable('character_positions', {
  agentId: uuid('agent_id').primaryKey().references(() => agents.id, { onDelete: 'cascade' }),
  layoutId: uuid('layout_id').references(() => officeLayouts.id, { onDelete: 'set null' }),
  deskId: uuid('desk_id').references(() => desks.id, { onDelete: 'set null' }),
  roomId: uuid('room_id').references(() => officeRooms.id, { onDelete: 'set null' }),
  x: real('x').notNull().default(0),
  y: real('y').notNull().default(0),
  facing: text('facing').notNull().default('down'),
  animation: text('animation').notNull().default('idle'),   // 'idle' | 'walk' | 'typing' | 'thinking' | 'talking' | 'sleeping' | 'error'
  targetX: real('target_x'), targetY: real('target_y'),
  speechBubble: text('speech_bubble'),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [index('character_positions_layout_idx').on(t.layoutId)]);
```

### 4.13 System: notifications, settings, limits

```ts
// schema/system.ts
export const notifications = pgTable('notifications', {
  id: id(),
  organizationId: orgId(),
  userId: uuid('user_id').references(() => users.id, { onDelete: 'cascade' }),   // null = toàn org
  level: e.notificationLevel('level').notNull().default('info'),
  kind: text('kind').notNull(),                          // 'approval.requested' | 'task.completed' | 'agent.failed' | 'budget.threshold'
  title: text('title').notNull(),
  body: text('body'),
  entityType: text('entity_type'),
  entityId: uuid('entity_id'),
  actionUrl: text('action_url'),
  isRead: boolean('is_read').notNull().default(false),
  readAt: timestamp('read_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  index('notifications_user_unread_idx').on(t.userId, t.createdAt).where(sql`${t.isRead} = false`),
]);

export const systemSettings = pgTable('system_settings', {
  organizationId: uuid('organization_id').references(() => organizations.id, { onDelete: 'cascade' }), // null = global
  key: text('key').notNull(),                            // 'office.tick_ms', 'events.retention_days', 'ui.theme'
  value: jsonb('value').$type<unknown>().notNull(),
  valueType: text('value_type').notNull().default('json'),   // 'string' | 'number' | 'boolean' | 'json'
  description: text('description'),
  updatedBy: uuid('updated_by').references(() => users.id, { onDelete: 'set null' }),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  uniqueIndex('system_settings_uq').on(sql`coalesce(${t.organizationId}, '${sql.raw(NIL)}')`, t.key),
]);

export const resourceLimits = pgTable('resource_limits', {
  id: id(),
  organizationId: orgId(),
  scopeType: text('scope_type').notNull(),               // 'organization' | 'provider' | 'department' | 'agent' | 'project'
  scopeId: uuid('scope_id'),                             // null khi scope_type = organization
  maxConcurrentAgents: integer('max_concurrent_agents'),
  maxConcurrentSessions: integer('max_concurrent_sessions'),
  maxTokensPerDay: bigint('max_tokens_per_day', { mode: 'number' }),
  maxCostUsdPerDay: numeric('max_cost_usd_per_day', { precision: 14, scale: 2 }),
  maxCostUsdPerMonth: numeric('max_cost_usd_per_month', { precision: 14, scale: 2 }),
  maxRequestsPerMinute: integer('max_requests_per_minute'),
  hardStop: boolean('hard_stop').notNull().default(true),   // false = chỉ cảnh báo
  ...timestamps,
}, (t) => [
  uniqueIndex('resource_limits_scope_uq').on(t.organizationId, t.scopeType, sql`coalesce(${t.scopeId}, '${sql.raw(NIL)}')`),
]);
```

### 4.14 FK vòng, trigger — thêm bằng custom migration

Các FK sau tạo vòng tham chiếu nên không khai báo trong Drizzle schema (tránh lỗi TS circular import), thêm bằng `drizzle-kit generate --custom`:

```sql
ALTER TABLE departments ADD CONSTRAINT departments_head_agent_fk
  FOREIGN KEY (head_agent_id) REFERENCES agents(id) ON DELETE SET NULL;
ALTER TABLE agents ADD CONSTRAINT agents_current_task_fk
  FOREIGN KEY (current_task_id) REFERENCES tasks(id) ON DELETE SET NULL;
ALTER TABLE skills ADD CONSTRAINT skills_current_version_fk
  FOREIGN KEY (current_version_id) REFERENCES skill_versions(id) ON DELETE SET NULL;
ALTER TABLE workflows ADD CONSTRAINT workflows_current_version_fk
  FOREIGN KEY (current_version_id) REFERENCES workflow_versions(id) ON DELETE SET NULL;
ALTER TABLE node_runs ADD CONSTRAINT node_runs_task_fk
  FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE SET NULL;
ALTER TABLE tasks ADD CONSTRAINT tasks_worktree_fk
  FOREIGN KEY (worktree_id) REFERENCES git_worktrees(id) ON DELETE SET NULL;
ALTER TABLE agent_sessions ADD CONSTRAINT agent_sessions_worktree_fk
  FOREIGN KEY (worktree_id) REFERENCES git_worktrees(id) ON DELETE SET NULL;
ALTER TABLE agent_memories ADD CONSTRAINT agent_memories_source_task_fk
  FOREIGN KEY (source_task_id) REFERENCES tasks(id) ON DELETE SET NULL;

-- updated_at trigger dùng chung, áp cho mọi bảng có cột updated_at
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END $$ LANGUAGE plpgsql;
DO $$ DECLARE r record; BEGIN
  FOR r IN SELECT table_name FROM information_schema.columns
           WHERE table_schema='public' AND column_name='updated_at' LOOP
    EXECUTE format('CREATE TRIGGER trg_%I_updated_at BEFORE UPDATE ON %I
                    FOR EACH ROW EXECUTE FUNCTION set_updated_at()', r.table_name, r.table_name);
  END LOOP; END $$;

-- Chặn UPDATE/DELETE trên bảng append-only (hàng rào thứ hai ngoài quyền role)
CREATE OR REPLACE FUNCTION reject_mutation() RETURNS trigger AS $$
BEGIN RAISE EXCEPTION 'table % is append-only', TG_TABLE_NAME; END $$ LANGUAGE plpgsql;
CREATE TRIGGER trg_agent_events_ro BEFORE UPDATE OR DELETE ON agent_events FOR EACH ROW EXECUTE FUNCTION reject_mutation();
CREATE TRIGGER trg_usage_records_ro BEFORE UPDATE OR DELETE ON usage_records FOR EACH ROW EXECUTE FUNCTION reject_mutation();
CREATE TRIGGER trg_audit_logs_ro BEFORE UPDATE OR DELETE ON audit_logs FOR EACH ROW EXECUTE FUNCTION reject_mutation();
```

---

## 5. Event store design (`agent_events`)

### 5.1 Nguyên tắc

- **Append-only**: không `UPDATE`/`DELETE` từ application. Role DB của API chỉ có `INSERT, SELECT` trên bảng này; role `aiw_maintenance` mới có quyền `DETACH/DROP PARTITION` (trigger `reject_mutation` không cản DROP partition vì đó là DDL).
- **Sequence per aggregate**: `(aggregate_type, aggregate_id, sequence)` unique. Aggregate chính là `session` (mỗi agent session là một stream), ngoài ra `task`, `run`, `agent` cho domain events. Sequence được cấp trong transaction bằng `UPDATE agent_sessions SET last_event_seq = last_event_seq + 1 ... RETURNING last_event_seq` (row lock tự nhiên → không race). Với aggregate khác, dùng bảng nhỏ `event_sequences(aggregate_type, aggregate_id, last_seq)` upsert — **[CẦN DUYỆT]** (hoặc chỉ stream theo session + `global_seq`).
- **Global cursor**: `global_seq bigserial` cho UI/stream consumer đọc "mọi thứ từ cursor X" không cần biết aggregate. Lưu ý `bigserial` không đảm bảo thứ tự commit tuyệt đối; consumer chấp nhận out-of-order nhỏ trong cửa sổ 1–2 giây (hoặc dùng `pg_current_xact_id` nếu cần nghiêm ngặt).
- **Correlation**: mọi event mang `agent_id, task_id, run_id, session_id, node_run_id, correlation_id, causation_id` denormalized → query theo bất kỳ chiều nào không cần join.
- **Payload** JSONB có `schema_version`; mỗi `type` có Zod schema trong `packages/contracts`. Không nhét stdout thô vào payload: chunk output > 8KB ghi vào artifact (`type = log`) và payload chỉ giữ `{artifactId, offset, length}`.

### 5.2 Event types (tiền tố theo aggregate)

> **[OWR-2 Q-12/Q-19/D-014]** Tên event trong DB là `lowercase.dot.notation` (danh sách dưới đây). `UPPER_SNAKE_CASE` chỉ là ánh xạ API/compat qua MỘT hàm xác định (`EVENT_SYSTEM.md` đầu tài liệu); không có hệ thống đặt tên thứ hai. Danh sách dưới là **đề xuất, chưa phải danh sách gốc của Owner** (mục 8.4 gốc chưa có trong repo: PENDING_OWNER_CLARIFICATION, xem `EVENT_SYSTEM.md`). Cần chốt: phân đoạn có `_` (`status_changed`) hay tách chấm (`status.changed`).

```
session.started | session.heartbeat | session.suspended | session.resumed | session.exited | session.crashed
llm.request | llm.response | llm.stream_chunk (chỉ khi bật debug) | llm.error
tool.called | tool.result | tool.denied
file.read | file.written | shell.exec | git.commit | git.push
task.created | task.status_changed | task.assigned | task.blocked | task.completed | task.failed
run.started | run.node_entered | run.node_finished | run.paused | run.finished
approval.requested | approval.decided | approval.expired
agent.status_changed | agent.message_sent | agent.memory_proposed
office.moved | office.animation (ephemeral, KHÔNG ghi DB — chỉ Redis pub/sub)
```

### 5.3 Partition

```sql
-- migration custom 0004
CREATE TABLE agent_events (...) PARTITION BY RANGE (occurred_at);
CREATE TABLE agent_events_default PARTITION OF agent_events DEFAULT;
CREATE TABLE agent_events_2026_11 PARTITION OF agent_events
  FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');

CREATE OR REPLACE FUNCTION ensure_month_partitions(tbl text, months_ahead int) RETURNS void AS $$
DECLARE d date := date_trunc('month', now())::date; i int; p text;
BEGIN
  FOR i IN 0..months_ahead LOOP
    p := format('%s_%s', tbl, to_char(d + (i || ' month')::interval, 'YYYY_MM'));
    EXECUTE format('CREATE TABLE IF NOT EXISTS %I PARTITION OF %I FOR VALUES FROM (%L) TO (%L)',
      p, tbl, d + (i || ' month')::interval, d + ((i + 1) || ' month')::interval);
  END LOOP;
END $$ LANGUAGE plpgsql;
```

- BullMQ repeatable job `partition-maintainer` hàng ngày gọi `ensure_month_partitions('agent_events', 2)` và `ensure_month_partitions('usage_records', 2)`. Không phụ thuộc extension `pg_partman` để image dev nhẹ; production có thể thay bằng pg_partman.
- **[OWR-2 Q-06 — retention mặc định audit]** `audit_logs`: giữ **365 ngày online**; **signed export giữ 2 năm**. Không xóa/ghi đè dữ liệu audit (OWR-2 §7).
- Retention (system_settings `events.retention_days`, mặc định 90 desktop / 365 server; áp cho `agent_events`, KHÔNG thay Q-06 cho audit): job `event-retention` chạy `ALTER TABLE agent_events DETACH PARTITION ... CONCURRENTLY` rồi `DROP` (hoặc export ra `.jsonl.gz` trước khi drop nếu `events.archive_on_drop = true`).
- `usage_records` **không** bị drop theo retention (dữ liệu chi phí), chỉ roll-up; partition cũ > 24 tháng được archive.

### 5.4 UI replay

1. Client mở session → `GET /sessions/:id/events?afterSeq=0&limit=500` đọc từ DB theo index `(session_id, sequence)`.
2. Đồng thời subscribe Redis channel `aiw:ch:session:{id}`; server publish mỗi event sau khi `COMMIT`. Hai phương án: (a) transactional outbox (insert `agent_events` + `outbox` cùng tx, worker publish rồi xoá) — bền; (b) publish-after-commit — đơn giản, mất event nếu process chết giữa chừng nhưng client tự bù qua bước 3. **[CẦN DUYỆT]**: (a) cho server mode, (b) cho desktop. **[OWR-2 D-014: ĐÃ DUYỆT hướng Transactional Outbox; phương án (b) không còn là lựa chọn mặc định, xem §5.5]**
3. Client merge: bỏ qua event có `sequence <= lastSeq` đã có; nếu phát hiện gap (`sequence > lastSeq + 1`) thì gọi lại REST với `afterSeq`.
4. Reconnect < 10 phút: đọc Redis Stream `aiw:evt:session:{id}:buffer` trước, chỉ fallback DB khi stream đã trim.
5. Timeline toàn cục (dashboard) dùng `global_seq` tương tự, channel `aiw:ch:org:{orgId}:events`.
6. "Time travel" debug: `node_runs.checkpoint.cursorEventSeq` trỏ tới sequence trong session stream, UI replay từ đầu tới cursor để tái hiện trạng thái.

---

### 5.5 Bảng `event_outbox` [Đề xuất, đồng bộ OWR-2 D-014 — cần Owner duyệt chi tiết cột]

Bảng này **chưa có** trong bản thiết kế gốc (44 bảng); được thêm như đề xuất nhất quán với `EVENT_SYSTEM.md` §7.1.

```ts
export const eventOutbox = pgTable('event_outbox', {
  id: bigserial('id', { mode: 'number' }).primaryKey(),
  eventId: uuid('event_id').notNull(),              // tham chiếu logic tới agent_events.id (agent_events partition nên không FK cứng)
  globalSeq: bigint('global_seq', { mode: 'number' }).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  attempts: integer('attempts').notNull().default(0),
}, (t) => [index('event_outbox_global_seq_idx').on(t.globalSeq)]);
```

- Ghi cùng transaction với `agent_events`; **một relay leader** (advisory lock) đọc theo `global_seq`, publish Redis Pub/Sub hoặc Streams, xóa dòng sau khi publish; at-least-once, consumer idempotent theo `event_id`.
- Replay bằng `lastEventId` (client gửi id event cuối đã nhận; server đọc `agent_events` theo `global_seq` > cursor của id đó).
- Role relay được `DELETE` trên bảng này (khác `agent_events` append-only).

---

## 6. Usage / cost aggregation

- **Ghi**: mỗi `llm.response` ghi 1 row `usage_records` trong cùng tx với event (nguồn sự thật duy nhất). `cost_usd` tính ở app từ `provider_configs.pricing` tại thời điểm ghi, lưu `pricing_snapshot` để tính lại được khi giá đổi. Claude Code CLI báo usage qua JSON stream → parse `usage` block; nếu provider không trả usage, ước lượng bằng tokenizer và đánh dấu `pricing_snapshot.estimated = true`.
- **Idempotency**: `(provider_id, request_id)` unique partial → retry không ghi trùng.
- **Rollup**: job BullMQ `usage-rollup` mỗi 5 phút: `INSERT ... SELECT ... FROM usage_records WHERE occurred_at >= watermark GROUP BY day, agent, department, project, provider, model ON CONFLICT DO UPDATE SET x = rollup.x + excluded.x`. Watermark lưu trong `system_settings['usage.rollup_watermark']`. Rollup ghi nhiều hàng với các dimension null (tổng theo org/day, theo agent/day...) để dashboard query O(1).
- **Realtime counters** (budget enforcement) nằm trong Redis (mục 10) vì cần check trước mỗi request: `HINCRBY aiw:usage:day:...`. DB rollup là nguồn đối soát, Redis counter được rebuild từ rollup khi khởi động.
- **Budget check** trước khi spawn/gọi LLM: so Redis counter với `resource_limits` (scope ưu tiên: agent > department > provider > organization), `departments.budget_monthly_*`, `agents.max_token_budget` (per-task budget, đếm qua `usage_records` theo task_id — cached trong Redis `aiw:usage:task:{taskId}`). Vượt ngưỡng với `hard_stop = true` → task chuyển `BLOCKED` + notification `budget.threshold`; `hard_stop = false` → chỉ notification ở 80%/100%.
- Dashboard: biểu đồ theo ngày đọc `usage_daily_rollups`; drill-down 1 ngày đọc `usage_records` theo partition (BRIN index hỗ trợ scan theo thời gian).

---

## 7. Soft delete / audit strategy

| Nhóm | Chiến lược |
|---|---|
| Master data (users, organizations, departments, agents, provider_configs, skills, projects, workflows, approval_policies, git_repositories, office_layouts, agent_memories, tasks) | `deleted_at` soft delete; partial unique index `where deleted_at is null`; repository layer luôn filter `deleted_at is null` qua helper `notDeleted(t)`; purge job xoá cứng sau `retention.soft_delete_days` (mặc định 30) nếu không còn FK restrict |
| Versioned (skill_versions, workflow_versions, project_revisions) | Immutable; không delete, chỉ `is_deprecated` / `is_published` |
| History / ledger (task_assignments, agent_events, usage_records, audit_logs, approvals sau khi decided) | Append-only; DB role app không có DELETE + trigger `reject_mutation`; dọn bằng partition drop |
| Hot state (character_positions, agent_sessions, notifications) | Hard delete / TTL job (`notifications` > 90 ngày đã đọc; `agent_sessions` exited > 180 ngày) |

- **Audit**: NestJS interceptor `AuditInterceptor` ghi `audit_logs` cho mọi mutation qua REST/WS của user (before/after diff chỉ các field whitelisted — không ghi `system_prompt` đầy đủ, chỉ hash + length; không bao giờ ghi `credential_ref` value). Agent actions không ghi `audit_logs` mà đã có `agent_events`; những action nhạy cảm (git push, shell exec, approval, memory approve) ghi **cả hai** (`audit_logs.actor_agent_id`).
- Cascade: xoá cứng `organization` cascade toàn bộ (chỉ cho phép qua CLI maintenance). Xoá agent chỉ soft delete, giữ `agent_events` / `usage_records` (không FK) để báo cáo lịch sử.

---

## 8. Multi-tenancy & single-user desktop mode

- Mọi bảng business có `organization_id NOT NULL`; bảng con (skill_documents, workflow_nodes, node_runs, desks...) suy ra qua parent, không lặp cột để tránh drift — trừ bảng ledger (`agent_events`, `usage_records`, `audit_logs`) denormalize `organization_id` để partition/query nhanh.
- Scoping ở app: `TenantContext` (AsyncLocalStorage) set từ JWT/session ở server mode (desktop mode dùng bearer per-launch; D-001/CF-041); repository base class tự thêm `where organization_id = ctx.orgId`. Thêm **PostgreSQL RLS** làm hàng rào thứ hai cho server mode: policy `USING (organization_id = current_setting('app.org_id', true)::uuid)`; Drizzle `db.transaction` chạy `SET LOCAL app.org_id = ...` đầu tx. **[CẦN DUYỆT]**: bật RLS từ v1 hay để v2.
- **Desktop mode** (`APP_MODE=desktop`): seed tạo org `local` (`is_default = true`) và user `local@localhost` (`is_local = true`, không password). Auth middleware tự inject user/org này, không màn hình login. Mọi query vẫn đi qua tenant scoping nên chuyển sang server mode không đổi code. Postgres chạy bằng Docker Desktop hoặc embedded Postgres (`embedded-postgres` npm) — **[CẦN DUYỆT]**: Docker bắt buộc hay bundle embedded Postgres cho bản desktop.
- Secrets (API key, git token) **không** vào DB: `credential_ref` là key trong Windows Credential Manager (qua `@napi-rs/keyring` / `keytar`); server mode map sang Vault/env.

---

## 9. Migration & seed strategy

### 9.1 Migration

```ts
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';
export default defineConfig({
  dialect: 'postgresql',
  schema: './apps/api/src/database/schema/index.ts',
  out: './apps/api/src/database/migrations',
  dbCredentials: { url: process.env.DATABASE_URL! },
  migrations: { table: '__drizzle_migrations', schema: 'public' },
  strict: true, verbose: true,
});
```

- `pnpm db:generate` → `drizzle-kit generate` (SQL từ schema); `pnpm db:generate:custom --name add_partitions` cho SQL tay (partition, FK vòng, trigger, RLS, plpgsql).
- `pnpm db:migrate` → chạy `migrate()` từ `drizzle-orm/postgres-js/migrator` trong script Node (không dùng `drizzle-kit push` ngoài prototype).
- API khởi động: `DatabaseModule.onModuleInit` chạy migrate tự động khi `APP_MODE=desktop` (user không chạy CLI); server mode chạy migrate trong init container.
- Migration được commit cùng PR; CI chạy migrate trên Postgres service container + `drizzle-kit check` (phát hiện conflict giữa 2 branch).
- Thứ tự migration ban đầu: `0000_enums_core`, `0001_identity_org`, `0002_agents_skills`, `0003_projects_workflows_tasks`, `0004_runtime_events_partitioned` (custom), `0005_approvals_artifacts_git`, `0006_usage_audit_partitioned` (custom), `0007_office_system`, `0008_circular_fks_triggers` (custom), `0009_rls` (custom, optional).
- Drizzle không biết `agent_events` là partitioned: khai báo table trong schema chỉ để có type; migration `0004` tự viết `CREATE TABLE ... PARTITION BY`, và bỏ phần CREATE TABLE do drizzle-kit sinh cho bảng này (hoặc `drizzle-kit generate` rồi sửa tay file SQL trước khi commit).

### 9.2 Seed (idempotent, upsert theo slug)

`pnpm db:seed` chạy `seeds/index.ts` theo thứ tự, mỗi seed kiểm tra `is_builtin` / slug:

1. **Org & user**: `local` org + local user (desktop) hoặc không gì cả (server, org tạo qua onboarding).
2. **Default departments** (tree):
   ```
   Executive (CEO agent)  → budget: unlimited
   ├── Product            (PM, Business Analyst)
   ├── Engineering        (Tech Lead, Backend Dev, Frontend Dev, Mobile Dev)
   │   └── QA             (QA Engineer, Test Automation)
   ├── DevOps             (DevOps Engineer, SRE)
   ├── Design             (UX Designer)
   └── Research           (Researcher)
   ```
   Kèm agent template (`is_template = true`) cho mỗi job title với `system_prompt` mặc định + `agent_profiles`; `head_agent_id` và `reporting_manager_agent_id` set sau khi insert xong agents.
3. **Default skills** (`is_builtin = true`, category, version 1.0.0, SKILL.md trong `seeds/skills/<slug>/`): `code-review`, `write-unit-tests`, `tdd-workflow`, `git-worktree-flow`, `write-prd`, `api-design`, `db-migration`, `security-review`, `commit-convention`, `release-notes`. Loader đọc thư mục → `skill_versions` + `skill_documents` (SKILL.md, references/, examples/, scripts/), checksum sha256; seed lại chỉ tạo version mới khi checksum đổi.
4. **Default FULL SDLC workflow template** (`workflows.slug = 'full-sdlc'`, `is_builtin`, version 1): nodes
   `start → pm.requirements (agent_task) → approval.prd (approval) → techlead.design → parallel[backend.impl, frontend.impl] → join → qa.test → condition(testsPassed) → [true] devops.release → approval.deploy → end ; [false] → backend.fix → qa.test`. Mỗi node `data.roleSelector = { jobTitle: 'Backend Developer', departmentSlug: 'engineering' }` để bind agent lúc run; `position` xếp theo lưới 260x140 để mở React Flow là đẹp ngay.
5. **Default approval policies**: `git-push` (require admin), `shell-destructive` (reject by default), `spend-over-5usd`, `deploy`.
6. **Default office layout**: 40x24 tiles, 1 room mỗi department (grid 2x3), 4 desks/room, lobby + meeting room, furniture cơ bản; `character_positions` tạo cho mỗi agent seeded, gán desk theo department.
7. **system_settings** mặc định & `resource_limits` org-level (`max_concurrent_agents = 5` theo D-010 / OWR-2 Q-07, trần cấu hình 18; `max_cost_usd_per_day = 20`). **Comment seed: "không phải xác nhận subscription"** (5 là giới hạn mặc định của ứng dụng, không phải xác nhận subscription cho phép 5 session đồng thời; không tự tăng lên 10/18 khi chưa xác minh điều khoản provider). Seed `provider_configs.max_concurrent`: `claude_code` 5; ~~`codex` 3~~ **[OWR-2 D-005: Codex hoãn khỏi MVP, không seed `codex` trong MVP; giá trị 3 chỉ còn là gợi ý cho tương lai, chưa xác nhận]**. **[OWR-2 D-016]** 18 agent role là **seed data, không hardcode**; Agent Identity, Provider Identity, User Identity là ba khái niệm tách biệt (không dùng chung khóa/bảng).

Dev fixtures (demo project, vài task, event mẫu) nằm ở `seeds/dev/` chỉ chạy với `pnpm db:seed --dev`.

---

## 10. Redis key design

Prefix toàn cục `aiw:` (configurable). Mọi key có org scope trừ global.

```
# Concurrency semaphores (Lua script acquire/release, TTL phòng crash)
aiw:sem:org:{orgId}:agents                 SET  session ids đang chạy; limit từ resource_limits
aiw:sem:provider:{providerId}              SET  session ids; limit = provider_configs.max_concurrent
aiw:sem:agent:{agentId}                    STRING lock = sessionId, TTL 60s, heartbeat gia hạn
aiw:sem:worktree:{worktreeId}              STRING lock = sessionId

# Rate limit (sliding window)
aiw:rl:provider:{providerId}:rpm           ZSET  timestamps, trim < now-60s

# Usage counters (rebuild từ rollup lúc boot)
aiw:usage:day:{yyyy-mm-dd}:org:{orgId}     HASH  {inputTokens, outputTokens, costUsd, requests}  TTL 3d
aiw:usage:day:{yyyy-mm-dd}:agent:{agentId} HASH  ...
aiw:usage:month:{yyyy-mm}:dept:{deptId}    HASH  ...
aiw:usage:task:{taskId}                    HASH  tokens dùng cho task (so với agents.max_token_budget)

# Presence / live state
aiw:presence:agent:{agentId}               HASH  {status, sessionId, taskId, pid, heartbeatAt}  TTL 90s
aiw:presence:user:{userId}                 STRING socket count, TTL 60s
aiw:office:{layoutId}:positions            HASH  agentId → JSON {x,y,facing,animation,ts}  (write-behind → character_positions mỗi 5s)

# Event stream buffer (replay ngắn hạn, tránh query DB khi reconnect < 10 phút)
aiw:evt:session:{sessionId}:buffer         STREAM  MAXLEN ~ 2000, field = event JSON

# Pub/Sub channels (Socket.IO adapter dùng kênh riêng aiw:sio:*)
aiw:ch:org:{orgId}:events                  mọi agent_events sau commit (fan-out dashboard)
aiw:ch:session:{sessionId}                 event của 1 session (console view)
aiw:ch:task:{taskId}                       status change + messages
aiw:ch:office:{layoutId}                   office.moved / office.animation (ephemeral, 10 tick/s)
aiw:ch:approvals:{orgId}                   approval.requested / decided
aiw:ch:control:agent:{agentId}             lệnh từ server → runner: pause|resume|cancel|inject_message

# BullMQ queues (prefix aiw:bull)
agent-runner         job = {sessionId}            concurrency = max_concurrent_agents (worker đọc semaphore)
workflow-engine      job = {workflowRunId, nodeKey}
usage-rollup         repeat every 5m
partition-maintainer repeat daily 02:00
event-retention      repeat daily 03:00
approval-expiry      delayed job per approval (expires_at)
presence-reaper      repeat every 30s — dọn session mất heartbeat, release semaphore
office-tick          repeat 100ms (chỉ khi có client đang xem office)

# Misc
aiw:cache:agent:{agentId}:resolved         JSON agent + skills + provider đã resolve, TTL 5m, invalidate khi update
aiw:idemp:{requestId}                      STRING, TTL 24h — idempotency REST
```

Semaphore acquire (Lua, atomic): kiểm tra `SCARD` của cả 3 cấp (org, provider, agent lock) rồi `SADD` + `SET NX` trong một script; release khi session exit và `presence-reaper` dọn session có heartbeat quá hạn. Redis chạy `maxmemory-policy noeviction` vì semaphore/counter không được phép bị evict.

---

## 11. Docker Compose (dev)

```yaml
# docker-compose.dev.yml
name: aiw-dev
services:
  postgres:
    image: postgres:16-alpine          # đổi sang pgvector/pgvector:pg16 nếu duyệt embedding
    container_name: aiw-postgres
    environment:
      POSTGRES_USER: aiw
      POSTGRES_PASSWORD: aiw_dev
      POSTGRES_DB: aiw
      TZ: UTC
      PGTZ: UTC
    command: >
      postgres -c shared_buffers=256MB -c work_mem=16MB
               -c max_connections=100 -c log_min_duration_statement=500
               -c timezone=UTC
    ports: ["5432:5432"]
    volumes:
      - aiw_pgdata:/var/lib/postgresql/data          # named volume: KHÔNG bind-mount trên Windows
      - ./docker/postgres/init:/docker-entrypoint-initdb.d:ro   # 00_extensions.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U aiw -d aiw"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7-alpine
    container_name: aiw-redis
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy noeviction
    ports: ["6379:6379"]
    volumes:
      - aiw_redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

  pgweb:                               # optional: docker compose --profile tools up
    image: sosedoff/pgweb:latest
    profiles: ["tools"]
    environment:
      PGWEB_DATABASE_URL: postgres://aiw:aiw_dev@postgres:5432/aiw?sslmode=disable
    ports: ["8081:8081"]
    depends_on:
      postgres: { condition: service_healthy }

volumes:
  aiw_pgdata:
  aiw_redisdata:
```

```sql
-- docker/postgres/init/00_extensions.sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;     -- gen_random_uuid
CREATE EXTENSION IF NOT EXISTS pg_trgm;      -- search name/description
CREATE EXTENSION IF NOT EXISTS btree_gin;
-- CREATE EXTENSION IF NOT EXISTS vector;    -- cần image pgvector/pgvector:pg16 [CẦN DUYỆT]
CREATE ROLE aiw_app LOGIN PASSWORD 'aiw_app_dev';
CREATE ROLE aiw_maintenance LOGIN PASSWORD 'aiw_maint_dev';
```

```env
# .env.example
APP_MODE=desktop
DATABASE_URL=postgres://aiw:aiw_dev@localhost:5432/aiw
REDIS_URL=redis://localhost:6379/0
EVENTS_RETENTION_DAYS=90
```

### Windows notes

- Dùng **Docker Desktop với WSL2 backend**; bind-mount thư mục Windows (`D:\...`) vào Postgres data dir **không hoạt động** (permission/fsync) → luôn dùng named volume như trên.
- Đường dẫn project có khoảng trắng (`D:\Code\Tuyen\Agent management`) → mọi script npm quote `"$INIT_CWD"`; Docker `./docker/...` tương đối là OK.
- Line ending: thêm `.gitattributes` với `*.sql text eol=lf` và `*.sh text eol=lf` để init script không lỗi CRLF.
- Port 5432 có thể bị Postgres cài local chiếm → đổi `ports: ["5433:5432"]` và `DATABASE_URL` tương ứng.
- `redis-server` trên Windows không chính thức — chỉ dùng qua Docker (hoặc Memurai nếu bắt buộc native).
- Chạy: `docker compose -f docker-compose.dev.yml up -d` → `pnpm db:migrate` → `pnpm db:seed` → `pnpm dev`. Reset: `docker compose -f docker-compose.dev.yml down -v`.
- Time zone: container + app đều UTC; UI convert theo `users.preferences.timezone`.
- `working_directory`, `local_path`, `cwd` lưu dạng forward-slash chuẩn hoá (`path.posix`), convert khi spawn process.

---

## 12. Các điểm cần user duyệt — tổng hợp

1. **ORM = Drizzle** (stable 0.45.x) thay vì Prisma (lý do mục 1); D-002 APPROVED (Owner, 2026-10-30). **Tác vụ bắt buộc trước khi khóa schema (chưa thực hiện):** spike 1 ngày cho bảng phân vùng `agent_events` + partial unique index bằng `drizzle-kit generate --custom` (chạy migrate hai lần) và FK vòng bằng SQL tay.
2. `task_status` giữ UPPERCASE theo spec trong khi enum khác lowercase — thống nhất hay giữ?
3. **pgvector** cho `agent_memories.embedding` (cần image `pgvector/pgvector:pg16`) — bật v1 hay để v2 (bỏ cột, dùng FTS).
4. **Transactional outbox** đầy đủ (server) hay publish-after-commit (desktop).
5. **RLS** ở Postgres từ v1 hay chỉ app-level scoping.
6. Desktop distribution: yêu cầu Docker Desktop hay bundle embedded Postgres/Redis.
7. Bảng phụ `event_sequences` cho aggregate không phải session (task/run/agent) — hoặc chỉ dùng session stream + `global_seq`.
8. Retention mặc định: events 90 ngày (desktop) / 365 ngày (server); usage không xoá. **[OWR-2 Q-06 ĐÃ DUYỆT cho audit: 365 ngày online + signed export 2 năm]**.
