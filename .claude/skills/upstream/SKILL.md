---
name: upstream
description: Upstream workflows for this fork. Use when the user asks to sync dev with upstream, merge upstream dev into develop, or cherry-pick a fix onto a branch from dev for an upstream pull request.
---

# Upstream workflows

The fork's policy lives in `CLAUDE.md` under Branching: `develop` is upstream's `dev` plus the fork-only files,
`dev` mirrors upstream, and fixes reach upstream as cherry-picks. This skill is the procedure for the three requests
that keep coming back. Each section ends on a report to the user; the user merges every pull request.

## Before any section

1. **Remotes.** A fresh clone has no `upstream` remote. Add it with its push URL disabled:

       git remote add upstream https://github.com/TecnaGamer/BeamNG.Earth
       git remote set-url --push upstream DISABLED-do-not-push-to-upstream

   Done when `git remote -v` shows `upstream` fetching from TecnaGamer and pushing to the disabled URL.
2. **Identity.** The container's git identity resets between sessions. Before every commit and push, check that
   `git config user.name` / `user.email` and `git log -1 --format='%an <%ae> / %cn <%ce>'` show the repository
   owner, and set it back when they don't.

## Sync dev with upstream

1. `git fetch upstream dev && git fetch origin dev`.
2. When `origin/dev` equals `upstream/dev`, report that dev already matches and stop.
3. `git merge-base --is-ancestor origin/dev upstream/dev` must succeed. When it fails, dev has diverged: report it
   and stop, leaving dev as it is.
4. `git checkout dev && git merge --ff-only upstream/dev && git push origin dev`, then return to the branch you
   were on.

Done when `origin/dev` and `upstream/dev` name the same commit. Report the commit and that it was a fast-forward.

## Merge upstream dev into develop

1. **Preconditions.** Fetch `upstream dev`, `origin dev` and `origin develop`.
   - `dev` behind `upstream/dev`: ask the user whether to sync it first (the section above).
   - `git merge-base --is-ancestor dev origin/develop` succeeds: report that develop already has upstream's dev and
     stop.
2. **Branch.** `git checkout develop && git pull origin develop`, then cut
   `feature/sync-upstream-dev-<YYYY-MM-DD>` from it, adding `b`, `c`, ... when that name exists locally or on origin.
   The new branch tracks nothing (`git rev-parse --abbrev-ref @{u}` fails) until its first push.
3. **Merge.** `git merge --no-ff --no-commit dev`. Resolve each conflict toward upstream: upstream owns its code,
   and develop only adds the fork-only files. Where upstream changed the same lines as a fork fix, keep upstream's
   version when it fixes the same problem and drop the fork's now-redundant code; when picking either side would
   lose behavior, ask the user.
4. **Fork-only check.** `git diff --cached --name-only dev` lists only fork-only files: `.github/workflows/tests.yml`,
   the agent files, `.claude/skills/`, and fork fixes upstream hasn't taken yet. Anything else means a resolution
   kept fork code by mistake.
5. **Commit** with the message `Bring upstream dev into develop`, plus one sentence per conflict saying which side
   was taken and why.
6. **Tests.** Run every `mod/tests/test_*.lua` under `luajit`, then
   `BNGE_SKIP_DRIVE=1 python3 tools/run_tests.py test_modsim test_protocol_agreement test_e2e test_v131_default
   test_gasstations test_geotiles`. CI runs the full suites on the pull request.
   - Without `luajit` on the machine, say which checks were skipped and rely on CI's `lua` job.
   - For each failure, rerun it on pure upstream dev (`git worktree add --detach <scratch dir> dev`, run there,
     then `git worktree remove --force <scratch dir>`). Failing there too makes it upstream's own failure: name the
     upstream commits that caused it and report it in the pull request and to the user, leaving the fix to the user.
     Failing only on the merge makes it the merge's: fix the resolution.
   - A network failure in this sandbox (hosts the proxy blocks) is the environment's; name the hosts.
7. **Pull request.** `git push -u origin <branch>`, then open a ready pull request into `develop` titled
   `Bring upstream dev into develop`, with a plain `## Summary` (which upstream commits, conflicts and how they were
   resolved) and `## Notes` (what still differs from dev, which tests ran, any upstream failures). Re-read the body
   and remove any attribution footer the tool appended. Keep status in the body; post no comments. Subscribe to the
   pull request's activity.

Done when the pull request is open with a clean body and the user has the report: conflicts, tests, upstream
failures. After the user merges, `git checkout develop && git pull origin develop`.

## Cherry-pick a fix onto a branch from dev

1. **Find the fix commits**: the commits of the fix's merged pull request into develop, excluding merge commits.
   None of them touches an agent file.
2. **Check upstream first.** `git fetch upstream dev` and read what upstream changed in the same files since the
   fork's fix (`git log` and `git diff` on those paths). When upstream already fixed the problem, or changed the same
   lines, ask the user before going on.
3. **Branch** `feature/<fix branch name>-dev` from the local `dev` after `git checkout dev && git pull origin dev`.
   The new branch tracks nothing (`git rev-parse --abbrev-ref @{u}` fails) until its first push.
4. `git cherry-pick <commits>`, oldest first.
5. **Checks.** Run both commands from `CLAUDE.md` Branching. Both print nothing. Also confirm
   `git merge-tree --write-tree upstream/dev HEAD` succeeds, so the pull request merges cleanly into upstream's
   current dev.
6. `git push -u origin <branch>` and return to develop.

Done when the branch is on origin with both checks empty. Report the branch name and how many commits dev is
behind upstream. The user opens the pull request into `TecnaGamer/BeamNG.Earth:dev`.
