# Git Interview Handbook — Senior Software Engineers (5–10 Years)

A practical, interview-focused guide for senior backend engineers. It covers Git's mental model and internals, collaboration, history management, release workflows, recovery, security, and engineering judgment. Commands assume a reasonably recent Git version; check `git help <command>` when version, configuration, or platform behavior matters.

## Contents

1. [Git fundamentals](#1-git-fundamentals)
2. [Internals and object model](#2-internals-and-object-model)
3. [Branching and merging](#3-branching-and-merging)
4. [Rebase and history management](#4-rebase-and-history-management)
5. [Remotes and collaboration](#5-remotes-and-collaboration)
6. [Branching strategies for product teams](#6-branching-strategies-for-product-teams)
7. [Commit quality and pull requests](#7-commit-quality-and-pull-requests)
8. [Reset, revert, restore, and checkout](#8-reset-revert-restore-and-checkout)
9. [Stash, cherry-pick, and selective changes](#9-stash-cherry-pick-and-selective-changes)
10. [History inspection and debugging](#10-history-inspection-and-debugging)
11. [Tags, releases, and versioning](#11-tags-releases-and-versioning)
12. [Configuration and repository hygiene](#12-configuration-and-repository-hygiene)
13. [Hooks, automation, and safeguards](#13-hooks-automation-and-safeguards)
14. [Security and sensitive-data recovery](#14-security-and-sensitive-data-recovery)
15. [Performance and large repositories](#15-performance-and-large-repositories)
16. [Common mistakes and recovery](#16-common-mistakes-and-recovery)
17. [Senior-level scenarios](#17-senior-level-scenarios)
18. [Most-asked interview questions](#18-most-asked-interview-questions)
19. [Mock interview rounds](#19-mock-interview-rounds)
20. [Revision checklist and official references](#20-revision-checklist-and-official-references)

---

## 1. Git fundamentals

### 1.1 What is Git, and why is it distributed?

Git is a distributed version-control system that records repository history as a graph of commits. Developers can commit locally, inspect history, create branches, and compare changes without a network connection; a remote is used to exchange objects and references with other repositories. Unlike a centralized system where the server is normally the primary history store, each ordinary clone contains the repository history it has fetched (a shallow clone is an intentional exception). Distribution improves offline work and flexibility, but it does not automatically provide code review, access control, backups, or release policy—those are team and hosting-platform responsibilities.

### 1.2 Explain the working tree, index, and repository.

The **working tree** is the checked-out files you edit. The **index** (staging area) is the proposed snapshot for the next commit. The **repository** stores Git objects and references, including commit history. A tracked file can be modified in the working tree, staged in the index, and then committed. These are separate states, so a file can have staged changes and additional unstaged changes at the same time.

```bash
git status
git diff                 # working tree versus index
git diff --staged        # index versus HEAD
git diff HEAD            # combined tracked changes versus HEAD
git add src/Order.java
git commit -m "Handle duplicate order requests"
```

Review both staged and unstaged changes before committing. `git diff HEAD` is useful, but `git status` makes the separate states clearer.

### 1.3 What are a repository, working tree, and bare repository?

A repository is the object database plus references and metadata. A working tree is a checked-out directory associated with a repository. A **bare repository** has no checked-out working tree and is commonly used as a shared remote repository. In a normal non-bare repository, Git metadata is usually in `.git/`; linked worktrees and submodules can use a `.git` file that points elsewhere. Do not assume `.git` is always a directory.

### 1.4 What do the core commands do?

- `git init` initializes a repository in the current directory.
- `git clone URL` creates a local copy and configures a remote, commonly named `origin`.
- `git status` reports staged, unstaged, and untracked changes.
- `git add` updates the index from the working tree; it does not itself create a commit.
- `git commit` records the index as a commit.
- `git diff` compares file states; the endpoints depend on options.
- `git log` displays reachable commit history.

Git generally determines tracked-file changes by comparing content and relevant metadata with the index and HEAD, using stat information to avoid rereading unchanged files where possible. It does not rely only on modification timestamps. Line-ending conversion, clean/smudge filters, file mode settings, and filesystem behavior can affect what appears as a change.

### 1.5 Does Git store diffs or snapshots?

Conceptually, each commit records a snapshot through a tree object. Git's storage is content-addressed and deduplicates identical objects; packfiles may delta-compress objects for efficiency. A commit is not simply a list of file edits. This model makes branches and history operations reliable and explains why removing a large file from the latest version does not necessarily remove its historical object.

---

## 2. Internals and object model

### 2.1 Explain blob, tree, commit, and annotated tag objects.

A **blob** stores file content, not its filename. A **tree** maps names and modes to blobs or other trees, representing a directory snapshot. A **commit** points to a root tree and records metadata and zero or more parent commits. An **annotated tag object** stores a target object plus tagger metadata, a message, and optionally a signature. A lightweight tag is just a reference to an object, commonly a commit; it is not a separate tag object.

### 2.2 How are commits linked?

A commit references its snapshot and parent commit(s). A normal commit has one parent; a root commit has none; a merge commit has two or more parents. This forms a directed acyclic graph. Two commits with identical file contents can still have different IDs because their parent, author/committer metadata, timestamps, message, or tree can differ. Commit IDs identify the complete commit object, not merely the file contents.

### 2.3 What are a branch, HEAD, and detached HEAD?

A branch is a movable reference to a commit. `HEAD` normally symbolically refers to the currently checked-out branch; committing advances that branch reference. In a detached HEAD state, HEAD points directly to a commit rather than a branch. You can inspect or test an older commit while detached, but commits made there may not be reachable from a named branch. Preserve such work by creating a branch: `git switch -c investigation`.

Example: if `main` points to C1, committing C2 while `main` is checked out moves `main` from C1 to C2. Creating `feature` at C2 creates another pointer; it does not copy the commit history.

### 2.4 What are symbolic references and remote-tracking branches?

A symbolic reference points to another reference; HEAD is the common example. A remote-tracking ref such as `origin/main` is a local record of the last fetched state of the remote branch. It is not a live network pointer and does not move merely because someone else pushed. `git fetch origin` updates these references and downloads needed objects without merging into your current branch.

### 2.5 What is the index, and what happens during add and commit?

The index is the snapshot Git will use for the next commit. `git add file` hashes the staged content as needed, stores the object, and updates the index entry. `git commit` creates a tree from the index and a commit object referencing that tree and the current parent, then updates the current branch reference. Hooks may run as part of the operation. This is why changes made after staging are not automatically included in the staged version.

### 2.6 How do hashes, packfiles, and garbage collection work?

Git computes object IDs from object type, size, and content using the repository's configured hash algorithm. Hashes support object identification and integrity checking; they are not by themselves a complete security guarantee against every malicious scenario. Modern repositories may use SHA-1 or SHA-256 depending on initialization and compatibility support. Loose objects are stored individually; packfiles group objects and may delta-compress similar content to reduce storage and transfer costs. Git's maintenance and garbage-collection mechanisms repack objects and eventually prune eligible unreachable objects. Exact maintenance behavior depends on Git version and configuration.

### 2.7 How does Git handle renames?

Git does not normally store a dedicated “rename object.” A commit stores a tree snapshot; commands such as diff infer a rename by comparing deleted and added paths and evaluating content similarity. Detection thresholds and options can affect the result. A rename can therefore appear as delete-plus-add in some comparisons, particularly when content changes substantially.

### 2.8 What are reflogs and unreachable objects?

Reflogs record recent local movements of references, including HEAD and branch refs, and are often the easiest way to recover after a reset or branch deletion. Inspect with `git reflog`, then preserve a found commit using `git branch rescue <sha>`. Reflogs are local, expire according to configuration, and are not a remote backup. If an object is no longer reachable from refs or reflogs and is eventually pruned, recovery may be impossible. Do not run aggressive cleanup while investigating lost work.

---

## 3. Branching and merging

### 3.1 Why are Git branches lightweight?

A branch is essentially a reference to a commit, so creating one is cheap; it does not duplicate the full project. Switching branches changes the index and working tree to the target snapshot, subject to uncommitted-change safety checks. Lightweight branches make short-lived feature branches practical, but cheap creation does not eliminate the integration cost of keeping a branch alive for weeks.

### 3.2 Create, switch, rename, and delete branches.

```bash
git switch -c feature/order-validation
git switch main
git branch -m feature/order-validation feature/order-validation-v2
git branch -d feature/order-validation
git branch -D abandoned-experiment   # force delete; verify first
```

The `-d` form refuses to delete a branch Git considers unmerged in the relevant context; `-D` overrides that safeguard and can make commits harder to find. Deleting a local branch does not delete the corresponding remote branch.

### 3.3 Fast-forward versus three-way merge.

A fast-forward is possible when the current branch tip is an ancestor of the branch being merged. Git can move the current branch pointer forward without creating a merge commit. If both branches have diverged, Git normally performs a three-way merge using the common merge base and the two branch tips; a successful merge usually creates a commit with both tips as parents. `git merge --no-ff feature` forces a merge commit even when fast-forwarding would be possible, preserving an explicit integration point. Team policy should determine whether that history is desirable.

### 3.4 What is merge-base?

The merge base is a best common ancestor used to compare the branches during a three-way merge. Inspect it with `git merge-base main feature`. In complex histories there can be more than one best common ancestor; Git's merge machinery can construct a virtual merge base. Do not assume the merge base is always the most recent commit visible on one branch.

### 3.5 How do you resolve conflicts safely?

First inspect `git status` and the conflicted paths. Open each file, understand both changes, and decide the intended combined behavior; conflict markers are only a starting point. Stage each resolved file with `git add`, inspect `git diff --staged`, run focused tests, then complete the merge. `git merge --abort` attempts to return to the pre-merge state, but can be unable to reconstruct pre-existing uncommitted changes cleanly. Start with a clean working tree or save work before integrating.

Conflicts can occur in the same lines, in file/rename/delete interactions, or because changes cannot be reconciled structurally. Even changes to different lines can conflict due to surrounding edits or file-level operations. Git resolves text, not program intent: two edits can merge cleanly but violate an API contract, transaction boundary, or business invariant. Those are **semantic conflicts** and require tests and code review.

### 3.6 What is squash merging? What is an octopus merge?

A squash merge combines the net changes from a branch into one new commit on the target branch, without recording the feature branch's individual commits as parents. The original commits remain in the graph only if reachable elsewhere. It creates a compact mainline history but loses the explicit commit ancestry between the feature branch and mainline. An octopus merge combines more than two heads in one merge commit; it is useful for several independent branches with no conflicts, but is rarely suitable for complex conflict resolution.

---

## 4. Rebase and history management

### 4.1 What does rebase do?

Rebase takes commits from a branch and replays their changes on a new base. It creates new commit objects, so the replayed commits normally have new IDs even when the resulting file content is the same. Rebase can produce a linear history, but it changes ancestry and can require conflict resolution. Example: `git fetch origin` followed by `git switch feature` and `git rebase origin/main` replays feature-only commits on the updated mainline.

### 4.2 Rebase versus merge: how do you choose?

Merge records the actual integration of divergent histories and preserves existing commit identities. Rebase rewrites the feature commits onto a new base and can make review history easier to follow. Rebase is commonly appropriate for private or single-owner feature branches; merge is safer when others already depend on the published commits. Neither is universally superior: the team should agree on history policy, review requirements, and release traceability.

### 4.3 How does interactive rebase help?

`git rebase -i HEAD~4` opens an instruction list for the last four commits. You can reorder, squash or fixup commits, edit a commit, or drop it. Use this to clean up a private pull-request branch before review or merge. Do not casually rewrite commits already used by teammates. Keep meaningful logical boundaries and avoid squashing away information needed to understand a complicated change.

### 4.4 Continue, skip, or abort a rebase?

After resolving conflicts, stage the resolutions and run `git rebase --continue`. `--skip` omits the current commit's patch entirely, so use it only when the change is already present or intentionally unwanted. `--abort` returns to the original branch state from before the rebase, subject to the state and any pre-existing changes. Inspect `git status` before acting; do not choose skip just to make the operation finish.

### 4.5 What about shared branches and force pushes?

Rebasing published commits changes their IDs and creates a different history from the one teammates fetched. If rewriting a branch is explicitly agreed and safe, use `git push --force-with-lease` rather than a blind force push; it checks that the remote ref still has the expected value. It is still not harmless: another update can be overwritten if the lease assumptions are stale or deliberately overridden. Never rewrite protected main or release branches as a routine cleanup. Coordinate first and preserve a recovery reference if needed.

### 4.6 What is `--rebase-merges`?

Ordinary rebase generally linearizes the selected commits and does not preserve the original merge topology. `git rebase --rebase-merges` attempts to recreate merge structure while replaying commits. It is useful when the branch's internal merge topology matters, but can still require conflict resolution and produces rewritten commits. Inspect the resulting graph and tests rather than assuming the original topology is reproduced perfectly.

---

## 5. Remotes and collaboration

### 5.1 Fetch versus pull versus push.

- `git fetch origin` downloads reachable objects and updates remote-tracking refs; it does not integrate them into the checked-out branch.
- `git pull` fetches and then integrates the selected upstream using merge or rebase behavior determined by options and configuration.
- `git push origin feature` sends reachable objects and updates the remote branch if the update is permitted, normally requiring a fast-forward unless a force option is used.

Fetch first when you want to inspect incoming changes before deciding how to integrate them.

### 5.2 What do origin and upstream mean?

They are conventional remote names, not special Git keywords. `origin` usually names the repository cloned from; in a fork workflow, `origin` may be your fork and `upstream` the canonical project. `git remote -v` lists configured URLs. A local branch can track a remote branch, allowing commands such as `git status` to report ahead/behind state.

### 5.3 Why is a push rejected as non-fast-forward?

The remote branch has commits not contained in the local branch, so updating it directly would discard remote history from that ref. Fetch and inspect the graph, then merge or rebase according to policy. For example: `git fetch origin`, `git log --oneline --graph --decorate --all`, then `git rebase origin/feature` for a private branch or merge when preserving shared history is important. Do not immediately force push.

### 5.4 What does force-with-lease protect against?

`git push --force` unconditionally requests a non-fast-forward ref update, subject to server policy. `--force-with-lease` checks that the remote ref still matches an expected value, reducing the chance of overwriting an update you have not observed. It does not replace coordination, review, or branch protection, and an explicitly supplied or stale expectation can weaken its safety.

### 5.5 What are shallow and partial clones?

A shallow clone limits history depth, reducing initial transfer but limiting ancestry-dependent operations and history inspection. A partial clone can omit some objects initially—for example, blob filtering—and fetch them on demand. Sparse checkout controls which paths are materialized in the working tree; it is not the same as filtering history or objects. These features can be combined but solve different problems.

### 5.6 What should you know about submodules?

A submodule records a specific commit of another repository, not a floating branch tip. Consumers must initialize and update submodules, and the referenced commit must remain available. This creates extra onboarding, CI, and dependency-update steps. Submodules can be useful for independently versioned dependencies; avoid them if the team cannot reliably manage nested repository state.

### 5.7 How do SSH keys and HTTPS credentials fit in?

Git transports data over a configured protocol; SSH typically authenticates with keys, while HTTPS commonly uses a credential manager or token-based authentication. Authentication identifies the caller, while repository permissions and server rules decide what they may do. Never embed credentials in remote URLs or commit them to config files. Use the organization's approved credential storage and rotate exposed credentials.

---

## 6. Branching strategies for product teams

### 6.1 Feature branching.

A feature branch isolates work and supports focused review. Keep it short-lived and integrate frequently to reduce drift. Long-lived branches accumulate incompatible assumptions and expensive merges. Feature branches work well with pull requests, but branch existence alone does not provide release isolation or prevent unfinished features from being deployed.

### 6.2 Trunk-based development.

Developers integrate small changes into a shared mainline frequently, using short-lived branches or direct integration under strong safeguards. Automated tests, rapid review, feature flags, and backward-compatible changes reduce risk. It supports frequent deployment, but requires disciplined testing and operational readiness; it is not “commit anything directly to main.”

### 6.3 Git Flow and release branches.

Git Flow uses long-lived development and main branches plus feature, release, and hotfix branches. It can suit scheduled releases and products that maintain release candidates, but adds branching and merge coordination. Release branches are useful when multiple versions need stabilization or support. For continuous delivery, the overhead of long-lived branches can outweigh their value.

### 6.4 How would you choose?

- **Several deployments per day:** favor trunk-based development, small changes, automated checks, feature flags, and quick rollback.
- **Monthly release trains:** short-lived feature branches plus a release branch for stabilization may fit; define what can be cherry-picked into release.
- **Several supported versions:** maintain explicit release/maintenance branches and backport fixes with traceable commits.
- **Several developers on one service:** small pull requests, clear ownership, frequent integration, protected mainline, and contract tests matter more than branch naming.
- **Urgent production fix while work continues:** create a hotfix from the deployed release tag or maintenance branch, test and release it, then propagate the fix to all maintained lines.
- **A feature takes weeks but must remain hidden:** split work into compatible increments and use a feature flag; avoid carrying a large divergent branch for weeks where feasible.

Branch protection, required reviews, status checks, and merge rules are hosting-provider features or server policies, not guarantees supplied by Git itself. A feature flag needs safe defaults, authorization where relevant, and a removal plan.

---

## 7. Commit quality and pull requests

### 7.1 What makes a good commit?

A good commit is a coherent, reviewable change with a clear message and an understandable purpose. Atomic commits help reviewers, bisect, revert, and backport work. Prefer `Add idempotency check to order creation` over `fix`, `updates`, or `final changes`. Avoid mixing formatting, generated output, refactoring, and business behavior unless there is a clear reason.

### 7.2 What are Conventional Commits?

Conventional Commits use a format such as `feat(order): reject duplicate submissions` or `fix(auth): validate expired tokens`. Teams often use this structure to automate changelogs or versioning. It is a convention, not a rule enforced by Git; the team must define its grammar and tooling.

### 7.3 Fixup and autosquash.

For a private branch, `git commit --fixup <commit>` creates a follow-up intended to be folded into a target commit. `git rebase -i --autosquash <base>` arranges fixup commits for cleanup. Since this rewrites commit IDs, do it before others depend on the history or coordinate any required force push.

### 7.4 How should a senior engineer review a pull request?

Review behavior, edge cases, API and data compatibility, error handling, security, observability, tests, and rollback implications—not just style. Keep the diff focused, explain why a change is needed, and call out generated files or lockfiles that materially affect dependencies. Large pull requests increase cognitive load and hide defects; split them where possible. A green CI run is evidence, not proof of correctness.

### 7.5 Squash merge, merge commit, or rebase merge?

A squash merge produces one target-branch commit and a concise mainline but does not preserve feature commits as ancestors. A merge commit preserves the branch topology and integration point. A rebase merge creates a linear mainline by replaying commits. Choose based on traceability, bisectability, backport practices, and the team's preferred audit trail. Hosting providers may implement and constrain these options differently.

### 7.6 Signed commits, tags, and hooks.

Cryptographic signatures can help verify that a commit or tag was signed by a key trusted under the team's verification policy. They do not prove that the code is safe or that the signer carefully reviewed it. Client-side hooks can check messages or run tests, but users can bypass them and they may not be installed. Enforce critical policies on the server or in required checks as well.

### 7.7 Why doesn't deleting a secret in a later commit solve exposure?

Earlier commits still contain the secret, and clones, forks, caches, logs, or artifacts may retain copies. Revoke or rotate the credential first, assess access logs, then decide whether to rewrite history and coordinate cleanup. History cleanup reduces future accidental exposure; it cannot make an already disclosed credential trustworthy again.

---

## 8. Reset, revert, restore, and checkout

These commands solve different problems. Before using one, run `git status`, inspect the target commit, and preserve work you may need.

### 8.1 Compare reset modes.

`git reset <commit>` moves the current branch (and HEAD with it) to the target commit when HEAD is attached. Modes control what happens to the index and working tree:

- **`--soft`**: move HEAD/branch only; index and working tree remain unchanged. The changes between the new HEAD and the staged snapshot remain staged.
- **`--mixed`** (default): move HEAD/branch and reset the index to the target; preserve working-tree files. Changes remain in the working tree as unstaged modifications.
- **`--hard`**: move HEAD/branch, reset the index, and make tracked working-tree files match the target. It can discard staged and unstaged tracked changes. Untracked files are not generally removed simply because of reset, but may obstruct checkout; do not treat this as a recovery-safe command.

Use `--soft` to undo a local commit while keeping its contents staged, `--mixed` to uncommit and unstage while keeping file edits, and `--hard` only when you intentionally want to discard tracked local changes and have verified the target.

### 8.2 Reset versus revert.

Reset changes where a branch points and is mainly appropriate for local history cleanup. Revert creates a new commit whose patch reverses an earlier commit; it preserves published history and is generally preferred for undoing changes already shared. Revert may not perfectly restore the old state if later commits depend on the change, so tests and follow-up fixes may be needed.

### 8.3 What does restore do?

`git restore file` restores working-tree content from the index by default. `git restore --staged file` removes a file's staged changes from the index while keeping working-tree content. `git restore --source=<commit> -- file` restores the chosen version into the working tree; it does not move the branch pointer. Review before staging or committing restored content.

### 8.4 What about checkout?

`git checkout` is a historical multi-purpose command used to switch branches, detach HEAD, and restore paths. Modern commands make intent clearer: `git switch` for branches and `git restore` for files. Existing teams may still use checkout, so understand it, but prefer the explicit commands in new instructions.

### 8.5 How do you revert a merge commit?

A merge commit has multiple parents. `git revert -m 1 <merge-commit>` tells Git which parent is the mainline to keep as the baseline; it does not mean “revert parent one.” Choosing the wrong mainline can reverse the wrong side of the merge. Inspect `git show --no-patch --pretty=raw <merge-commit>`, understand which parent represents the target branch, and test the resulting commit. Reverting a merge also affects how Git treats later attempts to merge the same ancestry; reintroducing the feature may require reverting the revert or adding new commits.

### 8.6 How do you recover uncommitted work?

First inspect `git status`, `git diff`, `git diff --staged`, editor local history, IDE backups, and any stash. Reflog generally recovers commits and reference movements, not arbitrary uncommitted edits. If a hard reset or cleanup overwrote uncommitted file contents, Git may never have stored those contents as objects, so recovery is uncertain. Stop writing to the affected working tree and avoid cleanup commands while investigating.

---

## 9. Stash, cherry-pick, and selective changes

### 9.1 How does stash work?

Stash saves selected working-tree and index changes so you can switch context. `git stash push -m "WIP order validation"` saves tracked modifications; add `-u` to include untracked files. Use `git stash list`, `git stash show -p stash@{0}`, `git stash apply stash@{0}`, or `git stash pop`. Apply retains the stash; pop removes it after a successful application. Conflicts can occur, and a stash is not a substitute for a durable commit or backup.

### 9.2 What is cherry-pick?

Cherry-pick applies the change introduced by a commit onto the current branch and records a new commit (unless options alter the operation). The new commit normally has a different ID because it has a different parent. It is useful for backporting a narrowly scoped fix without merging an entire feature branch. If conflicts occur, resolve and stage files, then use `git cherry-pick --continue`; use `--abort` to cancel the operation.

### 9.3 Merge versus rebase versus cherry-pick.

Merge integrates histories and preserves ancestry. Rebase replays a series of commits on a new base, rewriting their IDs. Cherry-pick applies selected commit changes without integrating the source branch's ancestry. Repeated cherry-picking can create duplicate or equivalent changes that later complicate merges. If a fix must be maintained on multiple release lines, cherry-pick is often appropriate; for regular synchronization between branches, merging is usually clearer.

---

## 10. History inspection and debugging

### 10.1 Commands senior engineers should know.

```bash
git log --oneline --graph --decorate --all
git show <commit>
git diff <base>...<branch>      # changes on branch since merge base
git diff <commit-a> <commit-b> # direct tree comparison
git blame -L 40,80 path/to/File.java
git grep "duplicate request" <revision>
git log --follow -- path/to/File.java
git reflog
```

The three-dot diff compares the merge base of the two revisions with the second revision; two-dot/range notation has different meanings in log commands. Be precise about endpoints when debugging. `git blame` shows the commit last responsible for each line in the selected file state; it does not prove who designed the behavior or why.

### 10.2 How does git bisect find a regression?

Bisect performs a binary search over commits between a known-good and known-bad point. You need a reproducible test or reliable criterion that classifies each checked-out revision.

```bash
git bisect start
git bisect bad <regressed-commit>
git bisect good <known-good-commit>
# At each checked-out revision, run the regression test:
git bisect good   # if the bug is absent
git bisect bad    # if the bug is present
git bisect skip   # if this revision cannot be classified
git bisect reset
```

Git selects revisions to test and narrows the range. If tests are flaky, environment-dependent, or cannot run on intermediate revisions, the result can be misleading; use `skip` for genuinely untestable commits and expect a less precise result. Always run `git bisect reset` to return to the original state. Automated `git bisect run` is useful when the test exits reliably with good/bad status codes.

### 10.3 How do you recover a deleted branch?

Run `git reflog --all` and inspect recent branch/HEAD movements. If you find the lost tip commit, create a rescue branch immediately: `git branch rescue/feature <sha>`. A deleted branch reference does not instantly delete its commit objects. Recovery depends on whether the commit remains reachable through reflogs or other refs and whether garbage collection has pruned it.

### 10.4 How do you compare a release with current code?

Use `git diff v2.4.0..main` to compare the two trees and `git log --oneline v2.4.0..main` to list commits reachable from main but not from the tag. Verify the tag's target with `git show v2.4.0`. For a production regression, combine release comparison with deployment metadata, logs, tests, and bisect rather than assuming the newest commit caused it.

---

## 11. Tags, releases, and versioning

### 11.1 Lightweight versus annotated tags.

A lightweight tag is a ref to an object. An annotated tag creates a tag object with a message and tagger metadata and can be signed. Annotated tags are usually better release markers because they can carry provenance and release notes. A tag points to a specific object, not a moving branch; creating a tag from a branch resolves the branch to its current commit at that time.

### 11.2 Create, inspect, and publish a tag.

```bash
git tag -a v2.4.0 -m "Release 2.4.0"
git show v2.4.0
git push origin v2.4.0
git push origin --tags
```

The last command pushes all tags Git is configured to push under that command's semantics; inspect tags before using it in release automation. Deleting or moving a published tag can break deployment traceability because consumers may have cached or acted on the original target. Prefer a new corrective tag/version unless the release policy explicitly permits repair.

### 11.3 How does Semantic Versioning relate to Git?

Semantic Versioning defines version-number conventions for public APIs; Git itself does not enforce them. A tag such as `v2.4.0` is a human/project convention that marks a commit. For backend services, a release record should tie the tag to the built artifact or image digest, source revision, configuration, and deployment. A tag alone does not guarantee that the deployed binary was built from that commit.

### 11.4 What if a release tag points to the wrong commit?

Do not silently move a tag that a pipeline or users have already consumed. Identify what was built and deployed, stop or correct further rollout if needed, publish a clear corrective release/tag according to policy, and ensure the artifact/source mapping is unambiguous. If the tag must be repaired, coordinate all consumers and audit systems that may have cached it. For several services in separate repositories, maintain a release manifest or deployment record tying compatible versions together.

---

## 12. Configuration and repository hygiene

### 12.1 Git configuration levels.

`git config --local` affects the current repository, `--global` affects the user's configuration, and `--system` affects the installation/system scope where supported. More specific configuration generally overrides broader settings. Set identity with `git config user.name "Name"` and `git config user.email "email@example.com"`; inspect with `git config --show-origin --list`. Identity fields are commit metadata and are not authentication credentials.

### 12.2 Why doesn't .gitignore untrack a file?

`.gitignore` prevents matching untracked files from being offered for addition; it does not remove files already tracked. To stop tracking a file while retaining the local copy, use `git rm --cached path`, then commit the index change and appropriate ignore rule. Be careful with broad patterns that may hide required files from future additions.

### 12.3 Line endings, attributes, and cross-platform changes.

Windows commonly uses CRLF while Unix-like systems use LF. Git's `core.autocrlf` and `.gitattributes` can normalize line endings; team-level attributes are often more reproducible than relying on each developer's personal configuration. For example, `* text=auto` lets Git detect text files and normalize them, while explicit rules can set LF for source files. Before committing a huge apparent diff, inspect whitespace and line-ending changes and agree on repository policy.

### 12.4 File modes, case sensitivity, and generated files.

Git generally tracks the executable bit for files, not every permission. `core.filemode` affects mode detection on filesystems that do not preserve modes reliably. Case-insensitive filesystems can make case-only renames awkward; use a deliberate two-step rename if necessary and verify the staged tree. Generated files should be committed only when project policy requires them. Large generated output inflates diffs and history and often belongs in a build pipeline or artifact store.

### 12.5 Git LFS, monorepos, submodules, and subtree.

Git LFS stores pointer files in Git while the large content is managed by an LFS server; access, quotas, and availability become additional dependencies. Removing a large file from the current tree does not remove prior blobs from history. A monorepo centralizes source and can simplify atomic cross-component changes but needs clear ownership and performance practices. Submodules pin an external repository commit; subtree workflows copy or merge external content into a directory and avoid requiring nested checkout, but can complicate synchronization. Choose based on release coupling, contributor experience, and tooling.

---

## 13. Hooks, automation, and team safeguards

### 13.1 Client-side versus server-side hooks.

Client hooks such as `pre-commit`, `commit-msg`, and `pre-push` can format code, validate commit messages, or run quick checks. They are useful feedback but can be bypassed and may not be installed by default. Server-side hooks and hosting-platform branch rules can reject prohibited updates; required CI checks and reviews enforce policy more consistently across contributors. Do not rely on a local hook as the only secret-scanning or quality gate.

### 13.2 How should a team prevent unsafe changes?

Protect main and release branches, require appropriate reviews and checks, limit who can bypass protections, and make rollback paths clear. Add secret scanning and dependency checks where appropriate. Hooks should be fast and actionable; slow full test suites often belong in CI. Git records history but does not itself provide a universal pull-request or required-check system—those are host and team capabilities.

### 13.3 What do signed commits prove?

A valid signature can establish that a commit object was signed by a particular key, subject to key trust and verification policy. It does not prove the author identity beyond that trust model, the code's safety, or that required tests passed. Signed tags can help authenticate release markers. Keep key management and verification policy documented.

---

## 14. Security and sensitive-data recovery

### 14.1 A secret was committed and pushed. What first?

**Revoke or rotate it immediately.** Treat it as compromised even if the repository is private. Check logs and access scope, replace the secret in the application/configuration system, and identify downstream consumers. Then remove it from current files and decide whether history rewriting is warranted. History rewriting is not a substitute for rotation: existing clones, forks, CI artifacts, caches, logs, and screenshots may retain the value.

### 14.2 When is git-filter-repo appropriate?

`git filter-repo` can rewrite repository history to remove sensitive files or replace exposed strings. It changes commit IDs and can affect tags, branches, and merge ancestry. Plan a maintenance window, back up the repository, coordinate contributors, update all relevant refs, and communicate how everyone should reclone or realign their work. Force pushing rewritten refs can overwrite concurrent updates; protect the repository during the operation and verify the result. Prefer the maintained `git-filter-repo` tool over obsolete recipes when a full history rewrite is required.

### 14.3 What does access control and least privilege mean for Git?

Grant write/admin rights only to people and automation that need them, protect important branches, use short-lived or narrowly scoped credentials where available, and audit access. Signed commits, branch protections, and review checks reduce certain risks but do not eliminate malicious changes or compromised accounts. Assess dependencies, build scripts, and repository content as supply-chain inputs; a clone can contain executable hooks only through explicit local setup, but build/test scripts and dependencies may execute when developers or CI run them.

---

## 15. Performance and large repositories

### 15.1 Why can a repository become slow or large?

Large binary files, generated output, many tiny files, expensive untracked-file scans, oversized history, and filesystem limitations can slow status, checkout, clone, or fetch. Diagnose the actual bottleneck before changing repository structure. Check `git status`, inspect tracked large files and history, and consider Git's trace/performance facilities when needed. Avoid committing build directories, IDE caches, and transient logs.

### 15.2 Shallow clone, partial clone, sparse checkout, and LFS are different.

- **Shallow clone** limits commit history, so ancestry and history operations may be constrained.
- **Partial clone** omits selected object types until needed, reducing initial transfer but requiring network access later.
- **Sparse checkout** limits which paths appear in the working tree; the repository may still know about other paths.
- **Git LFS** stores large content outside ordinary Git blobs, represented by pointers in the repository.

Choose based on whether the bottleneck is history depth, object transfer, working-tree file count, or large binary assets. CI jobs that need tags, merge-base calculations, changelogs, or bisect may fail or behave differently with a shallow history.

### 15.3 Why doesn't deleting a large file shrink history?

The deletion creates a new tree that no longer references the file, but older commits still reference the blob. Storage is reclaimed only after history is rewritten to remove the historical references and unreachable objects are eventually pruned; remote hosting retention and clones can still retain copies. Rewrite history only when the benefit justifies disruption, and coordinate the migration.

### 15.4 What do packfiles and garbage collection mean operationally?

Packfiles consolidate objects and use delta compression; maintenance can improve storage and access efficiency. Garbage collection may prune unreachable objects after grace periods and repack reachable ones. Do not run aggressive cleanup as a first response to lost work, because it can reduce recovery options. On hosted platforms, local maintenance does not guarantee immediate reclamation of server-side storage.

---

## 16. Common mistakes and recovery

### 16.1 I committed to the wrong branch.

If the commit is local and not shared, inspect status and the graph. One safe route is to create the intended branch at the current commit, then move the original branch back only after confirming no desired work will be lost. For example, `git branch feature/correct-branch` preserves the commit before you reset the mistaken branch. If already pushed or shared, avoid rewriting without coordination; a revert or follow-up commit may be safer.

### 16.2 I staged generated files by mistake.

Run `git diff --staged` to identify the exact staged files. Use `git restore --staged path` to unstage while keeping local content, add the correct ignore rule if appropriate, and inspect status again. If the file was already committed, removing it from the current tree does not erase its history; if it contains a secret, follow the security process in Module 14.

### 16.3 I am stuck during a merge or rebase.

Run `git status` first; it identifies the operation and conflicted files. For a merge, resolve and stage files then commit, or use `git merge --abort` if abandoning is appropriate. For a rebase, resolve/stage and run `git rebase --continue`, use `--skip` only if the commit's change should be omitted, or `--abort` to return to the pre-rebase state. Do not mix commands from different operations or delete Git metadata manually.

### 16.4 I reset the wrong commit or deleted a branch.

Stop and inspect `git reflog --all`. Identify the exact commit, verify it with `git show <sha>`, then create a rescue branch before further changes. Reflog recovery is best-effort and local; it cannot reliably recover edits that were never committed or stashed. Avoid garbage collection until recovery is complete.

### 16.5 I pushed an incomplete or incorrect commit.

If the branch is shared, prefer a follow-up fix or revert and communicate impact. If it is a private feature branch and no one depends on it, amend or rebase may be appropriate. Check CI and deployment state before deciding; a push does not necessarily mean deployment, but automation may trigger from it. Never assume a force push undoes an already deployed artifact.

### 16.6 I resolved a conflict but broke behavior.

Inspect the merge commit diff and compare each conflicted file against both parents using `git show <merge>^1:path` and `git show <merge>^2:path`. Run focused regression tests and validate contracts, data migrations, and error paths. If already released, use a tested revert or corrective commit according to deployment policy. A conflict-free merge is only a textual result.

---

## 17. Senior-level scenario-based interview questions

### 17.1 Your feature branch has diverged from main, and teammates have already pulled your commits. How do you update it?

Because teammates depend on the published commits, I would avoid rebasing the shared branch and force-pushing rewritten history. I would fetch the latest refs, inspect the graph, and merge the updated mainline into the feature branch, or follow the team's agreed shared-branch policy. For example, `git fetch origin`, `git switch feature`, `git merge origin/main`. I would resolve conflicts based on intended behavior, run tests, and push the merge commit. If the team explicitly requires a linear history, I would coordinate a planned rewrite and ensure every consumer understands how to realign; I would not surprise them with it.

### 17.2 A password was committed and pushed. What do you do?

I would revoke or rotate the credential first, check its permissions and usage logs, and replace it in the appropriate secret store. Then I would remove it from current source and use secret scanning to find other copies. If history cleanup is required, I would plan a coordinated `git filter-repo` rewrite, update affected refs, and instruct contributors to reclone or carefully realign. I would assume forks, caches, CI artifacts, and existing clones may retain the old value. Rewriting history without rotating the secret is insufficient.

### 17.3 A production release introduced a regression, but several commits were merged afterward. How do you isolate and reverse it?

I would identify the exact deployed artifact and source revision first, then compare the release tag to the prior known-good release. I would reproduce the failure with a focused test and use `git bisect` between good and bad revisions if the test classifies intermediate commits reliably. Once identified, I would assess whether reverting the offending commit is safe given dependent changes; a merge commit requires choosing the correct mainline parent. I would validate on a release branch, deploy through the normal rollback/roll-forward process, and backport the correction where necessary. If impact is ongoing, mitigation takes priority over completing a perfect root-cause analysis.

### 17.4 You must backport a critical fix to two maintained release branches. Merge, cherry-pick, or rebase?

If the fix is isolated in a single commit, I would usually cherry-pick it onto each maintenance branch, resolve any version-specific differences, and test both branches. The cherry-picked commits will have different IDs because their parents differ. If the fix depends on other commits, I would identify and port the full dependency set rather than cherry-picking blindly. I would avoid merging all of main into an older release unless that is the team's established policy, because it could introduce unrelated features. Record which branches received the fix.

### 17.5 A developer ran reset --hard and cannot find their commit.

I would ask them to stop cleanup and run `git reflog --all`. If the commit appears, inspect it and immediately create a rescue branch. I would also check other local branches, remote refs, teammates' clones, PR refs, and CI logs if relevant. Reflog can recover prior branch tips and committed objects, but not reliably recover uncommitted file edits overwritten by reset. I would avoid `git gc` or pruning during investigation.

### 17.6 A pull request has unrelated changes and repeated conflict-resolution commits. How do you clean it up?

First I would determine whether the branch is private or whether others have based work on it. For a private branch, I would create a backup ref and use interactive rebase to reorder, fixup, or drop accidental commits, then inspect the final diff against the merge base and rerun tests. If shared, I would prefer additive cleanup or coordinate the rewrite and use force-with-lease only after agreement. I would split unrelated concerns into separate pull requests where practical; polishing history must not hide behavior changes or erase useful rationale.

### 17.7 The team deploys many times per day. Git Flow or trunk-based development?

I would usually favor trunk-based development with short-lived branches, small reviewed changes, automated tests, protected mainline, feature flags, and backward-compatible database/API changes. This reduces long-lived branch drift and supports continuous delivery. Git Flow can work, but its long-lived development/release branches often create integration overhead for frequent deployments. The decision depends on regulatory requirements, release cadence, rollback capability, and whether multiple product versions must be maintained.

### 17.8 Rebase stopped on a conflict. Should the developer continue or abort?

I would first inspect `git status` and the current patch. If the conflict can be resolved according to intended behavior, stage the files and run `git rebase --continue`, then inspect the resulting history and run tests. If the base or operation was chosen incorrectly, or the resolution is uncertain, `git rebase --abort` is safer than improvising. `--skip` is appropriate only when the current commit's change is already present or intentionally omitted. I would not force-push until the rewritten result is reviewed and the branch's sharing status is clear.

### 17.9 A push is rejected as non-fast-forward. What is your process?

I would fetch and inspect the local and remote graphs. The remote may contain a teammate's commit, a server-generated change, or simply history I have not fetched. I would merge or rebase the remote-tracking branch according to whether the local commits are shared, resolve conflicts, test, and push again. I would not force-push just to bypass the rejection, because it may erase someone else's work.

### 17.10 A large file was deleted, but repository size remains high. Why?

The earlier commits still reference the blob. A normal deletion only changes the current snapshot. I would locate the file's historical paths and determine whether it is a binary artifact, sensitive file, or legitimate history. If removal from all history is justified, I would use a planned history rewrite, coordinate clones and refs, and account for hosting retention and LFS storage. If historical traceability matters and the file is not sensitive, moving future versions to artifact storage or LFS may be preferable to rewriting all history.

### 17.11 A merge succeeded but tests now fail. What next?

I would treat Git's merge success as a statement that it constructed a tree, not that the program is correct. I would inspect the merge commit against both parents, focus on resolved files, and run tests around the changed contracts and shared assumptions. I would also look for semantic conflicts such as inconsistent validation, changed error semantics, or a migration that is incompatible with one caller. Fix forward or revert the integration based on deployment state and release policy.

### 17.12 A release tag points to the wrong commit and deployment consumed it.

I would establish which artifact was built and deployed, pause further automation if it might repeat the error, and publish a clear corrective release or tag under the team's policy. I would not silently move a widely consumed tag because clients and pipelines may have cached the original target. I would record the source SHA and artifact digest, audit downstream consumers, and only repair the existing tag with explicit coordination if the release process allows it.

### 17.13 You need to undo a bad change already merged to main.

I would usually use `git revert <commit>` to create a new commit that reverses the change, because rewriting shared main history would disrupt collaborators and automation. For a merge commit, I would inspect its parents and choose the correct mainline with `git revert -m`. I would run tests because later commits may depend on the bad change. If the deployment is currently causing harm, I would also follow the incident rollback process; source-control reversal alone does not undo external side effects or database mutations.

### 17.14 Git shows a massive diff after switching operating systems.

I would check `.gitattributes`, `core.autocrlf`, and whether the changes are only line endings or whitespace. I would avoid committing a broad normalization accidentally. Agree on repository-level normalization, apply it as a separate intentional change if needed, and inspect the staged diff with whitespace checks. Case-only path changes and executable-bit differences can also behave differently across filesystems.

### 17.15 A long-running feature needs selective rollout while other work continues.

I would break the feature into backward-compatible, small changes behind a default-off feature flag. That allows integration with mainline without exposing the unfinished behavior. The flag should be tested, monitored, access-controlled if it gates sensitive behavior, and removed after rollout. If the feature cannot be made safe when partially integrated, I would minimize branch lifetime and define explicit integration points rather than leave an untested branch diverging indefinitely.

---

## 18. Most-asked Git interview questions

### 18.1 What is the difference between fetch and pull?

Fetch downloads remote objects and updates remote-tracking refs without integrating into the current branch. Pull performs fetch and then integrates via merge or rebase according to command options and configuration. Fetch is the better first step when you want to inspect changes before integration.

### 18.2 Merge versus rebase?

Merge combines histories and preserves commit identities, often with a merge commit when branches diverged. Rebase replays commits on a new base, rewriting their IDs and often creating linear history. Use merge for shared history or explicit integration records; use rebase mainly for private feature-branch cleanup under team policy.

### 18.3 Reset versus revert versus restore?

Reset moves the current branch and can also reset the index and working tree. Revert creates a new commit that reverses an earlier commit. Restore copies content into the working tree and/or index without moving the branch. Choose based on whether the goal is local history cleanup, safe undo of shared history, or file restoration.

### 18.4 Soft, mixed, and hard reset?

Soft moves HEAD/branch and leaves index and working tree unchanged. Mixed moves HEAD and resets the index while preserving working-tree edits. Hard moves HEAD and resets both index and tracked working-tree content, potentially discarding local changes. Inspect status and preserve work before any reset; hard reset is not a general undo command.

### 18.5 What is detached HEAD?

HEAD points directly to a commit rather than symbolically to a branch. It is useful for inspection or testing older revisions. If you create commits and want to retain them, create a branch at that commit before switching away.

### 18.6 What is reflog?

Reflog records recent local reference movements, making it useful for recovering commits after reset or branch deletion. It is local, expires, and does not guarantee recovery of uncommitted edits or objects already pruned.

### 18.7 Fast-forward versus three-way merge?

Fast-forward moves a branch pointer when its current tip is an ancestor of the target. A three-way merge combines two diverged tips using a merge base and typically creates a merge commit. Fast-forward does not create a merge commit unless options such as `--no-ff` require one.

### 18.8 Why use force-with-lease instead of force?

Force push can replace a remote ref even when it contains work you have not incorporated. Force-with-lease checks an expected remote value and reduces accidental overwrites, but it is still a history rewrite and should be coordinated on shared branches.

### 18.9 What is cherry-pick, and how does it differ from merge?

Cherry-pick applies one or more selected commit changes to the current branch and creates new commits. Merge integrates branch history and ancestry. Cherry-pick suits isolated hotfix backports; regular branch synchronization is often clearer with merge.

### 18.10 What is git bisect?

Bisect binary-searches a commit range using known-good and known-bad revisions. It works best with a deterministic test that can classify each revision. Flaky tests, environment differences, and unbuildable intermediate commits reduce confidence.

### 18.11 How do tags differ from branches?

Both are refs, but branches normally move as new commits are created, while tags are intended to identify a fixed point such as a release. Annotated tags add metadata and can be signed. Git does not enforce that tags are immutable, so teams should protect release tags by policy.

### 18.12 Why doesn't .gitignore ignore a tracked file?

Ignore rules affect untracked files. To stop tracking an already committed file while keeping it locally, use `git rm --cached path` and commit the index change plus ignore rule.

### 18.13 How do you resolve a merge conflict?

Inspect status, understand both sides, edit to the intended result, stage resolved files, review the staged diff, and run relevant tests. Textual resolution is not enough; check semantic behavior and contracts.

### 18.14 How do you undo a merge already pushed?

Usually create a revert commit. For a merge commit, inspect its parents and select the correct mainline using `git revert -m`. Test dependent changes afterward. Avoid resetting and force-pushing shared mainline history unless an explicitly coordinated recovery policy requires it.

### 18.15 How do you remove a secret from Git?

Rotate or revoke it immediately, then remove the secret from current code and assess history rewriting with `git filter-repo`. Coordinate rewritten refs and clones. History rewriting cannot erase all copies and does not make the old credential safe.

### 18.16 What are shallow clone, sparse checkout, and Git LFS?

Shallow clone limits commit history; sparse checkout limits materialized paths; Git LFS keeps large content outside normal Git blobs and stores pointers in Git. They solve different performance/storage problems and have distinct CI and collaboration trade-offs.

---

## 19. Mock interview rounds

### Round 1 — Fundamentals and internals

**Interviewer:** Walk me through what happens when you edit a Java file, stage it, and commit it.

**Model answer:** The edit changes the working tree. `git add` updates the index with the content I want in the next snapshot. `git commit` creates a tree from that index and a commit pointing to the tree and its parent, then advances the current branch. If I edit the file again after staging, the index and working tree differ, so I inspect both `git diff --staged` and `git diff`.

**Interviewer:** Why might identical files produce different commit IDs?

**Model answer:** A commit ID covers the whole commit object, not only the root tree. Different parent commits, metadata, timestamps, or messages can produce different IDs even when the checked-out files are identical. This is also why rebase normally changes commit IDs.

**Interviewer:** A developer says Git stores every complete copy of every file per commit. Is that accurate?

**Model answer:** It is a useful conceptual correction to say each commit represents a snapshot, but Git stores objects content-addressably and deduplicates identical content. Packfiles may delta-compress objects. It is neither simply a full independent copy per commit nor merely a list of file-level patches.

### Round 2 — Branching and collaboration

**Interviewer:** Your feature branch is three commits behind main and has two local commits. How do you integrate main?

**Model answer:** I fetch first and inspect the graph. If the feature commits are private, I can rebase them onto the updated mainline for a linear history. If others depend on the branch, I would normally merge main into it to preserve existing commit identities. I resolve conflicts based on behavior, test the result, and push using the team's policy.

**Interviewer:** Git reports non-fast-forward. Is force push the solution?

**Model answer:** Not by default. The rejection protects remote commits that are not in my local history. I fetch, inspect the remote-tracking ref, merge or rebase appropriately, and push again. Force-with-lease is reserved for an agreed history rewrite, not ordinary synchronization.

**Interviewer:** The merge has no conflicts, but integration tests fail. Why?

**Model answer:** Git can merge text without understanding business rules or API contracts. I compare the merge result with both parents, identify semantic conflicts, and test affected paths. I treat successful merge as a source-control result, not a correctness guarantee.

### Round 3 — Recovery and debugging

**Interviewer:** You need to find which commit introduced a reproducible production bug.

**Model answer:** I establish a known-good and known-bad source revision that correspond to the tested artifacts, create a deterministic regression test if possible, and use `git bisect`. I classify each revision as good or bad, skip revisions that cannot be tested, and reset bisect afterward. Then I verify the suspected commit and choose a tested revert or corrective fix based on dependencies and deployment impact.

**Interviewer:** A teammate accidentally deleted a branch after committing work.

**Model answer:** I inspect `git reflog --all`, other refs, and any remote or PR refs. If I locate the tip, I create a rescue branch immediately. Reflogs are local and expire, so this is not guaranteed; if the work was never committed, Git may not contain it.

**Interviewer:** When is reset appropriate?

**Model answer:** Reset is useful for local branch/history cleanup when I understand what will happen to the index and working tree. Soft preserves both, mixed resets the index but preserves working files, and hard resets tracked working files as well. For undoing a shared commit, revert is usually safer because it adds a new commit rather than rewriting shared history.

### Round 4 — Releases and senior judgment

**Interviewer:** A security fix must go to three supported service versions.

**Model answer:** I would identify the exact maintained branches and dependencies, make the fix as isolated as practical, and cherry-pick or otherwise port it to each line. Each target must be tested because code and APIs may differ. I would record which release tags contain the fix and verify the deployed artifact digest/source SHA. I would not merge all of main into old release lines unless policy calls for it.

**Interviewer:** A token was pushed to a private repository. Is rewriting history enough?

**Model answer:** No. I revoke or rotate it first, inspect use and scope, update secret storage, then assess history cleanup. Private repositories can still have clones, forks, CI logs, and caches. Rewriting is a cleanup measure, not a way to restore trust in a disclosed token.

**Interviewer:** The team wants linear history, but developers keep force-pushing over each other's work.

**Model answer:** I would clarify which branches are private and which are shared, require short-lived owner-controlled feature branches, and define when rewriting is permitted. Use force-with-lease only after checking the expected remote state and coordinating with collaborators. Protect main/release branches and make the policy explicit; a preference for linear history does not justify disrupting shared work.

---

## 20. Revision checklist and official references

### Revision checklist — every item links to a relevant section

- [ ] Explain working tree, index, HEAD, commits, and branches — [Modules 1–2](#1-git-fundamentals).
- [ ] Explain Git objects, commit ancestry, hashes, packfiles, and reflog — [Module 2](#2-internals-and-object-model).
- [ ] Distinguish fast-forward, three-way, squash, and octopus merges — [Module 3](#3-branching-and-merging).
- [ ] Choose merge or rebase based on whether history is shared — [Module 4](#4-rebase-and-history-management).
- [ ] Explain fetch, pull, push, tracking refs, and non-fast-forward rejection — [Module 5](#5-remotes-and-collaboration).
- [ ] Compare trunk-based development, feature branches, Git Flow, and release branches — [Module 6](#6-branching-strategies-for-product-teams).
- [ ] Describe atomic commits, review quality, merge strategies, and signatures — [Module 7](#7-commit-quality-and-pull-requests).
- [ ] State the effects of soft, mixed, and hard reset on HEAD, index, and working tree — [Module 8](#8-reset-revert-restore-and-checkout).
- [ ] Explain revert of a merge commit and mainline parent selection — [Module 8](#8-reset-revert-restore-and-checkout).
- [ ] Use stash and cherry-pick safely for context switching and backports — [Module 9](#9-stash-cherry-pick-and-selective-changes).
- [ ] Find regressions with log, blame, diff, reflog, and bisect — [Module 10](#10-history-inspection-and-debugging).
- [ ] Explain release tags, tag immutability policy, and artifact traceability — [Module 11](#11-tags-releases-and-versioning).
- [ ] Explain .gitignore, tracked files, line endings, attributes, and LFS — [Module 12](#12-configuration-and-repository-hygiene).
- [ ] Distinguish client hooks from server-enforced safeguards — [Module 13](#13-hooks-automation-and-team-safeguards).
- [ ] Respond to a leaked secret by rotating first and coordinating history cleanup — [Module 14](#14-security-and-sensitive-data-recovery).
- [ ] Distinguish shallow clone, partial clone, sparse checkout, and LFS — [Module 15](#15-performance-and-large-repositories).
- [ ] Recover a mistaken reset or deleted branch without worsening the situation — [Module 16](#16-common-mistakes-and-recovery).
- [ ] Practice realistic release, collaboration, and incident scenarios — [Module 17](#17-senior-level-scenarios).
- [ ] Review high-value question summaries — [Module 18](#18-most-asked-git-interview-questions).
- [ ] Answer connected questions out loud under interview conditions — [Module 19](#19-mock-interview-rounds).

### Official references

- [Git documentation](https://git-scm.com/doc) — official manuals and reference material.
- [Git reference manual](https://git-scm.com/docs) — command options and version-sensitive behavior.
- [Pro Git book](https://git-scm.com/book/en/v2) — repository model, branching, internals, and workflows.
- [GitHub documentation](https://docs.github.com/) — GitHub-specific pull requests, protected branches, authentication, and repository policies.
- [git-filter-repo project](https://github.com/newren/git-filter-repo) — history rewriting for repository cleanup.

Use Git's own manuals to verify command semantics. Treat pull requests, required reviews, branch protection, and host-side secret scanning as hosting-provider capabilities rather than universal Git features. In an interview, explain not just the command you would run, but whether it changes the working tree, index, local history, or shared remote history—and why that is the right risk for the situation.
