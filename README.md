# deep-plan

A Claude Code skill for planning complex features through structured decision-making with rich visual HTML pages.

## What it does

Instead of jumping straight into code, deep-plan forces a deliberate planning phase:

1. **Explores your codebase** — reads relevant files, greps for conventions, records observations
2. **Surfaces 6-12 decisions** across 6 categories (architecture, behavior, data, interface, visual, quality)
3. **Auto-decides trivials** — if the codebase already answers a question, it just picks it
4. **Presents non-trivial decisions as HTML pages** — each with 4 options, visual previews, pros/cons, and a comparison table
5. **Produces an executable implementation plan** with verifiable steps

Everything persists to a `.plan/` directory, so context loss (compaction, new session) is recoverable.

## Install

```bash
claude skill install /path/to/deep-plan
# or from a git repo
claude skill install github:user/deep-plan
```

## Usage

```
/deep-plan add a real-time notification system
/deep-plan migrate auth from sessions to JWT
/deep-plan build an admin dashboard for managing user accounts
```

## How decisions work

Each decision gets an HTML page opened in your browser with:

- **4 option cards** with visual previews (architecture diagrams, flow charts, UI mockups, code snippets)
- **Pros/cons** for each option
- **Comparison table** across 5-8 dimensions
- **A recommendation** with rationale
- **Copy-to-clipboard buttons** — click to copy your choice, paste it back in the terminal

You can also type `"Option A but [modification]"` or `"more options"` to get 4 more.

## What gets generated

```
.plan/
├── state.json              # Current phase, progress, all decision metadata
├── plan.md                 # Final implementation plan with steps
├── context/
│   ├── codebase-notes.md   # Observations from exploration
│   └── conventions.md      # Detected patterns and conventions
├── decisions/
│   └── N-slug.md           # Each decision with evidence, options, rationale
└── pages/
    ├── index.html           # Live dashboard (auto-refreshes every 3s)
    ├── decision-N-prompt.html  # Pre-decision comparison pages
    └── decision-N.html         # Post-decision summary pages
```

## Decision categories

| Category | Covers |
|---|---|
| Architecture | Tech stack, module boundaries, dependencies |
| Behavior | User flows, interaction patterns, state management |
| Data | Schema, storage, caching, migrations |
| Interface | API shape, types, validation, contracts |
| Visual | Design direction, layout, UX patterns |
| Quality | Testing strategy, observability, rollout |

## Templates

The `templates/` directory contains the HTML templates used for generating decision pages and the dashboard. The skill fills in content markers at generation time.
