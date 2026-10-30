# Data Label Authority: im-ctrl as policy administration point for the llm-tune library

**Date:** 2026-10-30
**Status:** Requested by llm-tune DCS project (`~/projects/llm-tune/.planning/`, `docs/DCS-PLAN.md`); design draft, not approved
**Scope:** New entity types, relations, validation and an unattended export so im-ctrl becomes the system of record for
data labels, need-to-know categories, document precedence and NCMS (STANAG 5636-shaped) core metadata. im-ctrl never
sits on a live access decision: it authors, exports, and the consumer (llm-tune generator → signed bundle → OPA) decides.

---

## Why im-ctrl, and why not yet

llm-tune (fine-tuning data library, Argilla review, models) is getting zero-trust, data-centric security: every record
carries a STANAG 4774-shaped label and a 4778-style binding, OPA decides per object, Argilla becomes a projection. The
user chose im-ctrl as the authority for labels, policy, precedence and core metadata (2026-10-30). Architecture research
on the llm-tune side found:

- im-ctrl has no machine auth: every `/api/v1` and `/api/data` route sits behind a session cookie
  (`src/auth/middleware.rs` `require_auth`), login is argon2 + CSRF, no bearer or OIDC.
- im-ctrl has no label concept; `entities.label` is a display name.
- The usable surface is `GET /api/data/export?format=json|jsonld&types=...` (needs `settings.manage` and a session).
- It is probably not deployed on three (unverified).

Consequence: llm-tune runs phases 0–3 (label module, schemas, OPA with policy compiled from git `labels.yaml`, manifest
and encryption) with **git as the interim writable authority**. im-ctrl is needed at the **authority cut-over** (llm-tune
phase "im-ctrl authority track", after schema freeze, estimated weeks away). If im-ctrl is not ready then, git stays the
authority; what is lost is precedence modelling and second-owner scoping, not security.

Rule inherited from the llm-tune pitfalls research: **exactly one writable authority at a time.** Every other copy
(`labels.yaml`, OPA data, Argilla stamps, authentik blueprint, manifest) is generated, carries a hash header, and a check
rejects hand edits.

---

## Label model (the part that shapes the ontology)

Two axes, as in STANAG 4774 and Janus 3.0 `src/shared/labels`:

| axis | type | test |
|---|---|---|
| classification (grading) | one value from an **ordered** list per policy (`ugradert < begrenset < konfidensielt < hemmelig`) | subject clearance ≥ object classification |
| categories (need-to-know) | a **set** of compartments (`public`, `fmn`, `mip4-client`, `personal-vidar`, …), flat, unordered | subject categories ⊇ object categories (allOf) |

**Compartments, not a linear rank.** llm-tune's current `labels.yaml` orders the need-to-know tags
(`order: [public, fmn, mip4-client, personal-vidar]`) and resolves a mixed dataset to the single most restrictive tag.
That makes `personal-vidar` dominate `mip4-client`, so someone cleared for personal data would see MIP4 client data. Under
the subset test a dataset built from fmn + mip4 sources requires **both** `fmn` and `mip4-client` clearances, and a
`personal-vidar` clearance grants nothing about MIP4. A derived asset's categories are the **union** of its inputs,
computed, never hand-declared. `public` is the empty compartment set, not a rank.

Policy identifier: `llm-tune-lab-v1` today; the model allows several policies (per owner, per coalition) with their own
classification lists and category vocabularies.

---

## Entity types to add (EAV, no schema migration)

| entity_type | purpose | key properties |
|---|---|---|
| `security_policy` | 4774 PolicyIdentifier | `identifier`, `version`, `description`, `owner` |
| `classification` | ordered grading value within a policy | `rank` (int), `marking` (display text, NO/EN), `policy` (via relation) |
| `category` | need-to-know compartment | `code`, `description`, `owner`, `clearance_group` (authentik group name) |
| `data_asset` | anything that carries a label: dataset, record shard, model, index, projection | `identifier` (stable id = llm-tune dataset name or manifest id), `kind` (terms, passage, traj, lesson, mix, model, index), `tier_allowed` (core, fog, edge), NCMS/5636 fields below |
| `document` (exists) | governing documents (CSRS Spiral 6, MIM 5.3, COPD, AJP) | add `doc_level` (policy, standard, specification, guidance, detail), `effective_from`, `effective_to`, `identifier` |
| `owner` | corpus owner (person or organisation) for second-owner readiness | `name`, `contact`, `rights`, `permitted_purposes` |

NCMS (STANAG 5636-shaped) properties on `data_asset` and `document`: `title`, `creator`, `created`, `modified`,
`language`, `rights` (licence), `source` (URL or path), `type`, `identifier`. Shaped by, not conformant to, the standard:
the ADatP text has not been read.

### Relations to add

| relation_type | source → target | meaning |
|---|---|---|
| `in_policy` | classification, category → security_policy | vocabulary membership |
| `classified_as` | data_asset, document → classification | exactly one per asset (validation) |
| `in_category` | data_asset, document → category | zero or more; set semantics |
| `owned_by` | data_asset, document → owner | one |
| `derived_from` | data_asset → data_asset | inputs of a mix, model, index, projection; label = union/high-water mark of inputs (computed) |
| `governed_by` | document → document | precedence: the target governs the source on conflict |
| `supersedes` | document → document | newer replaces older; `effective_from` decides which is in force on a date |
| `part_of` | document → document | hierarchy for "where to look for detail" |
| `cleared_for` | user → category | optional mirror of authentik groups; authentik stays the subject attribute source, this is for display and checks only |

### Validation (server-side, on create/update/import)

1. A `data_asset` or `document` has exactly one `classified_as`; its classification belongs to the asset's policy.
2. `in_category` targets belong to the same policy.
3. For an asset with `derived_from` edges: classification = max over inputs, categories = union over inputs; a stored
   value that differs fails validation (or is overwritten with an audit entry, decide at implementation).
4. `governed_by` and `supersedes` graphs are acyclic.
5. `tier_allowed` for a derived asset ⊆ intersection of its inputs' `tier_allowed`.
6. Deleting a classification or category that is in use is refused.

---

## Export for the consumer

The llm-tune generator reads one export and emits three generated files: `data/labels.yaml` (the contract the Python
scripts read), OPA `data.json` (policy data for Rego), and OpenTDF attribute definitions (when that PoC runs). A diff
check on a test matrix guards the two decision engines against drift.

Needed from im-ctrl:

1. **Export scope**: `GET /api/data/export?format=jsonld&types=security_policy,classification,category,data_asset,document,owner`
   including relations and properties above, plus an `export_revision` (monotonic counter or content hash) and the
   exporting user. The llm-tune bundle stamps this revision into every OPA decision.
2. **Unattended access**, cheapest first:
   - a service-account user with `settings.manage` plus a login script that keeps a session cookie (acceptable at
     cut-over; human-gated export is acceptable before that);
   - a scoped API token (bearer) for export only, stored in the three stack's `.env`;
   - OIDC via authentik, which also puts im-ctrl behind the same identity as Argilla and GitLab on three (see
     `2026-02-21-enterprise-infrastructure-design.md` if it covers OIDC; otherwise new work).
3. **Import path** for the initial load: `POST /api/data/import` through data-manager `upsert`, since the seed only runs
   on an empty database. Initial content = today's `labels.yaml` (32 datasets, 15 models, 4 workspaces) converted by a
   one-off script on the llm-tune side.
4. **Audit**: `entity.created/updated/deleted` call `audit::log`; verify that data-manager import and the export call do
   too, and that `audit_entry` records the export revision. Every label change must be attributable.

---

## Deployment

im-ctrl is not known to run on three. For the authority role it must: run on three in `/opt/stacks/im-ctrl` behind the
edge Caddy and authentik like the other apps; keep Postgres (and Neo4j if the graph sync is used) on the internal network;
be reachable by the llm-tune generator on three only. Check the memory list of three apps before assuming anything.

---

## Acceptance criteria (for the im-ctrl side)

- [ ] The six entity types and nine relations exist with the properties above; `make dev` seed includes the
      `llm-tune-lab-v1` policy with the four classifications and four categories.
- [ ] Validation rules 1–6 enforced; tests cover a derived asset whose stored label disagrees with its inputs.
- [ ] Export returns the labelled graph with `export_revision`; a second identical export yields the same revision.
- [ ] Unattended export works from a script on three with a non-interactive credential, and the call is in the audit log.
- [ ] Import of the converted `labels.yaml` round-trips: export → generator → `labels.yaml` byte-identical to the input.
- [ ] Documented in `docs/` per project rule; BACKLOG.md row F.4 updated.

---

## Open questions

- Should `cleared_for` live in im-ctrl at all, or is authentik the only subject-attribute source? (Proposal: authentik
  only; im-ctrl mirrors for display.)
- Norwegian classification values and NSM grunnprinsipper mapping: not researched; the ordered list above is a placeholder.
- Does the Neo4j graph sync need the new relation types, or is Postgres enough for the export?
- Policy rules themselves (which action a subject may take on an object class): stay in Rego on the llm-tune side, or
  become `policy_rule` entities here? Proposal: Rego first; move to entities when a second owner needs to author rules.
