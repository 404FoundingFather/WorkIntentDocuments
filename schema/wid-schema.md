# WID Schema Specification

**Version:** 1.0.0-draft
**Date:** 2026-03-02
**JSON Schema:** [`wid-frontmatter.schema.json`](wid-frontmatter.schema.json)

A Work Intent Document (WID) is a Markdown file with YAML frontmatter. The
frontmatter carries structured, machine-readable metadata. The body carries
human-readable (and agent-readable) narrative content. Together they form a
single artifact that serves as both specification and execution context.

---

## File Format

```
---
<YAML frontmatter>
---

<Markdown body>
```

The frontmatter block is delimited by `---` on its own line. Everything
between the delimiters is parsed as YAML. Everything after the closing
delimiter is GitHub-Flavored Markdown.

## File Naming

```
WID-NNN-slug-here.md
```

- **`WID-NNN`**: Matches the `id` field in frontmatter (authoritative)
- **`slug-here`**: Informational only — a human-readable hint at the content
- The `id` in frontmatter is always authoritative; the filename slug can
  drift without breaking anything

**Why not UUIDs?** WID IDs appear in conversation ("WID forty-two"), commit
messages (`feat(WID-042): ...`), branch names, and filenames. A 36-character
UUID adds noise in every context. Sequential integers are sufficient for
Git repos, which don't face the distributed collision problem UUIDs solve.

## File Organization

```
wids/           # All WID files, flat structure
  WID-001-oauth-google.md
  WID-002-rate-limiting.md
logs/           # Execution artifacts, organized by WID
  WID-001/
    exec-001.md
    exec-002.md
schema/         # This spec + JSON Schema
examples/       # Example WIDs (documentation + test fixtures)
```

WIDs are stored flat (no subdirectories) because hierarchy is expressed via
the `parent` field, not the filesystem. Flat structure simplifies tooling —
glob `wids/WID-*.md` finds everything.

---

## Frontmatter Fields

### Required Fields

Every WID must include these six fields.

#### `id`

| | |
|---|---|
| **Type** | `string` |
| **Pattern** | `WID-NNN` (zero-padded, minimum 3 digits) |
| **Example** | `WID-001`, `WID-042`, `WID-1337` |

The unique, authoritative identifier for this WID. Set once at creation,
never modified. The zero-padded format ensures lexicographic sorting aligns
with numeric sorting for the first 999 WIDs, and the pattern allows any
number of digits beyond three.

#### `title`

| | |
|---|---|
| **Type** | `string` |
| **Constraints** | 1–200 characters |
| **Example** | `"Implement Google OAuth2 Authentication"` |

Human-readable title. Should be specific enough to distinguish from related
WIDs without reading the body. Titles can be updated as understanding
evolves.

#### `type`

| | |
|---|---|
| **Type** | `enum` |
| **Values** | `epic`, `feature`, `task`, `bug`, `spike`, `chore` |

Determines valid parent/child relationships (see [Hierarchy Rules](#hierarchy-rules))
and influences how the execution runtime prioritizes and assigns work.

| Type | Purpose |
|------|---------|
| `epic` | Large initiative containing multiple features/tasks |
| `feature` | User-facing capability; may decompose into tasks |
| `task` | Discrete unit of implementation work |
| `bug` | Defect to be fixed |
| `spike` | Time-boxed investigation to reduce uncertainty |
| `chore` | Maintenance, refactoring, or infrastructure work |

#### `status`

| | |
|---|---|
| **Type** | `enum` |
| **Values** | `draft`, `ready`, `in-progress`, `review`, `done`, `cancelled` |

The current lifecycle state of the WID.

```
draft → ready → in-progress → review → done
  ↓       ↓         ↓            ↓
  └───────┴─────────┴────────────┴──→ cancelled
```

**Design rationale:** The README says status is "primarily inferred from
activity" — but the WID must be self-contained when read outside the engine
(e.g., browsing on Gitea). The inference engine *writes back* to this field
based on activity signals (commits, CI results, PR state). Humans rarely
touch it directly.

| Status | Meaning |
|--------|---------|
| `draft` | WID is being authored; not yet actionable |
| `ready` | Fully specified and available for execution pickup |
| `in-progress` | Actively being worked (agent executing or human coding) |
| `review` | Implementation complete; awaiting review or validation |
| `done` | Acceptance criteria met; work is complete |
| `cancelled` | Abandoned; reachable from any state |

#### `created`

| | |
|---|---|
| **Type** | `string` (ISO 8601 date) |
| **Format** | `YYYY-MM-DD` |
| **Example** | `2026-03-01` |

Set once at WID creation, never modified. The full modification history is
available via Git; this field records when the intent was first articulated.

**YAML gotcha:** Quote date values in frontmatter (`created: "2026-03-01"`)
to prevent YAML parsers from auto-converting them to `datetime.date`
objects, which will fail JSON Schema string validation.

#### `author`

| | |
|---|---|
| **Type** | `string` |
| **Example** | `"@ehammond"`, `"@claude-agent"` |

The creator of the WID. Use `@`-prefixed handles for consistency with
changelog entries and Git commit trailers. Can be a human or an agent.

---

### Optional Fields

#### `parent`

| | |
|---|---|
| **Type** | `string` |
| **Pattern** | `WID-NNN` |

Parent WID in the hierarchy. See [Hierarchy Rules](#hierarchy-rules) for
valid parent/child type combinations.

#### `depends_on`

| | |
|---|---|
| **Type** | `array` of `string` |
| **Items** | `WID-NNN` references, unique |

WIDs that must reach `done` before this WID can begin execution. This
expresses *execution ordering*, which is separate from hierarchy. A task
can depend on a WID that is not its parent or sibling.

#### `blocks`

| | |
|---|---|
| **Type** | `array` of `string` |
| **Items** | `WID-NNN` references, unique |

WIDs that cannot begin until this WID reaches `done`. The logical inverse
of `depends_on` — maintained explicitly for convenience and for tooling
that needs to traverse the dependency graph in either direction.

#### `priority`

| | |
|---|---|
| **Type** | `enum` |
| **Values** | `critical`, `high`, `medium`, `low` |

Used by the execution runtime to order the work queue when multiple WIDs
are in `ready` status. If unset, the runtime applies its own heuristics
(e.g., dependency depth, age).

#### `labels`

| | |
|---|---|
| **Type** | `array` of `string` (unique) |
| **Example** | `["auth", "backend", "security"]` |

Free-form tags for filtering and grouping. No controlled vocabulary is
enforced — conventions emerge per project. Useful for the projection layer
(e.g., filtering a board by label).

#### `assignee`

| | |
|---|---|
| **Type** | `string` |
| **Example** | `"@ehammond"`, `"@claude-agent"` |

Current owner responsible for execution. Blank means unassigned and
available for pickup by the execution runtime. Can be a human or an agent.

#### `estimate`

| | |
|---|---|
| **Type** | `string` |
| **Pattern** | `<number><unit>` where unit is `m` (minutes), `h` (hours), or `d` (days) |
| **Examples** | `"30m"`, `"2h"`, `"0.5d"` |

Estimated effort. Intentionally coarse — precise time estimates are false
precision, especially in the agent era where execution time varies by
model, context, and retry patterns. Useful primarily for the projection
layer's capacity planning views.

#### `due`

| | |
|---|---|
| **Type** | `string` (ISO 8601 date) |
| **Format** | `YYYY-MM-DD` |

Target completion date. A goal, not a guarantee. The projection layer uses
this for deadline-based views and overdue alerts.

#### `branch`

| | |
|---|---|
| **Type** | `string` |
| **Convention** | `wid/WID-NNN-slug` |
| **Example** | `"wid/WID-001-oauth-google"` |

Git branch associated with this WID's implementation. The naming convention
allows tools to correlate branches with WIDs automatically. Not enforced by
schema — some workflows may use different branch strategies.

#### `pr`

| | |
|---|---|
| **Type** | `string` (URI) or `array` of `string` (URIs) |
| **Example** | `"https://gitea.example.com/org/app/pulls/42"` |

Pull request URL(s) implementing this WID. A single string for one PR; an
array for split implementations across multiple repos or stacked PRs.

#### `context`

| | |
|---|---|
| **Type** | `object` |
| **Sub-fields** | `files`, `modules`, `apis` |

Structured codebase context. This is the critical bridge between the WID
and the codebase — it tells agents *where to start* and gives them
semantic descriptions that survive refactors.

**`context.files[]`** — Individual source files:
- `path` (required): Exact file path relative to repo root
- `role`: Semantic description of what this file does for this WID
- `glob`: Broader pattern capturing related files

**`context.modules[]`** — Logical groupings:
- `name` (required): Module or package name
- `description`: Relevance to this WID

**`context.apis[]`** — API endpoints:
- `endpoint` (required): Method + path (e.g., `POST /api/v1/auth/login`)
- `description`: What the endpoint does and why it matters

**Design rationale:** Three layers of specificity handle different failure
modes. `path` gives an exact starting point (fastest). If the file moves,
`role` lets agents relocate it via semantic search. `glob` captures file
sets that naturally include new files. All three together are resilient to
refactoring.

---

### Extension Fields (`x-` prefix)

Any field with a key starting with `x-` is a valid extension field. This
allows project-specific metadata without polluting the core namespace.

```yaml
x-team: platform
x-sprint: "2026-Q1-S3"
x-cost-center: ENG-042
```

The `x-` convention is well-established (HTTP headers, OpenAPI, Docker
labels). Tools that don't recognize an `x-` field ignore it; tools that
do can use it for project-specific filtering, grouping, and reporting.

Extension fields have no type constraints in the schema — they accept any
valid YAML value.

---

## Hierarchy Rules

WID types form a containment hierarchy via the `parent` field:

```
epic
├── feature
│   ├── task      (leaf)
│   └── bug       (leaf)
├── task          (leaf)
├── bug           (leaf)
├── spike         (leaf)
└── chore         (leaf)

feature
├── task          (leaf)
└── bug           (leaf)
```

| Parent Type | Valid Child Types |
|-------------|-----------------|
| `epic` | `feature`, `task`, `bug`, `spike`, `chore` |
| `feature` | `task`, `bug` |
| `task` | *(leaf — no children)* |
| `bug` | *(leaf — no children)* |
| `spike` | *(leaf — no children)* |
| `chore` | *(leaf — no children)* |

**Note:** These rules are enforced by tooling, not by the JSON Schema.
JSON Schema validates individual documents; hierarchy constraints span
multiple documents and require cross-document validation.

The `depends_on`/`blocks` fields are orthogonal to hierarchy. A task can
depend on any WID regardless of where it sits in the parent/child tree.

---

## Body Sections

The Markdown body uses conventional section headings. These are not
enforced by schema or tooling — they're conventions that agents and humans
can rely on for consistent document structure.

### `## Problem Statement`

Why this work exists. What user need, technical debt, or business
requirement motivates it. Should be understandable by someone with no
prior context.

### `## Design Discussion`

The reasoning chain: trade-offs evaluated, alternatives rejected, and why
the chosen approach was selected. This is the section that preserves the
semantic richness that traditional tickets lose. Include enough detail that
a future reader (human or agent) can understand not just *what* was decided
but *why*.

### `## Approach`

The chosen implementation plan, specific enough for an agent to execute
without additional clarification. Numbered steps are preferred. Each step
should be atomic and verifiable.

### `## Acceptance Criteria`

GFM checklists, optionally annotated with YAML code blocks:

```markdown
- [ ] User can authenticate via OAuth2
  ```yaml
  verify: test
  ref: tests/auth/test_oauth.py::test_full_flow
  ```
- [ ] Failed attempts are rate-limited to 5/min
- [ ] Tokens expire after 24 hours
```

**Without annotations:** A perfectly normal checklist — readable by any
human, parseable by any Markdown tool.

**With annotations:** Agents know *how* to verify each criterion:

| `verify` value | Meaning |
|----------------|---------|
| `test` | Run the referenced test |
| `command` | Execute the referenced command and check exit code |
| `manual` | Requires human verification |
| `ci` | Verified by CI pipeline (check referenced job) |

The `ref` field points to the specific test, command, or artifact. Each
criterion diffs independently in Git, making review straightforward.

### `## Codebase Context`

Narrative supplement to the structured `context` in frontmatter. Use this
section for context that doesn't fit the structured format — architectural
relationships, gotchas, relevant prior art, or explanations of why certain
files are included.

### `## Changelog`

Major decisions and inflection points, not every edit:

```markdown
- **2026-03-01** (author: @ehammond) -- Created from design session
- **2026-03-02** (author: @claude-agent) -- Added rate-limiting after spike
```

Git history captures every character change. The inline changelog captures
*decisions* and *why* — the kind of context Git diffs don't convey.

**Commit convention:** `feat(WID-001): add rate limiting criteria`

### `## Execution Log`

Reference table linking to detailed execution logs:

```markdown
| Session | Agent | Started | Duration | Status | Log |
|---------|-------|---------|----------|--------|-----|
| exec-001 | claude-code | 2026-03-04 | 23m | completed | [log](../logs/WID-001/exec-001.md) |
```

Execution logs can be massive (full agent transcripts, command output,
test results). Inlining them would make the WID unreadable. The summary
table gives at-a-glance execution history; detailed logs live in
`logs/{WID-ID}/`.

### `## Notes`

Catch-all for anything that doesn't fit elsewhere. Credentials references,
related reading, future considerations, warnings.

---

## Validation

The JSON Schema (`wid-frontmatter.schema.json`) validates frontmatter
structure and types. It does **not** validate:

- Cross-document constraints (hierarchy rules, dependency cycles)
- Body section presence or ordering
- Semantic consistency (e.g., `done` status with unchecked criteria)

These are the responsibility of the WID Engine's validation layer, which
operates on the full document set.

### Quick Validation

```bash
python -c "
import yaml, json, jsonschema
with open('wids/WID-001-oauth-google.md') as f:
    fm = yaml.safe_load(f.read().split('---')[1])
with open('schema/wid-frontmatter.schema.json') as f:
    schema = json.load(f)
jsonschema.validate(fm, schema)
print('Valid!')
"
```
