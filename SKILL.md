---
name: deep-plan
description: "Deep planning skill that explores the codebase first, surfaces decisions with rich visual comparison pages (4 options with previews, pros/cons, comparison tables), auto-decides trivials, tracks everything in .plan/ for context recovery, and produces executable implementation plans."
argument-hint: "[describe the feature or project you want to plan]"
---

# Deep Plan

You are planning a complex feature or project. Your job is to explore the codebase deeply, surface every meaningful decision, present them with rich visual HTML pages, get user input on non-trivial ones, auto-decide trivials, and produce an implementation plan with executable steps that survives context loss.

**The user's request is:** $ARGUMENTS

## Core Principles

- **Read before deciding.** Never propose anything without reading relevant code. Cite `file:line` as evidence.
- **Match conventions.** Grep for existing patterns. Follow them. Don't invent new ones.
- **Smallest viable change.** Plan the minimal diff that achieves the goal.
- **Every step is verifiable.** Each implementation step defines how to verify it works.
- **Persist everything.** Save state to `.plan/` after every decision. If context compresses, you can recover.
- **Auto-decide trivials.** If the codebase makes the answer obvious, decide it yourself.
- **Rich visual decisions.** Every non-trivial decision gets a full HTML page with 4 options, visual previews, pros/cons, and a comparison table.
- **Plain English everywhere.** Explain things like you're talking to a smart colleague.
- **Always 4 options.** Not 3, not 5. Exactly 4 per decision (unless user asks for more).
- **Always recommend.** Mark one option as recommended and explain why.

---

## PHASE 1 — Understand the Request

Read `$ARGUMENTS` carefully. If the request is clear enough to identify what part of the codebase is involved, proceed to Phase 2.

If too vague, ask 1-2 focused questions. Don't ask what you can grep:
- "Is this a new module or extending an existing one?"
- "Should this be user-facing or internal-only?"

**Then:** Create the `.plan/` directory and initialize state:

```bash
mkdir -p .plan/decisions .plan/context .plan/pages
```

Write `.plan/state.json`:
```json
{
  "feature": "[name]",
  "description": "[1-sentence summary]",
  "phase": "explore",
  "session_started_at": "[ISO timestamp]",
  "phase_entered_at": "[ISO timestamp]",
  "last_checkpoint_at": null,
  "decisions_total": 0,
  "decisions_made": 0,
  "decisions_auto": 0,
  "decisions": [],
  "plan_generated_at": null
}
```

Each entry in `decisions` array follows this schema:
```json
{
  "id": 1,
  "slug": "the-slug",
  "title": "Human Readable Title",
  "category": "architecture|behavior|data|interface|visual|quality",
  "status": "pending|decided|auto",
  "chosenOption": null,
  "chosenTitle": null,
  "options": ["A", "B", "C", "D"],
  "recommended": "B",
  "decidedAt": null,
  "summary": "One sentence about what this decision is about"
}
```

---

## PHASE 2 — Deep Codebase Exploration

Explore the codebase to understand what exists before surfacing decisions.

**Required actions:**
1. **Read every file the feature will likely touch.** Use Glob to find them, Read to understand them.
2. **Grep for conventions:** error handling, naming, test patterns, API styles, state management.
3. **Record observations** in `.plan/context/codebase-notes.md` with timestamps:
   ```markdown
   ## Observations
   - [2026-03-27T14:33:00Z] Found 12 REST endpoints in `src/api/` using Express
   - [2026-03-27T14:34:00Z] Tests use vitest with `describe/it` pattern
   ```
4. **Record conventions** in `.plan/context/conventions.md`:
   ```markdown
   ## Detected Conventions
   - [2026-03-27T14:33:00Z] **File naming:** kebab-case for files, PascalCase for components
   - [2026-03-27T14:34:00Z] **Error handling:** try/catch, errors thrown as `AppError`
   ```

**Update** `state.json`: set `phase` to `"decide"`, update `phase_entered_at`.

---

## PHASE 3 — Surface Decision Roadmap

Based on exploration, identify **6-12 decisions** across these 6 categories:

| Category | What it covers |
|---|---|
| **Architecture** | Tech stack, framework, module boundaries, code structure, dependencies |
| **Behavior** | How it works, user flows, interaction patterns, edge cases, state management |
| **Data** | Schema, queries, storage, caching, migrations |
| **Interface** | API shape, types, validation, public surface area, contracts |
| **Visual** | Design direction, component style, layout, navigation, UX patterns |
| **Quality** | Testing strategy, observability, rollout, monitoring |

### Classifying Trivial vs Non-Trivial

**Trivial** (auto-decide) when:
- Codebase already establishes a clear convention
- Only one sane option given the existing stack
- Easily reversible and low-impact

**Non-trivial** (ask user) when:
- Multiple valid approaches with real trade-offs
- Hard to reverse (schema changes, public API shape)
- Significantly affects architecture or user experience
- User likely has a preference

### Present the Roadmap

> Here's what we need to figure out for [feature]. I've identified N decisions:
>
> **Will ask you (M non-trivial):**
> 1. [Title] — [Category] — [1-sentence why]
> 2. ...
>
> **I'll decide (K trivial):**
> - [Title] — [reason it's obvious]
> - ...
>
> Let me know if you want to weigh in on any trivial ones, or if the list looks right.

Wait for user acknowledgment. **Update** `state.json`: set `decisions_total`, `decisions_auto`, populate `decisions` array.

**Generate the initial dashboard** now: read `templates/dashboard.html`, fill markers with all decisions as pending/upcoming, write to `.plan/pages/index.html`, and open it: `open .plan/pages/index.html`. The dashboard auto-refreshes every 3 seconds, so keep it open throughout the session — it will update live as decisions are made.

---

## PHASE 4 — Decision Loop

### 4a. Explore

Read code specifically relevant to THIS decision. Don't rely on Phase 2 alone — go deeper on specific files, interfaces, and patterns that matter here.

### 4b. Generate Decision Prompt Page

1. **Read** `templates/decision-prompt.html` from the skill's directory for the page structure and CSS
2. **Generate** `.plan/pages/decision-N-prompt.html` by filling the template markers:
   - `{{PROJECT_NAME}}` — feature name
   - `{{DECISION_NUMBER}}` — plain number (e.g. "3", NOT "003")
   - `{{DECISION_TOTAL}}` — total decisions
   - `{{DECISION_TITLE}}` — human-readable title
   - `{{CATEGORY}}` — one of: `architecture`, `behavior`, `data`, `interface`, `visual`, `quality`
   - `{{CATEGORY_LABEL}}` — display name (e.g. "Architecture")
   - `{{DECISION_DESCRIPTION}}` — 3-5 sentences in plain English
   - `{{EVIDENCE_CONTENT}}` — what the code says, citing `file:line`
   - `{{OPTION_CARDS}}` — 4 option cards (see Card Structure below)
   - `{{COMPARISON_TABLE}}` — comparison table (see Table Structure below)
   - `{{RECOMMENDED_LETTER}}` — e.g. "B"
   - `{{RECOMMENDATION_REASON}}` — 1-2 sentences

3. **Update the dashboard** (see Dashboard section)
4. **Open in browser:** `open .plan/pages/decision-N-prompt.html`
5. **Tell the user:**

> **Decision N: [Title]** ([Category])
>
> Opened comparison page in browser. 4 options with visual previews and trade-offs.
> I recommend **Option [X]** — [1-sentence reason].
>
> Click a button on the page to copy your choice, then paste it here. Or type: **"Option A but [change]"** · **"more options"**

**Wait for the user's response. Do not proceed until this decision is resolved.**

#### Option Card HTML Structure

Each card in `{{OPTION_CARDS}}` follows this structure:

```html
<article class="option-card option-a">
  <div class="card-header">
    <div class="card-header-top">
      <span class="option-label">Option A</span>
      <div class="card-badges">
        <!-- Include recommended-badge ONLY on the recommended option -->
        <span class="recommended-badge">Recommended</span>
        <!-- chosen-badge is always present but hidden via CSS until .chosen class added -->
        <span class="chosen-badge">Chosen</span>
      </div>
    </div>
    <h2 class="option-title">[2-4 Word Evocative Name]</h2>
  </div>

  <div class="visual-preview">
    <!-- CONTEXT-DEPENDENT VISUAL — see Visual Preview Rules -->
  </div>

  <div class="option-summary">
    [3-4 sentences. What does choosing this mean? How does it affect the project?
     Write conversationally — no jargon without explanation.]
  </div>

  <div class="verdict">
    <div class="pros">
      <h3>Works well when</h3>
      <ul>
        <li>[Specific context where this shines]</li>
        <li>[Another strength]</li>
        <li>[A third if genuinely useful]</li>
      </ul>
    </div>
    <div class="cons">
      <h3>Watch out for</h3>
      <ul>
        <li>[Honest trade-off]</li>
        <li>[Another trade-off]</li>
      </ul>
    </div>
  </div>

  <div class="card-footer">
    <button class="choose-btn" onclick="copyChoice('A', this)">
      <span class="btn-icon">&#x2398;</span> Choose Option A
    </button>
  </div>
</article>
```

The button copies "Option A" to the user's clipboard on click, with visual feedback (green confirmation + toast). The `copyChoice` JS function is included in the template.

Use classes `option-a`, `option-b`, `option-c`, `option-d` (and `option-e` through `option-l` for extra options).

#### Comparison Table Structure

```html
<table class="comparison-table">
  <thead>
    <tr>
      <th></th>
      <th class="col-a [col-recommended]">Option A: [Name]</th>
      <th class="col-b [col-recommended]">Option B: [Name]</th>
      <th class="col-c [col-recommended]">Option C: [Name]</th>
      <th class="col-d [col-recommended]">Option D: [Name]</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>[Dimension]</td>
      <td class="[col-recommended]">[Value]</td>
      <td class="[col-recommended]">[Value]</td>
      <td class="[col-recommended]">[Value]</td>
      <td class="[col-recommended]">[Value]</td>
    </tr>
    <!-- 5-8 rows -->
  </tbody>
</table>
```

Add `col-recommended` class to ALL cells in the recommended column. After choosing, replace with `col-chosen`.

### 4c. Handle User Response

#### Choosing an option

When the user picks (e.g. "Option B", "B", "the second one"):

1. **Update the prompt HTML**: Add `.chosen` to selected card, `.not-chosen` to others
2. **Generate post-decision page**: Read `templates/decision-page.html`, fill markers, write to `.plan/pages/decision-N.html`
3. **Write decision markdown** to `.plan/decisions/N-slug.md`:
   ```markdown
   # Decision N: [Title]

   - **Category:** [category]
   - **Decided at:** [ISO timestamp]
   - **Chosen:** Option [X] — [Name]
   - **Auto-decided:** no

   ## Evidence
   [What the agent found — specific files, lines, patterns]

   ## Options Considered
   - **A: [Name]** — [description]
   - **B: [Name]** — [description]
   - **C: [Name]** — [description]
   - **D: [Name]** — [description]

   ## Rationale
   [Why this option, referencing evidence]

   ## Acceptance Criteria
   [How to verify this was implemented correctly]

   ## Risks
   [Security, perf, race condition concerns, or "None"]
   ```
4. **Update state.json**: Set decision status to `"decided"`, increment `decisions_made`, update `last_checkpoint_at`
5. **Update dashboard**
6. **Confirm and proceed:**

> Got it — going with Option B ('[Name]'). Next up: **Decision N+1 — [Title]**...

#### "Option A but [modification]"

1. Regenerate the modified option in the HTML
2. Rewrite the prompt HTML with the updated card
3. Re-open: `open .plan/pages/decision-N-prompt.html`
4. Tell user to review the updated option

#### "More options"

1. Read existing prompt HTML to see current options
2. Determine next batch (E-H, I-L, etc.)
3. Generate 4 new meaningfully different options
4. Append to the grid and extend the comparison table
5. Rewrite the full HTML, re-open in browser
6. Update state.json: extend `options` array
7. Tell user: "Added Options E-H — now 8 options on the page."

#### Changing a past decision

1. Update the prompt HTML (move `.chosen`/`.not-chosen` classes)
2. Regenerate the post-decision page
3. Update decision markdown
4. Update state.json
5. Update dashboard
6. Re-open: `open .plan/pages/decision-N-prompt.html`
7. Flag downstream impacts if any

### 4d. Auto-decisions

For trivial decisions, write the same markdown to `.plan/decisions/N-slug.md` but with `Auto-decided: yes` and rationale. Generate a post-decision page but don't present individually — they were shown in the roadmap.

### 4e. Checkpoint

After EVERY decision:
- Update `.plan/state.json`
- Update `.plan/plan.md` (see Context Recovery)
- Regenerate `.plan/pages/index.html` (dashboard)

---

## PHASE 5 — Review Pass

After all decisions are made:

1. Read all decision files in `.plan/decisions/`
2. Check for conflicts (e.g. chose REST in decision 3 but tRPC types in decision 5)
3. Flag downstream impacts
4. Present review:

> All decisions locked in. Quick review:
> - [N] decided by you, [K] auto-decided
> - **No conflicts detected** (or: **Potential conflict:** [description])
> - [Risks worth highlighting]
>
> Ready to generate the implementation plan?

Wait for confirmation. **Update** `state.json`: set `phase` to `"plan"`.

---

## PHASE 6 — Generate Implementation Plan

### 6a. Write plan.md

Generate `.plan/plan.md`:

```markdown
# Implementation Plan: [Feature]

## Summary
[2-3 sentences. What we're building and why.]

## Decisions Summary
| # | Decision | Choice | Category |
|---|----------|--------|----------|
| 1 | [Title]  | [Chosen option] | [Category] |

## Steps

### Step N: [Title]
- **Phase:** [logical group]
- **Files:** `path/to/file.ts:L42` (modify), `path/to/new.ts` (create)
- **Convention:** Follows [pattern] seen in `existing-file.ts:L18`
- **What to do:** [Concrete description]
- **Verify:** [Command or check]
- **Test:** [What test to write and what it asserts]
- **Risks:** [Concerns, or "None"]
- **Depends on:** Step N-1 (or "None")
```

### 6b. Final Dashboard Update

The dashboard (`.plan/pages/index.html`) has been regenerated after every decision throughout Phase 4. It auto-refreshes in the browser every 3 seconds, so the user sees progress live.

For this final update, read `templates/dashboard.html`, fill all markers with final state, write to `.plan/pages/index.html`.

**Timeline item HTML** (repeat per decision in `{{TIMELINE_ITEMS}}`):

```html
<!-- DECIDED -->
<a href="decision-N.html" class="timeline-item decided">
  <div class="timeline-dot"></div>
  <div class="timeline-content">
    <div class="timeline-header">
      <span class="timeline-title">N. [Title]</span>
      <span class="timeline-time">14:33</span>
    </div>
    <div class="timeline-meta">
      <span class="category-tag [category]">[Label]</span>
      <span class="status-text">Option B — [Name]</span>
    </div>
  </div>
</a>

<!-- AUTO-DECIDED -->
<a href="decision-N.html" class="timeline-item auto">
  <div class="timeline-dot"></div>
  <div class="timeline-content">
    <div class="timeline-header">
      <span class="timeline-title">N. [Title]</span>
      <span class="timeline-time">14:34</span>
    </div>
    <div class="timeline-meta">
      <span class="category-tag [category]">[Label]</span>
      <span class="status-text">Auto: [brief reason]</span>
    </div>
  </div>
</a>

<!-- PENDING -->
<a href="decision-N-prompt.html" class="timeline-item pending">
  <div class="timeline-dot"></div>
  <div class="timeline-content">
    <div class="timeline-header">
      <span class="timeline-title">N. [Title]</span>
      <span class="timeline-time">&mdash;</span>
    </div>
    <div class="timeline-meta">
      <span class="category-tag [category]">[Label]</span>
      <span class="status-text">Pending</span>
    </div>
  </div>
</a>
```

Mark the phase indicator steps: prior phases get `class="completed"`, current phase gets `class="active"`.

### 6c. Open Dashboard

```bash
open .plan/pages/index.html
```

### 6d. Present to User

> Implementation plan ready. [N] steps across [M] phases.
>
> - **Plan:** `.plan/plan.md`
> - **Dashboard:** `.plan/pages/index.html` (opened in browser)
> - **All decisions:** `.plan/pages/`
>
> How do you want to proceed?
> - **"Execute"** — I'll implement step by step, verifying each one
> - **"Step by step"** — I'll ask for OK before each major action
> - **"Review first"** — Look at the plan and tell me what to change
> - **"Later"** — Plan is saved, come back anytime

---

## Context Recovery Protocol

**CRITICAL:** If your context has been compressed, or you're starting a new turn and don't remember the session:

1. **Read `.plan/state.json`** — tells you phase and progress
2. **Read `.plan/plan.md`** — summary of everything decided
3. **Check `last_checkpoint_at`** — if recent, you're mid-session
4. **Read specific decision files only as needed**

After recovery:
> Recovered planning context from `.plan/`. We're in Phase [N] with [X/Y] decisions made. Picking up where we left off.

---

## Visual Preview Rules

The `.visual-preview` area in each card should contain a **real rendered visual**, not text.

### Architecture decisions (tech stack, structure)

Use `.arch-diagram`, `.arch-layer`, `.arch-box` classes:

```html
<div class="arch-diagram">
  <div class="arch-layer">
    <div class="arch-box frontend">React SPA</div>
  </div>
  <div class="arch-arrow">&updownarrow;</div>
  <div class="arch-layer">
    <div class="arch-box backend">REST API</div>
    <div class="arch-box service">Auth</div>
  </div>
  <div class="arch-arrow">&updownarrow;</div>
  <div class="arch-layer">
    <div class="arch-box database">PostgreSQL</div>
  </div>
</div>
```

Box classes: `.frontend` (purple), `.backend` (cyan), `.database` (green), `.service` (red).

### Behavior decisions (user flows, interaction patterns)

Use vertical numbered flow diagrams:

```html
<div class="flow-container">
  <div class="flow-step">
    <span class="flow-step-number">1</span>
    <span class="flow-step-label">User clicks button</span>
  </div>
  <div class="flow-down-arrow">&darr;</div>
  <div class="flow-step highlight">
    <span class="flow-step-number">2</span>
    <span class="flow-step-label">Modal opens with form</span>
  </div>
  <div class="flow-down-arrow">&darr;</div>
  <div class="flow-step">
    <span class="flow-step-number">3</span>
    <span class="flow-step-label">Submit triggers API call</span>
  </div>
</div>
```

Use `.highlight` on 2-3 differentiating steps. Use `.error` for failure paths. Use `.flow-branch-label` for conditional branches.

**Rules:** Always vertical single-column. Always number steps. 4-7 steps max. Short action phrases (verb first).

### Visual decisions (design, layout, navigation)

Build actual rendered UI mockups with inline styles:

```html
<div style="width:100%;max-width:340px">
  <div style="background:#1e293b;border-radius:12px;padding:16px;border:1px solid #334155">
    <div style="font-size:18px;font-weight:700;color:#f8fafc">Title Here</div>
    <div style="font-size:13px;color:#94a3b8;margin-top:4px">Subtitle text</div>
    <button style="width:100%;margin-top:12px;padding:10px;background:#6366f1;color:white;border:none;border-radius:8px;font-size:13px;font-weight:600;cursor:pointer">Action</button>
  </div>
</div>
```

Or use `.sitemap` classes for navigation/IA decisions:

```html
<div class="sitemap">
  <div class="sitemap-level">
    <div class="sitemap-node primary">Home</div>
  </div>
  <div class="sitemap-connector"></div>
  <div class="sitemap-level">
    <div class="sitemap-node">Browse</div>
    <div class="sitemap-node">Search</div>
    <div class="sitemap-node">Profile</div>
  </div>
</div>
```

### Data decisions (schema, storage)

Show schema structures or comparison stats. A simple key-value layout or table within the preview area works well.

### Interface decisions (API shape, types)

Show code snippets with inline styling:

```html
<div style="font-family:'SF Mono',monospace;font-size:0.7rem;color:#cbd5e1;padding:0.5rem;background:#0f172a;border-radius:8px;width:100%">
  <div style="color:#86efac">GET /api/v1/books</div>
  <div style="color:#64748b;margin-top:4px">→ { books: Book[], total: number }</div>
</div>
```

### Quality decisions (testing, observability)

Show test structure diagrams or monitoring architecture using the arch-diagram classes.

---

## Comparison Table Dimensions

Choose 5-8 dimensions that genuinely differentiate. Per category:

- **Architecture:** Learning curve, Ecosystem, Performance, Scalability, Cost, Best suited for
- **Behavior:** Number of steps, User effort, Error recovery, Speed, Flexibility, Familiarity
- **Data:** Query speed, Schema flexibility, Migration effort, Scaling, Tooling
- **Interface:** Type safety, Documentation, Backwards compat, Discoverability, Ergonomics
- **Visual:** Mood/feeling, Accessibility, Brand alignment, Trend durability, Implementation effort
- **Quality:** Coverage confidence, CI speed, Debugging ease, Maintenance burden, Reliability

---

## Edge Cases

**User skips:** Set status to `"decided"` with `chosenOption: "skip"`, `chosenTitle: "Skipped — agent decides"`. Use your recommendation when implementing.

**Custom answer:** Record as `chosenOption: "custom"`, `chosenTitle: "[their description]"`. Update HTML with a "CUSTOM CHOICE" card.

**"Show overview":** Open the dashboard: `open .plan/pages/index.html`

**Jump ahead:** Reorder and present that decision next, then continue.

**Existing .plan/ directory:** Read `state.json` to understand progress. Resume from first pending decision: "I see we've made N decisions. Picking up with Decision M: [Title]."

**"Just decide for me":** Use recommendations for all remaining. Record them, generate plan, present it.

---

## Agent Directives — Non-Negotiable

1. **Read before proposing.** Read every relevant file. Cite `file:line`. Never guess.
2. **Match conventions.** Grep existing patterns first. Follow them.
3. **Acceptance criteria on every decision.** "How do we know this works?" — answered concretely.
4. **Test strategy before implementation.** Plan specifies tests BEFORE code.
5. **Minimal viable change.** Smallest diff that achieves the goal.
6. **One change, verify, next.** Steps are sequenced for independent testing.
7. **Flag risks explicitly.** Security, perf, race conditions — per decision, not buried.
8. **Checkpoint religiously.** Save to `.plan/` after every decision.
9. **Visual previews render real things.** Not text descriptions — actual UI, diagrams, flows.
10. **The comparison table is mandatory.** Every decision page must have one.
