# Connector: AirSync-AWSCostEngine

> **Provenance of this plan — read first.**
>
> The planner skill's Step 3 calls for probing the live API. That was **not
> possible**: the cost engine is not running on this machine and its source was
> not copied (`~/Downloads/aws/cost_agent` holds only dotfiles and a `.venv`
> whose editable install points at `/workshop/cost_agent/src`, a path from an
> AWS Workshop Studio environment).
>
> Everything below marked **[recorded]** is taken from the engine's own
> recorded request/response samples
> (`~/Downloads/aws/cost_agent/.local-state/api-samples/aws-cost-findings.json`)
> and its `.env.example`. Everything marked **[UNVERIFIED]** is an assumption
> that no observation supports. The three that matter most — pagination,
> incremental sync and rate limits — are all unverified, and all three change
> connector behaviour. **Confirm them against a live engine before trusting
> this connector with real data.**
>
> Also note the recorded samples were captured with `source_mode: fixture`, so
> they describe the contract faithfully but contain no real AWS data.

## 1. External System Overview

- **Name**: AWS Cost Engine (`cost_agent`)
- **API Base URL**: self-hosted; supplied per deployment via `COST_ENGINE_BASE_URL`. Routes are rooted at `/v1`. **[recorded]**
- **API Type**: REST/JSON **[recorded]**
- **API Maturity**: internal, undocumented. Its `.env.example` describes the HTTP surface as *"only needed if the optional HTTP surface is run"*, so it is an optional adjunct to a CLI tool rather than a primary product API. **[recorded]**
- **Authentication**: Bearer token. The engine's own config calls it `COST_AGENT_API_TOKEN` and suggests generating it with `secrets.token_urlsafe(32)`. **[recorded]**
- **Rate Limits**: **[UNVERIFIED]** — no `X-RateLimit-*` or `Retry-After` headers appear in the recorded samples. Treat as unknown and back off conservatively.
- **API Docs**: none published. The recorded samples are the contract of record.

### Observed routes **[recorded]**

| Method | Path | Status | Notes |
|---|---|---|---|
| POST | `/v1/preflight` | 200 | Identity, kill switches, action policy |
| GET | `/v1/coverage` | 200 | Technique catalog + coverage statuses |
| POST | `/v1/scans` | 202 | Body `{"demo_scenarios": true}`; **no scope arguments** |
| GET | `/v1/scans/{id}` | 200 | Scan status **and** its findings together |
| GET | `/v1/findings` | 200 | `{count, findings[], hidden_by_account_scope}` |
| GET | `/v1/findings/{id}` | 200 | Full finding |
| POST | `/v1/findings/{id}/investigations` | 201 | **Invokes the investigator** (see 2.2) |
| POST | `/v1/findings/{id}/plans` | 409 | Only ever recorded as a refusal |
| GET | `/v1/plans/{id}/verification` | 200 | Operational + financial status |
| POST | `/v1/findings/{id}/work-preview` | 201 | **The engine's own DevRev publisher** (see §8) |

Errors are `{"detail": "<message>"}`. **[recorded]**

## 2. Entity Inventory

### Cross-Reference Table

| # | External Entity | API Name | DevRev Target | Sync Direction | Dependencies |
|---|-----------------|----------|---------------|----------------|--------------|
| 1 | Finding | `findings` | `issue` | Extract only | None |
| 2 | Investigation | `investigations` | `timeline_comment` | **Skip — see 2.2** | Finding |
| 3 | Change plan | embedded `change_plan` | Fields on the parent `issue` | Extract only | Finding |
| 4 | Scan | `scans` | Not synced — supplies run context | — | None |
| 5 | Technique / coverage | `coverage` | Skip — reference data, no DevRev equivalent | — | None |
| 6 | Users | *(none)* | Skip — the engine exposes no user endpoint | — | — |

### Entity Details

#### 2.1 Finding

- **External type**: `findings` **[recorded]**
- **DevRev target**: `issue`. A cost finding is engineering work to triage and decide on, not a customer request, so `issue` fits better than `ticket`.
- **List endpoint**: `GET /v1/findings` → `{count, findings[], hidden_by_account_scope}` **[recorded]**
- **Detail endpoint**: `GET /v1/findings/{finding_id}` **[recorded]**
- **Pagination**: **none observed.** The recorded list response carries no cursor, page or offset field, and the request took no parameters. Planned as a single full enumeration. **[UNVERIFIED — see §10 Open]**
- **Incremental sync**: **no.** No `updated_since`, `modified_after` or equivalent parameter appears. Findings do carry `updated_at`, so client-side filtering is possible but still requires fetching everything. **[recorded + UNVERIFIED]**
- **Dependencies**: none.
- **External record id**: **`deduplication_key`**, not `finding_id`. This is the single most important decision in the plan: `finding_id` values such as `pb-ebs-54d7c5547941` are derived per scan, so keying on them would create a fresh DevRev issue on every scan. `deduplication_key` is the engine's own stable identity for "the same opportunity on the same resource". **[recorded]**

#### 2.2 Investigation — deliberately not extracted

- **External type**: `investigations` **[recorded]**
- **Would map to**: `timeline_comment` on the parent issue (the payload is a `narrative` with `cited_evidence_ids`, which is comment-shaped).
- **Why it is skipped**: retrieval is `POST /v1/findings/{id}/investigations`, and that call *runs* the investigator rather than reading a stored result. Two consequences:
  1. It can be **billable**. The engine gates `bedrock:InvokeModel` behind `COST_AGENT_MODEL_INFERENCE_ENABLED`; with that on, syncing N findings means N model invocations.
  2. **An extractor must be side-effect free.** AirSync re-runs extraction on every sync and on every resumed batch, so a POST-to-read endpoint would multiply those side effects without limit.
- **If it is wanted later**: add it as an explicit, default-off option (`COST_ENGINE_SYNC_INVESTIGATIONS=false`), and only with `COST_AGENT_MODEL_INFERENCE_ENABLED=false` so the deterministic offline investigator runs. Document the cost before enabling.

#### 2.3 Change plan — fields on the parent issue

- **External type**: `change_plan`, embedded in the finding. No separate call is needed, and `POST /v1/findings/{id}/plans` is **not** called: its only recorded response is a 409 ("*is needs_evidence and carries no change plan. A plan is 'a change you may approve'*"), so its success shape is unknown. **[recorded]**
- **DevRev target**: no DevRev leaf type represents a change plan. Its safety-bearing fields are rendered into the issue `body` and, where a custom schema exists, into custom fields: `plan_hash`, `evidence_hash`, `before_state_hash`, `approval_policy`, `irreversibility[]`, `operational_impact`, `validity_deadline`, and each `ordered_operations[].iam_action`.
- **Not negotiable in rendering**: the `irreversibility` statements and `approval_policy` must appear verbatim in the issue body. They are the reason a human is being asked, and summarising them away would defeat the point.

#### 2.4 Scan — context, not a record

`GET /v1/scans/{id}` returns status and findings together. `POST /v1/scans` has no documented list endpoint, so scans cannot be enumerated and are not sync targets. The scan's `source_mode`, `exhaustive` flag, `period` and `totals` are carried into the issue body as provenance.

## 3. Field Mappings Per Entity

### 3.1 Finding → `issue`

| External Field | DevRev Field | Type | Required | Transformation | Notes |
|---|---|---|---|---|---|
| `deduplication_key` | *(record id)* | text | yes | use_directly | Stable sync key. **Not** `finding_id`. |
| `summary` | `title` | text | yes | use_directly | Fallback: `"Untitled cost finding"`. `min_length: 1`. |
| *(composed)* | `body` | rich_text | no | compose | See the body composition below. |
| *(configured)* | `applies_to_part_id` | reference | **yes** | use_devrev_record | `leaf_type: product`. The engine has no product concept, so this is a configured constant — see §10 Open. |
| `state` | `stage` | enum | no | map_enum | Table below. |
| `risk_tier` | `priority` | enum | no | map_enum | Table below. |
| `owner_ref` | `owned_by_ids` | reference[] | no | use_as_array_value | Usually `null` in the samples; omit when absent rather than inventing an owner. |
| `created_at` | *(metadata)* | timestamp | no | use_directly | RFC 3339 UTC, microsecond precision. |
| `updated_at` | *(metadata)* | timestamp | no | use_directly | Used for client-side change detection. |
| — | `item_url_field` | text | no | — | **Omitted.** The engine has no UI, so there is no URL to link back to. Better empty than fabricated. |

#### Enum Mapping: `state` → `stage`

| External Value | DevRev Value | Note |
|---|---|---|
| `needs_evidence` | `in_development` | Evidence is still being gathered. |
| `ready_for_review` | `in_review` | The engine means "there is a change you may approve" — review, not done. |
| `rejected` | `done` | **Imperfect.** DevRev's stock issue stages have no "won't do", so a rejection lands in `done`. The rejection reason is written into the body so the distinction is not lost. Flagged in §10 Open. |

#### Enum Mapping: `risk_tier` → `priority`

| External Value | DevRev Value | Note |
|---|---|---|
| `high_impact` | `P1` | |
| `controlled_operational` | `P2` | |
| `advisory` | `P3` | |
| *(absent)* | `P3` | `use_fixed_value` default; never guess upward. |

#### Body composition

The body is assembled from the engine's own text, quoted rather than paraphrased, in this order:

1. **Provenance banner** — `source_mode`, scan id, `exhaustive`, and for `fixture`/`imported` modes an explicit line that the finding is **not** a live AWS observation.
2. **Root cause** — `root_cause_hypothesis`, then `alternative_hypotheses[]`, then `causal_confidence`.
3. **Savings** — `savings_estimate.steady_state_savings` as `{amount} {currency}` for the verbatim `period_label`, plus `price_basis` and `price_observed_at`. When `unavailable_reason` is set, print that reason and **no figure**. Never convert, never annualise, never total across findings.
4. **Evidence** — `evidence_ids[]` and `missing_evidence[]`.
5. **Change plan** — `irreversibility[]` and `approval_policy` verbatim, `operational_impact`, the ordered operations with their `iam_action`, and `plan_hash` / `evidence_hash`.
6. **Resource** — `service`, `region`, `resource_id`, `technique_id`, `playbook_id`.

#### Child Collections

| Child Type | Endpoint | Pagination | Trigger | Notes |
|---|---|---|---|---|
| change plan | *(inline in the finding)* | — | `change_plan` non-null | No extra request. |
| investigation | `POST /v1/findings/{id}/investigations` | — | **never, by default** | Side-effecting and possibly billable — see 2.2. |
| attachments | *(none)* | — | — | The engine exposes no attachments. Evidence is referenced by id only. |

## 4. Extraction Strategy

### Extraction Order

1. **findings** — no dependencies, single entity.

There is no user entity to extract first, and change plans arrive inline, so the dependency graph is trivial.

### Pagination Models

| Entity | Model | Page Size | Params | Termination |
|---|---|---|---|---|
| findings | **none observed** | n/a | none | Single response; `count` is expected to equal `len(findings)` |

**Generation must assert `count == len(findings)`** and fail loudly on mismatch. If the engine ever paginates without saying so, a silent mismatch would drop findings — a divergence worth crashing over, not logging.

### Historical Sync Strategy

| Entity | Date-range filter supported? | Historical sync approach |
|---|---|---|
| findings | **no** — full enumeration only | Artifact-based spillover (`docs/connector-guides/references/historical-sync-no-date-range.md`) |

Per the planner's Step 9: with no `updated_since` parameter, historical sync must not re-enumerate the whole dataset per batch. Use the artifact-based spillover pattern.

## 5. External Sync Units

- **Concept**: **AWS account scope.** The engine bounds a run by its own `COST_AGENT_SCOPE` file (accounts and regions), so the account is the natural top-level grouping.
- **Discovery endpoint**: `POST /v1/preflight` → `identity.account_id`. There is no list endpoint; one engine instance serves one configured scope. **[recorded]**
- **Expected count**: **1**.
- **Caveat [recorded]**: in the recorded fixture-mode preflight, `identity` is `{account_id: null, caller_arn: null, verified: false}`. So in fixture mode the account id is unavailable and the ESU id must fall back to the configured `COST_ENGINE_SYNC_UNIT_ID`. The extractor must not invent an account id.

| ESU Field | Source | Notes |
|---|---|---|
| `id` | `preflight.identity.account_id`, else `COST_ENGINE_SYNC_UNIT_ID` | Never fabricated. |
| `name` | `"AWS account {id}"`, else `"AWS cost engine ({source_mode})"` | Includes the mode so a fixture-backed unit is visibly not production. |
| `description` | `"Cost findings from the AWS cost engine"` + `source_mode` | |
| `item_count` | `0` | Per template: determined during extraction. |
| `item_type` | `"findings"` | |

## 6. Auth Configuration

- **Connection type**: `aws-cost-engine-connection`
- **Auth method**: PAT / bearer token
- **Key field**: the engine's `COST_AGENT_API_TOKEN`, sent as `Authorization: Bearer <token>`. **[recorded]**
- **Additional fields**: **base URL is required**, not optional — the engine is self-hosted with no canonical host, so the connection must carry `COST_ENGINE_BASE_URL`.

### OAuth2 Details

N/A — the engine supports a static bearer token only. **[recorded]**

### Security notes

- The engine's `.env.example` states it never accepts AWS credentials as configuration; AWS access is resolved through the standard SDK chain. This connector therefore holds **no** AWS credentials, only the engine token. **[recorded]**
- The engine's HTTP surface is optional and may be unauthenticated if `COST_AGENT_API_TOKEN` is unset. The connector must refuse to run without a token rather than silently connecting in the clear.

## 7. Manifest Draft

```yaml
version: '2'
name: 'AWS Cost Engine Extractor'
description: 'Extract AWS cost-optimization findings from the cost engine into DevRev'
service_account:
  display_name: 'AWS Cost Engine Bot'
functions:
  - name: extraction
    description: Extract findings from the AWS cost engine
imports:
  - slug: adaas_aws_cost_engine_import
    display_name: 'AWS Cost Engine Import'
    description: Extract cost-optimization findings from the AWS cost engine.
    extractor_function: extraction
    allowed_connection_types:
      - aws-cost-engine-connection
```

No `loading` function is declared, because nothing is written back — see §8.

## 8. Sync Direction Decision

- **Direction**: **Extract only.**
- **Loadable entities**: none.
- **Conflict resolution**: N/A — nothing is written to the engine.
- **Notes**: the engine exposes no write path for findings. Its only mutating routes create scans, invoke investigations and build plans; none of those is a write-back of DevRev state.

### Duplicate-publishing conflict — resolve before deploying

The engine already has its own DevRev publisher: `POST /v1/findings/{id}/work-preview` returns `published`, `remote_id` and `body`. **[recorded]** That is a second, independent path from finding to DevRev work item.

If both paths run, each finding can become **two** DevRev records, and neither side knows about the other's `remote_id`. Pick one before deploying:

1. **AirSync only** — do not call `work-preview`. Cleanest; AirSync owns identity via `deduplication_key`.
2. **work-preview only** — then this connector is redundant and should not be deployed.
3. **Both, reconciled** — requires the engine's `remote_id` to be surfaced to AirSync so it can adopt rather than re-create. No documented mechanism for this exists.

Recommended: option 1. Recorded in §10 Open as a decision for the engine's owners, not one this plan can settle.

## 9. Dynamic Metadata Strategy

- **Custom fields**: no — finding fields are fixed in engine code, not user-configurable. **[recorded]**
- **Configurable enums**: no — `state`, `risk_tier` and `causal_confidence` are code-level enumerations.
- **Stage diagrams**: no.
- **Scope**: N/A.
- **Notes**: metadata extraction is therefore **purely static**; no runtime enrichment helper is needed. One optional enhancement: `GET /v1/coverage` returns the technique catalog (`technique_id`, `estimation_boundary`, `safety_boundary`, `required_evidence`) and could enrich issue bodies with each technique's safety boundary. Deferred — it is not required for correctness, and the endpoint carries an explicit disclaimer that its statuses are *"per technique and are not transitive"*, which would need faithful rendering.

## 10. Assumptions and Open Questions

### Confirmed (from the recorded samples)

- Routes, methods and status codes in §1 were observed in recorded request/response pairs.
- Auth is a bearer token; the engine's config documents `COST_AGENT_API_TOKEN`.
- Finding states are exactly `needs_evidence`, `ready_for_review`, `rejected`.
- Risk tiers are exactly `advisory`, `controlled_operational`, `high_impact`.
- Money is `{"amount": "4.00", "currency": "USD"}` — a decimal **string** — with a **calendar** `period_label` such as `"2026-10"`, not a frequency.
- The engine refuses to total across incompatible periods: `totals.steady_state_savings` is `null` when `periods_present` holds more than one period.
- Findings carry `deduplication_key`, suitable as a stable external id.
- Errors are `{"detail": "..."}`; a plan request for a `needs_evidence` finding returns 409.
- `POST /v1/scans` accepts no region or service arguments.

### Open — **must be resolved before production use**

1. **Pagination.** No pagination was observed, but the sample scan held only 6 findings. A real account could return thousands. If `GET /v1/findings` paginates undocumented, this connector would silently truncate. *Resolve by:* running a scan with >100 findings and inspecting the response.
2. **Incremental sync.** No `updated_since` parameter was observed. If none exists, every sync enumerates everything. *Resolve by:* checking the engine's route definitions for query parameters.
3. **Rate limits.** No limit headers were observed. *Resolve by:* load-testing the engine, or reading its server config.
4. **Live reachability.** The engine has never been contacted from this machine. Nothing here proves the contract is still current.
5. **Duplicate publishing** vs `work-preview` — see §8. Needs a decision from the engine's owners.
6. **`applies_to_part_id`** is required by DevRev but has no engine equivalent. Which DevRev product should findings attach to? Needs a configured part id.
7. **`rejected` → `done`** loses the "won't do" distinction. Acceptable? Or should rejected findings be skipped at extraction instead?
8. **ESU identity in live mode.** Fixture-mode preflight returns a null `account_id`. Confirm a live preflight returns a real one, otherwise `COST_ENGINE_SYNC_UNIT_ID` is mandatory.
9. **`hidden_by_account_scope`** appears in the list response but its semantics are unobserved. If it counts findings withheld by scope, the connector should surface it rather than ignore it — an incomplete sync that looks complete is the failure mode to avoid.

## 11. File Structure Proposal

```
airdrop-aws-cost-engine-snap-in/
├── manifest.yaml
├── plans/
│   └── aws-cost-engine.md                  # this plan
├── code/
│   ├── package.json
│   ├── .env.example                        # COST_ENGINE_BASE_URL, COST_ENGINE_TOKEN, COST_ENGINE_SYNC_UNIT_ID
│   ├── src/
│   │   ├── function-factory.ts
│   │   ├── index.ts
│   │   └── functions/
│   │       ├── extraction/
│   │       │   ├── index.ts                # State: { findings: { completed, spilloverArtifactId } }
│   │       │   └── workers/
│   │       │       ├── external-sync-units-extraction.ts   # preflight -> 1 ESU
│   │       │       ├── metadata-extraction.ts              # static EDM only
│   │       │       ├── data-extraction.ts                  # findings, full enumeration
│   │       │       └── attachments-extraction.ts           # no-op: engine has no attachments
│   │       └── external-system/
│   │           ├── http-client.ts           # 4 methods: preflight, coverage, listFindings, getFinding
│   │           ├── types.ts                 # Finding, ChangePlan, SavingsEstimate, Preflight, ScanSummary
│   │           ├── data-normalization.ts    # 1 normalizer: finding -> issue
│   │           ├── external_domain_metadata.json
│   │           └── initial_domain_mapping.json
│   └── test/
│       ├── http-client.test.ts              # status mapping incl. 409 and {"detail": ...}
│       ├── normalizers.test.ts              # all three states; absent savings; verbatim period label
│       └── integration.test.ts              # recorded samples replayed as fixtures
```

No `loading/` directory: extraction only, so a loading stub would be dead code.

### Test fixtures

`code/test/fixtures/` should reuse the trimmed recorded samples already prepared at
`../cost-mcp-server/tests/fixtures/engine_samples.json` — one finding per state,
verified credential-free (placeholder account `000000000000`, `example.invalid`,
`source_mode: fixture`).

## 12. Governance and Approvals

- **App registration required**: no — an internal, self-hosted service with a static bearer token. **[recorded]**
- **Registration URL**: N/A.
- **Approval process**: none external. Internally, the §8 duplicate-publishing decision needs the engine owners' sign-off.
- **Required permissions/scopes**: the engine token only. This connector holds **no AWS credentials** and needs no AWS IAM permissions, because it never calls AWS — the engine does. **[recorded]**
- **Data handling requirements**: findings contain AWS account ids, resource ids, regions and cost figures — infrastructure and spend data, which is confidential even though it is not personal data. `owner_ref` may carry a person's identifier, so treat it as PII when populated. The connector must not log finding bodies at info level.
- **Workshop Studio action policy**: the engine sources its mutation allow-list from the Workshop Studio supported/unsupported service lists (`source: pdf_verified` in `config/workshop_actions.yaml`). That governs **execution**, which this read-only extractor never performs; noted so the relationship is not mistaken for a gap. **[recorded]**
- **Marketplace publishing tags**: do **not** publish. Suggested internal tags if listed privately: `aws`, `cost-optimization`, `finops`, `internal`.
- **Notes**: the engine defaults both kill switches off (`COST_AGENT_EXECUTION_ENABLED`, `COST_AGENT_MODEL_INFERENCE_ENABLED`) and defaults `COST_AGENT_SOURCE_MODE` to `fixture`. A connector pointed at a default-configured engine will therefore sync **synthetic** findings. The provenance banner in the issue body (§3.1) exists precisely so that is visible in DevRev rather than mistaken for live data.
