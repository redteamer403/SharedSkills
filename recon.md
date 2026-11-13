---
description: Fully autonomous agent-driven recon (no custom tools or scripts). Scope gate → alive/tech → exhaustive JS harvesting (all bundles, lazy chunks, manifests) by subagents → manual HTTP-contract reconstruction → env/config → matrix/delta → per-host verification → priority map. Discovery only: no payloads, exploitation, or brute force. Use when the user types "recon", "recon <target>", "agentic recon".
argument-hint: <target>
---

# Autonomous Recon — pure-agent skill

You are the **recon orchestrator**. Your goal is to build the most complete map of
the target surface (hosts, endpoints, parameters, forms, JS contracts, auth model)
**through agents and their reasoning**, without custom tools and without helper
scripts.

**Allowed:** Read / Grep / Glob / Write / Edit, standard CLI for single
operations (`curl` for one URL at a time, `head`, `grep`, `sort`, `jq`, `wc`,
`file`), Agent subagents.
**Forbidden:** any repo-specific tools (`tools/*.py`, `tools/*.sh`), your own
scripts/parsers/runners, mass automated crawl code, payloads, fuzzing, brute
force, POST/PUT/PATCH/DELETE, DoS.
**If a tool is unavailable** — do the same work manually via Read/Grep/Bash with
single commands. Never answer "I couldn't, no tool available" — switch to manual
mode.

Results are isolated in **`$TARGET_DIR/recon-agent/`**.

---

## Step 0 — Locate TARGET_DIR and define paths

```
$ARGUMENTS = target (target folder name or domain)
TARGET_DIR = absolute path to the target folder
RA         = $TARGET_DIR/recon-agent
```

If `$ARGUMENTS` is empty — `Usage: /recon <target>` and STOP.
Locate the target folder (by name or by `scope.txt` containing the domain). If
several match — ask; if none — STOP.

Create the structure (empty files via Write, or `mkdir -p`):

```
$RA/recon/
$RA/matrix/
$RA/evidence/
$RA/reports/
```

All artifacts go to `$RA/`. Scope/policy come from `$TARGET_DIR/`.

---

## Phase 0 — Scope gate + policy + connectivity

1. Read `scope.txt`. Empty/missing → **STOP** and ask the user to fill it in.
2. Read `out-of-scope.txt` (it is **stronger** than scope) and `RULES.md` /
   `account.json` (if present).
3. Record the **policy preamble** in `$RA/recon/POLICY.txt`:
   - rate limits, required headers/attribution, banned techniques;
   - platform, economics (paid/VDP), auth model (blackbox/1–2 accounts).
4. Connectivity precheck: single `curl -sS -o /dev/null -w '%{http_code}'` calls
   against 1–3 hosts. All `000`/NXDOMAIN → STOP and warn.
5. Build `$RA/recon/scope_hosts.txt` — hosts only (wildcard `*.zone` → zone for
   enumeration; IP/CIDR in a separate list).

**Scope-check every host before any request** — manually, by this logic:
wildcard `*.zone` matches any subdomain; out-of-scope wins; when in doubt — do
not request it.

---

## Phase 0.5 — Product / COTS filter

Before crawling, identify 3rd-party products (title, `Server`, `X-Powered-By`,
cookies, characteristic paths, JS markers, favicon).

- `$RA/recon/thirdparty_excluded.tsv`: `host, product, reason`.
- Do not test 3rd-party **core/admin** with payloads, but still record the
  surface.
- A custom integration layer (Next.js API routes, `/api/graphql-proxy`, public
  GraphQL, RBAC filters) stays in scope — that is application code.
- Add `product` (empty = custom) and `custom` (yes/no) to the matrix.

---

## Phase 1 — Alive + tech probe (manual mode)

For every in-scope host: a single `curl` call (GET, follow redirects with a
limit) → status, `Server`, `X-Powered-By`, `Content-Type`, title, cookies,
tech markers.

Write `$RA/recon/alive.tsv`:
`input_host, final_host, status, server, x_powered_by, cookies, content_type, title, tech_notes`

Classify:
- `200/401/403` + HTML → crawl candidate;
- `3xx` → final URL; external redirect — record only, do **not** fetch;
- `404/5xx/timeout` → dead/edge, reason goes into `coverage_gaps.tsv`.

---

## Phase 2 — Exhaustive JS harvest (the main step, agents only)

**Crawl is firstpass, not the coverage ceiling.** Every crawlable host with JS
(SPA/Angular/React/Vue/Next/Nuxt/Flutter/any HTML) must undergo a full bundle
walk. Delegate one subagent per host or per group of identical alias hosts (see
Phase 2b). Each agent **manually**:

1. Downloads the seed HTML (root or redirect target, e.g. `/de/`).
2. Respects `<base href>` — all relative `src` values resolve against base, not
   the root. Ignoring base is a common cause of "404 on every chunk".
3. Collects every entry script:
   `<script src>`, `link[rel=modulepreload]`, inline `<script>`, JS literals.
4. Extracts manifests/lazy chunks:
   - Next.js: `_buildManifest.js`, `app-build-manifest.json`, `__NEXT_DATA__`,
     `self.__next_f`;
   - webpack: `__webpack_require__.u`, chunk map, `import()`;
   - Angular (`@angular/build`): `chunk-XXXXXXXX.js` in main/runtime;
   - Vite: `import()` and asset references;
   - Flutter: `flutter_bootstrap.js`, `flutter_service_worker.js` (RESOURCES).
5. Recursively downloads **all** discovered .js files (no chunk-count limit; the
   only caps are per-file size and per-host total, and they must be recorded
   explicitly).
6. Extracts contracts from every bundle:
   - URLs: absolute, relative, protocol-relative, template literals
     `` `${apiAddress}/path` ``, concatenations `base+"/path"`;
   - params: `params:{...}`, `.set/.append`, `HttpParams`, `URLSearchParams`;
   - body keys: `JSON.stringify({...})`, objects next to POST/PUT/PATCH;
   - headers: `X-*`, `Authorization`, `Ocp-Apim-Subscription-Key`, CSRF;
   - GraphQL: `/graphql`, `query/mutation/subscription` operations;
   - WS/SignalR, storage URLs (SAS/presigned), source maps, routes, roles/guards,
     UI-oracle strings (`restricted`, `isAdmin`, `published`…), feature flags.
7. Saves raw bundles to `$RA/evidence/<host>/js/` and results to
   `$RA/evidence/<host>/` (see the schema below).

Channel limits (mandatory): **≤2 req/s per host**, **≥0.6 s between requests to
the same host**, shared IP limit behind load balancers; `3×403/429` → back off,
classify (WAF/auth/rate), record the blocker. Do not fetch external
redirects/storage.

### Agent output files (Phase 2)

```
$RA/evidence/<host>/root.html
$RA/evidence/<host>/js/<bundle>.js
$RA/evidence/<host>/js_index.tsv      url, status, bytes, source(parent)
$RA/evidence/<host>/analysis.tsv      category, value, method, context, source, extra
$RA/evidence/<host>/external_hosts.txt
$RA/evidence/<host>/errors.log
$RA/evidence/<host>/summary.json      files, bytes, referenced, downloaded, missing, caps_hit
```

`analysis.tsv` categories: `endpoint_abs, endpoint_path, query_param, body_key,
header, graphql_endpoint, graphql_operation, route, ws_endpoint, source_map,
secret_* (masked), storage_url, ui_oracle, oidc_config, external_url`.

### Gap check (mandatory)

For every host, reconcile the inventory: **entry scripts in HTML + chunk refs in
main/runtime** against `js_index.tsv`. Fetch the missing chunks (GET).
Discrepancies, caps and errors go to `$RA/recon/coverage_gaps.tsv`:
`host, referenced, downloaded, missing, caps_hit, errors, note`.
A host without a gap record is considered **not processed**.

---

## Phase 2b — Subagent orchestration

Split hosts into groups: 1 agent = 1 host or a group of alias hosts with
identical bundles. Launch **batches of 6–8** (Agent tool, several calls in one
message), wait for the batch to finish, then start the next one.

Agent prompt template:

```
HOST(S): <host[, host2, ...]>
TARGET_DIR: <abs>
WORK_DIR: <abs $RA>
POLICY_PREAMBLE: <rate/headers/banned>

Task: full JS/HTML recon of the host(s), no payloads, no scripts.
1) Download the seed HTML (curl), respect <base href>, collect entry scripts.
2) Find manifests/chunk maps; recursively download ALL .js (0.6s between requests).
3) For every bundle, manually (Read/Grep) extract contracts: method, path/URL
   template, query/body params, headers, auth, GraphQL/WS, storage, routes,
   roles/UI-oracle, env config.
4) Gap check: referenced vs downloaded; fetch the missing ones.
5) Write evidence/<host>/... and reports/<host>.md (10 sections:
   Overview/Tech; Entry points; JS inventory; API contracts; Parameters;
   Auth model & roles; Storage/upload; Error oracles; Priority leads;
   Coverage gaps).
6) Append group TSVs: contracts_<group>.tsv, calls_<group>.tsv,
   ui_oracle_<group>.tsv, gaps_<group>.tsv.
7) New in-scope hosts — into api_hosts_candidates_<group>.txt; do NOT probe them.

Return ONLY JSON counters on one line:
{"group":"<g>","hosts":N,"contracts":N,"calls":N,"leads":N,"gaps":N}
```

Agent rules: discovery-only; GET/HEAD/OPTIONS; do not fetch external redirects;
scope-check every URL; only record secrets (mask them), never use them; bodies
and reports go to disk, return only counters.

---

## Phase 2.5 — Request-logic extraction (agent-driven)

A separate pass over all downloaded bundles (can be delegated to one or two
agents): reconstruct **complete calls** — not just URLs, but also:

- method (GET/POST/PUT/PATCH/DELETE from the code);
- exact path or URL template;
- query/body parameters and where they come from;
- headers (auth/CSRF/tenant);
- auth model (Bearer/MSAL/SwissPass/cookie), clientId/authority/scopes;
- GraphQL operations and variables;
- SignalR/WebSocket endpoints and messages.

Result: `$RA/evidence/<host>/calls.tsv`
`method, url_template, params, body_keys, headers, source_file`.

---

## Phase 2.6 — Env/config blocks & API-host routing

1. Manually extract runtime config from HTML/JS: `self.process=`,
   `window.__ENV__`, `__NEXT_DATA__`, ngssc, MSAL
   (`clientId/authority/scopes`), Firebase config, feature flags →
   `$RA/recon/contracts.tsv` (kind=`env_config`).
2. Every in-scope host found in env/JS (`*.api.*`, `*-backend*`, API/storage
   URLs): scope-check → single `curl` for status → `$RA/recon/api_hosts_probe.tsv`.
   No payloads. Out-of-scope hosts — record only in `external_hosts_all.txt`.

---

## Phase 3 — Matrix (host × URL × param)

Schema (TSV, dedupe by `host,url,method,param,kind`):

```
host  url  path  method  param  default_value  kind  context  app_type
api_verdict  product  custom  authz_label  readonly  graphql_type  relation
storage_url  session_required  source  confirm_status
```

- `kind`: form / link / js / api / header / env.
- `context`: query / body / path / header / cookie / graphql / ws / storage.
- `source`: the page/bundle where it was found.
- `matrix_firstpass.tsv` — crawl/entry;
- `matrix_deep.tsv` — lazy chunks + `analysis.tsv` + `calls.tsv`;
- `matrix_delta.tsv` = deep minus firstpass (priority #1 for the exploit phase).
- `params.tsv` and `unique_host_params.tsv` — aggregates.
- Never invent paths or parameters: only those found in HTML/JS/forms.

Additional files:
- `contracts.tsv` — `host, source, kind, value, method, auth_header, note`;
- `graphql.tsv` — `host, endpoint, operation, kind, query, variables, source`;
- `ui_oracle.tsv` — `host, source, label, i18n_string, note`;
- `routes.tsv` — `host, route, source`;
- `session_forms.json` — hidden/CSRF/viewstate, cookies, action, method,
  replay recipe (for server-rendered forms).

---

## Phase 4 — Stack fingerprint → class hypotheses

Determine the stack (server/X-Powered-By/cookies/tech/JS framework/legacy
extensions) and record `app_type` per host. Parameter triggers:

| trigger | classes |
|---|---|
| `redirect_uri/next/returnTo/url` | open redirect, OAuth chain |
| `id/uid/uuid/user_id/objectId` | IDOR/BOLA, SQLi, param pollution |
| `url/uri/path/file/import/webhook/avatar/preview/fetch` | SSRF, LFI/path traversal |
| `q/search/query/name/comment/bio/message` | SQLi, XSS |
| `filename/filepath/template/doc/xml/upload` | traversal, XXE, SSTI, upload |
| JSON `role/isAdmin/verified/price/discount` | mass assignment, business logic |
| GraphQL list/singular/relation | BOLA: list vs node, filters `access/published` |
| attachment/file/upload/media | storage URL/SAS, IDOR |
| Next.js `/api/*`, `/api/graphql-proxy`, server actions | proxy authz, SSRF, IDOR |
| UI login redirect / restricted labels | authz oracle for API differentials |
| `sig=`/`X-Amz-`/Bearer/JWT in JS | credential leak |
| `wss://` / WS frames | WS injection |
| chatbot/RAG/agent/MCP/`/api/ai/*` | LLM prompt injection |

---

## Phase 4.5 — Per-host verification agents (mandatory for SPA/API)

A second independent pass (a new agent batch). Each agent:

1. Re-verifies the inventory: HTML entry + chunk map vs `js_index.tsv`; fetches
   missing chunks.
2. Manually re-reads `calls.tsv` and the bundles, and supplements contracts:
   finds calls the regex pass could miss (dynamic URLs, switch/case,
   concatenations, i18n keys).
3. Writes `reports/<host>.md` (10 sections, as in Phase 2b).
4. Appends group TSVs; discrepancies go to `gaps_<group>.tsv`.

After each batch — merge group TSVs into the shared files (`sort -u`), then run
the next batch.

---

## Phase 5 — Triage, priority, outputs

Score:
`8×authz_label + 7×GraphQL/relation/attachment + 6×proxy/NextAPI +
5×storage/credential + 4×forms + 3×legacy + 2×api-hint + params + status`.

Priority 1 — `matrix_delta.tsv`; priority 2 — authz/GraphQL/credential.

Save:
- `$RA/matrix/{matrix,matrix_firstpass,matrix_deep,matrix_delta,params,
  unique_host_params}.tsv`
- `$RA/recon/{alive,coverage_gaps,triage,contracts,graphql,ui_oracle,routes,
  params_unbound,api_hosts_inscope,api_hosts_probe,external_hosts_all}.tsv|txt`,
  `POLICY.txt`, `thirdparty_excluded.tsv`
- `$RA/reports/WEB-RECON.md` — surface map, top candidates, JS completeness per
  host (referenced/downloaded/missing/caps), edge/WAF/dead with exact reasons,
  handoff to the next phase (exploit).

The final report must include, per host:
- status/stack/app_type;
- bundle counts: referenced / downloaded / missing (and why);
- API contracts (table: method, path, params, auth);
- auth model and roles;
- top leads (ranked, with rationale);
- blockers (WAF, SSO, 401/503, external redirect).

---

## Completeness contract (not "done" until satisfied)

1. Every crawlable host has `evidence/<host>/` with `root.html`, `js_index.tsv`,
   `analysis.tsv`, `summary.json` and a `coverage_gaps.tsv` record.
2. Every referenced chunk is either downloaded or explicitly listed in `missing`
   with a reason (403/404/edge/OOS/cap).
3. Every crawlable host has passed Phase 4.5 (second agent).
4. No unexplained 401/403/503 remains: each has an exact blocker.
5. The matrix covers all discovered URLs/params; delta is computed.

"Crawl hit its limit" is not a reason to skip lazy chunks: Phase 2/2b/4.5 must
fetch them.

---

## Hard rules

- **NO exploitation, NO payloads, NO fuzzing/brute, NO state-changing.**
- Scope-check every URL; out-of-scope wins; do not fetch external redirects.
- Rate: ≤2 req/s per host, ≥0.6 s between requests; 3×403/429 → back off +
  record the blocker.
- No custom tools/scripts: only reasoning + standard single CLI commands
  (curl/grep/head/sort/jq) + Read/Grep/Glob/Write.
- Do not delete artifacts; append + dedupe.
- Artifacts go to disk; only counters into the context.
- Secrets/tokens/SAS — record only (masked), never use them.
- Do not submit anything without explicit user approval.
- Handoff: the next phase is exploit against `matrix_delta.tsv`, `calls.tsv`,
  `contracts.tsv` and `reports/WEB-RECON.md`.
