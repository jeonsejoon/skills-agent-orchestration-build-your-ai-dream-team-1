# Project Pulse implementation plan

Produced by the **Planner** at the Orchestrator's request, based on
`.github/project-pulse-brief.md` and the agent definitions in `.github/agents/`.

## Goal

Mona's team needs a lightweight **Project Pulse** dashboard that lets contributors
see, at a glance, which projects are active, who owns each one, its current
status, recent activity, and its priority or risk level.

The deliverable is a small static app that runs locally from VS Code:

- `app/index.html` — dashboard markup and the small script that renders project cards
- `app/styles.css` — the polished dashboard styling
- `app/project-data.json` — the project data source
- `.vscode/launch.json` — the **Run Project Pulse Dashboard** launch configuration

## File assignments

| File | Owner | What it must contain |
| --- | --- | --- |
| `app/project-data.json` | **Coder** | A top-level `projects` key holding an array. Each project object has `name`, `owner`, `status`, `recentActivity`, and `priority`. This is the contract every other file reads from. |
| `app/styles.css` | **Designer** | The `.dashboard` layout container and `.project-card` card styling, status badges, priority treatment, `border-radius`, `box-shadow`, readable spacing, responsive grid, and accessible contrast. |
| `app/index.html` | **Coder**, following the Designer's class contract | Semantic document skeleton, the `.dashboard` container, and the render logic that fetches `project-data.json` and emits one `project-card` element per project. |
| `.vscode/launch.json` | **Coder** | Strict JSON with no comments. One configuration named **Run Project Pulse Dashboard**, `cwd` set to `${workspaceFolder}/app`, serving the app directory and opening `index.html` rather than a directory listing. |

## Agent responsibilities

### Designer (Gemini 3.1 Pro — `.github/agents/designer.agent.md`)

- Owns `app/styles.css` outright.
- Defines the class contract the markup must use: `.dashboard`, `.project-card`,
  plus status badge and priority modifier classes.
- Decides information hierarchy: project name first, then owner and status, then
  recent activity, with priority visually distinct.
- Guarantees accessibility (contrast, focus states, readable type scale) and
  responsive behavior from narrow to wide viewports.
- Does **not** edit `app/index.html`, `app/project-data.json`, or
  `.vscode/launch.json`.

### Coder (GPT-5.5 — `.github/agents/coder.agent.md`)

- Owns `app/index.html`, `app/project-data.json`, and `.vscode/launch.json`.
- Writes the data file first so the render logic and the styling agree on field
  names.
- Keeps the render logic explicit and simple: fetch, guard against a failed
  fetch, iterate `projects`, build one card per entry.
- Writes `.vscode/launch.json` as strict JSON with no comments.
- Does **not** edit `app/styles.css`.

## Implementation phases

### Phase 1 — Data contract (sequential, blocks everything)

- **Coder** creates `app/project-data.json` with the top-level `projects` key and
  the five required fields per project.
- Nothing else can be finished until the field names are fixed, because both the
  markup and the styling key off them.

### Phase 2 — Design contract (sequential, blocks the markup)

- **Designer** publishes the class contract (`.dashboard`, `.project-card`, badge
  and priority class names) so the Coder can target it.

### Phase 3 — Parallel build

Once phases 1 and 2 are settled, two tracks run with **no overlapping file
scopes**, so they proceed in parallel:

- **Track A — Designer:** `app/styles.css`
- **Track B — Coder:** `app/index.html` and `.vscode/launch.json`

### Phase 4 — Integration (sequential)

- **Orchestrator** verifies the markup uses the Designer's classes, the data
  renders into visible cards, and the launch configuration opens the dashboard.

## Dependencies

- `app/index.html` **depends on** `app/project-data.json` — the render loop reads
  `projects[].name`, `.owner`, `.status`, `.recentActivity`, and `.priority`.
- `app/index.html` **depends on** the Designer's class contract from
  `app/styles.css` — the markup must emit `.dashboard` and `.project-card`.
- `app/styles.css` **depends on** the same class contract, which the Designer owns,
  so the dependency is one-directional and does not create a cycle.
- `.vscode/launch.json` **depends on** `app/index.html` existing at the path it
  opens, but on nothing else — it has no dependency on the styling or the data.

## Parallel work decisions

- **Runs in parallel:** `app/styles.css` (Designer) alongside `app/index.html` and
  `.vscode/launch.json` (Coder). Different owners, disjoint file scopes, and the
  shared contracts are already fixed by phases 1 and 2.
- **Runs in parallel:** `.vscode/launch.json` can be written at any point after the
  `app/` directory layout is agreed; it never contends with the other files.
- **Must run sequentially:** `app/project-data.json` before `app/index.html`,
  because changing a field name after the fact breaks the render loop.
- **Must run sequentially:** the Designer's class contract before the Coder writes
  markup, otherwise the two agents invent different class names.
- **Must run sequentially:** integration review after both tracks report done.
- **Never parallel:** two agents editing the same file. `app/index.html` stays with
  the Coder even though it carries the Designer's classes; the Designer supplies
  the contract, not the edit.

## Edge cases to handle

- `fetch` of `project-data.json` fails or the file is served from a `file://`
  origin — show an explicit error message instead of an empty page.
- `projects` is empty — show a clear empty state, not a blank dashboard.
- A project is missing an optional field — render a placeholder rather than
  `undefined`.
- Long project or owner names — wrap rather than overflow the card.
- Opening the server at the directory root — the launch configuration must open
  `index.html` so learners see the dashboard, not a directory listing.

## Validation expectations

The work is done when this validation passes:

1. All four files exist: `app/index.html`, `app/styles.css`,
   `app/project-data.json`, and `.vscode/launch.json`.
2. `app/project-data.json` parses as JSON and has a top-level `projects` array
   whose entries each carry `name`, `owner`, `status`, `recentActivity`, and
   `priority`.
3. `.vscode/launch.json` parses as strict JSON with no comments and contains a
   configuration named **Run Project Pulse Dashboard** with `cwd` set to
   `${workspaceFolder}/app`.
4. `app/index.html` contains the `.dashboard` container and produces
   `project-card` elements.
5. `app/styles.css` defines `.dashboard` and `.project-card` and uses
   `border-radius` and `box-shadow` for the polished treatment.
6. Running the **Run Project Pulse Dashboard** configuration opens `index.html`
   and shows populated project cards — not a directory listing, not an empty grid.
7. The layout stays readable and does not overflow at narrow widths.

## Open questions

- How many sample projects should ship in `app/project-data.json`? The plan
  assumes a handful, enough to show the grid wrapping.
- Should `priority` be a fixed vocabulary (High / Medium / Low) so the Designer can
  style each value, or free text? The plan assumes a fixed vocabulary.
- Which port should the launch configuration use? Any free port works; the plan
  assumes a single deterministic choice agreed with the Coder.
