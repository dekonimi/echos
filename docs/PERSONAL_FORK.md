# Personal Fork Workflow

A guide for following the upstream EchOS repo closely while carrying a small set of personal-only tweaks.

This workflow is meant for a private "patch stack" on top of the maintainer's work. It is a good fit when you want to:

- use the project as the maintainer ships it
- make a few local changes for your own setup
- avoid doing normal pull request work
- pick up upstream releases quickly
- see exactly which of your customizations still apply cleanly after each upstream update

It is not really about long-lived feature development in your fork. The goal is to keep `main` clean and treat your custom changes as a thin layer on top.

## Why This Workflow Works Well

The important idea is to separate "their code" from "your code":

```text
upstream/main        ← official releases from the maintainer
your fork/main       ← a clean mirror of upstream/main
your fork/personal   ← your personal tweaks, rebased on top
```

That separation gives you a few practical benefits:

- `main` always tells you what upstream currently looks like
- `personal` always tells you what you changed
- rebasing `personal` onto a new upstream release immediately shows where your changes no longer fit cleanly
- dropping a no-longer-needed tweak is easy because it lives in its own commit history

For a solo personal fork, this is usually better than committing custom changes directly onto `main`.

## One-time Setup

```bash
# 1. Fork the repo on GitHub (via UI), then clone your fork
git clone git@github.com:YOUR_USERNAME/echos.git
cd echos

# 2. Add upstream as a remote
git remote add upstream git@github.com:ORIGINAL_OWNER/echos.git

# 3. Create a personal branch — this is where all of your custom changes live
git checkout -b personal
```

### Important Rule

Treat `main` as read-only.

Do not commit personal changes on `main`. The point of the workflow is that `main` stays a mirror of upstream, so you can always compare against it and safely rebase your tweaks.

## Adding a Personal Tweak

Always work on `personal`, and prefer one commit per logical tweak:

```bash
git checkout personal

# make your change, then:
git add -p
git commit -m "[personal] add reminder tools to MCP server"
```

Why this helps:

- small commits are easier to rebase
- small commits are easier to drop if upstream later implements the same idea
- small commits make it obvious which tweak caused a conflict or regression

The `[personal]` prefix is optional, but useful. It makes your commits easy to spot during `git log`, `git rebase -i`, and conflict resolution.

## Syncing with an Upstream Release

When the maintainer ships a new version, update in two phases:

### 1. Move `main` forward to match upstream exactly

```bash
git checkout main
git fetch upstream
git merge --ff-only upstream/main
git push origin main
```

Use `--ff-only` here on purpose.

That protects the rule that `main` is only a mirror. If Git cannot fast-forward, something unusual happened and you should stop to inspect it instead of creating a merge commit.

### 2. Rebase your personal tweaks onto the new upstream version

```bash
git checkout personal
git rebase main
```

If a conflict happens:

```bash
git add <conflicted-file>
GIT_EDITOR=true git rebase --continue
```

For a personal fork, this is exactly the behavior you want. A conflict means upstream changed the same area that one of your personal commits touched, so Git is asking you whether your tweak should:

- stay as-is
- be adjusted to the new upstream code
- be removed because upstream now covers it

## What the Rebase Results Mean

This workflow is especially useful because the result tells you what kind of review is needed:

- If `git rebase main` stops with conflicts, one of your tweaks overlaps with upstream changes and needs manual review.
- If the rebase succeeds cleanly, your patches still apply structurally, but they may still need testing because upstream behavior may have changed around them.
- If a personal commit is no longer needed, you can drop it during an interactive rebase.

In other words, a clean rebase means "your edits still fit", not automatically "your edits still behave correctly".

## Check What Changed After Each Upstream Update

After rebasing, review what is still uniquely yours:

```bash
# See your personal commits on top of upstream
git log main..personal --oneline

# See the current diff between upstream mirror and your personal branch
git diff main..personal
```

This is usually the fastest way to answer:

- which changes are still mine
- which tweaks were reapplied cleanly
- which parts of the code I should test carefully

## Recommended Validation Step

After every upstream sync:

1. Rebase `personal` onto the updated `main`.
2. Review `git log main..personal --oneline`.
3. Review `git diff main..personal`.
4. Run the project's normal tests, build, or app startup flow.

That final validation matters. Rebase checks whether your code can be replayed on top of upstream, but runtime testing checks whether it still works.

## Keeping Changes Conflict-Resistant

The less you edit shared upstream files, the easier every future upgrade becomes.

**Prefer new files over editing existing ones.**

Instead of putting all of your custom code directly into a heavily shared file like `server.ts`, keep your logic in a new file and touch upstream files as lightly as possible:

```text
packages/core/src/mcp/personal-tools.ts   ← your additions live here
packages/core/src/mcp/server.ts           ← one import/use line only
```

This reduces conflict surface area because upstream is much more likely to edit `server.ts` than your new personal file.

**Prefer config or environment changes over code changes.**

If a tweak can be done in `.env`, config, or a plugin, that is usually safer than editing shared application code.

**One commit per tweak.**

Atomic commits let you keep, rewrite, or drop changes independently.

**Do not fork what you do not need to change.**

If a customization can live outside the shared codepath, keep it there.

## Handling a Commit Upstream Already Absorbed

Sometimes upstream adds the same feature you previously added for yourself. When that happens, your best move is usually to remove your version and rely on upstream's version instead.

```bash
git rebase -i main
```

In the editor, change `pick` to `drop` for the personal commit that upstream now makes unnecessary.

This is one of the main reasons to keep personal tweaks small and isolated: when upstream catches up, cleanup is easy.

## Useful Optional Improvement: Remember Conflict Resolutions

If you plan to keep this fork for a while, enable Git's recorded-resolution feature:

```bash
git config --global rerere.enabled true
```

`rerere` helps Git remember how you resolved repeated conflicts. For a personal fork that gets rebased over many upstream releases, this can save real time.

## Important Caveat

This workflow assumes `personal` is your own branch.

Rebasing rewrites history, which is totally reasonable for a solo personal patch branch, but can be awkward if multiple people are sharing and pulling that same branch. If this is strictly for your own use, rebasing is the right tradeoff.

## Summary

| Goal | Recommended practice |
|------|----------------------|
| Follow upstream closely | Keep `main` as a clean mirror of `upstream/main` |
| Keep private customizations | Put them on `personal`, not on `main` |
| Make upgrades easier | Keep tweaks small, isolated, and preferably in separate files |
| Sync a new upstream release | `git fetch upstream`, `git merge --ff-only upstream/main` on `main`, then `git rebase main` on `personal` |
| See what is still yours | `git log main..personal --oneline` and `git diff main..personal` |
| Remove a tweak upstream replaced | `git rebase -i main` and drop the commit |
