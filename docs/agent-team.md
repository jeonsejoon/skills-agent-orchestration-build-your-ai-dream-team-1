# Project Pulse agent team

This document summarizes the custom agent team defined in `.github/agents/` and how
the team will work together to build Mona's **Project Pulse** dashboard.

The goal is orchestration: instead of one large prompt, a coordinating agent
delegates scoped work to specialists, each with its own model, responsibility,
and file ownership.

## The team

| Agent | Model | Responsibility | Definition file |
| --- | --- | --- | --- |
| **Orchestrator** | Claude Opus 4.7 (copilot) | Coordinates the whole request. Breaks the Project Pulse brief into phases, delegates to the specialists, assigns explicit file scopes, decides what runs in parallel versus sequentially, and verifies the integrated result. Does not implement anything itself. | `.github/agents/orchestrator.agent.md` |
| **Planner** | Claude Opus 4.7 (copilot) | Researches the repository and produces the implementation plan: ordered steps, file assignments, dependencies, parallel versus sequential work, edge cases, validation expectations, and open questions. Writes no code. | `.github/agents/planner.agent.md` |
| **Coder** | GPT-5.5 (copilot) | Implements the static Project Pulse app files within the scope the Orchestrator assigns, and creates support configuration such as `.vscode/launch.json` as strict JSON when that work is assigned. | `.github/agents/coder.agent.md` |
| **Designer** | Gemini 3.1 Pro (copilot) | Owns UI/UX: layout, information hierarchy, accessibility, responsive behavior, and visual polish. Provides the deterministic CSS hooks (`.dashboard`, `.project-card`), status badges, priority treatment, rounded corners, and shadows that make the page read as a real dashboard. | `.github/agents/designer.agent.md` |

## Agent details

### Orchestrator — Claude Opus 4.7

- **Definition:** `.github/agents/orchestrator.agent.md`
- **Tools:** `read`, `agent`, `memory`
- **Role:** The entry point. I select it in Copilot CLI with `/agent`, hand it the
  Project Pulse request, and it delegates outward.
- **Rules it follows:** describe outcomes rather than techniques, give every
  specialist an explicit file scope, keep overlapping file scopes in separate
  phases, summarize progress after each phase, and surface blockers instead of
  hiding them.

### Planner — Claude Opus 4.7

- **Definition:** `.github/agents/planner.agent.md`
- **Tools:** `read`, `search`, `web`, `memory`, `todo`
- **Role:** Turns the brief in `.github/project-pulse-brief.md` into a plan the
  Orchestrator can execute, including ownership of `app/index.html`,
  `app/styles.css`, `app/project-data.json`, and `.vscode/launch.json`.
- **Output goes to:** `docs/project-pulse-plan.md`

### Coder — GPT-5.5

- **Definition:** `.github/agents/coder.agent.md`
- **Tools:** `read`, `edit`, `search`, `execute`, `web`, `memory`, `todo`
- **Role:** Builds the static app. Keeps control flow simple, names things
  descriptively, makes errors explicit, and validates before reporting done.
- **Likely files:** `app/index.html`, `app/project-data.json`, `.vscode/launch.json`
  (launch configuration named **Run Project Pulse Dashboard**, `cwd` set to
  `${workspaceFolder}/app`, opening `index.html`).

### Designer — Gemini 3.1 Pro

- **Definition:** `.github/agents/designer.agent.md`
- **Tools:** `read`, `edit`, `search`, `web`, `memory`, `todo`
- **Role:** Makes the first view unmistakably a Project Pulse dashboard rather
  than a bare HTML page.
- **Likely files:** `app/styles.css`, plus markup guidance for `app/index.html`.

## How the team will work together

1. **Kickoff.** In the Codespace terminal I run
   `copilot --allow-all --enable-all-github-mcp-tools`, then `/agent` and select
   **Orchestrator**, and give it the Project Pulse dashboard request.
2. **Plan.** The Orchestrator asks the **Planner** for an implementation plan with
   phases, dependencies, and file ownership — including `.vscode/launch.json` in
   the file assignments. The plan is saved to `docs/project-pulse-plan.md`.
3. **Phase split.** The Orchestrator parses the plan into phases. Work runs in
   **parallel** only when file scopes do not overlap and there is no data
   dependency; it runs **sequentially** when files overlap or one step depends on
   another's output.
4. **Design and implementation.** The **Designer** owns `app/styles.css` and the
   visual direction (`.dashboard` and `.project-card` hooks, status badges,
   `border-radius`, `box-shadow`, readable spacing, responsive layout). The
   **Coder** owns `app/index.html`, `app/project-data.json` (a top-level
   `projects` key where each project has `name`, `owner`, `status`,
   `recentActivity`, and `priority`), and `.vscode/launch.json`.
5. **Integration check.** The Orchestrator verifies the pieces hang together: the
   markup uses the Designer's classes, the data renders into project cards, and
   the **Run Project Pulse Dashboard** launch configuration serves from the app
   directory and opens `index.html`.
6. **Handoff.** The Orchestrator reports the final outcome, which becomes the
   validation and handoff summary in `docs/final-handoff.md`.

## Git control

None of the agents stage, commit, or push. Every git operation stays with me and
is requested through Copilot CLI prompts.
