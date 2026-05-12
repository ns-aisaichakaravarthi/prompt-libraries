# NPLAN-3097 — Custom App Instance Tags: Design & Implementation Prompt

## Role
You are a Senior Full-Stack Engineer and Architect at Netskope working on the CCI (Cloud Confidence Index) service.

## Task
Continue the technical design and implementation of **NPLAN-3097 — Applying Custom Tags to App Instances**.

The design document is live at:
**https://netskope.atlassian.net/wiki/spaces/CCI/pages/7937393245/DRAFT+NPLAN-3097+Custom+App+Instance+Tags+Technical+Design**

---

## Jira Tickets
- NPLAN: [NPLAN-3097](https://netskope.atlassian.net/browse/NPLAN-3097) — status: Under Review
- WebUI App Instances Page: [ENG-908181](https://netskope.atlassian.net/browse/ENG-908181) — owner: Yogendra Kumar Verma
- WebUI RTP Page: [ENG-908186](https://netskope.atlassian.net/browse/ENG-908186) — owner: Sukhdeep Singh Handa
- Security Impact Analysis: [SIA-924](https://netskope.atlassian.net/browse/SIA-924)
- UX Design: [UX-1299](https://netskope.atlassian.net/browse/UX-1299) — status: Done
- Proxy ticket: **TBC — to be created**
- RIS object registration ticket: **TBC — to be created**

---

## Source Code Locations
- **CCI Backend:** `/Users/aisaichakaravarthi/myworkdir/cci`
  - Primary service: `nsappinfo/src/nsappinfo/`
  - App Instance CRUD: `svc/services/casbinline/profiles.py`
  - Constraint profiles (RIS pattern reference): `svc/services/casbinline/constraints.py`
  - Routes: `svc/routes/casbinline_route.py`
  - Schema validation: `svc/schemas/casbinline_schema.py`
  - Constants (RIS object names): `svc/constants/constants.py`
  - DB connection factory: `database/connection.py`

- **CCI UI (micro-frontend):** `/Users/aisaichakaravarthi/myworkdir/mf-cci`
  - API URLs: `src/api/api-url.ts`
  - Existing tag manager (CCI attribute tags — unrelated): `src/pages/tag/TagManager.tsx`
  - Tab API client: `src/api/tab/tab.api.ts`
  - Common queries: `src/common/query/common.query.ts`

---

## Critical Context: Two Separate Tag Systems in One Repo

Do NOT confuse these:

| | CCI Catalog Tags (`cci_route.py`) | App Instance Tags (`casbinline/profiles.py`) |
|---|---|---|
| What they tag | Apps in the CCI catalog — global | App Instances — per tenant |
| DB tables | `tag_data`, `attribute_tag_data`, `static_apps_tag_data` | `app_instances` (`tag` column) |
| mf-cci consumer | `TagManager.tsx` | App Instance page |
| NPLAN-3097 touches? | **No** | **Yes** |

---

## Database Separation

Three distinct databases:

| Database | Connection | Scope |
|---|---|---|
| `core_data` | `getCoreDB()` | Global — shared infrastructure |
| `app_info` | `RetryTransaction()` no tenantId | Global — shared app catalog (`applications` table) |
| `tenantXXXdb` | `RetryTransaction(tenantId=N)` | Per-tenant — `app_instances`, `tag_data` etc. |

Both new tables (`app_instances.custom_tags` column and `app_instance_custom_tags`) live in **`tenantXXXdb`**.
Cross-DB JOINs against `app_info.applications` are normal — already used throughout `profiles.py`.

---

## Design Decisions Already Made

### 1. custom_tags Is an App Instance Attribute — Not a Separate Endpoint
`custom_tags` flows through the existing `create_app_instance()`, `update_app_instance()`, `get_app_instances()` as a peer of `tag` (predefined). No separate assignment endpoint.

### 2. Two Concerns Cleanly Separated
- **Tag Definition CRUD** — new endpoints on `casbinline_route.py` → `profiles.py` service functions, manage `app_instance_custom_tags` table.
- **Tag Assignment** — attribute on the existing App Instance CRUD, stored in `app_instances.custom_tags`.

### 3. All Backend Changes in casbinline — NOT cci_route.py
`cci_route.py` is for CCI catalog tagging (unrelated). All NPLAN-3097 backend work goes in:
- `casbinline/profiles.py`
- `casbinline_route.py`
- `casbinline_schema.py`

### 4. RIS Owns Policy Reference Tracking
App Instance Custom Tags must be registered as a RIS object (`APP_INSTANCE_TAG_PROFILE = "appinsttag"`). Policy reference enforcement on delete is handled by RIS — it returns `"Object is being referenced and cannot be deleted"` → service returns HTTP 409. No denormalized counter, no cross-service call needed. Pattern from `constraints.py` lines 221–228.

### 5. Proxy Header
Rename `x-cs-app-instance-tag` → `x-cs-app-instance-tags` (semicolon-separated, multi-value). Emit both headers during transition. Proxy ticket TBC.

---

## Schema Migrations (both in tenantXXXdb)

```sql
-- 1. Extend app_instances
ALTER TABLE app_instances
  ADD COLUMN custom_tags VARCHAR(2000) DEFAULT NULL
  COMMENT 'Semicolon-separated custom tag names. Peer of tag (predefined).';

-- 2. New tag definition catalog
CREATE TABLE app_instance_custom_tags (
  id          INT AUTO_INCREMENT PRIMARY KEY,
  tag_name    VARCHAR(75)  NOT NULL,
  description VARCHAR(255) DEFAULT NULL,
  created_at  DATETIME     DEFAULT CURRENT_TIMESTAMP,
  updated_at  DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uq_tag_name (tag_name)
);
```

---

## profiles.py Write Paths to Modify (4 locations)

| Location | Change |
|---|---|
| Line 413 — `create_app_instance()` INSERT | Add `custom_tags` column + `:custom_tags` param |
| Line 364–373 — `create_app_instance()` `instance_data` dict | Add `"custom_tags": ";".join(inst['tags'].get('custom', []))` |
| Line 511–520 — `update_app_instance()` `update_data` dict | Add `"custom_tags": ";".join(req_data['instance_tag'].get('custom', []))` |
| Line 602 — `update_app_instance()` UPDATE query | Add `custom_tags = :custom_tags` |
| Line 588 — sync-migration INSERT in `update_app_instance()` | Same as line 413 |
| Line 734 — sync-migration INSERT in `delete_app_instance()` | Same as line 413 |

Read paths to extend: `get_app_instances()` (line 251 columns, line 308 row map), `get_app_instances_for_check()` (line 106), `get_app_instances_for_matching()` (lines 1024, 1042, 1066).

---

## New Constant Required

```python
# svc/constants/constants.py — add alongside existing:
APP_INSTANCE_TAG_PROFILE = "appinsttag"  # NEW — for RIS registration
```

Existing for reference:
```python
APP_INSTANCE_PROFILE       = "appinst"
CONSTRAINT_USER_PROFILE    = "constraintuser"
CONSTRAINT_STORAGE_PROFILE = "constraintstorage"
```

---

## New API Endpoints (casbinline_route.py)

```
GET    /casbinline/appinstance/tags                — list tag definitions
POST   /casbinline/appinstance/tags                — create tag definition + RIS register
PATCH  /casbinline/appinstance/tags/<tag_name>     — update description
DELETE /casbinline/appinstance/tags?tags=a,b       — delete + RIS delete (409 if referenced)
```

Extended existing:
```
POST /casbinline/appinstance   — tags.custom array added (peer of tags.predefined)
PUT  /casbinline/appinstance   — instance_tag.custom array added (peer of instance_tag.predefined)
POST /casbinline/getappinstances — custom_tags returned in response rows
```

---

## mf-cci UI Changes (ENG-908181, owner: Yogendra Kumar Verma)

New files under `src/pages/app-instance-tags/` and `src/api/app-instance-tags/`.

Key Figma-grounded UX decisions:
- App Instance page gets a **"Manage Tags" second tab** (Figma: "Manage Tags Table" frame)
- **Policy count chip is clickable** → navigates to RTP Policy page filtered by tag
- **Mid-flow tag creation** during New App Instance form (creatable select, background call)
- **Bulk "Edit Tags"** action from instance table row selection
- Predefined tag (radio) and custom tags (multi-select) are **always separate controls**
- Delete disabled if RIS-referenced by a policy OR `instance_count > 0`

New API URL constants to add to `src/api/api-url.ts`:
```typescript
GET_APP_INSTANCE_TAGS: '/api/v2/services/cci/casbinline/appinstance/tags',
INS_APP_INSTANCE_TAG:  '/api/v2/services/cci/casbinline/appinstance/tags',
UPD_APP_INSTANCE_TAG:  '/api/v2/services/cci/casbinline/appinstance/tags/',
DEL_APP_INSTANCE_TAG:  '/api/v2/services/cci/casbinline/appinstance/tags',
```

---

## RTP Page Changes (ENG-908186, owner: Sukhdeep Singh Handa)

Angular shell (`ms-app-core`) — not in mf-cci:
1. **"App Instance Tag" as Destination** in RTP rule builder → multi-select from tag list
2. **New Filter Type** in RTP Policies list filter panel
3. Once "App Instance Tag" selected as Destination → remove it from Criteria & Constraints (frontend state only)

---

## Dependency Map Summary

| Component | Owner | Ticket |
|---|---|---|
| casbinline/profiles.py + casbinline_route.py + casbinline_schema.py | CCI Backend | ENG-908181 |
| RIS — register `appinsttag` object type | RIS team | TBC |
| mf-cci WebUI | Yogendra Kumar Verma | ENG-908181 |
| WebUI RTP Page | Sukhdeep Singh Handa | ENG-908186 |
| Proxy header changes | Proxy team | TBC |
| SkopeIT / Advanced Analytics | Data team | — |

---

## Skills / Tools Available
- `/jira` — Jira CLI at `~/.claude/plugins/cache/netskope/eng-skills/1.4.0/skills/jira/scripts/jira`
- `/confluence` — Confluence CLI at `~/.claude/plugins/cache/netskope/eng-skills/1.4.0/skills/confluence/scripts/confluence`
- Confluence page ID for this design: `7937393245`
