---
description: "Fork curation decisions for Zurackus/hve-core: branch workflow, plugin.json membership policy, excluded packages, and upstream sync procedure."
applyTo: '**'
---

# Fork State

This repository is a curated fork of `microsoft/hve-core`. It serves as the AI workflow foundation for a TypeScript/Astro/Payload CMS/Cloudflare Pages marketing agency stack.

## Branch Workflow

| Branch | Role |
|---|---|
| `main` | Clean upstream mirror. Accepts only fast-forward merges from `microsoft/hve-core`. No fork-specific commits live here. |
| `production` | Curated working branch. Receives cherry-picked upstream changes and all fork-specific additions. |

Never commit fork-specific changes to `main`. The two branches diverge solely at `plugin.json` content and any fork-specific artifacts added to `production`.

## Plugin.json Membership

`plugin.json` is the sole mechanism controlling which agents, rules, and skills activate in Copilot sessions. File presence under `.github/` alone does not activate anything.

The file carries `merge=ours` in `.gitattributes`. An upstream `git merge main` never silently overwrites the curated membership; each new upstream entry requires deliberate review.

Do not run `npm run plugin:sync` in this fork. That script derives the manifest from all tracked `.github/` files and would re-add every excluded package. Maintain `plugin.json` manually.

## Excluded Packages

These upstream packages are absent from `plugin.json` because they are not relevant to the target stack. Their files remain on disk in both branches so upstream diffs remain legible.

| Excluded package | Reason |
|---|---|
| `accessibility` | No WCAG/AT compliance workflow on this stack |
| `data-science-engineering` | No data-science or ML workloads |
| `design-thinking` | No DT coaching workflow in active use |
| `engagement-reporting` | No client engagement reporting workflow |
| `experimental` | Unstable; each item evaluated individually before adoption |
| `privacy` | No formal DPIA or privacy-plan workflow needed |
| `sssc` / supply-chain security | Covered by the security skills already included |
| ADR planning | ADRs tracked informally; no formal ADR workflow |
| Jira / ADO / GitLab backlog | GitHub Issues is the sole tracker |
| `bicep` / `terraform` coding standards | No IaC in this stack |
| `csharp` coding standards | TypeScript/Astro only; no C# |

When a future session adds Astro, CSS, or MDX coding-standard artifacts, register them in `plugin.json` directly without running `plugin:sync`.

## Upstream Sync Procedure

Before running the sync, confirm the `upstream` remote is configured:

```bash
git remote get-url upstream
# If missing:
git remote add upstream https://github.com/microsoft/hve-core.git
```

1. Fast-forward `main` to upstream:

   ```bash
   git fetch upstream
   git checkout main
   git merge --ff-only upstream/main
   ```

2. Generate the diff between `main` and `production`:

   ```bash
   git diff production..main --name-only
   git diff production..main -- plugin.json
   ```

3. Invoke the `/hve-core:upstream-sync-review` skill with those outputs (or `/upstream-sync-review` if working directly in this repo window). It categorizes each changed file and emits cherry-pick commands for approved items.

4. Cherry-pick approved commits to `production`:

   ```bash
   git checkout production
   git cherry-pick <sha> ...
   ```

5. Apply any new in-scope `plugin.json` entries manually and commit to `production`.
6. Bump the version in `plugin.json` and update the installed plugin — see [Updating the Installed Plugin](#updating-the-installed-plugin).

## Updating the Installed Plugin

VS Code does not automatically pick up pushed changes. After any commit to `production` that modifies `plugin.json`, adds or removes agents/rules/skills, or changes skill content, follow these steps:

**Step 1 — Bump the version in `plugin.json`**

Use [semantic versioning](https://semver.org) relative to the current fork version (not Microsoft's upstream version):

| Change type | Bump |
|---|---|
| New skill, agent, or rule added | Minor — e.g. `3.2.3` → `3.3.0` |
| Skill content or rule updated | Patch — e.g. `3.2.3` → `3.2.4` |
| Agent or skill removed | Minor |
| Breaking restructure | Major |

**Step 2 — Commit and push to `production`**

Include the version bump in the same commit as the change.

**Step 3 — Update the plugin in VS Code**

1. Open the Agent Customizations editor (Command Palette → **Chat: Open Customizations**)
2. Select the **Plugins** tab
3. Find **hve-core** and open its details
4. Select **Update** — VS Code re-pulls from `production` and installs the new version

If **Update** is greyed out, the installed version already matches. Confirm the version number in the plugin details matches what was pushed.

**Step 4 — Verify**

Open a fresh chat session, type `/` and confirm new or changed skills appear as expected. If a skill is missing, check that its folder name matches the `name` field in its `SKILL.md` exactly.

## Fork-specific Artifacts

These files exist only in `production` and have no upstream equivalent. They must never be removed during a sync.

| File | Purpose |
|---|---|
| `.github/instructions/hve-core/fork-state.instructions.md` | This file. Persistent memory of curation decisions. |
| `.github/skills/hve-core/upstream-sync-review/SKILL.md` | Skill for reviewing upstream diffs and producing cherry-pick commands. |
| `.github/prompts/hve-core/upstream-sync-review.prompt.md` | Native prompt version of the sync review; auto-loads in the hve-core workspace window. |

When new fork-specific artifacts are added (e.g. Astro, CSS, MDX coding standards), add them to this table.
