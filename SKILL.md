---
name: skill-publishing
description: "Use when pushing a Hermes skill to GitHub or opening its PR."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
categories: [devops, github]
metadata:
  hermes:
    tags: [hermes, skills, github, git, publish, pr]
    related_skills: [github/github-pr-triage, devops/repo-initialization, github/github-issue-to-pr]
---

# Skill Publishing — sync a Hermes skill to GitHub

Hermes skills live under `~/.hermes/skills/<owner>/<skill-name>/`. They are **not git
repositories by default** — even though some sibling skills (e.g. `superpowers/`,
`hermes-agent/`) ship with their own `.git`. When you need to version-control,
share, or back up a skill, you must git-init it yourself and reconcile with any
repo that already exists on GitHub.

## When to use
- "sync this skill to GitHub" / "push the skill to main"
- First-time publish of a skill you've been editing locally
- Open/update a PR for a skill, then verify it still loads via `hermes skills list`

## Core workflow (see references/skill-to-github.md for the exact command sequence)

0. **Publish from an isolated staging dir — never from the skills tree.**
   Copy the skill's files by EXPLICIT name into a scratch dir and `git init`
   there:

   ```bash
   STAGE=~/.hermes/cache/scratch/<skill>-stage
   rm -rf "$STAGE" && mkdir -p "$STAGE/scripts"
   SRC=~/.hermes/skills/<owner>/<skill-name>
   cp "$SRC/SKILL.md" "$SRC/.gitignore" "$STAGE/"
   cp "$SRC/scripts"/*.py "$SRC/scripts"/*.sh "$STAGE/scripts/"
   cd "$STAGE" && rm -rf .git && git init -q -b main
   ```

   Copy by explicit name, never `cp -r`: a skills dir can contain nested repos
   (`hermes-agent/`, `kanban/workspaces/…`) that abort `git add` with *"does not
   have a commit checked out"*. Assert the staged set before the first commit
   (`git diff --cached --name-only | wc -l`) against the file count you meant
   to publish.

   **Executable guard — run this BEFORE any `git add`.** It converts the
   prose warning into a hard stop, because a session can read the warning and
   still stage into the installation repo:

   ```bash
   TOP=$(git -C "$STAGE" rev-parse --show-toplevel 2>/dev/null)
   [ "$TOP" = "$STAGE" ] || { echo "ABORT: $TOP is not the skill's own repo"; exit 1; }
   ```

   `[ "$TOP" = "$STAGE" ]` is the whole point: a toplevel that prints an
   ancestor means every `git add` would land in that ancestor's repo.

1. **Check git state — and that the repo is the SKILL's, not a parent's.**
   `git -C <dir> rev-parse --show-toplevel` succeeds for ANY dir nested under a
   git repo, and `~/.hermes` IS a git repo — so `… || git init -q` is skipped
   and every later `git add` stages into the Hermes *installation* repo. The
   printed toplevel must BE `<dir>`; if it prints an ancestor, you are about to
   commit infrastructure files. Undo with
   `git -C <parent> reset -q HEAD <skill-path>/` and confirm `git log -1` is
   unchanged before continuing. Step 0's staging dir makes this a non-issue.
2. **Check the remote.** `gh repo view TeacherEvan/<skill-name>` — the repo may
   already exist (another session, or a prior empty scaffold) with a `main`
   branch. If so, `git fetch origin` and **rebase** onto it; do not recreate.
3. **Align branch name.** `git init` defaults to `master`; GitHub remotes are
   usually `main`. `git branch -m master main` before pushing.
4. **Resolve the add/add rebase conflict.** Because the remote history is
   unrelated, every shared file shows as "both added." During `git rebase`,
   `--theirs` is the commit being replayed (YOUR local work). Resolve each
   conflict with `git checkout --theirs <file> && git add <file>`, then
   `GIT_EDITOR=true git rebase --continue`. Prefer the local copy — it is the one
   you corrected this session.
5. **.gitignore BEFORE committing.** A `bench/` or test dir pulls in
   `node_modules/`, `package-lock.json`, `package.json`, `results/`. Write a
   `.gitignore` first; never commit those.
6. **Commit scoped sources.** `git add` explicit paths (SKILL.md, references/,
   docs/) — never `git add -A` blindly.
7. **Push** `git push -u origin main`.
8. **PR (honor the "open a PR" workflow).** If you already pushed `main`, a
   branch cut afterward fails with *"No commits between main and &lt;branch&gt;"*.
   To get a real PR: rewind `origin/main` to the pre-push SHA
   (`git reset --hard <prev>` + `git push --force origin main`), then push the
   feature branch and `gh pr create --base main --head feat/...`, `gh pr merge --squash`.
9. **Verify it loads.** `hermes skills list | grep <skill-name>` should show it
   as `local / enabled`.

## Pitfalls (scars)

- **`~/.hermes` is a git repo, so `git init` in a skills dir is skipped.**
  `rev-parse --show-toplevel` returns the PARENT (`/home/ewaldt/.hermes`) for
  every path under it, so the `|| git init -q` fallback never fires and
  `git add` stages the skill into the Hermes installation repo — where it lands
  alongside cron scripts, auth state and gateway config. Detect it by comparing
  the printed toplevel to the target dir, and publish from a scratch staging
  dir (step 0) so the question never arises.
- **`cp -r` + `git add -A` in a staging dir sweeps in nested repos.** A scratch
  dir copied recursively can contain `hermes-agent/` or
  `kanban/workspaces/<id>/`, and `git add` aborts the whole index with
  *"does not have a commit checked out"*. Build the file list explicitly and
  verify the staged count before committing.
- **Unrelated histories on first sync — rebase, do not force-push.** When the
  remote already holds a commit from an earlier session and your staging dir
  started from `git init`, the histories share no ancestor. The step-4
  `--theirs` resolution applies, but the PR recipe's `git push --force` does
  NOT: force-pushing `main` clobbers the remote's commit. Rebase onto
  `origin/main`, resolve with `--theirs` (your corrected copy), then push.
- **Publish from a copy, then re-verify the INSTALLED skill.** Edits made after
  the staging copy was taken are not what you shipped. After pushing, re-run
  the skill's tests from `~/.hermes/skills/…` and confirm
  `hermes skills list` still shows it `local / enabled`.

- **2026-08-13 — skill dir is not a git repo by default.** Unlike `superpowers/`
  or `hermes-agent/`, a freshly edited skill under `~/.hermes/skills/` has no
  `.git`. `git init` it before any push; don't assume `git rev-parse` works.
- **2026-08-13 — add/add rebase conflict on first sync.** The remote repo already
  contained the same files (from an earlier session's commit) while your local
  history started at `git init`, so git sees two independent "adds" of identical
  paths. Resolve with `git checkout --theirs` (your local) — never take the
  remote's stale copy over your corrected one.
- **2026-08-13 — "No commits between main and branch" when PR-ing after push.**
  Cutting a feature branch from an already-pushed `main` yields a zero-diff PR.
  Rewind `origin/main` one commit and force-push, then the branch has a real diff.
- **2026-08-13 — node_modules leaked into the skill repo.** A `bench/` dir with a
  local `npm install --no-save` left `node_modules/` + `package.json` untracked;
  add `.gitignore` (`node_modules/`, `package-lock.json`, `package.json`,
  `results/`) and verify with `git status --short` before committing.
- **Subagent timeouts while building test harnesses.** A `delegate_task` subagent
  tasked with authoring a test file can hit the 600s timeout and leave stray
  scratch files (`_probe.ts`, `_measure.ts`) without writing the target. If a
  subagent times out, build the file directly in the parent session and `rm` its
  scratch files — don't wait for a retry that will likely time out again.

## References
- [Skill → GitHub command sequence](./references/skill-to-github.md) — copy-paste terminal recipe for steps 1–9.
