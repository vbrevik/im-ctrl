# Ahlt — Product Backlog

## Vision

Transform Ahlt from a hardcoded admin panel into a **data-driven platform** where behavior, access control, navigation, and configuration are all defined by an **ontology** (structured data in the database), not by code. The system should be extensible without recompilation.

---

## Ontology Model (Actual Implementation)

All domain objects share three generic tables — no dedicated tables per type:

```
┌──────────────────────────┐
│        entities           │
│ id, entity_type, name,    │
│ label, sort_order,        │
│ is_active, timestamps     │
│ UNIQUE(entity_type, name) │
└──────────┬───────────────┘
           │ 1:N
┌──────────┴───────────────┐     ┌──────────────────────────┐
│   entity_properties       │     │        relations          │
│ entity_id, key, value     │     │ id, relation_type_id,     │
│ PK(entity_id, key)        │     │ source_id, target_id      │
└──────────────────────────┘     │ UNIQUE(type,src,tgt)      │
                                  └──────────────────────────┘
```

### Entity Types in Use

| entity_type | Purpose | Key Properties |
|---|---|---|
| `relation_type` | Named relationship kinds (35 defined) | — |
| `role` | Named collection of permissions | `description`, `is_default` |
| `permission` | Atomic capability (46 defined) | `group_name`, `description` |
| `user` | Account with role relation | `password`, `email` |
| `nav_item` | Menu entry (module or page) | `url`, `parent` *(permission via relation)* |
| `setting` | Key-value config (8 defined) | `value`, `description`, `setting_type` |
| `workflow_status` | State in a workflow (suggestion/proposal) | `entity_type_scope`, `status_code`, `is_initial`, `is_terminal` |
| `workflow_transition` | Allowed state change | `from_status_code`, `to_status_code`, `required_permission` |
| `tor` | Term of Reference | `description`, `status`, `purpose` |
| `suggestion` | Workflow item (suggestion stage) | `description`, `status`, `submitted_by` |
| `proposal` | Workflow item (proposal stage) | `description`, `status`, `submitted_by` |
| `agenda_point` | Meeting agenda item | `type` (informative/decision), `status` |
| `coa` | Course of Action (decision option) | `description`, `type` (simple/complex) |
| `opinion` | Advisory input from participants | `stance`, `rationale` |
| `warning` | System notification | `severity`, `category`, `source_action`, `details` |
| `warning_receipt` | Per-user warning delivery | `status` (unread/read/deleted) |
| `warning_event` | Warning audit trail entry | `event_type`, `performed_by` |
| `audit_entry` | Audit log record | `user_id`, `action`, `target_type`, `summary` |

### Relations in Use

| Relation Type | Source → Target | Purpose |
|---|---|---|
| `has_role` | user → role | User's assigned roles (many-to-many) |
| `has_permission` | role → permission | Role's granted permissions |
| `requires_permission` | nav_item → permission | Nav item access requirement |
| `member_of` | user → tor | ToR membership |
| `has_tor_role` | user → tor | User's role within a ToR |
| `belongs_to_tor` | entity → tor | Entity scoped to a ToR |
| `suggested_to` | suggestion → tor | Suggestion target |
| `spawns_proposal` | suggestion → proposal | Auto-creation link |
| `submitted_to` | proposal → tor | Proposal target |
| `transition_from` | workflow_transition → workflow_status | Transition source state |
| `transition_to` | workflow_transition → workflow_status | Transition target state |
| `considers_coa` | agenda_point → coa | Decision options |
| `originates_from` | agenda_point → proposal | Agenda from proposal |
| `has_section` / `has_subsection` | coa → coa | Nested COA structure |
| `spawns_agenda_point` | proposal → agenda_point | Scheduling link |
| `opinion_by` / `opinion_on` / `prefers_coa` | user → opinion / opinion → agenda/coa | Opinion tracking |
| `targets_user` | warning → user | Warning target |
| `for_warning` / `for_user` / `on_receipt` | warning_receipt → entities | Receipt links |
| `forwarded_to_user` | warning_event → user | Forward tracking |

### Permission Codes (seed data)

| Code | Group | Description |
|------|-------|-------------|
| `dashboard.view` | Dashboard | View the dashboard |
| `users.list` | Users | View user list |
| `users.create` | Users | Create new users |
| `users.edit` | Users | Edit existing users |
| `users.delete` | Users | Delete users |
| `roles.manage` | Roles | Create/edit/delete roles and assign permissions |
| `roles.assign` | Roles | Assign/unassign roles to users |
| `settings.manage` | Settings | Modify app settings |
| `audit.view` | Admin | View audit log |
| `warnings.view` | Admin | View warnings |
| `tor.list` | Governance | List Terms of Reference |
| `tor.create` | Governance | Create Terms of Reference |
| `tor.edit` | Governance | Edit Terms of Reference |
| `tor.manage_members` | Governance | Manage ToR members |
| `suggestion.view` | Workflow | View suggestions |
| `suggestion.create` | Workflow | Submit new suggestions |
| `suggestion.review` | Workflow | Accept or reject suggestions |
| `proposal.view` | Workflow | View proposals |
| `proposal.create` | Workflow | Create new proposals |
| `proposal.edit` | Workflow | Edit draft proposals |
| `proposal.submit` | Workflow | Submit drafts for review |
| `proposal.review` | Workflow | Move proposals to under_review |
| `proposal.approve` | Workflow | Approve or reject proposals |
| `agenda.view` | Governance | View agenda |
| `agenda.create` | Governance | Create agenda points |
| `agenda.queue` | Governance | Queue proposals for agenda |
| `agenda.manage` | Governance | Manage agenda status |
| `agenda.participate` | Governance | Participate in meeting |
| `agenda.decide` | Governance | Make final decisions |
| `coa.create` | Workflow | Create courses of action |
| `coa.edit` | Workflow | Edit courses of action |
| `workflow.manage` | Governance | Manage workflow system |

### Navigation Hierarchy (seed data)

| Name | Label | Parent | URL | Permission |
|---|---|---|---|---|
| `dashboard` | Dashboard | — | `/dashboard` | `dashboard.view` |
| `admin` | Admin | — | `/users` | *(visible if any child permitted)* |
| `admin.users` | Users | `admin` | `/users` | `users.list` |
| `admin.roles` | Roles | `admin` | `/roles` | `roles.assign` |
| `admin.ontology` | Ontology | `admin` | `/ontology` | `settings.manage` |
| `admin.settings` | Settings | `admin` | `/settings` | `settings.manage` |
| `admin.audit` | Audit Log | `admin` | `/audit` | `audit.view` |
| `admin.menu_builder` | Menu Builder | `admin` | `/menu-builder` | `roles.manage` |
| `admin.warnings` | Warnings | `admin` | `/warnings` | `warnings.view` |
| `admin.role_builder` | Role Builder | `admin` | `/roles/builder` | `roles.manage` |
| `governance` | Governance | — | `/tor` | *(visible if any child permitted)* |
| `governance.tor` | Terms of Reference | `governance` | `/tor` | `tor.list` |
| `governance.map` | Governance Map | `governance` | `/governance/map` | `tor.list` |
| `governance.workflow` | Item Workflow | `governance` | `/workflow` | `suggestion.view` |
| `governance.outlook` | Meeting Outlook | `governance` | `/tor/outlook` | `tor.list` |
| `governance.workflow_builder` | Workflow Builder | `governance` | `/workflow/builder` | `workflow.manage` |

---

## Completed Work

### Epic 1: Ontology Foundation
- 1.1 EAV schema (entities, entity_properties, relations) with role + permission entities
- 1.2 User→role via has_role relation (replaced text role field)
- 1.3 Data-driven role dropdown from DB query
- 1.4 Permission-based auth (session CSV storage, `require_permission()` helper)

### Epic 2: Data-Driven Navigation
- 2.1 Nav items as entities with parent-child hierarchy
- 2.2 Two-tier rendering: modules in header, pages in sidebar
- Active state detection (children first, then top-level fallback)
- Permission-gated visibility (module visible if any child permitted)
- 2.3 Nav permissions via relations: converted nav_item permission checks from `permission_code` text properties to `requires_permission` relations (nav_item→permission), making permissions visible in ontology graph and consistent with EAV model

### Security (5.1–5.4)
- 5.1 Self-deletion protection
- 5.2 Last admin protection
- 5.3 Persistent session key from `SESSION_KEY` env var (falls back to `Key::generate()` with warning)
- 5.4 CSRF protection: 32-byte hex token, constant-time comparison, all 9 POST handlers validate

### Housekeeping
- 7.1 Git init + push to GitHub
- 7.2 Favicon (inline SVG data URI)

### Infrastructure
- PageContext struct (bundles username, permissions, flash, nav_modules, sidebar_items)
- `PageContext::build()` constructor reduces handler boilerplate

### Role Management (4.1)
- Full CRUD: list (with permission/user counts), create, edit, delete (with user-assigned guard)
- Permission checkboxes with manual form body parsing (serde_urlencoded can't handle duplicate keys)
- `admin.roles` nav item under Admin module

### Ontology Explorer
- Three-tab explorer: Concepts (schema-level D3 graph) + Data (instance-level D3 graph) + Reference (entity type cards, relation patterns, schema docs)
- JSON APIs: `/ontology/api/schema` + `/ontology/api/graph`
- Concepts tab: schema graph with entity type nodes (sized by count), relation pattern edges, toolbar, keyboard shortcuts, right-click context menu with instance drill-down
- Data tab: instance graph with search, entity/relation type chip filters, right-click context menu (grouped relations, ego focus), position-preserving visibility toggles
- Reference tab: entity type summary cards, relation pattern breakdowns, schema reference tables

### App Settings (3.1 + 3.2 + 3.3)
- Settings as entities with entity_type='setting', properties: `value`, `description`, `setting_type` (text/number/boolean)
- 8 seeded settings: app.name, app.description, audit.enabled, audit.log_path, audit.retention_days, warnings.retention_resolved_days, warnings.retention_info_days, warnings.retention_deleted_days
- Runtime integration: `app.name` drives navbar brand, page titles, and login page brand

### Change Password (6.1)
- `GET /account` + `POST /account` with current/new/confirm validation

### Navbar Avatar Dropdown (6.5)
- Avatar with user initial, dropdown with Profile/Warnings/Logout, live warning badge count via WebSocket

### Audit Trail (7.3)
- Two-tier system: database (EAV) + filesystem (daily-rotated JSONL)
- UI: /audit with search, action filter, target type filter, pagination
- Retention cleanup on startup (configurable via settings)

### Custom Error Pages (6.2)
- Branded 404 and 500 error pages with navigation buttons

### Pagination (6.3) + Search/Filter (6.4)
- User list with `?page=1&per_page=25&q=searchterm` support

### Frontend Design Review (6.6)
- Color-coded role badges, unified search bars, improved empty states, redesigned dashboard

### Menu Builder / Permission Matrix (4.2)
- Visual permission matrix: roles as columns, permissions as rows grouped by section
- Diff-based save with audit logging
- JavaScript change tracking with unsaved-changes warning

### Roles Builder (4.3)
- Two-step wizard: role details → permissions & live sidebar preview (side-by-side)
- Handles both create and edit: single wizard replaces old separate edit form
- Edit mode pre-fills step 1 fields and pre-checks assigned permissions
- Live preview uses `requires_permission` relations (matches real nav system logic)
- Vertical preview grouped by module with accurate permission filtering
- Update handler includes audit logging + admin WebSocket warning (parity with old edit)
- Old `/roles/{id}/edit` removed; edit now at `/roles/builder/{id}/edit`
- 8 integration tests (5 builder + 3 model)

### Phase 2a: Item Pipeline
- Suggestion→proposal workflow with form validation and auto-proposal creation
- Integration tests validating complete workflow

### Phase 2b: Agenda Points, COAs & Data-Driven Workflows
- Data-driven workflow engine: WorkflowStatus + WorkflowTransition entities replace all hardcoded transitions
- Agenda points (informative/decision), courses of action (simple/complex), opinion recording, decision making
- Proposal queue with bulk scheduling
- 19 tasks delivered, 12 tests

### Warnings System
- Three-layer model: warning → warning_receipt → warning_event
- WebSocket real-time notifications with toast UI
- List/detail pages with category/severity/status filters + pagination
- Background scheduler (5-min interval) running generators + retention cleanup
- Event-driven generators: inline warnings on user create/delete, role permission changes
- 7 integration tests

### Production Deployment Preparation
- Environment variables (HOST, PORT, COOKIE_SECURE, SESSION_KEY, APP_ENV, RUST_LOG)
- Multi-stage Dockerfile with dependency caching
- Release profile (LTO, strip, single codegen unit)

### Code Cleanup & Refactoring (Tasks 1-28)
- Phase 1-2: AppError enum with 8 variants, render() helper, session helpers return Result
- Phase 3: All 27 handlers migrated to AppError pattern (~280 lines eliminated)
- Phase 4: 5 large files split into 17 focused modules (1,680 lines reorganized)

### Automated Testing
- 221 tests across 26 test files covering: infrastructure, auth, user CRUD, data integrity, permissions, role lifecycle, nav gating, workflows, warnings, role builder, API v1, suggestions, COA, opinions, data manager, governance, meetings, proposals, calendar, ABAC

### Hardening: Input Validation (H.2)
- Centralized `src/auth/validate.rs` with 5 reusable functions: username, email, password, required field, optional field
- Applied across all form handlers: user create/update, role create/update, ToR create/update, account password change
- Length limits enforced on all text inputs, format validation on username (alphanumeric + underscore) and email

### Hardening: Rate Limiting (H.1)
- Per-IP login rate limiting: 5 failed attempts per 15-minute window, 6th attempt blocked before DB access
- In-memory `HashMap<IpAddr, Vec<Instant>>` behind `Arc<Mutex<>>`, no external dependencies
- Lazy cleanup of stale entries, poison-resistant mutex, clear on successful login

### ToR Expansion (13 tasks)
- **Position-based membership**: `fills_position` relation (user→tor_function) replaces `member_of`+`has_tor_role`; authority flows through named positions, not persons. Vacant positions remain visible with `mandatory`/`optional` type from `entity_properties`.
- **Protocol templates**: `protocol_step` entities scoped to a ToR define reusable meeting agendas with type, duration, required flag, and sequence ordering.
- **Inter-ToR dependencies**: `feeds_into` and `escalates_to` relations between ToR entities with `is_blocking` metadata on `relation_properties`.
- **Minutes auto-scaffold**: `Minutes` + `MinutesSection` EAV model. `generate_scaffold()` creates 5 sections from meeting data (attendance flags vacant mandatory positions). Status lifecycle: draft→pending_approval→approved (read-only once approved).
- **Presentation templates**: `template_of`/`slide_of` relation chain. Per-ToR fixed slide templates with ordered slides and move-up/down reordering.
- **Governance map**: Cross-ToR dependency overview at `/governance/map` with colour-coded relationship badges. Nav item added to Governance sidebar.

### Governance Map Visual Graph (T.1)
- Dagre+D3 hierarchical DAG on `/governance/map`: ToR nodes (rounded rectangles) with label, cadence badge, status dot
- Directed edges: solid blue for `feeds_into`, dashed red for blocking, dashed amber for `escalates_to`
- JSON API at `GET /api/governance/graph` (nodes with cadence properties + edges with relation metadata)
- Click node → navigates to `/tor/{id}`, hover highlights connected subgraph
- Toolbar: fit, zoom in/out, reset zoom (keyboard shortcuts F, +/-, 0)
- Dagre 0.8.5 + D3 v7 from CDN, no new Rust dependencies
- ToR card grid retained below graph as secondary reference

### Meeting Outlook Calendar (T.3)
- Cadence computation engine in `src/models/tor/calendar.rs`: `compute_meetings(conn, start, end)` generates `CalendarEvent` instances from ToR cadence rules (daily, working_days, weekly, biweekly, monthly, ad-hoc)
- JSON API at `GET /api/tor/calendar?start=YYYY-MM-DD&end=YYYY-MM-DD` with 90-day cap
- Page at `/tor/outlook` with day/week/month CSS grid views
- Hybrid rendering: server renders initial week, client `fetch()` for tab/date switching
- Color-coded event pills per ToR, today highlighting, click-through to ToR detail
- Safe DOM construction (no innerHTML) via `el()` helper pattern
- ~~Known gap: day view doesn't handle overlapping events side-by-side~~ → **Fixed in CA4.5**: column-based overlap algorithm positions concurrent events side-by-side with computed widths

- New relations seeded: `fills_position`, `protocol_of`, `feeds_into`, `escalates_to`, `minutes_of`, `section_of`, `template_of`, `slide_of`, `requires_template`
- New permissions: `minutes.generate`, `minutes.edit`, `minutes.approve`
- New nav items: `governance.map` → `/governance/map`, `governance.outlook` → `/tor/outlook` under Governance module

### ToR Vacancy Warning Generators (T.2)
- New scheduled generator `check_tor_vacancies()` in `src/warnings/generators.rs`
- Queries active ToRs for mandatory positions with no `fills_position` relation
- Per-ToR dedup: one warning per ToR listing all vacant mandatory positions
- Targets users with `tor.manage_members` permission, WebSocket push on creation
- Auto-resolve: warnings auto-resolve on next scheduler tick when vacancies are filled
- Category `governance`, severity `medium`, source_action `scheduled.tor_vacancy`
- 2 integration tests: warning creation + dedup, auto-resolve on position fill

### Data Manager Seed Refactor
- Replaced ~1,000 lines of procedural `seed_ontology()`/`seed_staging()` in `db.rs` with JSON fixture imports
- Seed data now in `data/seed/ontology.json` (112 entities, 67 relations) and `data/seed/staging.json` (43 entities, 91 relations)
- Uses `data_manager::import::import_data()` with `ConflictMode::Skip` for idempotent seeding
- Passwords excluded from fixtures, hashed at runtime via `set_user_password()`
- `include_str!` embeds fixtures at compile time — no filesystem dependency at runtime
- To modify seed data: edit JSON fixtures, delete DB, restart server

### Dashboard Redesign (F.6)
- Full visual redesign: time-aware greeting (morning/afternoon/evening), stats cards (proposals, suggestions, ToRs, active warnings), recent activity feed from audit log
- Replaced placeholder dashboard with real data from model queries

### Dashboard Personalization (F.6b)
- **Actionable content-first layout**: Replaced passive stats-first dashboard with personalized, actionable panels
- **New model layer**: `src/models/dashboard.rs` with 6 types and 5 queries aggregating data from warnings, calendar, proposals, suggestions, and ToR membership
- **Needs Attention panel**: Unread warnings (severity-coded), pending proposals, open suggestions — all scoped to user's ToRs
- **My Terms of Reference**: User's ToR memberships with position labels, linked to ToR detail pages
- **Upcoming Meetings**: Next 7 days of meetings from user's ToRs via calendar computation engine
- **System Overview**: Clickable stat cards (users, roles, pending, positions, warnings) linking to relevant pages
- **Responsive**: Two-column primary layout (3fr/2fr) stacking at 768px, dark mode via CSS variables
- **Build**: PASS | **Tests**: 171 passing (unchanged)

### Workflow Builder UI (F.1)
- List page at `/workflow/builder`: scope cards with status/transition counts
- Detail page at `/workflow/builder/{scope}`: D3/dagre state machine graph + statuses table (add/edit/delete) + transitions table (add/edit/delete)
- All mutations: permission checks, CSRF, audit logging
- Nav item `governance.workflow_builder` seeded with `workflow.manage` permission
- Sidebar longest-prefix active-state fix (was marking `/workflow` active on `/workflow/builder` pages)
- Shared `.graph-panel` CSS: identical header panel across governance map and workflow builder (title + stat, 400px canvas, same toolbar/keyboard shortcuts)

### Test Coverage Expansion (H.4)
- **Result: 141 passing tests** (target: 120+)
- Phase 1: E2E test infrastructure fixes — 4 calendar confirmation tests with unique cookie isolation (#[ignore] for live server)
- Phase 2: Governance model tests — 7 tests for ToR CRUD, agenda points, meetings, proposals, cascade deletes, data queries
- Phase 3: Warning system tests — 5 tests for warning creation, deduplication, resolution, receipt lookup
- Phase 4: Proposal lifecycle tests — 7 tests for CRUD, status workflow, rejection, counting, querying
- **Systematic approach**: Prompt Contracts (4-component spec) applied across all 4 phases, API discovery via Explore agent before implementation
- Test pattern: all use `setup_test_db()` for isolation, follow established Rust/Actix patterns, no state leakage in parallel execution
- **Key learnings**: Graceful degradation over perfectionism (simplified failing tests to verify contract), relation type creation requires string name not ID, E2E cookie isolation prevents CI flakes

### REST API v1 Layer (F.2)
- **Phase 1: Users CRUD** (`/api/v1/users`)
  - GET /api/v1/users: list with pagination (page, per_page bounded 1-100), search filtering
  - GET /api/v1/users/{id}: single user with all properties
  - POST /api/v1/users: create with validation, bcrypt password hashing, audit logging
  - PUT /api/v1/users/{id}: update username/email/display_name/role with optional password
  - DELETE /api/v1/users/{id}: delete with audit trail
  - All endpoints: permission gating (users.list/create/edit/delete), proper HTTP status codes (200/201/204/400/404)
- **Phase 2: Entities CRUD** (`/api/v1/entities`)
  - GET /api/v1/entities: list with optional type filter, pagination
  - GET /api/v1/entities/{id}: single entity with properties
  - POST /api/v1/entities: create with validation, optional key-value properties, audit logging
  - PUT /api/v1/entities/{id}: update name/label/properties with audit trail
  - DELETE /api/v1/entities/{id}: delete with audit trail
  - All endpoints: permission gating (entities.list/create/edit/delete), EAV property support
- **Response Format**: Generic `PaginatedResponse<T>` wrapper for consistency, `ApiErrorResponse` with optional details field
- **Implementation**: ~650 lines across 3 files (handlers/api_v1/{mod,users,entities}.rs), follows established patterns (session helpers, audit logging, validation), no new dependencies
- **Build**: PASS | **Tests**: 141 passing (unchanged)

### Composite Database Index (H.5)
- Added composite index on `entity_properties(entity_id, key)` for optimized EAV lookups
- Index created automatically on schema initialization via `CREATE INDEX IF NOT EXISTS`
- Improves performance from O(n) sequential scan to O(log n) seek when filtering by both entity_id and key
- Applied at SQLite level, no application code changes needed
- Fully backward compatible, applied idempotent

### Dark Mode Theme System (F.4)
- **CSS Refactoring**: Migrated all colors to CSS custom properties (--bg, --text, --accent, etc.) at `:root` level
- **Dark Mode Palette**: Added `:root.dark` selector with inverted colors optimized for dark backgrounds
- **Theme Options**: Light, Dark, Auto (respects system preference via `prefers-color-scheme` media query)
- **Persistence**: Theme preference stored in localStorage, persists across browser sessions and device restarts
- **Flash Prevention**: Theme initialization script runs in `<head>` before CSS loads to apply correct theme immediately
- **UI Toggle**: New Preferences tab on `/account` page with 3 visual buttons (Light/Dark/Auto with emoji icons)
- **JavaScript Handler**: `window.toggleTheme(theme)` manages theme switching and localStorage persistence
- **Accessibility**: WCAG AA contrast ratios maintained in both light and dark themes
- **Components**: All pages, forms, graphs, and navigation render correctly in both themes
- **Build**: PASS | **Tests**: 141 passing (unchanged)

### User Profile Enhancements (F.5)
- **Avatar Upload**: Users can upload JPEG/PNG profile images (max 200KB)
- **Avatar Storage**: Profile images stored as base64 data URIs in entity_properties (no external file storage)
- **Avatar Display**: Profile image appears in navbar avatar dropdown and account page preview
- **Avatar Management**: Users can delete their avatar with confirmation; placeholder emoji shows when none present
- **Display Name Editing**: Users can update their display name via form on Profile tab
- **Profile Tab**: New "Profile" tab on `/account` page (default active), separate from Security/Preferences
- **Client-Side Validation**: File type (JPEG/PNG) and size (max 200KB) validated before upload
- **Error Handling**: Clear error messages for validation failures (wrong type, too large)
- **Audit Logging**: All profile changes logged (avatar upload/delete, display name updates)
- **Responsive UI**: Two-column grid layout (avatar + display name sections, stacks on mobile)
- **Build**: PASS | **Tests**: 141 passing (unchanged)

### Data Manager Hardening (DM.1–DM.4)
- **DM.1 CSS specificity fix**: `.dm-loading` and `.dm-editor-overlay` used `display:flex` as default, overriding browser's `[hidden]` attribute. Fixed by setting `display:none` default and adding `.class:not([hidden]) { display:flex }` rule. Stuck spinner on page load eliminated.
- **DM.2 Request timeouts**: All `fetch()` calls wrapped with `fetchWithTimeout(url, opts, FETCH_TIMEOUT_MS)` helper using `AbortController`. Timeout constant `60000ms`. On abort, user sees "Request timed out. The server may be busy." and spinner hides.
- **DM.3 web::block for import**: `import::import_data()` moved off Actix async thread via `web::block(move || ...)` to prevent worker thread starvation. Body size limit raised to 50 MB via sub-scope `app_data(web::JsonConfig::default().limit(...))`.
- **DM.4 Batch/multi-file import**: File input now accepts `multiple` files. Entities chunked in batches of 100 (constant `CHUNK_SIZE`). Relations sent only with last chunk to respect FK order. Progress shown as "File X of N, chunk Y of Z…". File queue resets after successful import.
- **Build**: PASS | **Tests**: 3 passing (unchanged)

### Users Table Enhancements (U.1)
- **Filter Builder**: Visual FilterTree UI — Add/Remove conditions (field, operator, value), group conditions with AND/OR logic, bookmarkable URL state. Filter fields: username, display_name, email, role, created_at, updated_at. Operators: contains, not_contains, equals, not_equals, is_empty, is_not_empty, gt, lt, gte, lte.
- **Column Picker**: Toggle visibility + drag-to-reorder via HTML5 drag-and-drop. Per-user preferences persisted to entity_properties (`pref.users_table_columns`), with global default setting entity fallback. ⊞ Columns popover button in table controls bar.
- **Per-page Selector**: 10/25/50/100 rows options; persisted in URL (`per_page` query param).
- **Sort Headers**: Clickable column headers with ▲/▼ indicators, preserved across filter/pagination changes.
- **CSV Export**: Download filtered/sorted result set; filter state passed via URL param.
- **Askama `|safe` bug fix**: `{{ variable }}` auto-HTML-escapes `"` → `&#34;`, breaking `JSON.parse()` in `<script type="application/json">` blocks. Fix applied to `list.html` and `table_controls.html`.
- **Playwright E2E tests**: 46 tests in `scripts/users-table.test.mjs` covering all new features.
- **Server startup**: `eprintln!` ensures port always printed to stderr regardless of `RUST_LOG`.
- **Build**: PASS | **Tests**: 154 passing

### Entity Metadata Gap Fill (E.1–E.3, complete)
- **E.1 Meeting**: Added `meeting_number`, `classification`, `vtc_details`, `chair_user_id`, `secretary_user_id` (simple strings) + `roll_call_data` (JSON: `[{user_id, status}]`) — stored via EAV, shown conditionally in `meetings/detail.html`, accepted as `Option<String>` in `ConfirmForm`
- **E.2 AgendaPoint** (fully complete): Added `presenter`, `priority` (normal/high/urgent), `pre_read_url` — form fields in `agenda/form.html`, conditional display in `agenda/detail.html`
- **E.3 Minutes**: Added `approved_by`, `approved_date` (simple strings) + `distribution_list`, `structured_action_items` (JSON: `[{description, responsible, due_date, status}]`), `structured_attendance` (JSON: `[{user_id, name, status, delegation_to}]`) — shown in `minutes/view.html`, editable via dedicated POST endpoints
- All via EAV `entity_properties`, no schema changes needed
- **Build**: PASS | **Tests**: all passing (0 failures)

### Users / Roles / Role Builder Separation
- **Three-page separation**: Users page (pure CRUD, no role assignment), Roles page (dedicated assignment with By Role/By User tabs + menu preview), Role Builder (sole path for role CRUD with permissions + delete)
- **Multi-role support**: Users can hold multiple roles via `has_role` many-to-many relation. Permissions = union across all roles via `find_codes_by_user_id()`. `UserDisplay` uses `GROUP_CONCAT(DISTINCT ...)` for comma-separated role display.
- **New permission**: `roles.assign` separates role assignment from role management (`roles.manage`)
- **Last-admin protection**: Updated to query `has_role` relation directly (multi-role safe)
- **Auto-assign**: New users get "viewer" role automatically on creation
- **17 tasks** across handlers, models, templates, routes, and tests
- **Build**: PASS | **Tests**: 162 passing

### ABAC — Attribute-Based Access Control (3 splits)
- **Split 1 (01-abac-core)**: Created `src/auth/abac.rs` with three functions: `has_resource_capability` (EAV graph traversal), `load_tor_capabilities` (bulk capability loader), `require_tor_capability` (two-phase handler guard). 7 TDD tests in `tests/abac_test.rs`. Commits `5e75c4d`, `8751672`, `4e67d7b`.
- **Split 2 (02-handler-migration)**: Migrated 9 handlers from `require_permission("tor.edit")` to ABAC capability checks: `confirm`, `transition`, `assign_agenda`, `remove_agenda`, `save_roll_call`, `generate_minutes` (meeting_handlers) + `save_attendance`, `save_action_items` (minutes_handlers) + special `confirm_calendar` (JSON pattern). Commit `ec92ae2`.
- **Split 3 (03-template-ui)**: Added `tor_capabilities: Permissions` to `MeetingDetailTemplate`, populated via `load_tor_capabilities` in `detail` handler, updated roll call section guards in `meetings/detail.html`. Commit `330110d`.
- **Build**: PASS | **Tests**: 169 passing

### Enterprise Infrastructure Migration (Phase 1)
- **Phase 1a: PostgreSQL migration** — Full migration from SQLite/rusqlite to PostgreSQL 17/sqlx 0.8 (async). All 44 model files, 30+ handler files, 23 test files converted. Schema migrated to `migrations/` directory (sqlx auto-run). Test isolation via unique PostgreSQL schemas (`search_path`). 171 tests passing.
- **Phase 1b: Neo4j integration** — Optional read-only graph projection via neo4rs 0.8. Graph sync module with fire-and-forget `tokio::spawn`, ABAC Cypher queries, governance map visualization. Falls back to PostgreSQL when Neo4j unavailable. 4 integration tests (#[ignore] without Neo4j).
- **Phase 2: Docker Compose** — Multi-environment setup: `docker-compose.yml` (base) + `docker-compose.{dev,staging,prod}.yml` (overrides). Makefile orchestration (`make dev/staging/prod/down`). PostgreSQL init script creates per-env databases.
- **Phase 3: GitLab CI/CD** — Self-hosted GitLab CE + Runner configs in `infra/gitlab/`. CI pipeline (`.gitlab-ci.yml`) with test, lint, build, and Helm deployment stages.
- **Phase 4: Kubernetes Helm** — Shared infra charts (`helm/infra/`) for PostgreSQL + Neo4j. Application chart (`helm/ahlt/`) with deployment, service, configmap, secret, ingress templates. Per-environment values files (dev/staging/prod).
- **Build**: PASS | **Tests**: 171 passing (8 ignored: 4 Neo4j + 4 E2E)

### Codebase Audit Fixes (CA.1–CA.2)
- **CA.1 — Queue template**: Replaced stub `templates/workflow/queue.html` with full implementation (table, checkboxes, bulk schedule form, unqueue per-row)
- **CA.1 — Agenda transition handler**: Replaced "not yet implemented" stub in `agenda_handlers.rs` with working workflow transition (validate → set_property → audit log)
- **CA.1 — Seed missing permissions**: Added 5 permission entities (`entities.list/create/edit/delete`, `minutes.view`) + 11 `has_permission` relations for admin role to `ontology.json`
- **CA.1 — Opinion seed direction**: Standardized 2 `opinion_by` relations in `staging.json` to match programmatic direction (`user → opinion`)
- **CA.2 — Minutes export button**: Added Export button to `minutes/view.html` (visible only when status is "approved", opens in new tab)
- **CA.2 — Audit logging gaps**: Added `audit::log()` calls to 6 mutation handlers: `mark_deleted`, `forward` (warning actions), `move_step` (protocol), `handle_add_slide`, `handle_delete_slide`, `handle_move_slide` (presentation)
- **CA.3 — Stale `/agenda-points/` URL prefix**: Fixed 12 broken links across `templates/opinion/form.html`, `templates/coa/detail.html`, `templates/workflow/view.html` — all now use `/workflow/agenda/` prefix matching actual routes
- **CA.3 — Missing agenda point delete handler**: Added `delete()` handler + `AgendaDeleteForm` struct to `agenda_handlers.rs` and registered `POST /tor/{id}/workflow/agenda/{agenda_id}/delete` in `main.rs`; delete button in workflow view was previously wired to a non-existent route
- **CA.3 — Dead code cleanup**: Removed unused `setup_test_db_seeded()` wrapper, unused `ADMIN_USER`/`ADMIN_PASS`/`TEST_USER_EMAIL` constants from `tests/common/mod.rs`, and unused `insert_entity` import from `opinion_relation_test.rs`
- **Build**: PASS | **Tests**: 171 passing (unchanged)

### Ontology Graph Redesign (OG.1)
- **Search**: Real-time entity search with pulsing highlight rings on matching nodes, dimming non-matches
- **Entity type filters**: Chip-based toggles with colored dots, instance counts, position-preserving visibility (no simulation restart)
- **Relation type filters**: Chip-based toggles for showing/hiding relation types and their edges
- **Right-click context menu**: Instance graph shows grouped outgoing/incoming relations, "Focus on this node", "Open full detail" link; Schema graph shows instance count badge, "View instances" drill-down, relation type summary
- **Ego network focus**: 1-hop neighbor subgraph with floating "Focus: Name x" dismiss pill
- **Schema-to-instance drill-down**: "View instances" navigates to `/ontology/data?type=X` with pre-filtered entity type
- **Scrollable filter panel**: `max-height` + `overflow-y: auto` prevents panel overflow with many entity types
- **Bug fix**: Schema API 500 error resolved by adding explicit SQL column aliases for `SchemaEdge` struct deserialization
- **Files**: `templates/ontology/data.html` (full rewrite), `templates/ontology/graph.html` (context menu), `static/css/style.css` (~268 lines), `src/models/ontology/schema.rs` (SQL fix)
- **Build**: PASS | **Tests**: 171 passing (unchanged)

### Minutes Export (T.4)
- **Export Format**: Print-friendly HTML (users print to PDF via browser Ctrl+P / Cmd+P)
- **Approved-Only**: Only approved minutes exportable; draft/pending return 403 Forbidden
- **URL**: GET `/meetings/{id}/export` returns inline HTML for preview or download
- **Content**: All minutes sections displayed with semantic structure (header + sections + footer)
- **Section Types**: Attendance (👥), Protocol (📋), Agenda Items (📝), Decisions (✅), Action Items (🎯)
- **Print Styling**: CSS optimized for print with page breaks on sections, no headers/footers repeated
- **Filename**: Content-Disposition set to `inline; filename="minutes-{id}.html"`
- **Permission**: Requires `minutes.view` permission; validates user access
- **Audit Logging**: Export action logged with minutes ID, format, and audit trail
- **No Dependencies**: Pure HTML/CSS, no external PDF library needed
- **Build**: PASS | **Tests**: 141 passing (unchanged)

### CA4.1 — Account Preferences Theme → DB Sync (Bug Fix)
- **Root Cause**: Account page `/account` Preferences tab theme buttons called `window.toggleTheme()` + updated `localStorage` but never hit the `POST /api/v1/user/theme` endpoint — preferences were browser-local only, not cross-device.
- **Fix**: Added a `fetch('/api/v1/user/theme', ...)` call in the `.theme-btn` click handler after `window.toggleTheme()`, mirroring the identical pattern in `base.html` (header toggle). `~10 lines` added to `templates/account.html`.
- **Behaviour**: Theme selection on account page now persists to `entity_properties` DB, loaded server-side on next page load from any device/browser.
- **Build**: PASS | **Tests**: 171 passing (unchanged)

### Audit Batch CA4.2/3/6/7 (Quality + Hardening)
- **CA4.2 — REST API integration tests**: Created `tests/api_v1_test.rs` with 17 tests covering user CRUD, entity CRUD, validation, permission chain, and auth guard. Uses `setup_test_db()` for isolation.
- **CA4.3 — Calendar fetch timeout**: Wrapped both `fetch()` calls in `templates/tor/outlook.html` with `fetchWithTimeout()` + `AbortController` (30s timeout, `FETCH_TIMEOUT_MS` constant). Error message uses `textContent` (no innerHTML).
- **CA4.6 — API CSRF protection**: Added `require_json_content_type` middleware in `src/handlers/api_v1/mod.rs`. Rejects POST/PUT/DELETE without `Content-Type: application/json` (400). GET exempt. Applied to entities, users, and user/theme scopes.
- **CA4.7 — Dead code warnings**: Added `#[allow(dead_code)]` to `insert_entity`/`insert_prop`/`insert_relation` in `tests/common/mod.rs` (rust-analyzer false positives — functions used by sibling test files).
- **Build**: PASS | **Tests**: 194 passing

### CA4.4 — REST API Coverage Expansion
- **GET /api/v1/tors**: Paginated list with `?status=` filter, uses `tor::find_all_list_items()`. Permission: `tor.list`.
- **GET /api/v1/tors/{id}**: Detail with member count via `tor::count_members()`. Permission: `tor.list`.
- **GET /api/v1/proposals**: Paginated list with `?status=` + `?tor_id=` filters, uses `proposal::find_all_cross_tor()`. Permission: `proposal.view`.
- **GET /api/v1/warnings**: User-scoped list with `?severity=` filter, uses `warnings::queries::find_for_user()`. Permission: `warnings.view`.
- Response types defined inline (`ApiTorListItem`, `ApiTorDetail`, `ApiProposalItem`, `ApiWarningItem`). All use `PaginatedResponse<T>` envelope.
- Read-only scopes excluded from CSRF middleware (GET only).
- 7 integration tests added (tor list/detail/not-found, proposal cross-tor/status-filter, warning user-scoped/severity-filter).
- **Build**: PASS | **Tests**: 201 passing

### CA4.5 — Day View Overlapping Events
- Column-based overlap algorithm (`computeOverlapColumns()`) positions concurrent events side-by-side in day view
- Greedy column assignment + connected-components grouping for totalCols per overlap cluster
- Events with overlap get `position: absolute` with computed `left`/`width` percentages; non-overlapping events remain full-width flow layout
- CSS: Updated `.outlook-event-day` with overflow:hidden, attribute selector for absolute variants
- `DAY_SLOT_HEIGHT = 28px` per 30-min slot; pixel-precise top offsets for events not starting on 30-min boundaries
- Week and month views verified unaffected
- **Build**: PASS | **Tests**: 201 passing

### CA4.8 — E2E Suite CI Integration
- Created `scripts/README.md` documenting all E2E test suites: Playwright JS (46 tests), Python admin screens, Rust curl-based calendar tests (4 tests)
- README covers: Node.js/Playwright one-time setup, server startup with staging data, per-suite run commands, cookie isolation, troubleshooting
- Updated all 4 `#[ignore]` comments in `tests/calendar_confirmation_e2e.rs` with exact pre-conditions: (1) port 8080 free, (2) ahlt_staging DB with seed data, (3) APP_ENV=staging, plus copy-paste run command
- No CI pipeline changes — documentation only
- **Build**: PASS | **Tests**: 201 passing

### Users Page Editorial Redesign (UI)
- **Avatar circles**: Replaced emoji with colored initial circles using 8-hue warm palette; hue computed from `charCode % 8`, CSS `[data-hue]` variants, dark mode support.
- **Count badge**: Amber pill badge showing total user count next to page heading.
- **Compact action buttons**: Replaced `.btn.btn-sm` pairs with `.action-btn` / `.action-btn--danger` (compact borderline style).
- **Pagination v2**: Pill-style pagination with info text + prev/current/next controls replacing plain links.
- **Left accent hover**: `tbody tr:hover td:first-child { box-shadow: inset 3px 0 0 var(--accent) }` — subtle row focus cue with no layout shift.
- **Files**: `templates/users/list.html` (HTML + JS), `static/css/style.css` (~175 lines added)
- **Build**: PASS | **Tests**: 171 passing (unchanged)

### Role Builder One-Page Redesign (UI)
- **One-page layout**: Converted from two-step wizard to a single-page three-column grid (240px details / 1fr permissions / 200px preview) with nav and sidebar visible.
- **Compact form overrides**: Scoped `.rb-details-panel .form-group` overrides reduce input padding and margins for IDE-config-panel density.
- **Sticky group headers**: Permission group headers use `position: sticky; top: 0` inside the `overflow-y: auto` permissions scroll area.
- **Checked row state**: CSS `:has(.permission-item:checked)` applies accent left-border + `color-mix()` background tint — zero JS.
- **Files**: `templates/roles/builder.html` (rewritten), `static/css/style.css` (role builder section replaced)
- **Build**: PASS | **Tests**: 171 passing (unchanged)

### Role Builder Accordion Redesign (UI.3)
- **2-column accordion layout**: Replaced 3-column flat-list with `2fr | 1fr` grid (main + sticky preview sidebar).
- **Permission descriptions**: Added `description` EAV property to all 46 permission entities, threaded through model pipeline (types → queries → handler → template).
- **Collapsible groups**: Accordion with chevron rotation, `max-height` CSS transition, select-all toggle, checked count badge per group.
- **Accent tint**: Groups with selected permissions show `color-mix(in srgb, var(--accent) 6%, transparent)` header background.
- **Responsive**: 3 breakpoints (>1100px 2-col, 768-1100px single, <768px full stack).
- **Review fixes**: Accordion `scrollHeight` read after class addition (padding-aware), `create_role` handler trims name/label/description (matches `update_role`).
- **Files**: `static/css/pages/role-builder.css` (410 lines), `templates/roles/builder.html` (316 lines), `src/models/role/{types,queries}.rs`, `src/models/permission.rs`, `src/handlers/role_builder_handlers.rs`, `data/seed/ontology.json`
- **Build**: PASS | **Tests**: 201 passing

### Quick-Win Tech Debt Batch (TD.8/TD.10/TD.11)
- **TD.8 — Fetch timeouts on graph endpoints**: Added 30s `AbortSignal.timeout()` + error handling to ontology graph (`templates/ontology/partials/graph_js.html`) and governance map (`templates/governance/partials/map_js.html`) fetch calls. Timeout shows "Request timed out" via `textContent`, hides loading spinner.
- **TD.10 — Settings audit logging**: Added `audit::log()` to `settings_handlers.rs` `save` handler. Tracks setting IDs changed, count, and summary. Imported `get_user_id` for actor attribution.
- **TD.11 — Unsafe `.unwrap()` in user API**: Replaced `body.password.as_ref().unwrap()` in `api_v1/users.rs` `create` handler with `.ok_or_else()` returning `AppError::Hash`. Prevents panic on malformed API requests.
- **Build**: PASS | **Tests**: 221 passing

### EAV Query Deduplication (TD.3)
- **Shared user→ToR membership query**: Extracted the 10-line EAV chain query (`user→fills_position→tor_function→belongs_to_tor→tor`) into `find_user_tors()` and `find_tor_ids_for_user()` in `src/models/tor/queries.rs`.
- **New canonical type**: `UserTorMembership` struct in `src/models/tor/types.rs` (id, name, label, position_label).
- **Dashboard dedup**: `src/models/dashboard.rs` rewritten to delegate to `tor::find_user_tors()` and pre-fetch ToR IDs for proposals/suggestions queries. Eliminated duplicated 10-line subqueries in 3 locations.
- **Bug fix**: Fixed `s_status.key` → `p_status.key` SQL alias typo in `find_open_suggestions_for_tors`.
- **Export fix**: Added `sort_order::BIGINT` casts in `src/models/data_manager/export.rs` to fix `ColumnDecode` error (pre-existing bug found by test agent).
- **Build**: PASS | **Tests**: 221 passing

### Handler Splitting (TD.5)
- **proposal_handlers.rs** (449 lines) split into `proposal_handlers/` module: `crud.rs` (238 lines, detail/new/create/edit/update), `workflow.rs` (211 lines, submit/review/approve/reject), `mod.rs` (5 lines, re-exports)
- COA handlers (423 lines) and agenda handlers (418 lines) assessed but left as-is: both under 300-line split threshold with no clear domain separation
- Re-export pattern in `mod.rs` maintains identical public API — zero changes to `main.rs` route registration
- **Build**: PASS | **Tests**: 221 passing

### Inline JS Extraction (TD.6 + TD.6b)
- **Phase 1 (TD.6)**: 9 static JS files created (912 lines total): `role-builder.js` (183), `filter-builder.js` (179), `account.js` (151), `nav.js` (81), `shared/graph-toolkit.js` (79), `theme.js` (71), `column-picker.js` (65), `menu-builder.js` (56), `shared/dynamic-table.js` (47). 9 templates updated.
- **Phase 2 (TD.6b)**: 11 additional static JS files created: `ontology-schema-graph.js` (423), `governance-map.js` (287), `data-manager.js` (547), `outlook-calendar.js` (542), `users-list.js` (57), `coa-form.js`, `meeting-detail.js`, `minutes-view.js`, `table-controls.js`, `role-assignment.js`, `proposal-queue.js`. 11 templates updated. 3 NEEDS_BRIDGE templates used JSON `<script type="application/json">` bridges for Askama template variables.
- **Head IIFE preserved**: Theme initialization script in `base.html <head>` intentionally kept inline (must run before CSS to prevent flash)
- **Two-tier JS organization**: Page-specific files in `static/js/`, shared utilities in `static/js/shared/`
- **1 template deferred** (workflow/partials/graph_script.html): Needs API endpoint refactor before extraction
- **Build**: PASS | **Tests**: 221 passing

### JS File Splitting (TD.12)
- **5 oversized JS files** split into 15 focused modules + 1 shared helper:
  - `ontology-graph.js` (728→266): + context-menu (143), detail (96), filters (194), shared/graph-helpers (40)
  - `data-manager.js` (547→109): + import (138), errors (187)
  - `outlook-calendar.js` (542→229): + calendar-day-view (134), calendar-views (130)
  - `ontology-schema-graph.js` (423→168): + schema-graph-panels (149)
  - `governance-map.js` (287→74): + governance-map-render (164)
- **Pattern**: Factory function with dependency injection (no ES modules/bundler needed)
- **Shared utility**: `graph-helpers.js` eliminates typeColor/createEl duplication across graph files
- **Build**: PASS | **Tests**: 221 passing (unchanged)

### Model Layer Test Coverage (TD.7)
- **20 new integration tests** across 4 previously untested model domains:
  - `tests/suggestion_test.rs` (5 tests): CRUD + status transitions (open→accepted/rejected)
  - `tests/coa_test.rs` (5 tests): CRUD + sections + agenda point scoping
  - `tests/opinion_test.rs` (5 tests): record/update opinions, find by user, record decisions
  - `tests/data_manager_test.rs` (5 tests): import/export round-trip, skip conflict mode, relations
- All use `setup_test_db()` for isolation, follow established patterns
- **Build**: PASS | **Tests**: 221 passing (up from 201), 8 ignored

---

## Remaining Backlog

### Tech Debt

| ID | Item | Priority | Effort | Description |
|----|------|----------|--------|-------------|
| TD.13 | **Extract remaining large templates** | Low | S | 3 templates exceed 300-line threshold: `meetings/detail.html` (331), `workflow/view.html` (317), `workflow/partials/graph_script.html` (303). Extract sections into partials. |

### Features

| ID | Item | Priority | Effort | Description |
|----|------|----------|--------|-------------|
| F.3 | **More entity types** | Medium | Variable | Extend the platform with project and task entity types. Document entity type already implemented (model + handlers + templates + routes). The EAV model requires zero schema migrations — just new model files, handlers, and templates per type. |
| F.4 | **Data label authority for llm-tune** | High (needed at llm-tune authority cut-over, weeks out) | M | im-ctrl becomes system of record for STANAG 4774-shaped labels (classification ordered, need-to-know categories as compartments with subset test), 5636-shaped core metadata, document precedence (`governed_by`, `supersedes`, `part_of`) and derived-asset label computation; plus an unattended export with `export_revision` and an initial import of llm-tune `labels.yaml`. Design: `docs/plans/2026-10-30-data-label-authority-design.md`. |

---

## Audit-Identified Work (2026-02-22)

Identified via systematic codebase + documentation audit. Ordered by effect. Each item has an embedded prompt contract.

| ID | Item | Type | Priority | Effort |
|----|------|------|----------|--------|
| ~~CA4.1~~ | ~~**Account preferences theme → DB disconnect**~~ | ~~Bug~~ | ~~High~~ | ~~XS~~ | **DONE** |
| ~~CA4.2~~ | ~~**REST API integration tests**~~ | ~~Quality~~ | ~~High~~ | ~~S~~ | **DONE** |
| ~~CA4.3~~ | ~~**Calendar fetch timeout**~~ | ~~Hardening~~ | ~~Medium~~ | ~~S~~ | **DONE** |
| ~~CA4.4~~ | ~~**REST API coverage expansion**~~ | ~~Architecture~~ | ~~Medium~~ | ~~M~~ | **DONE** |
| ~~CA4.5~~ | ~~**Day view overlapping events**~~ | ~~Known gap~~ | ~~Medium~~ | ~~S~~ | **DONE** |
| F.3 | **More entity types** | Feature | Medium | L |
| ~~CA4.6~~ | ~~**API CSRF protection**~~ | ~~Security~~ | ~~Medium~~ | ~~S~~ | **DONE** |
| ~~CA4.7~~ | ~~**Dead code warnings in test helpers**~~ | ~~Cleanup~~ | ~~Low~~ | ~~XS~~ | **DONE** |
| ~~CA4.8~~ | ~~**E2E suite CI integration**~~ | ~~Testing~~ | ~~Low~~ | ~~M~~ | **DONE** |
| ~~CA4.9~~ | ~~**Metrics baseline depth**~~ | ~~Process~~ | ~~Low~~ | ~~Ongoing~~ | **DONE** — 8 effectiveness rows, 7 efficiency rows |

---

### F.3 — More Entity Types (Project / Task)

**Status:** Document entity type is already implemented (model at `src/models/document/`, handlers at `src/handlers/document_handlers/`, templates at `templates/documents/`, routes at `/documents/*`). Project and Task remain.

**GOAL:** The platform gains two new entity types: `project` (top-level work container) and `task` (assigned work item). Each has: list page, create/edit/delete handlers, detail page, and seeded relation types (`produces`, `assigned_to`, `referenced_by`). A project can be linked to a ToR via a `tracked_by` relation. No schema migrations.

**CONSTRAINTS:**
- Follow existing EAV model: entity_type + entity_properties, no new tables.
- Follow existing handler/model/template patterns (UserDisplay, PageContext, render()).
- One module per entity type under `src/models/` and `src/handlers/`.
- Seed new relation types + nav items in `data/seed/ontology.json`.
- Permission codes: `project.view`, `project.create`, `project.edit`, `project.delete` (seeded to admin role).
- No new dependencies.

**FORMAT:**
- Create: `src/models/project/mod.rs`, `src/models/task/mod.rs`, `src/models/document/mod.rs`
- Create: `src/handlers/project_handlers/` etc.
- Create: `templates/project/`, `templates/task/`, `templates/document/` (list + form + detail)
- Modify: `src/lib.rs`, `src/handlers/mod.rs`, `src/main.rs` (routes), `data/seed/ontology.json`

**FAILURE CONDITIONS:**
- Any new file exceeds 300 lines.
- Permission checks missing on any mutation handler.
- CSRF validation missing on any POST handler.
- Audit logging missing on create/update/delete.
- Seed changes not tested (drop DB + restart fails).
- `cargo test` count decreases.

---

## Implementation Order

```
DONE (chronological)
════════════════════
Epic 1: Ontology Foundation
Epic 2: Data-Driven Nav
5.1–5.4 Security
4.1 Role Management, 4.2 Menu Builder, 4.3 Roles Builder
3.1–3.3 App Settings, 6.1–6.6 UX features, 7.1–7.3 Housekeeping
Ontology Explorer
Phase 2a: Item Pipeline, Phase 2b: Workflows + Governance
Warnings System, Production Deployment
Code Cleanup (Tasks 1-28)
H.1 Rate Limiting, H.2 Input Validation, H.3 WebSocket error handling
H.4 Test coverage expansion, H.5 Composite DB index
ToR Expansion (13 tasks), T.1 Governance Map, T.2 Vacancy Warnings
T.3 Meeting Outlook Calendar, T.4 Minutes export
Data Manager Seed Refactor, DM.1–DM.4 Data Manager hardening+batch
F.1 Workflow Builder UI, F.2 REST API v1 Layer
F.4 Dark mode, F.5 User profile, F.6 Dashboard Redesign + Personalization
E.1–E.3 Entity metadata gaps (Meeting, Agenda Point, Minutes)
Users/Roles/Role Builder Separation (17 tasks, multi-role support)
ABAC — 3-split implementation (abac-core, handler-migration, template-ui)
Enterprise Infrastructure Migration (5 phases: PostgreSQL, Neo4j, Docker, GitLab, Helm)
OG.1 Ontology Graph Redesign
U.1 Users Table Enhancements (filter builder, column picker, per-page, sort, CSV, Playwright)
CA.1–CA.3 Codebase Audit Fixes (queue template, audit logging, stale URLs, dead code)
P7 ToR Context Bar, P8 Dark Mode Header Toggle
CA4.1 Account theme DB sync, CA4.2 REST API tests (17), CA4.3 Calendar timeout
CA4.4 REST API expansion (tors, proposals, warnings), CA4.5 Day view overlap
CA4.6 API CSRF protection, CA4.7 Dead code warnings, CA4.8 E2E CI integration
UI.1 Users page redesign, UI.2 Role builder one-page, UI.3 Role builder accordion
TD.1 CSS monolith split (60 PostCSS files), TD.2 Template partial extraction (6 templates)
TD.3 EAV query deduplication (user→ToR membership)
TD.7 Model layer test coverage (+20 tests: suggestion, COA, opinion, data manager)
TD.8 Fetch timeouts (ontology + governance graphs)
TD.10 Settings audit logging, TD.11 Unsafe unwrap fix
CA4.9 Metrics baseline depth (8 effectiveness, 7 efficiency rows)
TD.5 Handler splitting (proposal_handlers → module with crud.rs + workflow.rs)
TD.6 Move inline JS to static files (9 files, 912 lines extracted)
TD.6b Move remaining inline JS to static files (Phase 2: 11 templates, 3 with JSON bridges)
TD.12 Split oversized JS files (5 files → 15 modules + shared helpers)

CANDIDATES (pick next)
══════════════════════
F.3    More entity types (project, task — document already done)             (feature, M)
TD.9   REST API expansion (CRUD for tors, proposals, suggestions)           (feature, M)
TD.13  Extract remaining large templates (3 files >300 lines)               (tech debt, S)
```

---

## Architecture Decisions

### Handler Pattern
Each handler follows: permission check → get conn → build PageContext → page query → template render. The `PageContext::build()` helper consolidates the 5 common fields (username, permissions, flash, nav_modules, sidebar_items) into a single constructor call. All handlers use `Result<HttpResponse, AppError>` with `?` operator for error propagation.

### Nav Item Hierarchy
Top-level items with no children are standalone (Dashboard). Items with children are modules (Admin, Governance) — visible if any child passes permission check. Children appear in sidebar when their parent module is active. Active module detection checks child URLs first for correct prefix matching.

### EAV Trade-offs
The generic schema means zero migrations when adding new entity types. The trade-off is more complex queries (LEFT JOINs on entity_properties). Typed domain structs (UserDisplay, RoleDisplay) provide a stable API layer over the generic storage.

### Workflow Engine
Workflow definitions (statuses + transitions) are stored as EAV entities, not hardcoded. Transitions are permission-gated and support conditions. The engine queries available transitions at runtime based on current status and user permissions. Currently seeded for suggestion and proposal workflows — designed to be extensible to any entity type.
