---
description: "Review upstream changes from microsoft/hve-core after pulling main, categorize each changed file, and produce a decision table with cherry-pick commands for production."
argument-hint: "[file-list] [plugin-diff]"
---

# Upstream Sync Review

Categorize every file changed between `production` and `main`, then produce a decision table with cherry-pick commands for approved items and a list of any `plugin.json` additions needed.

## Success Criteria

Done when:

- Every changed file has exactly one category: **cherry-pick**, **skip**, or **evaluate**.
- A decision table lists each file, its category, and a one-line reason.
- Cherry-pick commands cover all **cherry-pick** items in commit order.
- Any new in-scope `plugin.json` entries are listed with the exact JSON to add and the array they belong to.

## Stop Rules

- If no diff output is provided, ask for both outputs listed in Inputs before proceeding.
- If every changed file belongs to an excluded package, report that and confirm no cherry-picks are needed.
- If a file belongs to the `experimental` package, category is **evaluate** unless the user explicitly approves it.

## Inputs

Paste both of these into the conversation before or after invoking this prompt:

**File list** — the output of:

```bash
git diff production..main --name-only
```

**Plugin diff** — the output of:

```bash
git diff production..main -- plugin.json
```

## Categorization Rules

| Category | When to apply |
|---|---|
| **cherry-pick** | File is in an in-scope package and the change is relevant to this stack |
| **skip** | File is in an excluded package or clearly irrelevant to this stack |
| **evaluate** | File is in an unfamiliar or new package; requires explicit review before adoption |

**In-scope packages** — changes here are cherry-pick candidates unless the specific change is irrelevant:

- `coding-standards`: TypeScript, Python, Rust, Bash, PowerShell, code-review
- `hve-core`: all sub-packages including rpi, hve-builder, documentation
- `project-planning`: backlog (GitHub only), architecture review, UX/UI designer
- `rai-planning`, `security`, `rpi`, `shared`

**Excluded packages** — changes here are skip unless the user overrides:
accessibility, data-science-engineering, design-thinking, engagement-reporting, experimental, privacy, sssc, ADR planning, Jira/ADO/GitLab backlog, bicep/terraform, csharp

## Output

### Decision Table

Produce one row per changed file:

| File | Category | Reason |
|---|---|---|
| `.github/agents/hve-core/rpi-agent.agent.md` | cherry-pick | Core RPI agent update |
| `.github/agents/accessibility/...` | skip | Accessibility excluded from this stack |

### Cherry-pick Commands

List the commits for all **cherry-pick** files, in chronological order. Identify them with:

```bash
git log --oneline production..main -- <file-paths>
```

Then emit the commands:

```bash
git checkout production
git cherry-pick <sha1> <sha2> ...
```

### Plugin.json Additions

For each new entry in the plugin diff that falls in an in-scope package, list:

- The exact JSON value to add
- Which array it belongs to: `agents`, `rules`, `skills`, or `commands`

If no additions are needed, state that explicitly.
