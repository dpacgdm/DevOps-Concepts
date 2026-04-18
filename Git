# PHASE 2 — Lesson 1B: Git Commands & Workflows (The Missing Half)

---

### Fetch vs Pull — The Distinction That Matters

```bash

\git fetch: downloads objects and refs from remote. Touches NOTHING local.

git fetch origin

\Updates: .git/refs/remotes/origin/main

\Does NOT update: your local main branch, your working directory

\Safe. Always safe. You're just getting information.



\git pull: fetch + merge (or fetch + rebase if configured)

git pull origin main

\Equivalent to:

git fetch origin

git merge origin/main



\git pull --rebase:

git pull --rebase origin main

\Equivalent to:

git fetch origin

git rebase origin/main



\WHY THIS MATTERS:

\git pull with merge creates merge commits on your local branch

\Your history:    A → B → C (local)

\Remote history:  A → B → D (someone pushed)

\git pull creates: A → B → C → M (merge commit)

\                       └→ D ↗

\Ugly. Pollutes history with "Merge branch 'main' of origin..."

\\#

\git pull --rebase:

\Replays your C on top of D:  A → B → D → C'

\Clean linear history.

\\#

\Configure globally:

git config --global pull.rebase true

\Now git pull always rebases instead of merging.

\This is the standard at most serious engineering orgs.

```

---

### Reset — The Three Modes

```bash

\git reset moves the branch pointer AND optionally changes

\the staging area and working directory.



\SETUP: You have 3 commits

\A → B → C (HEAD)



\--soft: moves HEAD only. Staging and working dir UNTOUCHED.

git reset --soft HEAD~1

\HEAD now at B

\Staging area: still has C's changes (ready to commit)

\Working directory: unchanged

\Use case: "I want to redo that commit with a different message"

\          or "I want to combine the last 2 commits"



\--mixed (DEFAULT): moves HEAD + resets staging. Working dir UNTOUCHED.

git reset HEAD~1        # --mixed is the default

git reset --mixed HEAD~1

\HEAD now at B

\Staging area: reset to match B (C's changes are UNSTAGED)

\Working directory: still has C's changes as modified files

\Use case: "I want to unstage everything and re-add selectively"



\--hard: moves HEAD + resets staging + resets working directory.

git reset --hard HEAD~1

\HEAD now at B

\Staging area: matches B

\Working directory: matches B

\C's changes are GONE from working dir

\Use case: "Throw everything away, go back to this state"

\DANGEROUS: working directory changes are NOT recoverable

\           (unless they were committed — then reflog saves you)



\COMMON PATTERNS:

\Undo last commit but keep changes staged:

git reset --soft HEAD~1



\Unstage a file (without losing changes):

git reset HEAD myfile.txt     # old way

git restore --staged myfile.txt  # new way (Git 2.23+)



\Nuke everything and match remote exactly:

git fetch origin

git reset --hard origin/main

\Your local main is now identical to remote. All local changes gone.

```

Visual:

```

\                  --soft       --mixed      --hard

HEAD (branch ptr)  MOVES        MOVES        MOVES

Staging area       unchanged    RESET        RESET

Working directory  unchanged    unchanged    RESET

```

---

### Revert vs Reset — Undo Strategies

```bash

\RESET: rewrites history (moves the branch pointer backward)

\REVERT: creates a NEW commit that undoes a previous commit



\Reset:

\A → B → C → D (HEAD)

git reset --hard HEAD~2

\A → B (HEAD)

\C and D are orphaned. History is rewritten.

\If C and D were pushed, you need --force to push.

\NEVER reset shared branches.



\Revert:

\A → B → C → D (HEAD)

git revert HEAD

\A → B → C → D → D' (HEAD)

\D' is a new commit that undoes D's changes

\History is PRESERVED. No force push needed.

\Safe for shared branches.



\Revert a specific commit (not the latest):

git revert abc1234

\Creates a new commit that undoes abc1234's changes

\May conflict if later commits depend on abc1234



\Revert a merge commit:

git revert -m 1 <merge-commit-hash>

\-m 1 means "keep the first parent's side" (usually main)

\This undoes the merged branch's changes

\GOTCHA: if you later want to re-merge that branch,

\you must REVERT THE REVERT first, or Git thinks

\those changes are already in history



\THE RULE:

\Pushed to shared branch? → git revert

\Local only, not pushed? → git reset

\No exceptions.

```

---

### Cherry-Pick

```bash

\Copy a specific commit from one branch to another

\WITHOUT merging the entire branch



\Scenario: hotfix commit on develop needs to go to main NOW

git checkout main

git cherry-pick abc1234

\Creates a NEW commit on main with the same diff as abc1234

\Different hash (it's a new commit), same changes



\Cherry-pick multiple commits:

git cherry-pick abc1234 def5678



\Cherry-pick a range:

git cherry-pick abc1234..def5678

\Applies everything AFTER abc1234 up to and including def5678



\Cherry-pick without committing (stage only):

git cherry-pick --no-commit abc1234

\Changes are staged but not committed

\Useful when you want to combine multiple cherry-picks into one commit



\CONFLICTS during cherry-pick:

git cherry-pick abc1234

\CONFLICT in file.txt

\Fix the conflict, then:

git add file.txt

git cherry-pick --continue

\Or abort:

git cherry-pick --abort



\WHEN TO USE:

\- Hotfixes that need to go to release branch

\- Extracting a single useful commit from an abandoned branch

\- Backporting fixes to older versions



\WHEN NOT TO USE:

\- Moving large sets of commits (use merge or rebase)

\- Cherry-picking creates duplicate commits in history

\  (same diff, different hashes on different branches)

```

---

### Conflict Resolution — The Full Workflow

```bash

\WHEN CONFLICTS HAPPEN:

\merge, rebase, cherry-pick, stash pop — any operation that

\combines changes from two sources



\Git marks conflicts in the file:

<<<<<<< HEAD

const port = 3000;

=======

const port = 8080;

>>>>>>> feature-branch



\HEAD section: what's on your current branch

\======= divider

\feature-branch section: what's on the incoming branch



\RESOLUTION STEPS:

\1. Open the file

\2. Choose which version (or combine both)

\3. Remove the conflict markers entirely

\4. git add the resolved file

\5. git merge --continue (or git rebase --continue)



\USEFUL TOOLS:

\See all conflicted files:

git diff --name-only --diff-filter=U



\Use a merge tool:

git mergetool

\Opens configured diff tool (vimdiff, meld, VS Code, IntelliJ)



\Accept one side entirely:

git checkout --ours file.txt     # keep current branch version

git checkout --theirs file.txt   # keep incoming branch version



\During rebase conflicts — CAREFUL:

\"ours" and "theirs" are SWAPPED during rebase!

\Because rebase replays YOUR commits onto THEIR base

\--ours = the branch you're rebasing ONTO (main)

\--theirs = YOUR commits being replayed

\This confuses everyone. Every time.



\ABORT if things go wrong:

git merge --abort

git rebase --abort

git cherry-pick --abort

\Returns to pre-operation state. No damage done.

```

---

### Git Log — Actually Useful

```bash

\Basic:

git log --oneline              # short hash + message, one line each

git log --graph --oneline      # ASCII branch visualization

git log --graph --oneline --all  # ALL branches, not just current



\Filter by author:

git log --author="john"        # partial match works



\Filter by date:

git log --since="2024-01-01" --until="2024-06-01"

git log --since="2 weeks ago"



\Filter by message:

git log --grep="hotfix"        # commits whose message contains "hotfix"



\Filter by file:

git log -- path/to/file.txt    # only commits that touched this file

git log -p -- path/to/file.txt # show the actual diff for each commit



\Filter by content change (pickaxe):

git log -S "DATABASE_URL"      # commits that added/removed this STRING

git log -G "port.*8080"        # commits matching this REGEX in diff

\Incredibly powerful for: "who changed this config value and when?"



\Show stats:

git log --stat                 # files changed, insertions/deletions

git log --shortstat            # summary only



\Limit output:

git log -5                     # last 5 commits

git log main..feature          # commits in feature NOT in main

git log feature..main          # commits in main NOT in feature



\Format for scripts:

git log --format="%H %an %s"  # full hash, author name, subject

```

---

### Git Blame, Show, Diff

```bash

\BLAME — who wrote each line:

git blame file.txt

\a1b2c3d (John 2024-01-15 14:30:00 +0000  1) const port = 3000;

\b2c3d4e (Jane 2024-02-20 09:15:00 +0000  2) const host = "0.0.0.0";



\Blame a specific range of lines:

git blame -L 10,20 file.txt    # lines 10-20 only



\Ignore whitespace changes:

git blame -w file.txt



\SHOW — inspect any object:

git show HEAD                   # latest commit diff

git show abc1234                # specific commit

git show abc1234:path/to/file   # file contents at that commit

git show v1.0.0                 # tag details



\DIFF:

git diff                        # working dir vs staging (unstaged changes)

git diff --staged               # staging vs last commit (staged changes)

git diff HEAD                   # working dir vs last commit (all changes)

git diff main..feature          # difference between two branches

git diff abc1234 def5678        # difference between two commits

git diff --stat main..feature   # summary: files changed, lines +/-

git diff --name-only HEAD~3     # just filenames changed in last 3 commits

```

---

### Restore and Switch (Git 2.23+ — The Modern Way)

```bash

\Git 2.23 split `git checkout` into two focused commands:



\OLD (overloaded):

git checkout main                 # switch branches

git checkout -- file.txt          # discard working dir changes

git checkout abc1234 -- file.txt  # restore file from specific commit



\NEW (clear intent):

\git switch — for branch operations:

git switch main                   # switch to main

git switch -c new-branch          # create and switch

git switch -                      # switch to previous branch



\git restore — for file operations:

git restore file.txt              # discard working dir changes (from index)

git restore --staged file.txt     # unstage (move from index to working dir)

git restore --source=HEAD~3 file.txt  # restore from specific commit

git restore --staged --worktree file.txt  # unstage AND discard changes



\git checkout still works. But switch/restore make intent explicit.

\In scripts and CI, prefer the new commands.

```

---

### Git Clean — Remove Untracked Files

```bash

git clean -n          # dry run — show what WOULD be deleted

git clean -f          # delete untracked files

git clean -fd         # delete untracked files AND directories

git clean -fdx        # delete untracked files, dirs, AND ignored files

\                     # WARNING: -x removes things in .gitignore too

\                     # (node_modules, build artifacts, .env files)



\Use case: "I want a pristine state matching the repo exactly"

git reset --hard HEAD

git clean -fd

\Working directory now matches HEAD commit perfectly

```

---

### Git LFS — Large File Storage

```bash

\Problem: Git stores full copies of every version of every file

\Binary files (images, videos, ML models, JARs) bloat the repo

\A 100MB model file × 50 versions = 5GB repo



\Git LFS replaces large files with lightweight POINTERS in the repo

\Actual file content stored on a separate LFS server



\Setup:

git lfs install                              # one-time setup

git lfs track "*.psd"                        # track Photoshop files

git lfs track "models/**"                    # track ML models directory

\This creates/updates .gitattributes:

*.psd filter=lfs diff=lfs merge=lfs -text



git add .gitattributes                       # commit the tracking rules

git add large-file.psd

git commit -m "add design file"

git push                                     # pushes pointer to Git, content to LFS



\What's in the repo:

git cat-file -p HEAD:large-file.psd

\version https://git-lfs.github.com/spec/v1

\oid sha256:abc123...

\size 104857600

\← That's it. A 130-byte pointer, not 100MB.



\CI CONSIDERATION:

\git clone fetches LFS files automatically (if lfs is installed)

\To skip LFS in CI (if you don't need the actual binaries):

GIT_LFS_SKIP_SMUDGE=1 git clone repo.git

\Saves bandwidth and time in pipelines that don't need binary assets

```

---

### .gitattributes — Beyond LFS

```bash

\Line ending normalization (cross-platform teams):

* text=auto                  # Git auto-detects text files, normalizes endings

*.sh text eol=lf             # Shell scripts always LF (even on Windows)

*.bat text eol=crlf          # Batch files always CRLF

*.png binary                 # Never treat as text, never normalize



\Custom merge strategies per file:

database/schema.sql merge=ours    # always keep our version on conflict

package-lock.json merge=ours      # avoid lockfile merge nightmares

\(requires: git config merge.ours.driver true)



\Diff drivers:

*.zip diff=zip                # custom diff for zip files

*.md diff=markdown            # better diff formatting for markdown

```

---

### Git Hooks — In Detail

```bash

\.git/hooks/ contains sample scripts (*.sample)

\Remove .sample suffix to activate



\CLIENT-SIDE HOOKS:

pre-commit        # runs before commit is created

\                 # exit 1 = abort commit

\                 # use for: lint, format, secrets scan



commit-msg        # runs after message entered, before commit finalized

\                 # receives message file path as arg

\                 # use for: enforce message format (JIRA-123: description)



pre-push          # runs before push

\                 # use for: run tests, prevent push to main



post-checkout     # runs after git checkout / git switch

\                 # use for: npm install after branch switch



\SERVER-SIDE HOOKS (Bitbucket/GitLab/self-hosted):

pre-receive       # runs before accepting a push

\                 # use for: enforce branch naming, reject force push

\                 # exit 1 = reject the entire push



update            # like pre-receive but per-branch

\                 # use for: per-branch policies



post-receive      # runs after push is accepted

\                 # use for: trigger CI, notify Slack, deploy



\SHARED HOOKS (team-wide):

\.git/hooks/ is NOT committed (it's inside .git/)

\Solution: use a framework



\pre-commit framework (Python):

\.pre-commit-config.yaml (committed to repo):

repos:

\ - repo: https://github.com/pre-commit/pre-commit-hooks

\   rev: v4.5.0

\   hooks:

\     - id: trailing-whitespace

\     - id: end-of-file-fixer

\     - id: check-yaml

\     - id: check-added-large-files

\       args: ['--maxkb=500']

\ - repo: https://github.com/gitleaks/gitleaks

\   rev: v8.18.1

\   hooks:

\     - id: gitleaks        # scan for secrets



\Install: pre-commit install

\Now every developer who runs pre-commit install

\gets these hooks locally



\Husky (Node.js projects):

\package.json:

"husky": {

\ "hooks": {

\   "pre-commit": "lint-staged",

\   "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"

\ }

}

```

---

### Submodules vs Subtrees

```bash

\SUBMODULES: embed another repo as a subdirectory

git submodule add https://github.com/org/shared-lib.git libs/shared

\Creates .gitmodules file:

[submodule "libs/shared"]

\  path = libs/shared

\  url = https://github.com/org/shared-lib.git



\The parent repo stores a POINTER (commit hash) to the submodule

\NOT the submodule's contents



\Clone with submodules:

git clone --recurse-submodules repo.git

\Or after clone:

git submodule update --init --recursive



\Update submodule to latest:

cd libs/shared

git pull origin main

cd ../..

git add libs/shared

git commit -m "update shared-lib to latest"



\PAIN POINTS (and there are many):

\- Developers forget --recurse-submodules on clone

\- CI must explicitly init submodules

\- Detached HEAD inside submodule is the default

\- Merge conflicts on submodule pointer are confusing

\- Nested submodules compound all these problems



\SUBTREES: copy another repo INTO your repo as actual files

git subtree add --prefix=libs/shared https://github.com/org/shared-lib.git main --squash



\Pull updates:

git subtree pull --prefix=libs/shared https://github.com/org/shared-lib.git main --squash



\Push changes back upstream:

git subtree push --prefix=libs/shared https://github.com/org/shared-lib.git main



\SUBTREE ADVANTAGES:

\- No special clone commands — files are in the repo

\- No .gitmodules, no detached HEAD confusion

\- Works with existing tooling without modifications



\SUBTREE DISADVANTAGES:

\- Repo size grows (full copy of subtree code)

\- History can be messy without --squash

\- Push back to upstream is awkward



\RECOMMENDATION:

\Small shared libraries → subtree (simpler)

\Large external dependencies → submodule (saves space)

\Or better: use a proper package manager (npm, pip, Maven)

\  and publish shared code as packages

```

---

### Git Worktrees

```bash

\Problem: you're working on feature-x, urgent hotfix needed on main

\Without worktrees: stash, switch, fix, switch back, pop stash

\With worktrees: check out another branch in a SEPARATE directory



git worktree add ../hotfix-dir main

\Creates ../hotfix-dir with main checked out

\SHARES the same .git database (no re-clone)



cd ../hotfix-dir

\fix the issue

git commit -am "hotfix"

git push



\Go back to original directory — feature-x is untouched

cd ../original-dir



\Clean up when done:

git worktree remove ../hotfix-dir



\List all worktrees:

git worktree list



\USE CASES:

\- Hotfixes without disrupting current work

\- Running tests on one branch while coding on another

\- Comparing behavior across branches side by side

\- CI that needs multiple branches checked out simultaneously

```

# 📋 QUICK REFERENCE — Git Commands (Lesson 1B)

```

FETCH vs PULL:

\ fetch: download, touch nothing local

\ pull: fetch + merge (or + rebase if configured)

\ Always: git config --global pull.rebase true



RESET:

\ --soft:  HEAD moves. Staging + working dir untouched.

\ --mixed: HEAD moves. Staging reset. Working dir untouched.

\ --hard:  HEAD moves. Staging reset. Working dir reset. DESTRUCTIVE.



UNDO:

\ Pushed? → git revert (new commit, safe)

\ Local only? → git reset (rewrite history)

\ Revert a merge: git revert -m 1 <hash>

\ Re-merge after revert: must revert the revert first



CHERRY-PICK:

\ git cherry-pick <hash>           single commit

\ git cherry-pick --no-commit      stage only

\ Use for hotfixes, backports. Not for bulk moves.



CONFLICTS:

\ ours/theirs SWAP during rebase (everyone forgets this)

\ git checkout --ours/--theirs for bulk resolution

\ Always: git merge --abort / git rebase --abort if lost



LOG:

\ -S "string"    pickaxe: who added/removed this string

\ -G "regex"     grep through diffs

\ main..feature  commits in feature not in main

\ --author, --since, --grep for filtering



LFS:

\ git lfs track "*.bin"

\ GIT_LFS_SKIP_SMUDGE=1 git clone   (skip in CI)

\ Repo stores 130-byte pointer, LFS server stores content



SUBMODULES vs SUBTREES:

\ Submodules: pointer to external repo (saves space, painful workflow)

\ Subtrees: full copy in your repo (simple workflow, larger repo)

\ Best: publish shared code as packages instead



WORKTREES:

\ git worktree add ../dir branch

\ Multiple branches checked out simultaneously

\ Shares .git database — no re-clone

```

---

# 📝 Retention Questions — Lesson 1B

**Q1:** A junior developer on your team ran `git reset --hard HEAD~3` on the `main` branch, then did `git push --force`. Three commits are gone from remote. Two other developers have already pulled the old `main`. Walk through the full recovery process.

**Q2:** A critical bug is found in production. The fix exists as a single commit on the `develop` branch, but `develop` has 40 other untested commits. How do you get ONLY that fix onto the `release` branch? What happens if the fix conflicts?

**Q3:** Your team reverted a merge to `main` because it caused issues. The feature branch was fixed and they want to re-merge it. But `git merge feature` says "Already up to date." Explain why and how to fix it.

**Q4:** Explain the difference between `git reset --mixed HEAD~1` and `git restore --staged .` — when would you use each?
