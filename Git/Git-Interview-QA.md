# Git Interview Handbook — Senior Software Engineers (5–10 Years)

**Audience:** Senior software engineers and Java backend engineers with 5–10 years of experience.  
**Purpose:** A practical interview-first reference for explaining Git internals, collaborating safely, maintaining useful history, debugging regressions, and supporting reliable releases.

Questions are grouped by topic, but their `Q` numbers run continuously from the beginning to the end of the handbook. Each answer starts with the direct explanation, then develops the mechanism, trade-offs, and practical engineering implications in natural language. Examples are illustrative; command behavior can vary with Git version, configuration, operating system, and hosting-provider policy. Check `git help <command>` for exact options in your environment.

## Contents

Question numbering continues globally across all topic sections, scenario questions, and mock interview rounds; it does not restart when a new section begins.

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

### Q1. What is Git, and why is it distributed?

Git is a distributed version-control system that records project snapshots as commits connected in a directed acyclic graph. Its central benefit is that most everyday operations—committing, branching, inspecting history, comparing revisions, and experimenting—happen locally without waiting for a server. A remote repository is used to exchange objects and references, not to perform every operation on behalf of the developer.

In a centralized version-control system, the central server is usually the primary history store and many operations depend on reaching it. In Git, a normal clone contains the history it has fetched and can support local work even when offline; a shallow clone intentionally has limited history. This gives teams flexibility, but it does not itself provide code review, access control, backups, or release governance. Those depend on repository hosting, team policy, and operational practices.

### Q2. Explain the working tree, index, and repository.

The **working tree** is the set of checked-out files you edit. The **index**, also called the staging area, records the exact content and file metadata Git intends to include in the next commit. The **repository** stores objects—such as blobs, trees, and commits—and references that make history reachable.

These are three distinct layers, which is why a file can contain staged changes and additional unstaged edits at the same time. `git diff` compares the working tree with the index, while `git diff --staged` compares the index with HEAD. This separation lets you build a focused commit from only part of your current work. Before committing, inspect both staged and unstaged changes so that you know exactly what the commit will contain.

```bash
git status
git diff                 # working tree versus index
git diff --staged        # index versus HEAD
git diff HEAD            # combined tracked changes versus HEAD
git add src/Order.java
git commit -m "Handle duplicate order requests"
```

Review both staged and unstaged changes before committing. `git diff HEAD` is useful, but `git status` makes the separate states clearer.

### Q3. What are a repository, working tree, and bare repository?

A repository is the object database plus references and metadata. A working tree is a checked-out directory associated with a repository. A **bare repository** has no checked-out working tree and is commonly used as a shared remote repository. In a normal non-bare repository, Git metadata is usually in `.git/`; linked worktrees and submodules can use a `.git` file that points elsewhere. Do not assume `.git` is always a directory.

### Q4. What do the core commands do, and how do they fit together?

I use `git status` first to understand the current state. I edit files in the working tree, use `git diff` to inspect unstaged changes, and use `git add` to select the exact content that belongs in the next snapshot. I then review that snapshot with `git diff --staged` and create a commit with `git commit`. `git log` helps me verify the resulting history, while `git clone` obtains an existing repository and `git init` initializes a new one.

The key distinction is that `git add` changes the index, not the commit history. If I edit a file after staging it, the index still contains the previously staged version until I stage again. This is intentional: staging is a deliberate selection step, not merely a synonym for “save the file.”

Git generally determines tracked-file changes by comparing content and relevant metadata with the index and HEAD, using stat information to avoid rereading unchanged files where possible. It does not rely only on modification timestamps. Line-ending conversion, clean/smudge filters, file mode settings, and filesystem behavior can affect what appears as a change.

### Q5. Does Git store diffs or snapshots?

Conceptually, each commit records a snapshot through a tree object. Git's storage is content-addressed and deduplicates identical objects; packfiles may delta-compress objects for efficiency. A commit is not simply a list of file edits. This model makes branches and history operations reliable and explains why removing a large file from the latest version does not necessarily remove its historical object.

---

## 2. Internals and object model

### Q6. Explain blob, tree, commit, and annotated tag objects.

A **blob** stores file content, not its filename. A **tree** maps names and modes to blobs or other trees, representing a directory snapshot. A **commit** points to a root tree and records metadata and zero or more parent commits. An **annotated tag object** stores a target object plus tagger metadata, a message, and optionally a signature. A lightweight tag is just a reference to an object, commonly a commit; it is not a separate tag object.

### Q7. How are commits linked?

A commit references its snapshot and parent commit(s). A normal commit has one parent; a root commit has none; a merge commit has two or more parents. This forms a directed acyclic graph. Two commits with identical file contents can still have different IDs because their parent, author/committer metadata, timestamps, message, or tree can differ. Commit IDs identify the complete commit object, not merely the file contents.

### Q8. What are a branch, HEAD, and detached HEAD?

A branch is a named reference that normally points to the latest commit on that line of work. `HEAD` usually points symbolically to the currently checked-out branch; when you commit, Git creates a commit whose parent is the previous tip and advances that branch reference.

In a **detached HEAD** state, HEAD points directly to a commit instead of a branch name. This is useful for checking an old release or testing a historical revision. You can even create commits while detached, but no branch automatically advances to retain them. Before switching away, preserve useful work with `git switch -c investigation`. Detached HEAD is not an error by itself; the risk is forgetting to anchor new commits to a branch.

Example: if `main` points to C1, committing C2 while `main` is checked out moves `main` from C1 to C2. Creating `feature` at C2 creates another pointer; it does not copy the commit history.

### Q9. What are symbolic references and remote-tracking branches?

A symbolic reference points to another reference; HEAD is the common example. A remote-tracking ref such as `origin/main` is a local record of the last fetched state of the remote branch. It is not a live network pointer and does not move merely because someone else pushed. `git fetch origin` updates these references and downloads needed objects without merging into your current branch.

### Q10. What is the index, and what happens during add and commit?

The index is the snapshot Git will use for the next commit. `git add file` hashes the staged content as needed, stores the object, and updates the index entry. `git commit` creates a tree from the index and a commit object referencing that tree and the current parent, then updates the current branch reference. Hooks may run as part of the operation. This is why changes made after staging are not automatically included in the staged version.

### Q11. How do hashes, packfiles, and garbage collection work?

Git computes object IDs from the object's type, size, and content using the repository's configured hash algorithm. Hashes support object identification and integrity checking, but an object ID alone is not a complete authenticity or security guarantee. Traditional repositories use SHA-1; SHA-256 repository support exists in modern Git, but interoperability with tools and hosting providers must be checked rather than assumed. A commit signature and trusted verification policy address a different question—who signed a commit—not whether the code is safe. Loose objects are stored individually; packfiles group objects and may delta-compress similar content to reduce storage and transfer costs. Git's maintenance and garbage-collection mechanisms repack objects and eventually prune eligible unreachable objects. Exact maintenance behavior depends on Git version and configuration.

### Q12. How does Git handle renames?

Git does not normally store a dedicated “rename object.” A commit stores a tree snapshot; commands such as diff infer a rename by comparing deleted and added paths and evaluating content similarity. Detection thresholds and options can affect the result. A rename can therefore appear as delete-plus-add in some comparisons, particularly when content changes substantially.

### Q13. What are reflogs and unreachable objects?

Reflogs record recent local movements of references, including HEAD and branch refs, and are often the easiest way to recover after a reset or branch deletion. Inspect with `git reflog`, then preserve a found commit using `git branch rescue <sha>`. Reflogs are local, expire according to configuration, and are not a remote backup. If an object is no longer reachable from refs or reflogs and is eventually pruned, recovery may be impossible. Do not run aggressive cleanup while investigating lost work.

---

## 3. Branching and merging

### Q14. Why are Git branches lightweight?

A branch is essentially a reference to a commit, so creating one is cheap; it does not duplicate the full project. Switching branches changes the index and working tree to the target snapshot, subject to uncommitted-change safety checks. Lightweight branches make short-lived feature branches practical, but cheap creation does not eliminate the integration cost of keeping a branch alive for weeks.

### Q15. Create, switch, rename, and delete branches.

```bash
git switch -c feature/order-validation
git switch main
git branch -m feature/order-validation feature/order-validation-v2
git branch -d feature/order-validation
git branch -D abandoned-experiment   # force delete; verify first
```

The `-d` form refuses to delete a branch Git considers unmerged in the relevant context; `-D` overrides that safeguard and can make commits harder to find. Deleting a local branch does not delete the corresponding remote branch.

### Q16. Fast-forward versus three-way merge.

A fast-forward is possible when the current branch tip is an ancestor of the branch being merged. Git can move the current branch pointer forward without creating a merge commit. If both branches have diverged, Git normally performs a three-way merge using the common merge base and the two branch tips; a successful merge usually creates a commit with both tips as parents. `git merge --no-ff feature` forces a merge commit even when fast-forwarding would be possible, preserving an explicit integration point. Team policy should determine whether that history is desirable.

### Q17. What is merge-base?

The merge base is a best common ancestor used to compare the branches during a three-way merge. Inspect it with `git merge-base main feature`. In complex histories there can be more than one best common ancestor; Git's merge machinery can construct a virtual merge base. Do not assume the merge base is always the most recent commit visible on one branch.

### Q18. How do you resolve conflicts safely?

First inspect `git status` and the conflicted paths. Open each file, understand both changes, and decide the intended combined behavior; conflict markers are only a starting point. Stage each resolved file with `git add`, inspect `git diff --staged`, run focused tests, then complete the merge. `git merge --abort` attempts to return to the pre-merge state, but can be unable to reconstruct pre-existing uncommitted changes cleanly. Start with a clean working tree or save work before integrating.

Conflicts can occur in the same lines, in file/rename/delete interactions, or because changes cannot be reconciled structurally. Even changes to different lines can conflict due to surrounding edits or file-level operations. Git resolves text, not program intent: two edits can merge cleanly but violate an API contract, transaction boundary, or business invariant. Those are **semantic conflicts** and require tests and code review.

### Q19. What is squash merging? What is an octopus merge?

A squash merge combines the net changes from a branch into one new commit on the target branch, without recording the feature branch's individual commits as parents. The original commits remain in the graph only if reachable elsewhere. It creates a compact mainline history but loses the explicit commit ancestry between the feature branch and mainline. An octopus merge combines more than two heads in one merge commit; it is useful for several independent branches with no conflicts, but is rarely suitable for complex conflict resolution.

---

## 4. Rebase and history management

### Q20. What does rebase do?

Rebase identifies the commits unique to the branch being rebased, then reapplies their changes one by one onto a new base. Conceptually, if main advances from C to D while a feature branch has commits F1 and F2 based on C, rebasing replays the changes from F1 and F2 on top of D. The resulting commits have new parent relationships and therefore normally new commit IDs, even if their file content is unchanged.

A common workflow is `git fetch origin`, `git switch feature`, and `git rebase origin/main`. This updates the feature branch without creating a merge commit, but conflicts may occur while replaying individual commits. Rebase is most straightforward when the branch is private or its owner can coordinate the rewrite. If teammates already depend on the published commits, rewriting them can disrupt their work.

### Q21. Rebase versus merge: how do you choose?

Merge records the actual integration of divergent histories and preserves existing commit identities. Rebase rewrites the feature commits onto a new base and can make review history easier to follow. Rebase is commonly appropriate for private or single-owner feature branches; merge is safer when others already depend on the published commits. Neither is universally superior: the team should agree on history policy, review requirements, and release traceability.

### Q22. How does interactive rebase help?

`git rebase -i HEAD~4` opens an instruction list for the last four commits. You can reorder, squash or fixup commits, edit a commit, or drop it. Use this to clean up a private pull-request branch before review or merge. Do not casually rewrite commits already used by teammates. Keep meaningful logical boundaries and avoid squashing away information needed to understand a complicated change.

### Q23. Continue, skip, or abort a rebase?

After resolving conflicts, stage the resolutions and run `git rebase --continue`. `--skip` omits the current commit's patch entirely, so use it only when the change is already present or intentionally unwanted. `--abort` returns to the original branch state from before the rebase, subject to the state and any pre-existing changes. Inspect `git status` before acting; do not choose skip just to make the operation finish.

### Q24. What about shared branches and force pushes?

Rebasing published commits changes their IDs and creates a different history from the one teammates fetched. If rewriting a branch is explicitly agreed and safe, use `git push --force-with-lease` rather than a blind force push; it checks that the remote ref still has the expected value. It is still not harmless: another update can be overwritten if the lease assumptions are stale or deliberately overridden. Never rewrite protected main or release branches as a routine cleanup. Coordinate first and preserve a recovery reference if needed.

### Q25. What is `--rebase-merges`?

Ordinary rebase generally linearizes the selected commits and does not preserve the original merge topology. `git rebase --rebase-merges` attempts to recreate merge structure while replaying commits. It is useful when the branch's internal merge topology matters, but can still require conflict resolution and produces rewritten commits. Inspect the resulting graph and tests rather than assuming the original topology is reproduced perfectly.

---

## 5. Remotes and collaboration

### Q26. Fetch versus pull versus push.

These commands operate at different stages of collaboration. `git fetch origin` downloads objects needed from the remote and updates local remote-tracking references such as `origin/main`; it does not merge anything into your current branch. That makes fetch useful when you want to inspect incoming work first.

`git pull` is a convenience operation that fetches and then integrates the configured upstream. Integration may use merge or rebase depending on command-line options and configuration such as `pull.rebase`; teams should make this behavior explicit rather than assuming every developer's pull works the same way. `git push` sends local objects and asks the remote to update a reference. Normal pushes generally require a fast-forward update, and the server may apply additional permissions or branch rules. A push rejection is a signal to inspect the remote state, not an invitation to force-push blindly.

### Q27. What do origin and upstream mean?

They are conventional remote names, not special Git keywords. `origin` usually names the repository cloned from; in a fork workflow, `origin` may be your fork and `upstream` the canonical project. `git remote -v` lists configured URLs. A local branch can track a remote branch, allowing commands such as `git status` to report ahead/behind state.

### Q28. Why is a push rejected as non-fast-forward?

The remote branch has commits not contained in the local branch, so updating it directly would discard remote history from that ref. Fetch and inspect the graph, then merge or rebase according to policy. For example: `git fetch origin`, `git log --oneline --graph --decorate --all`, then `git rebase origin/feature` for a private branch or merge when preserving shared history is important. Do not immediately force push.

### Q29. What does force-with-lease protect against?

`git push --force` unconditionally requests a non-fast-forward ref update, subject to server policy. `--force-with-lease` checks that the remote ref still matches an expected value, reducing the chance of overwriting an update you have not observed. It does not replace coordination, review, or branch protection, and an explicitly supplied or stale expectation can weaken its safety.

### Q30. What are shallow and partial clones?

A shallow clone limits history depth, reducing initial transfer but limiting ancestry-dependent operations and history inspection. A partial clone can omit some objects initially—for example, blob filtering—and fetch them on demand. Sparse checkout controls which paths are materialized in the working tree; it is not the same as filtering history or objects. These features can be combined but solve different problems.

### Q31. What should you know about submodules?

A submodule records a specific commit of another repository, not a floating branch tip. Consumers must initialize and update submodules, and the referenced commit must remain available. This creates extra onboarding, CI, and dependency-update steps. Submodules can be useful for independently versioned dependencies; avoid them if the team cannot reliably manage nested repository state.

### Q32. How do SSH keys and HTTPS credentials fit in?

Git transports data over a configured protocol; SSH typically authenticates with keys, while HTTPS commonly uses a credential manager or token-based authentication. Authentication identifies the caller, while repository permissions and server rules decide what they may do. Never embed credentials in remote URLs or commit them to config files. Use the organization's approved credential storage and rotate exposed credentials.

---

## 6. Branching strategies for product teams

### Q33. Feature branching.

A feature branch isolates work and supports focused review. Keep it short-lived and integrate frequently to reduce drift. Long-lived branches accumulate incompatible assumptions and expensive merges. Feature branches work well with pull requests, but branch existence alone does not provide release isolation or prevent unfinished features from being deployed.

### Q34. Trunk-based development.

Developers integrate small changes into a shared mainline frequently, using short-lived branches or direct integration under strong safeguards. Automated tests, rapid review, feature flags, and backward-compatible changes reduce risk. It supports frequent deployment, but requires disciplined testing and operational readiness; it is not “commit anything directly to main.”

### Q35. Git Flow and release branches.

Git Flow uses long-lived development and main branches plus feature, release, and hotfix branches. It can suit scheduled releases and products that maintain release candidates, but adds branching and merge coordination. Release branches are useful when multiple versions need stabilization or support. For continuous delivery, the overhead of long-lived branches can outweigh their value.

### Q36. How would you choose?

- **Several deployments per day:** favor trunk-based development, small changes, automated checks, feature flags, and quick rollback.
- **Monthly release trains:** short-lived feature branches plus a release branch for stabilization may fit; define what can be cherry-picked into release.
- **Several supported versions:** maintain explicit release/maintenance branches and backport fixes with traceable commits.
- **Several developers on one service:** small pull requests, clear ownership, frequent integration, protected mainline, and contract tests matter more than branch naming.
- **Urgent production fix while work continues:** create a hotfix from the deployed release tag or maintenance branch, test and release it, then propagate the fix to all maintained lines.
- **A feature takes weeks but must remain hidden:** split work into compatible increments and use a feature flag; avoid carrying a large divergent branch for weeks where feasible.

Branch protection, required reviews, status checks, and merge rules are hosting-provider features or server policies, not guarantees supplied by Git itself. A feature flag needs safe defaults, authorization where relevant, and a removal plan.

---

## 7. Commit quality and pull requests

### Q37. What makes a good commit?

A good commit is a coherent, reviewable change with a clear message and an understandable purpose. Atomic commits help reviewers, bisect, revert, and backport work. Prefer `Add idempotency check to order creation` over `fix`, `updates`, or `final changes`. Avoid mixing formatting, generated output, refactoring, and business behavior unless there is a clear reason.

### Q38. What are Conventional Commits?

Conventional Commits use a format such as `feat(order): reject duplicate submissions` or `fix(auth): validate expired tokens`. Teams often use this structure to automate changelogs or versioning. It is a convention, not a rule enforced by Git; the team must define its grammar and tooling.

### Q39. Fixup and autosquash.

For a private branch, `git commit --fixup <commit>` creates a follow-up intended to be folded into a target commit. `git rebase -i --autosquash <base>` arranges fixup commits for cleanup. Since this rewrites commit IDs, do it before others depend on the history or coordinate any required force push.

### Q40. How should a senior engineer review a pull request?

Review behavior, edge cases, API and data compatibility, error handling, security, observability, tests, and rollback implications—not just style. Keep the diff focused, explain why a change is needed, and call out generated files or lockfiles that materially affect dependencies. Large pull requests increase cognitive load and hide defects; split them where possible. A green CI run is evidence, not proof of correctness.

### Q41. Squash merge, merge commit, or rebase merge?

A squash merge produces one target-branch commit and a concise mainline but does not preserve feature commits as ancestors. A merge commit preserves the branch topology and integration point. A rebase merge creates a linear mainline by replaying commits. Choose based on traceability, bisectability, backport practices, and the team's preferred audit trail. Hosting providers may implement and constrain these options differently.

### Q42. Signed commits, tags, and hooks.

Cryptographic signatures can help verify that a commit or tag was signed by a key trusted under the team's verification policy. They do not prove that the code is safe or that the signer carefully reviewed it. Client-side hooks can check messages or run tests, but users can bypass them and they may not be installed. Enforce critical policies on the server or in required checks as well.

### Q43. Why doesn't deleting a secret in a later commit solve exposure?

Earlier commits still contain the secret, and clones, forks, caches, logs, or artifacts may retain copies. Revoke or rotate the credential first, assess access logs, then decide whether to rewrite history and coordinate cleanup. History cleanup reduces future accidental exposure; it cannot make an already disclosed credential trustworthy again.

---

## 8. Reset, revert, restore, and checkout

These commands solve different problems. Before using one, run `git status`, inspect the target commit, and preserve work you may need.

### Q44. Compare reset modes.

`git reset <commit>` moves the current branch (and HEAD with it) to the target commit when HEAD is attached. Modes control what happens to the index and working tree:

- **`--soft`**: move HEAD/branch only; index and working tree remain unchanged. The changes between the new HEAD and the staged snapshot remain staged.
- **`--mixed`** (default): move HEAD/branch and reset the index to the target; preserve working-tree files. Changes remain in the working tree as unstaged modifications.
- **`--hard`**: move HEAD/branch, reset the index, and make tracked working-tree files match the target. It can discard staged and unstaged tracked changes. Untracked files are not generally removed simply because of reset, but an untracked file or directory that obstructs writing a tracked path can be removed; do not treat this as a recovery-safe command.

Use `--soft` to undo a local commit while keeping its contents staged, `--mixed` to uncommit and unstage while keeping file edits, and `--hard` only when you intentionally want to discard tracked local changes and have verified the target.

### Q45. Reset versus revert.

Reset and revert both help undo work, but they operate on history in different ways. `git reset` moves the current branch reference to another commit and, depending on the mode, also changes the index and working tree. It is therefore commonly used to reshape local, unpublished history. If the commit has already been shared, resetting and force-pushing can make collaborators' histories diverge.

`git revert` leaves existing commits in place and creates a new commit that applies the inverse of an earlier change. That is usually safer for shared branches because other developers can continue building on the same history. Revert is not a magic time machine: later commits may depend on the reverted behavior, and reversing a code change cannot automatically undo external side effects or database changes. Run relevant tests and plan any data or deployment recovery separately.

### Q46. What does restore do?

`git restore` is for restoring file content, not moving branch history. By default, `git restore path` copies the index version into the working tree, discarding unstaged edits to that path. `git restore --staged path` resets the index entry to HEAD while leaving the working-tree file untouched, which is useful when you staged the wrong file or staged too much.

To retrieve a file from another revision, use `git restore --source=<commit> -- path`; this normally updates the working tree, and you still need to stage and commit if you want the restored content recorded in history. Because restore can overwrite edits, inspect the file and status first. Unlike reset, it does not move the current branch pointer.

### Q47. What about checkout?

`git checkout` is a historical multi-purpose command used to switch branches, detach HEAD, and restore paths. Modern commands make intent clearer: `git switch` for branches and `git restore` for files. Existing teams may still use checkout, so understand it, but prefer the explicit commands in new instructions.

### Q48. How do you revert a merge commit?

A merge commit has multiple parents. `git revert -m 1 <merge-commit>` tells Git which parent is the mainline to keep as the baseline; it does not mean “revert parent one.” Choosing the wrong mainline can reverse the wrong side of the merge. Inspect `git show --no-patch --pretty=raw <merge-commit>`, understand which parent represents the target branch, and test the resulting commit. Reverting a merge also affects how Git treats later attempts to merge the same ancestry; reintroducing the feature may require reverting the revert or adding new commits.

### Q49. How do you recover uncommitted work?

First inspect `git status`, `git diff`, `git diff --staged`, editor local history, IDE backups, and any stash. Reflog generally recovers commits and reference movements, not arbitrary uncommitted edits. If a hard reset or cleanup overwrote uncommitted file contents, Git may never have stored those contents as objects, so recovery is uncertain. Stop writing to the affected working tree and avoid cleanup commands while investigating.

---

## 9. Stash, cherry-pick, and selective changes

### Q50. How does stash work?

Stash saves selected working-tree and index changes so you can switch context. `git stash push -m "WIP order validation"` saves tracked modifications; add `-u` to include untracked files. Use `git stash list`, `git stash show -p stash@{0}`, `git stash apply stash@{0}`, or `git stash pop`. Apply retains the stash; pop removes it after a successful application. Conflicts can occur, and a stash is not a substitute for a durable commit or backup.

### Q51. What is cherry-pick?

Cherry-pick applies the change introduced by a commit onto the current branch and records a new commit (unless options alter the operation). The new commit normally has a different ID because it has a different parent. It is useful for backporting a narrowly scoped fix without merging an entire feature branch. If conflicts occur, resolve and stage files, then use `git cherry-pick --continue`; use `--abort` to cancel the operation.

### Q52. Merge versus rebase versus cherry-pick.

Merge integrates histories and preserves ancestry. Rebase replays a series of commits on a new base, rewriting their IDs. Cherry-pick applies selected commit changes without integrating the source branch's ancestry. Repeated cherry-picking can create duplicate or equivalent changes that later complicate merges. If a fix must be maintained on multiple release lines, cherry-pick is often appropriate; for regular synchronization between branches, merging is usually clearer.

---

## 10. History inspection and debugging

### Q53. Commands senior engineers should know.

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

For `git diff A...B`, Git compares the merge base of A and B with B (the right-hand revision). This is commonly useful for reviewing what a feature branch adds since it diverged from main. It is not the same as comparing the two tips directly with `git diff A B`; range notation also has different meanings in `git log`. Be precise about endpoints when debugging. `git blame` shows the commit last responsible for each line in the selected file state; it does not prove who designed the behavior or why.

### Q54. How does git bisect find a regression?

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

### Q55. How do you recover a deleted branch?

Run `git reflog --all` and inspect recent branch/HEAD movements. If you find the lost tip commit, create a rescue branch immediately: `git branch rescue/feature <sha>`. A deleted branch reference does not instantly delete its commit objects. Recovery depends on whether the commit remains reachable through reflogs or other refs and whether garbage collection has pruned it.

### Q56. How do you compare a release with current code?

Use `git diff v2.4.0..main` to compare the two trees and `git log --oneline v2.4.0..main` to list commits reachable from main but not from the tag. Verify the tag's target with `git show v2.4.0`. For a production regression, combine release comparison with deployment metadata, logs, tests, and bisect rather than assuming the newest commit caused it.

---

## 11. Tags, releases, and versioning

### Q57. Lightweight versus annotated tags.

A lightweight tag is simply a reference to an object, usually a commit. An annotated tag creates a separate tag object containing the target, tagger identity and timestamp, a message, and optionally a cryptographic signature. For a release, an annotated tag is often preferable because it can record release context and can be verified under a signing policy.

A tag created from a branch resolves to the branch's current target at that moment; it does not follow the branch as it moves. Git itself does not enforce immutability, so protecting release tags and preventing accidental retagging is a repository or hosting-policy responsibility. For production traceability, record the tag alongside the exact source commit and built artifact identifier.

### Q58. Create, inspect, and publish a tag.

```bash
git tag -a v2.4.0 -m "Release 2.4.0"
git show v2.4.0
git push origin v2.4.0
git push origin --tags
```

The last command pushes all tags Git is configured to push under that command's semantics; inspect tags before using it in release automation. Deleting or moving a published tag can break deployment traceability because consumers may have cached or acted on the original target. Prefer a new corrective tag/version unless the release policy explicitly permits repair.

### Q59. How does Semantic Versioning relate to Git?

Semantic Versioning defines version-number conventions for public APIs; Git itself does not enforce them. A tag such as `v2.4.0` is a human/project convention that marks a commit. For backend services, a release record should tie the tag to the built artifact or image digest, source revision, configuration, and deployment. A tag alone does not guarantee that the deployed binary was built from that commit.

### Q60. What if a release tag points to the wrong commit?

Do not silently move a tag that a pipeline or users have already consumed. Identify what was built and deployed, stop or correct further rollout if needed, publish a clear corrective release/tag according to policy, and ensure the artifact/source mapping is unambiguous. If the tag must be repaired, coordinate all consumers and audit systems that may have cached it. For several services in separate repositories, maintain a release manifest or deployment record tying compatible versions together.

---

## 12. Configuration and repository hygiene

### Q61. Git configuration levels.

`git config --local` affects the current repository, `--global` affects the user's configuration, and `--system` affects the installation/system scope where supported. More specific configuration generally overrides broader settings. Set identity with `git config user.name "Name"` and `git config user.email "email@example.com"`; inspect with `git config --show-origin --list`. Identity fields are commit metadata and are not authentication credentials.

### Q62. Why doesn't .gitignore untrack a file?

`.gitignore` prevents matching untracked files from being offered for addition; it does not remove files already tracked. To stop tracking a file while retaining the local copy, use `git rm --cached path`, then commit the index change and appropriate ignore rule. Be careful with broad patterns that may hide required files from future additions.

### Q63. Line endings, attributes, and cross-platform changes.

Windows commonly uses CRLF while Unix-like systems use LF. Git's `core.autocrlf` and `.gitattributes` can normalize line endings; team-level attributes are often more reproducible than relying on each developer's personal configuration. For example, `* text=auto` lets Git detect text files and normalize them, while explicit rules can set LF for source files. Before committing a huge apparent diff, inspect whitespace and line-ending changes and agree on repository policy.

### Q64. File modes, case sensitivity, and generated files.

Git generally tracks the executable bit for files, not every permission. `core.filemode` affects mode detection on filesystems that do not preserve modes reliably. Case-insensitive filesystems can make case-only renames awkward; use a deliberate two-step rename if necessary and verify the staged tree. Generated files should be committed only when project policy requires them. Large generated output inflates diffs and history and often belongs in a build pipeline or artifact store.

### Q65. Git LFS, monorepos, submodules, and subtree.

Git LFS stores pointer files in Git while the large content is managed by an LFS server; access, quotas, and availability become additional dependencies. Removing a large file from the current tree does not remove prior blobs from history. A monorepo centralizes source and can simplify atomic cross-component changes but needs clear ownership and performance practices. Submodules pin an external repository commit; subtree workflows copy or merge external content into a directory and avoid requiring nested checkout, but can complicate synchronization. Choose based on release coupling, contributor experience, and tooling.

---

## 13. Hooks, automation, and team safeguards

### Q66. Client-side versus server-side hooks.

Client hooks such as `pre-commit`, `commit-msg`, and `pre-push` can format code, validate commit messages, or run quick checks. They are useful feedback but can be bypassed and may not be installed by default. Server-side hooks and hosting-platform branch rules can reject prohibited updates; required CI checks and reviews enforce policy more consistently across contributors. Do not rely on a local hook as the only secret-scanning or quality gate.

### Q67. How should a team prevent unsafe changes?

Protect main and release branches, require appropriate reviews and checks, limit who can bypass protections, and make rollback paths clear. Add secret scanning and dependency checks where appropriate. Hooks should be fast and actionable; slow full test suites often belong in CI. Git records history but does not itself provide a universal pull-request or required-check system—those are host and team capabilities.

### Q68. What do signed commits prove?

A valid signature can establish that a commit object was signed by a particular key, subject to key trust and verification policy. It does not prove the author identity beyond that trust model, the code's safety, or that required tests passed. Signed tags can help authenticate release markers. Keep key management and verification policy documented.

---

## 14. Security and sensitive-data recovery

### Q69. A secret was committed and pushed. What first?

**Revoke or rotate the credential immediately.** Treat it as compromised even if the repository is private, because other users, automation, forks, logs, or caches may already have copied it. Check the credential's scope and usage logs, replace it through the approved secret-management system, and verify that dependent services are using the replacement.

After containment, remove the secret from the current source and search for other occurrences. Decide whether to rewrite history using a tool such as `git filter-repo`; this changes commit IDs and requires coordination across branches, remotes, and contributors. History rewriting reduces future accidental exposure, but cannot guarantee removal from existing clones, forks, CI artifacts, caches, logs, or backups. Rotation is the security control; history cleanup is a separate hygiene step.

### Q70. When is git-filter-repo appropriate?

`git filter-repo` can rewrite repository history to remove sensitive files or replace exposed strings. It changes commit IDs and can affect tags, branches, and merge ancestry. Plan a maintenance window, back up the repository, coordinate contributors, update all relevant refs, and communicate how everyone should reclone or realign their work. Force pushing rewritten refs can overwrite concurrent updates; protect the repository during the operation and verify the result. Prefer the maintained `git-filter-repo` tool over obsolete recipes when a full history rewrite is required.

### Q71. What does access control and least privilege mean for Git?

Grant write/admin rights only to people and automation that need them, protect important branches, use short-lived or narrowly scoped credentials where available, and audit access. Signed commits, branch protections, and review checks reduce certain risks but do not eliminate malicious changes or compromised accounts. Assess dependencies, build scripts, and repository content as supply-chain inputs; a clone can contain executable hooks only through explicit local setup, but build/test scripts and dependencies may execute when developers or CI run them.

---

## 15. Performance and large repositories

### Q72. Why can a repository become slow or large?

Large binary files, generated output, many tiny files, expensive untracked-file scans, oversized history, and filesystem limitations can slow status, checkout, clone, or fetch. Diagnose the actual bottleneck before changing repository structure. Check `git status`, inspect tracked large files and history, and consider Git's trace/performance facilities when needed. Avoid committing build directories, IDE caches, and transient logs.

### Q73. Shallow clone, partial clone, sparse checkout, and LFS are different.

- **Shallow clone** limits commit history, so ancestry and history operations may be constrained.
- **Partial clone** omits selected object types until needed, reducing initial transfer but requiring network access later.
- **Sparse checkout** limits which paths appear in the working tree; the repository may still know about other paths.
- **Git LFS** stores large content outside ordinary Git blobs, represented by pointers in the repository.

Choose based on whether the bottleneck is history depth, object transfer, working-tree file count, or large binary assets. CI jobs that need tags, merge-base calculations, changelogs, or bisect may fail or behave differently with a shallow history.

### Q74. Why doesn't deleting a large file shrink history?

The deletion creates a new tree that no longer references the file, but older commits still reference the blob. Storage is reclaimed only after history is rewritten to remove the historical references and unreachable objects are eventually pruned; remote hosting retention and clones can still retain copies. Rewrite history only when the benefit justifies disruption, and coordinate the migration.

### Q75. What do packfiles and garbage collection mean operationally?

Packfiles consolidate objects and use delta compression; maintenance can improve storage and access efficiency. Garbage collection may prune unreachable objects after grace periods and repack reachable ones. Do not run aggressive cleanup as a first response to lost work, because it can reduce recovery options. On hosted platforms, local maintenance does not guarantee immediate reclamation of server-side storage.

---

## 16. Common mistakes and recovery

### Q76. I committed to the wrong branch.

If the commit is local and not shared, inspect status and the graph. One safe route is to create the intended branch at the current commit, then move the original branch back only after confirming no desired work will be lost. For example, `git branch feature/correct-branch` preserves the commit before you reset the mistaken branch. If already pushed or shared, avoid rewriting without coordination; a revert or follow-up commit may be safer.

### Q77. I staged generated files by mistake.

Run `git diff --staged` to identify the exact staged files. Use `git restore --staged path` to unstage while keeping local content, add the correct ignore rule if appropriate, and inspect status again. If the file was already committed, removing it from the current tree does not erase its history; if it contains a secret, follow the security process in Module 14.

### Q78. I am stuck during a merge or rebase.

Run `git status` first; it identifies the operation and conflicted files. For a merge, resolve and stage files then commit, or use `git merge --abort` if abandoning is appropriate. For a rebase, resolve/stage and run `git rebase --continue`, use `--skip` only if the commit's change should be omitted, or `--abort` to return to the pre-rebase state. Do not mix commands from different operations or delete Git metadata manually.

### Q79. I reset the wrong commit or deleted a branch.

Stop and inspect `git reflog --all`. Identify the exact commit, verify it with `git show <sha>`, then create a rescue branch before further changes. Reflog recovery is best-effort and local; it cannot reliably recover edits that were never committed or stashed. Avoid garbage collection until recovery is complete.

### Q80. I pushed an incomplete or incorrect commit.

If the branch is shared, prefer a follow-up fix or revert and communicate impact. If it is a private feature branch and no one depends on it, amend or rebase may be appropriate. Check CI and deployment state before deciding; a push does not necessarily mean deployment, but automation may trigger from it. Never assume a force push undoes an already deployed artifact.

### Q81. I resolved a conflict but broke behavior.

Inspect the merge commit diff and compare each conflicted file against both parents using `git show <merge>^1:path` and `git show <merge>^2:path`. Run focused regression tests and validate contracts, data migrations, and error paths. If already released, use a tested revert or corrective commit according to deployment policy. A conflict-free merge is only a textual result.

---

## 17. Senior-level scenario-based interview questions

### Q82. Your feature branch has diverged from main, and teammates have already pulled your commits. How do you update it?

Because teammates depend on the published commits, I would avoid rebasing the shared branch and force-pushing rewritten history. I would fetch the latest refs, inspect the graph, and merge the updated mainline into the feature branch, or follow the team's agreed shared-branch policy. For example, `git fetch origin`, `git switch feature`, `git merge origin/main`. I would resolve conflicts based on intended behavior, run tests, and push the merge commit. If the team explicitly requires a linear history, I would coordinate a planned rewrite and ensure every consumer understands how to realign; I would not surprise them with it.

### Q83. A password was committed and pushed. What do you do?

I would revoke or rotate the credential first, check its permissions and usage logs, and replace it in the appropriate secret store. Then I would remove it from current source and use secret scanning to find other copies. If history cleanup is required, I would plan a coordinated `git filter-repo` rewrite, update affected refs, and instruct contributors to reclone or carefully realign. I would assume forks, caches, CI artifacts, and existing clones may retain the old value. Rewriting history without rotating the secret is insufficient.

### Q84. A production release introduced a regression, but several commits were merged afterward. How do you isolate and reverse it?

I would identify the exact deployed artifact and source revision first, then compare the release tag to the prior known-good release. I would reproduce the failure with a focused test and use `git bisect` between good and bad revisions if the test classifies intermediate commits reliably. Once identified, I would assess whether reverting the offending commit is safe given dependent changes; a merge commit requires choosing the correct mainline parent. I would validate on a release branch, deploy through the normal rollback/roll-forward process, and backport the correction where necessary. If impact is ongoing, mitigation takes priority over completing a perfect root-cause analysis.

### Q85. You must backport a critical fix to two maintained release branches. Merge, cherry-pick, or rebase?

If the fix is isolated in a single commit, I would usually cherry-pick it onto each maintenance branch, resolve any version-specific differences, and test both branches. The cherry-picked commits will have different IDs because their parents differ. If the fix depends on other commits, I would identify and port the full dependency set rather than cherry-picking blindly. I would avoid merging all of main into an older release unless that is the team's established policy, because it could introduce unrelated features. Record which branches received the fix.

### Q86. A developer ran reset --hard and cannot find their commit.

I would ask them to stop cleanup and run `git reflog --all`. If the commit appears, inspect it and immediately create a rescue branch. I would also check other local branches, remote refs, teammates' clones, PR refs, and CI logs if relevant. Reflog can recover prior branch tips and committed objects, but not reliably recover uncommitted file edits overwritten by reset. I would avoid `git gc` or pruning during investigation.

### Q87. A pull request has unrelated changes and repeated conflict-resolution commits. How do you clean it up?

First I would determine whether the branch is private or whether others have based work on it. For a private branch, I would create a backup ref and use interactive rebase to reorder, fixup, or drop accidental commits, then inspect the final diff against the merge base and rerun tests. If shared, I would prefer additive cleanup or coordinate the rewrite and use force-with-lease only after agreement. I would split unrelated concerns into separate pull requests where practical; polishing history must not hide behavior changes or erase useful rationale.

### Q88. The team deploys many times per day. Git Flow or trunk-based development?

I would usually favor trunk-based development with short-lived branches, small reviewed changes, automated tests, protected mainline, feature flags, and backward-compatible database/API changes. This reduces long-lived branch drift and supports continuous delivery. Git Flow can work, but its long-lived development/release branches often create integration overhead for frequent deployments. The decision depends on regulatory requirements, release cadence, rollback capability, and whether multiple product versions must be maintained.

### Q89. Rebase stopped on a conflict. Should the developer continue or abort?

I would first inspect `git status` and the current patch. If the conflict can be resolved according to intended behavior, stage the files and run `git rebase --continue`, then inspect the resulting history and run tests. If the base or operation was chosen incorrectly, or the resolution is uncertain, `git rebase --abort` is safer than improvising. `--skip` is appropriate only when the current commit's change is already present or intentionally omitted. I would not force-push until the rewritten result is reviewed and the branch's sharing status is clear.

### Q90. A push is rejected as non-fast-forward. What is your process?

I would fetch and inspect the local and remote graphs. The remote may contain a teammate's commit, a server-generated change, or simply history I have not fetched. I would merge or rebase the remote-tracking branch according to whether the local commits are shared, resolve conflicts, test, and push again. I would not force-push just to bypass the rejection, because it may erase someone else's work.

### Q91. A large file was deleted, but repository size remains high. Why?

The earlier commits still reference the blob. A normal deletion only changes the current snapshot. I would locate the file's historical paths and determine whether it is a binary artifact, sensitive file, or legitimate history. If removal from all history is justified, I would use a planned history rewrite, coordinate clones and refs, and account for hosting retention and LFS storage. If historical traceability matters and the file is not sensitive, moving future versions to artifact storage or LFS may be preferable to rewriting all history.

### Q92. A merge succeeded but tests now fail. What next?

I would treat Git's merge success as a statement that it constructed a tree, not that the program is correct. I would inspect the merge commit against both parents, focus on resolved files, and run tests around the changed contracts and shared assumptions. I would also look for semantic conflicts such as inconsistent validation, changed error semantics, or a migration that is incompatible with one caller. Fix forward or revert the integration based on deployment state and release policy.

### Q93. A release tag points to the wrong commit and deployment consumed it.

I would establish which artifact was built and deployed, pause further automation if it might repeat the error, and publish a clear corrective release or tag under the team's policy. I would not silently move a widely consumed tag because clients and pipelines may have cached the original target. I would record the source SHA and artifact digest, audit downstream consumers, and only repair the existing tag with explicit coordination if the release process allows it.

### Q94. You need to undo a bad change already merged to main.

I would usually use `git revert <commit>` to create a new commit that reverses the change, because rewriting shared main history would disrupt collaborators and automation. For a merge commit, I would inspect its parents and choose the correct mainline with `git revert -m`. I would run tests because later commits may depend on the bad change. If the deployment is currently causing harm, I would also follow the incident rollback process; source-control reversal alone does not undo external side effects or database mutations.

### Q95. Git shows a massive diff after switching operating systems.

I would check `.gitattributes`, `core.autocrlf`, and whether the changes are only line endings or whitespace. I would avoid committing a broad normalization accidentally. Agree on repository-level normalization, apply it as a separate intentional change if needed, and inspect the staged diff with whitespace checks. Case-only path changes and executable-bit differences can also behave differently across filesystems.

### Q96. A long-running feature needs selective rollout while other work continues.

I would break the feature into backward-compatible, small changes behind a default-off feature flag. That allows integration with mainline without exposing the unfinished behavior. The flag should be tested, monitored, access-controlled if it gates sensitive behavior, and removed after rollout. If the feature cannot be made safe when partially integrated, I would minimize branch lifetime and define explicit integration points rather than leave an untested branch diverging indefinitely.

---

## 18. Most-asked Git interview questions

### Q97. What is the difference between fetch and pull?

Fetch downloads remote objects and updates remote-tracking refs without integrating into the current branch. Pull performs fetch and then integrates via merge or rebase according to command options and configuration. Fetch is the better first step when you want to inspect changes before integration.

### Q98. Merge versus rebase?

Merge combines histories and preserves commit identities, often with a merge commit when branches diverged. Rebase replays commits on a new base, rewriting their IDs and often creating linear history. Use merge for shared history or explicit integration records; use rebase mainly for private feature-branch cleanup under team policy.

### Q99. Reset versus revert versus restore?

Reset moves the current branch and can also reset the index and working tree. Revert creates a new commit that reverses an earlier commit. Restore copies content into the working tree and/or index without moving the branch. Choose based on whether the goal is local history cleanup, safe undo of shared history, or file restoration.

### Q100. Soft, mixed, and hard reset?

Soft moves HEAD/branch and leaves index and working tree unchanged. Mixed moves HEAD and resets the index while preserving working-tree edits. Hard moves HEAD and resets both index and tracked working-tree content, potentially discarding local changes. Inspect status and preserve work before any reset; hard reset is not a general undo command.

### Q101. What is detached HEAD?

HEAD points directly to a commit rather than symbolically to a branch. It is useful for inspection or testing older revisions. If you create commits and want to retain them, create a branch at that commit before switching away.

### Q102. What is reflog?

Reflog records recent local reference movements, making it useful for recovering commits after reset or branch deletion. It is local, expires, and does not guarantee recovery of uncommitted edits or objects already pruned.

### Q103. Fast-forward versus three-way merge?

Fast-forward moves a branch pointer when its current tip is an ancestor of the target. A three-way merge combines two diverged tips using a merge base and typically creates a merge commit. Fast-forward does not create a merge commit unless options such as `--no-ff` require one.

### Q104. Why use force-with-lease instead of force?

Force push can replace a remote ref even when it contains work you have not incorporated. Force-with-lease checks an expected remote value and reduces accidental overwrites, but it is still a history rewrite and should be coordinated on shared branches.

### Q105. What is cherry-pick, and how does it differ from merge?

Cherry-pick applies one or more selected commit changes to the current branch and creates new commits. Merge integrates branch history and ancestry. Cherry-pick suits isolated hotfix backports; regular branch synchronization is often clearer with merge.

### Q106. What is git bisect?

Bisect binary-searches a commit range using known-good and known-bad revisions. It works best with a deterministic test that can classify each revision. Flaky tests, environment differences, and unbuildable intermediate commits reduce confidence.

### Q107. How do tags differ from branches?

Both are refs, but branches normally move as new commits are created, while tags are intended to identify a fixed point such as a release. Annotated tags add metadata and can be signed. Git does not enforce that tags are immutable, so teams should protect release tags by policy.

### Q108. Why doesn't .gitignore ignore a tracked file?

Ignore rules affect untracked files. To stop tracking an already committed file while keeping it locally, use `git rm --cached path` and commit the index change plus ignore rule.

### Q109. How do you resolve a merge conflict?

Inspect status, understand both sides, edit to the intended result, stage resolved files, review the staged diff, and run relevant tests. Textual resolution is not enough; check semantic behavior and contracts.

### Q110. How do you undo a merge already pushed?

Usually create a revert commit. For a merge commit, inspect its parents and select the correct mainline using `git revert -m`. Test dependent changes afterward. Avoid resetting and force-pushing shared mainline history unless an explicitly coordinated recovery policy requires it.

### Q111. How do you remove a secret from Git?

Rotate or revoke it immediately, then remove the secret from current code and assess history rewriting with `git filter-repo`. Coordinate rewritten refs and clones. History rewriting cannot erase all copies and does not make the old credential safe.

### Q112. What are shallow clone, sparse checkout, and Git LFS?

Shallow clone limits commit history; sparse checkout limits materialized paths; Git LFS keeps large content outside normal Git blobs and stores pointers in Git. They solve different performance/storage problems and have distinct CI and collaboration trade-offs.

---

## 19. Mock interview rounds

These rounds test not only command knowledge, but also whether you can explain Git's underlying model, reason about collaboration risks, and choose a safe recovery strategy. In an interview, state your assumptions, explain why the command fits the situation, and mention how you would validate the result.

### Round 1 — Fundamentals and internals

### Q113. Walk me through what happens when you edit a Java file, stage it, and commit it.

Editing a Java file changes the working tree: the files on disk no longer necessarily match the version recorded by the current commit. Git does not automatically put those edits into the next commit. When I run `git add src/main/java/com/example/OrderService.java`, Git takes the file content as it exists at that moment and updates the index (also called the staging area). The index is the proposed snapshot for the next commit, so it is possible for one file to have staged changes and additional unstaged changes at the same time.

Before committing, I use `git diff --staged` to review what is in the index and `git diff` to review remaining unstaged changes. `git status` summarizes the relationship between the working tree, index, and current commit. When I run `git commit`, Git writes the necessary blob and tree objects from the indexed content, creates a commit object containing the root tree, parent commit, author and committer metadata, message, and any applicable headers, then advances the current branch reference to that new commit. In an ordinary commit, the parent is the previous tip of the branch. `HEAD` normally identifies that current branch. I would then run the relevant unit tests and checks; Git records the snapshot, not whether the Java code compiles or behaves correctly.

### Q114. Why might identical files produce different commit IDs?

A commit ID is a hash of the commit object, not just a hash of the files in the working tree. A commit object points to a root tree and records its parent commit or parents, author and committer identities and timestamps, the commit message, and potentially additional headers such as a signature. Therefore, two commits can point to exactly the same tree—the same tracked paths, file contents, and file modes—and still have different commit IDs because their parent, metadata, message, or other commit-object data differs.

This matters during history operations. Rebasing normally replays commits onto a different parent, creating new commit objects and therefore new commit IDs even when the resulting source files look identical. Amending a commit also normally creates a new object. By contrast, two branches can point to the same commit if they refer to the same existing commit object. For a senior engineer, the key distinction is between **content identity** (the tree and its objects) and **history identity** (the commit object and its ancestry/metadata). A changed commit ID does not by itself mean the source code changed, but I would inspect the tree and diff before concluding that two revisions are functionally equivalent.

### Q115. A developer says Git stores every complete copy of every file per commit. Is that accurate?

Not as a description of Git's physical storage. Conceptually, each commit represents a snapshot of the tracked project at a point in time. A commit points to a tree, and tree objects describe directory entries, file modes, and references to blobs or subtrees. A blob stores file content. If an unchanged file has exactly the same content, Git can reuse the same content-addressed blob rather than creating a fresh copy just because a new commit was made. This object model is why it is more accurate to say that Git records snapshots than to describe its history as a sequence of file-level patches.

The physical object database also uses storage optimizations. When Git packs objects, it can delta-compress objects against other objects to reduce disk usage; the logical object model remains the same even though the on-disk representation is more compact. Git does not need to record a special rename object for every file move either: rename detection is generally inferred from similarity when Git presents diffs or merges. So the useful interview answer is that commits behave like snapshots at the logical level, while content reuse and packfile compression keep storage efficient. It is neither a naive full independent copy of every file in every commit nor merely a permanent list of patches that must be replayed to understand the latest state.

### Round 2 — Branching and collaboration

### Q116. Your feature branch is three commits behind main and has two local commits. How do you integrate main?

First I would fetch the latest remote state rather than make a decision using a potentially stale local branch:

```bash
git fetch origin
git log --oneline --graph --decorate --left-right origin/main...feature/my-feature
```

The graph helps establish whether the feature branch has diverged, which commits exist only on each side, and whether anyone else relies on the feature commits. If the two feature commits are private to me and the team prefers linear history, I can rebase them onto the updated mainline with `git switch feature/my-feature` followed by `git rebase origin/main`. Git replays each feature commit on top of the new base; conflicts may need to be resolved for individual commits, after which I would use `git add` and `git rebase --continue`, or `git rebase --abort` if the operation should be abandoned.

If the branch is shared and teammates may already have based work on those commits, I would normally merge `origin/main` into the feature branch instead, preserving the existing commit identities. In either case, I would review the final diff, run unit and integration tests, and check that the merge or rebase did not accidentally omit changes. A normal push is appropriate after a merge. Rebasing a previously published branch can require a coordinated force push; I would only do that under team policy and with an explicit lease, never as an automatic response to a rejected push.

### Q117. Git reports non-fast-forward. Is force push the solution?

No. A non-fast-forward rejection is a safeguard: the remote branch has moved in a way that is not an ancestor of the commit I am trying to push, so updating it directly would discard or overwrite remote history from the branch's point of view. The first step is to fetch and inspect both histories, not to bypass the check:

```bash
git fetch origin
git log --oneline --graph --decorate --left-right HEAD...origin/my-branch
```

I would determine whether the remote contains teammates' commits, whether I am pushing the intended local branch to the intended remote branch, and whether the remote changed because of another push or an earlier history rewrite. If the histories should be combined, I merge or rebase according to whether the local commits are private and the team's workflow, then test and push normally.

A force push is only appropriate when rewriting that remote branch is intentional and coordinated—for example, after rebasing my own published feature branch where the team permits it. In that case, `git push --force-with-lease` is safer than `--force` because it checks the expected remote state before replacing it. For particularly sensitive cases, I would supply the exact expected remote commit to the lease. Even then, it is not a substitute for inspecting the graph and coordinating with collaborators. It protects against a class of concurrent updates; it does not make overwriting shared history harmless.

### Q118. The merge has no conflicts, but integration tests fail. Why?

A conflict-free merge only means Git was able to combine the relevant changes according to its merge rules. Git primarily operates on file content and repository history; it does not understand whether two independent edits remain compatible at the application level. For example, one branch might change a Java method's return contract while another branch updates its caller, or one branch might introduce a database column while another changes the query or migration order. If changes touch different lines or different files, Git may merge them cleanly even though the combined application is invalid.

I would first keep the merge result available for diagnosis rather than assume the merge itself is technically wrong. I would reproduce the failing integration test, identify which contract or behavior broke, and review the merge result against each parent—for example, `git diff HEAD^1 HEAD` and `git diff HEAD^2 HEAD` when `HEAD` is the merge commit. I would inspect API schemas, configuration, database migrations, event payloads, test fixtures, and dependency versions as relevant. Then I would add or correct tests that exercise the combined behavior, make the smallest sound fix, and rerun the affected test suite. The senior-level point is that merge success proves source-control compatibility, not semantic or runtime correctness.

### Round 3 — Recovery and debugging

### Q119. You need to find which commit introduced a reproducible production bug.

I would first establish that I am debugging the right source history. Ideally, the deployed artifact records its source commit SHA, build metadata, and dependency versions. I would identify a known-good revision and a known-bad revision that can both be built and tested under a comparable environment, then create or identify a deterministic regression test. If the issue is reproducible across the range, `git bisect` can narrow the search by repeatedly testing revisions between those two endpoints:

```bash
git bisect start
git bisect bad <known-bad-commit>
git bisect good <known-good-commit>
# Build and run the regression test for each checked-out revision
git bisect good   # or: git bisect bad
git bisect reset
```

Git selects intermediate commits in a search that usually reduces the number of revisions that need testing substantially compared with checking each commit in order. I would mark a revision as skipped if it cannot be built or tested reliably; I would not label an untestable revision good or bad just to move forward. Once bisect identifies a likely introducing commit, I would verify the behavior at that commit and its parent, inspect the code and relevant dependency/configuration changes, and determine the actual root cause. Finally, I would choose a tested corrective change or revert based on release impact and dependent changes, then verify the fix against the production-relevant test path. Bisect finds a likely point of introduction; engineering validation establishes the cause.

### Q120. A teammate accidentally deleted a branch after committing work.

I would first look for a surviving reference to the commit rather than recreate the work manually. Branch deletion removes the branch name; it does not necessarily delete the commits immediately. I would inspect reflogs and any other references:

```bash
git reflog --all --date=iso
git log --all --oneline --decorate --graph
```

If I find the former tip commit, I would create a rescue branch immediately so that the commit remains reachable by a named reference:

```bash
git branch rescue/recovered-feature <commit-sha>
git switch rescue/recovered-feature
```

I would also check whether the branch still exists on the remote, whether a pull request references it, or whether a teammate has a local clone with the commit. Reflogs are local records and can expire; a deleted branch's own reflog may no longer be available, so recovery is not guaranteed. If normal references and reflogs fail, an experienced Git user may investigate unreachable objects with `git fsck --no-reflogs --unreachable`, but this is a more advanced recovery path and can be affected by object pruning and garbage collection.

The important distinction is that a commit that was created can often be recovered while its object remains in the repository, even if its branch name was deleted. Work that was never committed may not exist as a Git object at all; in that case I would also check the editor's local history, backups, or other filesystem recovery options.

### Q121. When is reset appropriate?

I use `git reset` mainly when adjusting local history or moving the current branch reference, after checking what I want to keep. The three commonly discussed modes differ in how they update the index and working tree. `git reset --soft <commit>` moves the current branch/HEAD to the target commit but leaves both the index and working tree as they were, which is useful when I want to recombine or rewrite recent commits while retaining the staged snapshot. `git reset --mixed <commit>`—the default—also resets the index to match the target, but leaves the tracked working files in place; the changes are then unstaged relative to the target. `git reset --hard <commit>` updates the branch, index, and tracked working-tree content to the target. It can discard tracked uncommitted changes, so I would inspect `git status` and preserve anything valuable before running it. It should not be described as a general command for deleting all untracked files.

The context matters as much as the mode. If a commit has not been shared and I need to reorganize local history, reset can be appropriate. If a commit has already been pushed to a shared branch, rewriting the branch can disrupt everyone who based work on it; `git revert <commit>` is generally safer because it records a new commit that applies the inverse change without removing the published commit from history. If I only need to discard or restore changes to specific paths, `git restore` may communicate the intent more clearly. Before a destructive operation, I verify the target SHA, branch, status, and recovery options rather than rely on memory.

### Round 4 — Releases and senior judgment

### Q122. A security fix must go to three supported service versions.

I would treat this as a controlled backport, not as a reason to merge the entire development branch into every release line. First I would identify the three supported branches, the exact deployed versions, the vulnerability's root cause, and whether the fix depends on APIs, libraries, database changes, or configuration added after those release branches diverged. I would create a small, focused fix and follow the team's policy for which branch receives the canonical implementation first. If the fix is represented by an isolated commit and each target branch has compatible code, `git cherry-pick <fix-commit>` is often practical; if the branches have diverged substantially, I may need to port the change manually and preserve separate commits with clear references to the original fix.

A successful cherry-pick only means Git applied the patch; it does not prove that the fix is correct for that release. I would resolve any conflicts with the old version's behavior in mind, run the supported tests and security checks on each branch, and validate dependency and configuration differences. Then I would create or update the release candidates according to the deployment process, record which commit and release tag contain the fix, and verify the deployed artifact's digest and source revision. I would not blindly merge all of main into older support branches unless that is the team's explicitly chosen release strategy, because doing so can introduce unrelated and unvalidated changes.

### Q123. A token was pushed to a private repository. Is rewriting history enough?

No. I would assume the token is exposed and revoke or rotate it immediately, before spending time on history cleanup. A private repository limits access, but it does not guarantee that nobody copied the commit or that the token was never exposed through a clone, fork, CI job, build log, cache, artifact, pull-request view, or developer workstation. Rewriting Git history cannot recall an already copied secret and cannot make an active credential safe again.

After rotation, I would assess the token's privileges and audit available access or use logs to determine whether it was used unexpectedly. I would update the appropriate secret manager or CI configuration and make sure the replacement is not committed to source control. Then, following the incident-response and repository policies, I would decide whether to remove the secret from repository history using an appropriate history-rewriting tool, coordinate the rewrite with collaborators, and arrange for affected clones and references to be cleaned up. Hosting-provider support may be needed for cached views or retained references. I would also add or improve secret scanning and prevention controls so the same path is less likely to recur. The order is essential: revoke first, investigate and contain, then clean up the history. History rewriting is hygiene and exposure reduction—not revocation.

### Q124. The team wants linear history, but developers keep force-pushing over each other's work.

I would address the collaboration model before changing Git commands. Linear history is a property of the history we choose to publish; it does not require developers to rewrite a branch that other people are actively using. I would keep the protected `main` and release branches free of direct force pushes, require review and automated checks, and agree on a merge policy such as squash merging or rebase merging through pull requests. For feature work, I would prefer short-lived branches with clear ownership. If several developers need to collaborate on the same branch, I would generally avoid rebasing its published commits because other branches may already contain the old commit IDs; merging new work into that shared branch is often less disruptive.

When rewriting my own published feature branch is permitted, I would fetch and inspect the remote state first, coordinate with anyone using the branch, and use `--force-with-lease` rather than a blind `--force`. For a high-risk update, an explicit expected SHA makes the condition clear:

```bash
git fetch origin
git push --force-with-lease=refs/heads/my-feature:<expected-remote-sha> origin HEAD:my-feature
```

The expected SHA must be the remote branch tip I intentionally inspected. If the branch has advanced since that point, the push should fail and I should review the new work rather than overwrite it. I would document when rebasing is allowed, protect important branches with repository rules, and use CI plus required reviews to maintain quality. The principle is to achieve a readable history without treating teammates' published commits as disposable.

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

### Quick command-state reference

| Command | Primary effect | Shared-history guidance |
|---|---|---|
| `git fetch` | Downloads objects and updates remote-tracking refs | Safe for the current branch; does not integrate changes |
| `git merge` | Integrates histories into the current branch | Usually preserves existing commits |
| `git rebase` | Replays commits onto a new base | Rewrites replayed commits; coordinate if published |
| `git reset --soft` | Moves current branch/HEAD; leaves index and working tree | Prefer for local, unpublished cleanup |
| `git reset --mixed` | Moves branch/HEAD and resets index; preserves working files | Prefer for local, unpublished cleanup |
| `git reset --hard` | Moves branch/HEAD and resets index and tracked working tree | Can discard local work; never use casually |
| `git revert` | Adds a commit that reverses another commit | Usually safer for shared history |
| `git restore` | Restores file content in working tree and/or index | Does not move the branch pointer |
| `git cherry-pick` | Applies selected commit changes as new commit(s) | Useful for selective backports; creates new IDs |
| `git push --force-with-lease` | Requests a non-fast-forward remote ref update with an expected-value check | Only for coordinated history rewrites |

This table is a memory aid, not a substitute for the detailed explanations above. Before a potentially destructive command, inspect `git status`, verify the target commit, and preserve work you may need.

### Official references

- [Git documentation](https://git-scm.com/doc) — official manuals and reference material.
- [Git reference manual](https://git-scm.com/docs) — command options and version-sensitive behavior.
- [Pro Git book](https://git-scm.com/book/en/v2) — repository model, branching, internals, and workflows.
- [GitHub documentation](https://docs.github.com/) — GitHub-specific pull requests, protected branches, authentication, and repository policies.
- [git-filter-repo project](https://github.com/newren/git-filter-repo) — history rewriting for repository cleanup.

Use Git's own manuals to verify command semantics. Treat pull requests, required reviews, branch protection, and host-side secret scanning as hosting-provider capabilities rather than universal Git features. In an interview, explain not just the command you would run, but whether it changes the working tree, index, local history, or shared remote history—and why that is the right risk for the situation.
