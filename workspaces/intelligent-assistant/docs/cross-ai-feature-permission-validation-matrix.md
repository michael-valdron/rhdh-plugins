# Cross-AI Feature Permission Validation Matrix

Source: [RHIDP-15674](https://redhat.atlassian.net/browse/RHIDP-15674).
Execution of these cases: [RHIDP-15675](https://redhat.atlassian.net/browse/RHIDP-15675).

This matrix maps each Intelligent Assistant feature-linked permission to the
exact rights it grants and denies. Use the same validation approach for every
integration (chat, notebooks, MCP, and skills). Partial-rollout personas are
included so a user with only a subset of permissions cannot reach another
feature.

## Prerequisites

- Permission framework enabled (`permission.enabled: true`) with RBAC.
- Policy CSV action is always `use`, matching
  `workspaces/intelligent-assistant/rbac-policy.csv`.
- Test users are **non-admin**. RHDH plugin-permission admins typically bypass
  these checks and are not valid negative-case subjects.
- Base API path: `/api/intelligent-assistant`.
- Notebooks API path: `/api/intelligent-assistant/notebooks` (mounted under the
  plugin router).

```csv
p, role:default/<role>, intelligent-assistant.chat, use, allow
p, role:default/<role>, intelligent-assistant.notebooks, use, allow
p, role:default/<role>, intelligent-assistant.mcp.tools, use, allow
p, role:default/<role>, intelligent-assistant.skills, use, allow
g, user:default/<user>, role:default/<role>
```

## Feature-linked permission sets

Defined in
`plugins/intelligent-assistant-common/src/permissions.ts`. Each set is a
Backstage basic permission with empty attributes. Holding a set grants **full**
use of that feature (read, create, update, delete, and configure within the
feature). There is no per-action split (`access` / `use` / `manage`).

| Permission name                   | Constant                | Feature         |
| --------------------------------- | ----------------------- | --------------- |
| `intelligent-assistant.chat`      | `iaChatPermission`      | Lightspeed chat |
| `intelligent-assistant.notebooks` | `iaNotebooksPermission` | Notebooks       |
| `intelligent-assistant.mcp.tools` | `iaMcpToolsPermission`  | MCP tools       |
| `intelligent-assistant.skills`    | `iaSkillsPermission`    | Skills          |

Related (not a feature-linked IA set): MCP tools that query the catalog still
need `catalog.entity.read` / `catalog.location.read`. Those catalog checks are
out of scope for this matrix except as a note when MCP tools fail after IA
permissions are granted.

## Consistent validation approach

Apply this contract to every integration.

| Outcome                | HTTP (authenticated)                                         | UI                                                             |
| ---------------------- | ------------------------------------------------------------ | -------------------------------------------------------------- |
| **Grant**              | Success (`200` / `202`) for a well-formed request            | Feature is usable                                              |
| **Deny**               | `403` with `{ "error": "Unauthorized" }`                     | Feature blocked, read-only, or missing-permissions empty state |
| **Unauthenticated**    | `401` `{ "error": "Unauthorized" }` from identity middleware | Sign-in / guest denied                                         |
| **Identity forbidden** | `403` `{ "error": "Forbidden" }` from identity middleware    | Not a permission-set failure                                   |
| **Health**             | `GET …/health` is ungated (`200`)                            | N/A                                                            |

Isolation rule: a grant for permission set **A** must not authorize any
endpoint or UI surface gated by permission set **B**.

Ordering exception (chat query only): `POST /v1/query` runs body validation
**before** authorization. A malformed body returns `400` even when the user is
denied `intelligent-assistant.chat`. A well-formed body with deny still returns
`403`. Other routes authorize first.

## Rights each permission grants and denies

Legend: **G** = grant, **D** = deny. Health endpoints are ungated for every
persona.

| Capability                                                                                    | `chat` | `notebooks` | `mcp.tools` | `skills` |
| --------------------------------------------------------------------------------------------- | ------ | ----------- | ----------- | -------- |
| Open plugin / chat drawer (after FAB)                                                         | G      | D           | D           | D        |
| List/send/rename/delete conversations, saved prompts, feedback, models, shields, vision check | G      | D           | D           | D        |
| Conversation menu rename / delete                                                             | G      | D           | D           | D        |
| Notebooks tab content                                                                         | D\*    | G           | D           | D        |
| Notebook sessions, documents, notebook query                                                  | D      | G           | D           | D        |
| `GET /notebook-conversation-ids`                                                              | D      | G           | D           | D        |
| List / toggle / token-configure MCP servers                                                   | D      | D           | G           | D        |
| `GET /v1/skills`                                                                              | D      | D           | D           | G        |

\*The notebooks **tab** is reachable only after chat UI access. Without
`intelligent-assistant.chat`, the plugin renders `PermissionRequiredState` for
`intelligent-assistant.chat` and never evaluates the notebooks tab. Direct
notebook **API** calls still follow the notebooks column.

## Partial-rollout personas

| ID         | Granted sets          | Chat UI             | Chat API | Notebooks UI        | Notebooks API | MCP UI               | MCP API | Skills API |
| ---------- | --------------------- | ------------------- | -------- | ------------------- | ------------- | -------------------- | ------- | ---------- |
| P0         | none                  | Missing permissions | 403      | Unreachable         | 403           | Unreachable          | 403     | 403        |
| P-CHAT     | chat                  | Usable              | 200      | Missing permissions | 403           | Read-only / 403 load | 403     | 403        |
| P-NB       | notebooks             | Missing permissions | 403      | Unreachable         | 200           | Unreachable          | 403     | 403        |
| P-MCP      | mcp.tools             | Missing permissions | 403      | Unreachable         | 403           | Unreachable          | 200     | 403        |
| P-SK       | skills                | Missing permissions | 403      | Unreachable         | 403           | Unreachable          | 403     | 200        |
| P-CHAT-NB  | chat + notebooks      | Usable              | 200      | Usable              | 200           | Read-only / 403 load | 403     | 403        |
| P-CHAT-MCP | chat + mcp.tools      | Usable              | 200      | Missing permissions | 403           | Manage               | 200     | 403        |
| P-CHAT-SK  | chat + skills         | Usable              | 200      | Missing permissions | 403           | Read-only / 403 load | 403     | 200        |
| P-NB-MCP   | notebooks + mcp.tools | Missing permissions | 403      | Unreachable         | 200           | Unreachable          | 200     | 403        |
| P-ALL      | all four              | Usable              | 200      | Usable              | 200           | Manage               | 200     | 200        |

`P-CHAT` is the minimum UI persona. Notebooks-only, MCP-only, and skills-only
are **API-only** rollouts until chat is also granted.

---

## Test cases

Expected deny body unless noted: HTTP `403`, JSON `{ "error": "Unauthorized" }`.

### Shared / identity

| ID         | Type     | Steps                                                 | Expected                            |
| ---------- | -------- | ----------------------------------------------------- | ----------------------------------- |
| TC-ID-N-01 | Negative | Call any gated endpoint with no Backstage credentials | `401` `{ "error": "Unauthorized" }` |
| TC-ID-P-01 | Positive | `GET /health` as P0 (no IA permissions)               | `200` `{ "status": "ok" }`          |
| TC-ID-P-02 | Positive | `GET /notebooks/health` as P0                         | `200` `{ "status": "ok" }`          |

### Lightspeed chat

Gated by `intelligent-assistant.chat`. Frontend:
`useLightspeedViewPermission`, `useLightspeedUpdatePermission`, and
`useLightspeedDeletePermission` all resolve that same permission.

#### Positive

| ID           | Surface | Steps                                                           | Expected                                                |
| ------------ | ------- | --------------------------------------------------------------- | ------------------------------------------------------- |
| TC-CHAT-P-01 | UI      | P-CHAT opens `/intelligent-assistant` or FAB drawer             | Chat UI loads (not missing-permissions)                 |
| TC-CHAT-P-02 | UI      | P-CHAT sends a message                                          | Stream succeeds; conversation appears in history        |
| TC-CHAT-P-03 | UI      | P-CHAT renames a conversation                                   | Rename enabled and succeeds                             |
| TC-CHAT-P-04 | UI      | P-CHAT deletes a conversation                                   | Delete enabled and succeeds                             |
| TC-CHAT-P-05 | UI      | P-CHAT opens notebooks tab                                      | Chat still works; notebooks follow notebooks permission |
| TC-CHAT-P-06 | API     | P-CHAT `GET /v1/models`                                         | 200, model list                                         |
| TC-CHAT-P-07 | API     | P-CHAT `GET /v1/shields`                                        | 200                                                     |
| TC-CHAT-P-08 | API     | P-CHAT `GET /v2/conversations`                                  | 200                                                     |
| TC-CHAT-P-09 | API     | P-CHAT `GET /v2/conversations/:id`                              | 200                                                     |
| TC-CHAT-P-10 | API     | P-CHAT `PUT /v2/conversations/:id` with topic summary           | 200                                                     |
| TC-CHAT-P-11 | API     | P-CHAT `DELETE /v2/conversations/:id`                           | 200                                                     |
| TC-CHAT-P-12 | API     | P-CHAT `GET /v1/feedback/status`                                | 200                                                     |
| TC-CHAT-P-13 | API     | P-CHAT `POST /v1/feedback` well-formed                          | 200                                                     |
| TC-CHAT-P-14 | API     | P-CHAT `GET /v1/saved-prompts/config`                           | 200                                                     |
| TC-CHAT-P-15 | API     | P-CHAT `GET /v1/saved-prompts`                                  | 200                                                     |
| TC-CHAT-P-16 | API     | P-CHAT `POST /v1/saved-prompts` well-formed                     | 200                                                     |
| TC-CHAT-P-17 | API     | P-CHAT `DELETE /v1/saved-prompts/:prompt_id`                    | 200                                                     |
| TC-CHAT-P-18 | API     | P-CHAT `POST /v1/query` well-formed                             | 200 stream                                              |
| TC-CHAT-P-19 | API     | P-CHAT `POST /v1/query/interrupt` well-formed                   | 200                                                     |
| TC-CHAT-P-20 | API     | P-CHAT `POST /v1/validate-model-vision` with model and provider | 200                                                     |

#### Negative

| ID           | Surface   | Steps                                                       | Expected                                                                      |
| ------------ | --------- | ----------------------------------------------------------- | ----------------------------------------------------------------------------- |
| TC-CHAT-N-01 | UI        | P0 / P-NB / P-MCP / P-SK opens plugin or FAB                | Title **Missing permissions**; description names `intelligent-assistant.chat` |
| TC-CHAT-N-02 | UI        | P-CHAT-NB with chat revoked mid-session, reload             | Missing-permissions for chat; notebooks tab not shown                         |
| TC-CHAT-N-03 | API       | P0 or P-NB `GET /v1/models`                                 | 403                                                                           |
| TC-CHAT-N-04 | API       | P0 `GET /v1/shields`                                        | 403                                                                           |
| TC-CHAT-N-05 | API       | P0 `GET /v2/conversations`                                  | 403                                                                           |
| TC-CHAT-N-06 | API       | P0 `GET /v2/conversations/:id`                              | 403                                                                           |
| TC-CHAT-N-07 | API       | P0 `PUT /v2/conversations/:id`                              | 403                                                                           |
| TC-CHAT-N-08 | API       | P0 `DELETE /v2/conversations/:id`                           | 403                                                                           |
| TC-CHAT-N-09 | API       | P0 `GET /v1/feedback/status`                                | 403                                                                           |
| TC-CHAT-N-10 | API       | P0 `POST /v1/feedback`                                      | 403                                                                           |
| TC-CHAT-N-11 | API       | P0 `GET /v1/saved-prompts/config`                           | 403                                                                           |
| TC-CHAT-N-12 | API       | P0 `GET /v1/saved-prompts`                                  | 403                                                                           |
| TC-CHAT-N-13 | API       | P0 `POST /v1/saved-prompts`                                 | 403                                                                           |
| TC-CHAT-N-14 | API       | P0 `DELETE /v1/saved-prompts/:prompt_id`                    | 403                                                                           |
| TC-CHAT-N-15 | API       | P0 `POST /v1/query` well-formed body                        | 403                                                                           |
| TC-CHAT-N-16 | API       | P0 `POST /v1/query` missing `provider` / `model` / `query`  | **400** (validation before auth)                                              |
| TC-CHAT-N-17 | API       | P0 `POST /v1/query/interrupt`                               | 403                                                                           |
| TC-CHAT-N-18 | API       | P0 `POST /v1/validate-model-vision` with model and provider | 403 (auth before body 400)                                                    |
| TC-CHAT-N-19 | Isolation | P-NB, P-MCP, or P-SK repeats TC-CHAT-N-03–N-15, N-17–N-18   | 403 on every chat route                                                       |

### Notebooks

Gated by `intelligent-assistant.notebooks`. Frontend:
`useLightspeedNotebooksPermission`. Denied notebooks tab shows
`PermissionRequiredState` for `intelligent-assistant.notebooks` with **Go
back**.

#### Positive

| ID         | Surface | Steps                                                 | Expected                                            |
| ---------- | ------- | ----------------------------------------------------- | --------------------------------------------------- |
| TC-NB-P-01 | UI      | P-CHAT-NB opens Notebooks tab                         | Notebook list / create UI (not missing-permissions) |
| TC-NB-P-02 | UI      | P-CHAT-NB creates, opens, renames, deletes a notebook | All succeed                                         |
| TC-NB-P-03 | UI      | P-CHAT-NB uploads, lists, renames, deletes a document | All succeed                                         |
| TC-NB-P-04 | UI      | P-CHAT-NB asks a question in a notebook               | Stream succeeds                                     |
| TC-NB-P-05 | API     | P-NB or P-CHAT-NB `POST /notebooks/v1/sessions`       | 200                                                 |
| TC-NB-P-06 | API     | Same persona `GET /notebooks/v1/sessions`             | 200                                                 |
| TC-NB-P-07 | API     | `GET /notebooks/v1/sessions/:sessionId`               | 200                                                 |
| TC-NB-P-08 | API     | `PUT /notebooks/v1/sessions/:sessionId`               | 200                                                 |
| TC-NB-P-09 | API     | `DELETE /notebooks/v1/sessions/:sessionId`            | 200                                                 |
| TC-NB-P-10 | API     | `GET /notebooks/v1/sessions/:sessionId/documents`     | 200                                                 |
| TC-NB-P-11 | API     | `PUT /notebooks/v1/sessions/:sessionId/documents`     | 202 processing                                      |
| TC-NB-P-12 | API     | `GET …/documents/:documentId/status`                  | 200                                                 |
| TC-NB-P-13 | API     | `PATCH …/documents/:documentId`                       | 200                                                 |
| TC-NB-P-14 | API     | `DELETE …/documents/:documentId`                      | 200                                                 |
| TC-NB-P-15 | API     | `POST /notebooks/v1/sessions/:sessionId/query`        | 200 stream                                          |
| TC-NB-P-16 | API     | `GET /notebook-conversation-ids`                      | 200                                                 |

#### Negative

| ID         | Surface   | Steps                                             | Expected                                                                                   |
| ---------- | --------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| TC-NB-N-01 | UI        | P-CHAT (no notebooks) opens Notebooks tab         | **Missing permissions** for `intelligent-assistant.notebooks`; **Go back** returns to chat |
| TC-NB-N-02 | UI        | P-NB (no chat) opens plugin                       | Chat missing-permissions; notebooks UI never shown                                         |
| TC-NB-N-03 | API       | P-CHAT `POST /notebooks/v1/sessions`              | 403                                                                                        |
| TC-NB-N-04 | API       | P-CHAT `GET /notebooks/v1/sessions`               | 403                                                                                        |
| TC-NB-N-05 | API       | P-CHAT `GET /notebooks/v1/sessions/:sessionId`    | 403                                                                                        |
| TC-NB-N-06 | API       | P-CHAT `PUT /notebooks/v1/sessions/:sessionId`    | 403                                                                                        |
| TC-NB-N-07 | API       | P-CHAT `DELETE /notebooks/v1/sessions/:sessionId` | 403                                                                                        |
| TC-NB-N-08 | API       | P-CHAT `GET …/documents`                          | 403                                                                                        |
| TC-NB-N-09 | API       | P-CHAT `PUT …/documents`                          | 403                                                                                        |
| TC-NB-N-10 | API       | P-CHAT `GET …/documents/:documentId/status`       | 403                                                                                        |
| TC-NB-N-11 | API       | P-CHAT `PATCH …/documents/:documentId`            | 403                                                                                        |
| TC-NB-N-12 | API       | P-CHAT `DELETE …/documents/:documentId`           | 403                                                                                        |
| TC-NB-N-13 | API       | P-CHAT `POST …/query`                             | 403                                                                                        |
| TC-NB-N-14 | API       | P-CHAT `GET /notebook-conversation-ids`           | 403                                                                                        |
| TC-NB-N-15 | Isolation | P-MCP or P-SK repeats TC-NB-N-03–N-14             | 403                                                                                        |

### MCP

Gated by `intelligent-assistant.mcp.tools`. Frontend: `usePermission({
permission: iaMcpToolsPermission })` in `McpServersSettings`. Denied users see
the **You have read-only access to MCP servers.** alert; enable toggles and
configure actions are disabled. `GET /mcp-servers` is also gated, so the table
does not populate (load error plus read-only alert).

#### Positive

| ID          | Surface | Steps                                                     | Expected                                                             |
| ----------- | ------- | --------------------------------------------------------- | -------------------------------------------------------------------- |
| TC-MCP-P-01 | UI      | P-CHAT-MCP opens MCP settings                             | Server list loads; toggles and configure enabled; no read-only alert |
| TC-MCP-P-02 | UI      | P-CHAT-MCP enables / disables a server                    | PATCH succeeds; row updates                                          |
| TC-MCP-P-03 | UI      | P-CHAT-MCP sets or clears a personal token                | PATCH succeeds; validation runs when a token is set                  |
| TC-MCP-P-04 | API     | P-MCP or P-CHAT-MCP `GET /mcp-servers`                    | 200 `{ servers: […] }`                                               |
| TC-MCP-P-05 | API     | `POST /mcp-servers/validate` with LCS-known URL and token | 200                                                                  |
| TC-MCP-P-06 | API     | `POST /mcp-servers/:name/validate`                        | 200                                                                  |
| TC-MCP-P-07 | API     | `PATCH /mcp-servers/:name` `{ "enabled": false }`         | 200                                                                  |

#### Negative

| ID          | Surface   | Steps                                     | Expected                                                             |
| ----------- | --------- | ----------------------------------------- | -------------------------------------------------------------------- |
| TC-MCP-N-01 | UI        | P-CHAT (no mcp.tools) opens MCP settings  | Read-only alert; toggles and configure disabled; list fails with 403 |
| TC-MCP-N-02 | UI        | P-MCP (no chat) opens plugin              | Chat missing-permissions; MCP settings unreachable                   |
| TC-MCP-N-03 | API       | P-CHAT `GET /mcp-servers`                 | 403                                                                  |
| TC-MCP-N-04 | API       | P-CHAT `POST /mcp-servers/validate`       | 403                                                                  |
| TC-MCP-N-05 | API       | P-CHAT `POST /mcp-servers/:name/validate` | 403                                                                  |
| TC-MCP-N-06 | API       | P-CHAT `PATCH /mcp-servers/:name`         | 403                                                                  |
| TC-MCP-N-07 | Isolation | P-NB or P-SK repeats TC-MCP-N-03–N-06     | 403                                                                  |

### Skills

Gated by `intelligent-assistant.skills`. There is no dedicated frontend
`usePermission` hook; enforcement is API-only (`GET /v1/skills`). Include these
cases so all four feature-linked sets are covered with the same grant/deny
approach.

#### Positive

| ID         | Surface | Steps                              | Expected                             |
| ---------- | ------- | ---------------------------------- | ------------------------------------ |
| TC-SK-P-01 | API     | P-SK or P-CHAT-SK `GET /v1/skills` | 200 `{ skills: […] }` (may be empty) |

#### Negative

| ID         | Surface   | Steps                          | Expected |
| ---------- | --------- | ------------------------------ | -------- |
| TC-SK-N-01 | API       | P-CHAT `GET /v1/skills`        | 403      |
| TC-SK-N-02 | Isolation | P-NB or P-MCP `GET /v1/skills` | 403      |

---

## Endpoint catalog (source of truth)

All plugin routes except health go through identity middleware, then
`requirePermission`.

### Ungated

| Method | Path                |
| ------ | ------------------- |
| GET    | `/health`           |
| GET    | `/notebooks/health` |

### `intelligent-assistant.mcp.tools`

| Method | Path                          |
| ------ | ----------------------------- |
| GET    | `/mcp-servers`                |
| POST   | `/mcp-servers/validate`       |
| POST   | `/mcp-servers/:name/validate` |
| PATCH  | `/mcp-servers/:name`          |

### `intelligent-assistant.notebooks`

| Method | Path                                                             |
| ------ | ---------------------------------------------------------------- |
| GET    | `/notebook-conversation-ids`                                     |
| POST   | `/notebooks/v1/sessions`                                         |
| GET    | `/notebooks/v1/sessions`                                         |
| GET    | `/notebooks/v1/sessions/:sessionId`                              |
| PUT    | `/notebooks/v1/sessions/:sessionId`                              |
| DELETE | `/notebooks/v1/sessions/:sessionId`                              |
| GET    | `/notebooks/v1/sessions/:sessionId/documents`                    |
| PUT    | `/notebooks/v1/sessions/:sessionId/documents`                    |
| GET    | `/notebooks/v1/sessions/:sessionId/documents/:documentId/status` |
| PATCH  | `/notebooks/v1/sessions/:sessionId/documents/:documentId`        |
| DELETE | `/notebooks/v1/sessions/:sessionId/documents/:documentId`        |
| POST   | `/notebooks/v1/sessions/:sessionId/query`                        |

### `intelligent-assistant.chat`

| Method | Path                                 |
| ------ | ------------------------------------ |
| GET    | `/v1/models`                         |
| GET    | `/v1/shields`                        |
| GET    | `/v2/conversations`                  |
| GET    | `/v2/conversations/:conversation_id` |
| DELETE | `/v2/conversations/:conversation_id` |
| PUT    | `/v2/conversations/:conversation_id` |
| GET    | `/v1/feedback/status`                |
| POST   | `/v1/feedback`                       |
| GET    | `/v1/saved-prompts/config`           |
| GET    | `/v1/saved-prompts`                  |
| POST   | `/v1/saved-prompts`                  |
| DELETE | `/v1/saved-prompts/:prompt_id`       |
| POST   | `/v1/query`                          |
| POST   | `/v1/query/interrupt`                |
| POST   | `/v1/validate-model-vision`          |

### `intelligent-assistant.skills`

| Method | Path         |
| ------ | ------------ |
| GET    | `/v1/skills` |

---

## UI gating catalog

| Surface                       | Permission                        | Denied behavior                                                              |
| ----------------------------- | --------------------------------- | ---------------------------------------------------------------------------- |
| Plugin page / chat container  | `intelligent-assistant.chat`      | `PermissionRequiredState` for the plugin; names `intelligent-assistant.chat` |
| FAB                           | none                              | FAB still opens; denied users then hit the chat empty state                  |
| Conversation rename menu item | `intelligent-assistant.chat`      | Item disabled (`hasUpdateAccess`)                                            |
| Conversation delete menu item | `intelligent-assistant.chat`      | Item disabled (`hasDeleteAccess`)                                            |
| Notebooks tab                 | `intelligent-assistant.notebooks` | `PermissionRequiredState` for notebooks; **Go back**                         |
| MCP settings                  | `intelligent-assistant.mcp.tools` | Read-only alert; controls disabled; list fetch 403                           |
| Skills                        | `intelligent-assistant.skills`    | No frontend hook; API 403 only                                               |

---

## Existing automated coverage vs this matrix

Use this when executing [RHIDP-15675](https://redhat.atlassian.net/browse/RHIDP-15675).
Long-term E2E/smoke automation is out of scope for 15675; unit coverage can
still be reused.

| Area                  | Already covered (unit)                                                               | Gaps vs this matrix                                                                    |
| --------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| Chat API deny         | shields, conversations CRUD, feedback, saved-prompts, query, query/interrupt, skills | `GET /v1/models`, `POST /v1/validate-model-vision`                                     |
| Chat query ordering   | malformed body → 400 while denied                                                    | —                                                                                      |
| Chat UI deny          | `LightspeedPage` missing-permissions                                                 | FAB still visible without chat                                                         |
| Notebooks API deny    | sessions POST/GET/GET:id, documents PUT, query                                       | session PUT/DELETE, document GET/PATCH/DELETE/status, `GET /notebook-conversation-ids` |
| Notebooks UI deny     | notebooks tab missing-permissions + Go back                                          | —                                                                                      |
| MCP API deny          | GET list, PATCH, POST `:name/validate`                                               | `POST /mcp-servers/validate`                                                           |
| MCP UI deny           | not asserted for read-only alert                                                     | TC-MCP-N-01                                                                            |
| Isolation (wrong set) | tests deny **all** permissions, not a single other set                               | P-CHAT vs notebooks/MCP/skills cross-grants                                            |
| Partial rollouts      | —                                                                                    | All P-\* personas in the matrix                                                        |
| Skills                | GET 200 and 403                                                                      | —                                                                                      |
| Health ungated        | plugin and notebooks health 200                                                      | —                                                                                      |
