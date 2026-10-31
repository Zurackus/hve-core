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

`plugin.json` is the sole mechanism controlling which agents, rules, skills, and commands activate in Copilot sessions. File presence under `.github/` alone does not activate anything.

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

3. Invoke the `upstream-sync-review` prompt with those outputs. It categorizes each changed file and emits cherry-pick commands for approved items.

4. Cherry-pick approved commits to `production`:

   ```bash
   git checkout production
   git cherry-pick <sha> ...
   ```

5. Apply any new in-scope `plugin.json` entries manually and commit to `production`.
