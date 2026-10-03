---
topic: Git submodules — how they work, daily usage, and the "not our ref" failure
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Git submodules

**One-line idea:**
A submodule is a separate git repo nested inside a parent repo; the parent does not store the child's files or branch, only a **single pinned commit SHA** (plus the URL in `.gitmodules`). In pikaros2, `src/pika_warehouse_description` is one.

**Why it exists:**
Lets the parent repo depend on an exact, reproducible version of another repo (shared config/description data) without copying it in. The cost: two repos must be kept in sync by hand, and the pointer is only valid if that commit exists on the child's remote.

**Math / mechanics:**
Operational mechanics:
- `.gitmodules` (tracked) holds path + URL. The parent's commit tree holds a special "gitlink" entry: path -> child commit SHA. Check with `git ls-tree HEAD <path>`.
- A submodule checkout is normally a **detached HEAD** at that SHA. This is expected, not an error.
- `git pull` / `checkout` / `reset` in the parent moves the pointer but does **not** move the submodule's working tree. You must run `git submodule update` afterwards, or `git status` will show the submodule as "modified" (stale checkout).
- `git submodule update` fetches the child's remote and checks out the pinned SHA. If the SHA is not on the remote, it fails (see gotchas).

**Code:**
```bash
# clone with submodules
git clone --recurse-submodules <url>
# or, in an existing clone
git submodule update --init --recursive

# after EVERY pull / branch switch / rebase / reset in the parent
git submodule update --init --recursive

# inspect
git submodule status                     # leading '-' = not init, '+' = checkout differs from pinned SHA
git ls-tree HEAD src/pika_warehouse_description   # SHA the parent pins

# CHANGING a submodule (order matters!) — land the child change in ITS OWN repo first
cd src/pika_warehouse_description
git checkout -b my-change                # leave detached HEAD before committing
# ...edit, commit...
git push origin my-change                # 1) push the CHILD first
# open/merge a PR against the submodule's own repo (its own master), independent of the parent
cd ../..
git -C src/pika_warehouse_description fetch origin
git -C src/pika_warehouse_description checkout origin/master   # point at the now-merged SHA
git add src/pika_warehouse_description   # 2) stage the new pointer in the parent
git commit -m "Bump warehouse description submodule"
git push                                 # 3) then push/merge the PARENT (contains code + one gitlink change)
# everyone else, after pulling the parent:
git pull
git submodule update --init src/pika_warehouse_description

# safety nets
git config --global submodule.recurse true          # pull/checkout also update submodules
git config --global push.recurseSubmodules check    # refuse to push parent if child commit isn't pushed
# (or 'on-demand' to push the child automatically)

# diagnosing "not our ref"
git ls-tree HEAD src/pika_warehouse_description
cd src/pika_warehouse_description
git fetch origin
git branch -r --contains <sha>           # empty output => remote doesn't have it
git log --oneline -5 origin/HEAD
```

```bash
# Resolving a submodule pointer CONFLICT during a merge or rebase.
# Git only stores one commit SHA per submodule in the parent, so when two
# branches move that SHA differently it can't auto-merge — fix by checking
# out the desired SHA inside the submodule, then staging the gitlink.
cd src/pika_warehouse_description
git fetch origin
git checkout <desired-submodule-sha>
cd ../..
git add src/pika_warehouse_description
git commit -m "Resolve warehouse submodule pointer"      # for a merge conflict
# or, during a rebase: stage the SHA of the commit currently being replayed
# (not necessarily the final target SHA — later picks will move it further anyway)
git rebase --continue
```

```bash
# Local branch cleanup
git branch -d my-old-branch                                     # delete if merged
git branch -D my-old-branch                                     # force-delete if unmerged
git branch --merged master | grep -v '^\*' | grep -v 'master$' | xargs -r git branch -d
git branch --no-merged master            # see what's left (not yet merged)
git branch -a | grep SOME-TICKET-123     # find a ref not shown by plain `git branch`
git fetch --prune                        # drop stale remote-tracking refs
```

**Gotchas:**
- **Push the child before the parent.** If the parent's pointer is pushed first (or the child is never pushed), everyone else gets `fatal: remote error: upload-pack: not our ref <sha>` / `Direct fetching of that commit failed` on `git submodule update`. The commit then exists only on the author's machine. Fix: author pushes the submodule; or someone repoints the parent to a SHA that exists on the remote.
- Same error appears if the child's history was **rebased/force-pushed**, orphaning the pinned SHA.
- Rebasing/force-pushing the parent does not break submodules by itself, but you must re-run `git submodule update` after resetting to the rewritten branch, and any new submodule added upstream needs `--init`.
- **Pointer conflicts in merge/rebase are not text conflicts.** Resolve by checking out the desired SHA inside the submodule, then `git add <submodule-path>` in the parent. During a rebase specifically, three different submodule SHAs can be in play at once ("ours," "theirs," and the current stale working-tree checkout) — stage the SHA of the commit *currently being replayed*, not necessarily the rebase's final target, since later picks will move the pointer further anyway.
- **`git commit` can fail with "nothing to commit" even though `git status` shows the submodule as modified**, right after a rebase finishes. This happens when the branch's recorded submodule pointer is already correct but the submodule's own working-tree checkout is stale from an earlier manual `git checkout <sha>` step inside it — fix by re-checking out the correct SHA *in the submodule* (no new parent commit needed).
- `git branch` (no flags) only lists **local** branches. A name that doesn't show up there (but e.g. tab-completes) may be a remote-tracking ref (`refs/remotes/origin/...`) — check `git branch -r` or `git branch -a`.
- `git add -A` / `git commit -a` in the parent will happily commit an accidentally moved submodule pointer. Check `git status` / `git diff` for the submodule line before committing.
- Commits made while the submodule is in detached HEAD are easy to lose; create a branch first.
- Do not `git reset --hard` as a fix for a stale submodule: it moves the parent only. Use `git submodule update`.
- A `git diff` of just the submodule gitlink shows as `Subproject commit <old> -> <new>` — that's the normal, expected way a submodule bump appears in a parent-repo diff/PR, not a sign of a corrupted diff.

**Related:** [[_software-hub]]

**Still unclear:**
Trigger for this note was a real `git submodule update --init --recursive` failure ("not our ref ...") for `src/pika_warehouse_description` after resetting a local branch to a rebased remote — root cause (unpushed child commit vs. rewritten child history) was not confirmed in that instance; diagnostic commands were never run to pin it down. Everything else is general git submodule mechanics, not independently verified against this project's exact `.gitmodules`/CI setup (e.g. whether `submodule.recurse` is actually configured team-wide) — verify before marking `reviewed`.
