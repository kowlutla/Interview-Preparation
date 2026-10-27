# Git Interview Revision Handbook — Senior Software Engineers (5–10 Years)

**Audience:** Senior software engineers, especially Java backend engineers preparing for interviews at product companies.  
**Purpose:** A focused, reusable revision guide: 55 high-value questions with detailed answers, core commands, trade-offs, and realistic scenarios.

This handbook intentionally consolidates overlapping fundamentals, frequently asked questions, scenarios, and mock rounds. Question numbers run continuously from Q1 to Q55. The goal is not to memorize commands; it is to understand what each command changes, when it is safe, what can go wrong, and how to verify the result.

## Contents

1. [Git fundamentals and object model — Q1–Q5](#1-git-fundamentals-and-object-model)
2. [Branches and merging — Q6–Q10](#2-branches-and-merging)
3. [Rebase and history management — Q11–Q15](#3-rebase-and-history-management)
4. [Remotes and team workflows — Q16–Q21](#4-remotes-and-team-workflows)
5. [Commits, pull requests, and safeguards — Q22–Q26](#5-commits-pull-requests-and-safeguards)
6. [Undoing and selectively applying changes — Q27–Q33](#6-undoing-and-selectively-applying-changes)
7. [History, debugging, and recovery — Q34–Q39](#7-history-debugging-and-recovery)
8. [Tags, releases, and backports — Q40–Q44](#8-tags-releases-and-backports)
9. [Configuration, repository hygiene, and performance — Q45–Q49](#9-configuration-repository-hygiene-and-performance)
10. [Security — Q50–Q52](#10-security)
11. [Senior-level scenarios — Q53–Q55](#11-senior-level-scenarios)
12. [Mock interview rounds](#mock-interview-rounds)
13. [Last-minute revision checklist and command reference](#last-minute-revision-checklist-and-command-reference)
14. [Official references](#official-references)

---

## 1. Git fundamentals and object model

### Q1. What is Git, and why is it a distributed version-control system?

Git is a version-control system that records changes to a project as a history of commits. It is distributed because a normal clone contains the repository's commit history and objects, not merely a working copy connected to one central server. Developers can inspect history, create commits, compare branches, and experiment locally without network access; a remote such as GitHub or GitLab is used to exchange commits and references, rather than being the only place where history exists.

This model makes branching and committing inexpensive and supports parallel development. It also means that a local commit is not automatically shared with the team: creating a commit and pushing it are separate actions. A distributed repository can have multiple remotes or branches that advance independently, so engineers must understand which revision is local, which revision is tracked from a remote, and which revision is deployed.

Git is optimized around content-addressed objects and snapshots of tracked project state. It is not a deployment system, a backup guarantee, or a replacement for code review and CI. A senior engineer should explain both the benefit—local, flexible history—and the operational responsibility: synchronize deliberately, protect shared branches, and verify that the commit being released is the commit that was reviewed and tested.

### Q2. Explain the working tree, index, local repository, and remote repository.

The **working tree** is the set of checked-out files I edit on disk. The **index**, commonly called the staging area, records the content and file modes that will be included in the next commit. The **local repository** stores Git objects and references such as branches and tags. A **remote repository** is another repository, usually hosted centrally, with which I exchange commits and references.

These are separate states. Editing a Java file changes the working tree, but not the index or current commit. **git commit** records the index as a new commit; **git push** transfers commits to a remote.

This separation is central to safe daily work. Before committing, inspect both staged and unstaged changes rather than assuming that everything on disk will be committed. Before pushing, confirm the branch and remote. Before a destructive operation, use **git status** and preserve any work that is not safely committed.

### Q3. What do add, commit, fetch, pull, and push do?

These commands act at different stages of collaboration. **git add** copies the selected current file content into the index; it does not create a commit or upload anything. **git push** sends local commits and asks the remote to update a reference, subject to ancestry checks and server policy.

A safe routine is to review the actual changes, commit a coherent unit, fetch before integrating recent remote work, run tests, and push to the intended destination. For example:

    git status
    git diff
    git diff --staged
    git add src/main/java/com/example/OrderService.java
    git diff --staged
    git commit -m "Handle duplicate order requests"
    git push origin feature/idempotent-orders

The command sequence is not a substitute for judgment. I should not stage every file blindly, and I should understand whether a pull will merge or rebase. A successful push means the remote accepted the update; it does not prove that CI passed or the change is ready to deploy.

### Q4. Does Git store diffs or snapshots?

At the logical level, Git commits represent snapshots of the tracked project at particular points in history. A commit points to a root tree that records paths, file modes, and references to file content objects. This is why Git is often described as snapshot-based rather than as a database that permanently stores only a sequential list of patches.

Physically, Git uses storage optimizations such as packfiles and delta compression to reduce object-database size. Those optimizations do not change the logical model: each commit still describes a complete project state through its tree. Git also generally infers renames when showing diffs or performing merges, based on similarity between deleted and added paths; a rename is not usually a special standalone object in the committed tree.

The distinction matters in interviews and during recovery. A commit can have the same tree as another commit but a different identity because its parent or metadata differs. The old object can remain reachable through history, so shrinking repository history may require a deliberate history rewrite rather than a simple file deletion.

### Q5. Explain Git's blob, tree, commit, and tag objects.

A **blob** stores file content, without the file's path. A **tree** describes a directory level by mapping names and modes to blobs or other trees. A lightweight tag, by contrast, is simply a reference to an object.

Git identifies objects using a hash of their object representation. Repositories created with different object formats can use different hash algorithms, and not every Git host or tool necessarily supports every format in the same way. The practical lesson is not to depend on manually interpreting object IDs; use Git commands to inspect objects and references.

These object types explain Git's behavior. A branch is not a copy of all project files; it is a reference to a commit. These distinctions help explain branching, merging, rebasing, tags, and why commits can become unreachable after their references are moved or deleted.

---

## 2. Branches and merging

### Q6. How are commits linked, and why can identical source files have different commit IDs?

A commit points to a tree representing the snapshot and normally to one parent commit; a merge commit has two or more parents. Following parent pointers gives the commit history. The branch reference normally moves to a new commit when I create a commit on that branch. This makes history a graph, not just a list of filenames or a single universal timeline.

A commit ID hashes the commit object, not only the source files. Two commits can point to identical trees yet have different IDs because their parent commits, author or committer metadata, timestamps, messages, signatures, or other headers differ. Rebasing typically changes parentage, so it creates new commit objects and changes the IDs of the replayed commits even if the source-level changes remain the same.

For investigation, separate three questions: are the checked-out files the same, are the root trees the same, and are the commit histories the same? These are not interchangeable. Inspect the graph, tree, and diff to establish what actually differs.

### Q7. What are a branch, HEAD, and detached HEAD?

A branch is a movable reference to a commit. When I make a normal commit while on a branch, Git creates a commit whose parent is the previous tip and advances that branch reference. **HEAD** normally points symbolically to the current branch, which then points to a commit. This is why switching branches changes which commit the working tree is based on without copying an entire repository for each branch.

In **detached HEAD** state, HEAD points directly to a commit rather than symbolically to a branch. This commonly happens when checking out a tag or a specific commit. If I switch away without creating a reference, the commits may eventually become difficult to find and can be pruned after they become unreachable.

Check with **git status** or **git branch --show-current**. To keep work created in detached HEAD, create a branch at the current commit, for example **git switch -c rescue/my-experiment**. Detached HEAD is useful for testing a historical revision; it is not inherently an error. The risk is assuming a branch name will preserve new commits when HEAD is detached.

### Q8. What are merge-base, fast-forward merges, and three-way merges?

The **merge base** is a best common ancestor used by Git to compare the histories being merged. If a feature branch is strictly ahead of the target branch, Git can do a **fast-forward**: it moves the target reference forward to the feature commit because no divergent histories need to be combined. No merge commit is required for that operation.

If both branches have diverged, Git typically performs a **three-way merge** using the merge base and the two branch tips. It combines the changes from both sides and may create a merge commit with both tips as parents. The merge-base concept explains why Git compares each side's changes since a common ancestor, rather than treating the merge as a simple comparison of two final files.

A fast-forward is simple and keeps history linear, but it does not preserve a separate merge boundary. A merge commit records that two lines of development were integrated. Inspect the graph with **git log --oneline --graph --decorate --all** when the relationship between branches is unclear.

### Q9. How do you resolve merge conflicts safely, and why can a clean merge still be wrong?

A conflict means Git could not safely combine particular changes automatically. I first inspect **git status** to identify conflicted paths, then read the surrounding code and understand both changes before editing the file. If I realize the operation was started on the wrong branch or cannot resolve it safely, I can abort the operation rather than leave a partial result.

A merge can also finish without conflicts while producing a broken application. Git reasons about file content and merge rules, not business invariants or service contracts. For example, one branch may change a REST response or Java method signature while another modifies its caller in a different file. The edits can combine cleanly but still fail at runtime or violate an API contract.

After an important merge, validate behavior, not merely Git status. Review the combined diff, run the affected unit and integration tests, and check related migrations, configuration, event schemas, and compatibility requirements. A conflict-free merge proves that Git found a textual combination; it does not prove that the application is correct.

### Q10. What are merge commits, squash merges, and octopus merges?

A normal merge commit records the integration of divergent histories by referencing both parent commits. This retains the branch's commit history and makes the integration point visible. Squash merging is not the same as using Git's ordinary merge to create a two-parent merge commit.

An **octopus merge** combines multiple heads in one merge operation and is intended mainly for cases where the changes can be resolved without complex conflicts. It is not the normal choice for integrating a large set of independently changing feature branches.

Choose the method based on the kind of history the team needs. Merge commits show when integration happened and preserve the branch graph. Do not confuse a clean or linear-looking history with code quality; the important result is traceable, validated changes with a history the team can maintain.

---

## 3. Rebase and history management

### Q11. What does rebase do, and why does it change commit IDs?

Rebase takes commits from a source branch that are not already part of the chosen upstream history and replays their changes on top of a new base. For example, if main has advanced while my feature branch has two private commits, I can fetch and then run:

    git fetch origin
    git switch feature/my-change
    git rebase origin/main

Git reconstructs the feature commits on top of origin/main. The resulting commits normally have new parents, so their commit objects and IDs change even when their patches are equivalent to the old commits. I resolve the relevant files, stage them, then run **git rebase --continue**; I can use **git rebase --abort** to return to the pre-rebase state if I need to abandon the operation.

Rebase is useful for organizing private work and keeping a branch up to date, but it rewrites history. It should not be treated as a harmless equivalent of fetching. After rebasing, inspect the graph and diff and rerun tests, because conflict resolution can change behavior.

### Q12. Merge versus rebase: how do you choose?

Merge combines histories and can create a merge commit that records the integration of two lines of development. It preserves existing commit identities and is usually appropriate for shared branches where others may already have based work on the published commits. It is often useful for private or owner-controlled feature branches before review or integration.

Neither is always better. A team that values explicit integration history may prefer merge commits. The important constraint is coordination: rebasing commits that other people already consume forces them to reconcile the changed history and can create duplicate-looking commits or confusion.

I decide by asking: is the branch private or shared, does the repository policy require a specific history shape, and can I safely rewrite the commits? I fetch first, inspect the graph, perform the integration, resolve conflicts with behavior in mind, run tests, and review the final diff. Linear history is a useful convention, not a reason to rewrite teammates' work without agreement.

### Q13. How does interactive rebase help clean up commits?

Interactive rebase lets me edit a series of commits before they become part of protected or widely shared history. It can reorder commits, combine them with **squash** or **fixup**, reword messages, edit a commit, or drop an unwanted commit. A **fixup** commit is useful for a small correction to an earlier commit; autosquash can place fixup commits next to the target commit for cleanup.

The goal is to make the history reviewable: each final commit should represent a coherent change and have a useful message. I would not squash everything automatically if separate commits document important steps or provide useful revert boundaries. Nor would I rewrite a shared branch without coordinating with its consumers.

After the rebase, inspect **git log --oneline --graph** and compare the branch with its base using **git diff origin/main...HEAD** when that three-dot comparison is what I intend to review. Run tests because reordered or combined commits can expose missing dependencies or hide a bad intermediate assumption. Interactive rebase is a history-editing tool, not a substitute for understanding the patch being changed.

### Q14. What is force-with-lease, and why should shared history rewrites be handled carefully?

A normal push is rejected when it would move the remote branch in a way that is not a fast-forward. That check prevents a developer from accidentally discarding commits that exist on the remote but are not reachable from the local branch. **git push --force** bypasses this protection and can overwrite someone else's published work.

**git push --force-with-lease** adds a check that the remote reference still has the value Git expects, commonly the value observed through the local remote-tracking reference. If the remote branch has advanced unexpectedly, the push should be rejected. If the expected remote state is stale or misunderstood, a destructive rewrite can still happen.

Before rewriting history, fetch, inspect remote and local commits, check whether anyone else depends on the branch, and agree on the operation. Use this approach for an intentional rewrite of a branch that the team permits you to rewrite. For protected main or release branches, follow repository policy and normally use a corrective commit or revert rather than force-pushing.

### Q15. During a rebase, when should you continue, skip, abort, or preserve merge structure?

When a rebase stops, Git is asking me to resolve or decide what to do with the commit currently being replayed. I inspect **git status** and the conflict details, edit the files to preserve intended behavior, stage the resolved paths, and run **git rebase --continue**. **git rebase --abort** abandons the rebase and restores the branch to its state before that rebase began.

The choice must come from the intent of the changes, not simply from which command makes the prompt go away. A commit may fail to apply because its change is obsolete, because a later commit has already included it, or because the branch has real behavior conflicts. Those cases need different decisions.

Ordinary rebase does not preserve the original merge topology by default. Where preserving merge structure is important, **git rebase --rebase-merges** can attempt to recreate merge commits while replaying the branch. I would still inspect the resulting history and test it. Before complex history operations, ensure valuable work is committed or otherwise backed up and know how to abort.

---

## 4. Remotes and team workflows

### Q16. What is the difference between fetch, pull, and push?

**Fetch** downloads commits and related objects from a remote and updates the appropriate remote-tracking references, such as origin/main. It does not by itself change the current branch or working tree. **Push** sends local commits to a remote and requests that the destination reference move, subject to fast-forward rules and server policy.

Fetch is often the best first step when I want to inspect changes before integrating them:

    git fetch origin
    git log --oneline --graph --decorate HEAD..origin/main

That lets me understand what is incoming before choosing merge or rebase. Pull is convenient when the intended integration policy is known, but a developer should understand what it will do in that repository. A rejected push should trigger inspection and reconciliation, not an automatic force push.

These commands are distinct parts of collaboration: fetch observes remote changes locally, merge or rebase integrates history, and push publishes commits. Keeping those steps conceptually separate helps diagnose divergence and avoids accidentally changing the current branch while trying only to see what teammates have committed.

### Q17. What do origin, upstream, and remote-tracking branches mean?

A remote is a named connection to another repository. **origin** is simply the conventional name Git gives the remote from which a repository was cloned; it is not a reserved server or a special GitHub feature. In a fork-based workflow, **origin** often refers to my fork and **upstream** refers to the original project, but teams may choose different names. The actual URLs can be inspected with **git remote -v**.

A remote-tracking branch such as **origin/main** is a local reference recording the last fetched state of the remote's main branch. It is not a live remote connection and normally does not move until I fetch. An upstream tracking configuration links a local branch with the remote branch it normally pulls from or pushes to; **git branch -vv** helps inspect that setup.

This distinction is important during troubleshooting. If my local main is behind origin/main, I need to integrate the new commits. If origin/main itself is stale, I should fetch before assuming it accurately reflects the server. Never infer the destination from a familiar branch name alone; verify the remote and branch before publishing changes.

### Q18. A push is rejected as non-fast-forward. What should you do?

A non-fast-forward rejection usually means the remote branch has commits that are not ancestors of the update I am trying to push. The rejection prevents a normal push from deleting or bypassing that remote history. I start by fetching and inspecting the two histories:

    git fetch origin
    git log --oneline --graph --decorate --left-right HEAD...origin/my-branch

Then I determine why the branch diverged. Perhaps a teammate has pushed commits, my local branch is stale, I am pushing to the wrong destination, or someone rewrote history. If remote work must be retained, I integrate it with merge or rebase according to whether the local commits are private and the team's workflow, resolve conflicts, test, and push normally.

I would not immediately add **--force**. That bypasses the safety check and could discard published work. The core interview answer is to diagnose divergence, preserve other developers' commits, and only rewrite history when that is an explicit, coordinated choice.

### Q19. How do you choose a branching strategy: feature branches, trunk-based development, or Git Flow?

With **feature branching**, work happens in isolated branches and reaches the main branch through review. This gives developers space to work independently, but long-lived branches accumulate divergence and can make integration painful. It can fit products with deliberate release trains or multiple supported release lines, but it creates more branch-management overhead.

I would choose based on deployment cadence, release requirements, team size, test automation, and whether older versions must be maintained. A service deploying many times a day often benefits from trunk-based development and feature flags. A product with scheduled releases and several maintained versions may need release branches even if day-to-day development is trunk-oriented.

No branching model removes the need for integration testing and clear ownership. The senior-level consideration is how the workflow manages risk, integration cost, and rollback—not whether one branching diagram is universally correct.

### Q20. What are shallow clones, partial clones, sparse checkout, and submodules?

These features solve different repository-management problems. A **shallow clone** limits history depth, which can make an initial checkout faster but means older ancestry may not be present until more history is fetched. These options can be combined in supported workflows, but they should not be treated as synonyms.

A **submodule** records a specific commit from another repository at a path in the parent repository. Cloning or switching the parent repository does not necessarily fetch and check out the submodule content unless the submodule workflow is used. Git subtree is a different approach that integrates another project's content into the repository history.

I choose based on what is expensive: history depth, object download, working-tree size, or independent repository ownership. Reduced local data can improve speed, but it can complicate builds or history inspection when commands expect objects that have not been fetched.

### Q21. How do SSH keys and HTTPS credentials work when accessing a remote?

Git uses the remote URL to determine how to communicate with the server. With **SSH**, the client authenticates using an SSH key and the server's configured account or key permissions. The authentication method is separate from repository authorization: successfully authenticating does not guarantee that the account can push to a protected branch or repository.

I would inspect the URL with **git remote -v**, use the company's supported credential method, and avoid putting tokens directly into remote URLs or shell history. Tokens and private keys should be stored in an approved credential manager or secret-management system, rotated when exposed, and scoped to the minimum permissions needed. On a build agent, credentials should be short-lived or tightly scoped where possible and never printed into logs.

For troubleshooting, distinguish network or host-key failures from authentication failure, repository permission denial, and branch protection. Changing SSH to HTTPS will not solve missing authorization. A senior engineer should also consider the blast radius of credentials in developer laptops and CI systems, not just whether one push succeeds.

---

## 5. Commits, pull requests, and safeguards

### Q22. What makes a good commit, and when are Conventional Commits useful?

A good commit represents one coherent, reviewable change. Its message explains the intent and, where necessary, the reason for the change rather than simply repeating the file names. Before committing, I inspect the staged diff and confirm that the index contains only the intended changes.

Conventional Commits adds a common message structure, often using a type such as **feat**, **fix**, **docs**, **refactor**, **test**, or **chore**, optionally with a scope and a body or footer. For example, a message might begin with **fix(order): reject duplicate request IDs**. Teams may use this format to drive changelog generation or semantic-version automation, but the format does not itself guarantee that a commit is correct or backward compatible.

Commit quality is a team productivity practice. Clear, atomic commits help code review and make it easier to understand why a line was introduced. I would follow the repository's conventions rather than enforce a personal style. If I need to revise local commits, interactive rebase or fixup commits can help, but I avoid rewriting published history without agreement.

### Q23. What should a senior engineer look for when reviewing a pull request?

A senior review checks correctness and maintainability, not just formatting. I start with the problem statement and expected behavior, then inspect the overall diff and the most important code paths. I check whether tests cover meaningful success cases, failures, boundaries, and regressions instead of merely increasing line coverage.

I also consider the operational impact: migrations, configuration, feature flags, rollout order, observability, and rollback strategy. A change may compile but still introduce a race condition, breaking schema change, expensive query, or sensitive-data leak. Automated checks provide useful evidence, but they cannot replace reasoning about requirements and integration with neighboring services.

Review comments should explain the risk and offer a practical improvement, while distinguishing a blocker from a preference. If a change needs a design discussion, I raise that early rather than burying it in minor inline comments. The outcome should be an understandable, tested change whose risks are visible—not merely an approval checkbox.

### Q24. Squash merge, merge commit, or rebase merge: what are the trade-offs?

A **merge commit** preserves the source branch's existing commits and records the integration with a merge commit. This is helpful when the team wants to see where a branch entered main or preserve meaningful branch history. A **rebase merge** typically replays commits on top of the target branch to maintain a linear history, changing commit IDs as needed and avoiding a merge commit.

The best policy depends on whether main's history is used for detailed investigation, whether feature commits are already well organized, and how the team wants to revert or backport changes. Squash merges make reverting one pull request conceptually simple, but can obscure a useful sequence of logical commits. Merge commits preserve integration topology, but may add noise if many tiny branch commits are retained.

Whichever method is chosen, the branch must be reviewed and tested. A linear history is not automatically better, and a merge commit is not automatically messy. Consistency matters because the team relies on the history for debugging, releases, audits, and recovery.

### Q25. What do hooks, CI checks, and signed commits provide—and what do they not prove?

Git hooks are programs that run at specific points in local Git workflows, such as before a commit or push. They can enforce formatting, run fast checks, or reject an invalid message. CI then runs repeatable checks in a controlled environment, while protected-branch settings can require reviews and successful checks before accepting changes.

Signed commits or tags use cryptographic signatures to allow a verifier to check that the signed object was signed by a key associated with a particular identity or trust configuration. A valid signature does not prove that the code is correct, safe, or reviewed. Likewise, a green build proves only that the configured checks passed for that build's inputs; test gaps and environment differences can remain.

For important branches, combine enforceable repository rules, required reviews, CI, and least-privilege access. Keep fast local feedback for developer productivity, but do not rely on a hook running only on one developer's machine to protect production. Explain what each control guarantees and its limits.

### Q26. Why does .gitignore not stop Git from tracking a file?

The ignore file tells Git which **untracked** paths to ignore during normal discovery. It does not remove a file that has already been added to the index and committed. If a developer adds a build output or local configuration file to the repository, adding its name to .gitignore does not stop future changes to that tracked file from appearing in status.

To stop tracking the file while leaving it on disk, remove it from the index and commit that repository change. For example:

    git rm --cached application-local.yml
    echo application-local.yml >> .gitignore
    git status
    git commit -m "Stop tracking local configuration"

Before doing this, confirm that the file should not be versioned and that the ignore pattern is appropriately scoped. Shared configuration templates can belong in Git; real secrets and machine-specific values usually should not. The key distinction is ignored versus untracked versus tracked: .gitignore controls discovery, while the index determines what Git is already tracking.

---

## 6. Undoing and selectively applying changes

### Q27. Explain soft, mixed, and hard reset.

Reset moves the current branch reference to a target commit and, depending on the mode, updates the index and working tree. **git reset --soft <commit>** moves the branch but leaves the index and working tree unchanged; it is useful when I want to combine or rewrite recent local commits while retaining the staged snapshot. It can discard tracked uncommitted work.

I choose the mode based on exactly what I want to preserve. Before a hard reset, inspect **git status**, verify the target SHA and branch, and save valuable work in a commit, patch, or another safe form. A hard reset is not a general-purpose command to remove every untracked file; untracked paths can remain, although they may obstruct a tracked path being checked out.

Reset is typically most appropriate for local history that I am intentionally changing. For a commit already shared with teammates, a revert is usually safer because it records a corrective commit rather than rewriting the shared branch's ancestry.

### Q28. Compare reset, revert, restore, and checkout.

These commands address different kinds of changes. **reset** moves the current branch and can also change the index and working tree, depending on its mode. Modern Git workflows often use **switch** for branch changes and **restore** for file restoration to make intent clearer.

Examples:

    git restore src/main/java/com/example/OrderService.java
    git restore --staged src/main/java/com/example/OrderService.java
    git revert <commit-sha>
    git switch main

The first command restores the working-tree path from the index; the second unstages the path while leaving working-tree content in place. Neither is a general substitute for reverting a shared commit. Before choosing a command, ask whether I am trying to change branch history, add an inverse commit, unstage a file, or discard an edit. Being precise about the target state prevents accidental data loss.

### Q29. How do you undo a merge commit that has already been pushed?

For a shared branch, I normally use **git revert** rather than reset-and-force-push. Reverting an ordinary merge commit needs special care because Git must know which parent should be treated as the mainline. For a merge where the first parent is the target branch before integration, the command is commonly:

    git revert -m 1 <merge-commit-sha>

The **-m 1** option tells Git to treat parent 1 as the mainline and reverse the changes introduced relative to that parent. Verify the parent order with **git show --no-patch --pretty=raw <merge-commit-sha>** and inspect the merge before running the command; choosing the wrong mainline can reverse the wrong set of changes.

After the revert, review the diff and run tests. A revert does not erase the original merge from history. For a production incident, I would also coordinate the rollback with deployment and data changes; undoing source code alone cannot reverse irreversible external side effects or database migrations automatically.

### Q30. How does git stash work, and what are its risks?

Stash temporarily stores local changes so I can switch context without making a normal commit. **git stash** commonly saves tracked working-tree and index changes, while untracked files need **-u** and ignored files need additional care. The staged and working-tree state can be preserved so that changes can be reapplied later, but conflicts may occur when applying the stash to a changed base.

Common commands include:

    git stash push -m "WIP: order validation"
    git stash list
    git stash show -p stash@{0}
    git stash apply stash@{0}
    git stash drop stash@{0}

**apply** keeps the stash entry, whereas **pop** attempts to apply it and then removes the entry when successful. For changes I may need again, apply first and verify the result before dropping the stash. Always inspect the working tree after applying a stash, especially when it contains staged changes or touches files that have changed on the new branch.

### Q31. What is cherry-pick, and when is it preferable to merge or rebase?

Cherry-pick applies the change introduced by one or more existing commits onto the current branch and records the result as a new commit. It is particularly useful for backporting an isolated bug or security fix to a supported release branch without integrating every change from the source branch. For example:

    git switch release/2.x
    git cherry-pick <fix-commit-sha>

The new commit usually has a different ID because it is a new commit object with a different parent, even when the patch is equivalent. A conflict may occur if the target branch has different code or lacks a prerequisite change. I must validate the result in the target branch instead of assuming a successful cherry-pick proves the fix is complete.

A **merge** integrates the history of a branch, while **rebase** replays a series of commits onto a new base. **Cherry-pick** selectively transfers specific changes. Keep a traceable link to the original fix or pull request, and record which release branches contain the patch.

### Q32. I committed to the wrong branch. How do you recover safely?

First I determine whether the commit is local only or already pushed, and whether the intended branch contains other work that must be preserved. If the commit is local and I am on the wrong branch, I can create or switch to the correct branch at the current commit so the work has a named reference, then restore the original branch to its previous state using a safe, verified target. For example, I might create the destination branch first:

    git branch feature/correct-target
    git log --oneline --decorate -5

Then I check the original branch's prior tip and use an appropriate reset only after confirming which commit belongs there. If the commit has already been published or others have based work on it, I avoid rewriting shared history casually. I can instead apply the desired change to the correct branch with cherry-pick and coordinate how to handle the mistaken branch.

The important practice is to protect the commit before moving references. Branches are pointers, so creating a new branch at the current commit is a low-risk way to preserve access while investigating. I would inspect the graph and working tree rather than blindly copy commands from memory. The exact recovery depends on whether the target commit is private, shared, or already integrated elsewhere.

### Q33. What should you do if an operation is stuck or you accidentally stage the wrong files?

Start with **git status**. It tells you whether a merge, rebase, or cherry-pick is in progress and which files are staged, unstaged, or conflicted. If a merge or rebase is no longer appropriate, use the corresponding abort command while the operation is still in progress.

    git status
    git diff
    git diff --staged
    git restore --staged path/to/file
    git merge --abort
    git rebase --abort

Do not run abort commands mechanically: first confirm which operation is active and whether you want to abandon it. If conflict resolution was partly completed, make sure important manual work is captured before aborting. The recovery principle is to identify the repository state, decide which data must be preserved, and then use the narrowest command that achieves the intended result.

---

## 7. History, debugging, and recovery

### Q34. Which Git commands are most useful for reviewing history and comparing changes?

For quick history, **git log --oneline --graph --decorate --all** shows branch structure and references. **git show <commit>** presents a commit and its patch, while **git diff** compares working-tree or index states and **git diff A B** compares two specified revisions. The three-dot behavior is not identical to a direct A-versus-B comparison, so choose the form based on the question you are trying to answer.

**git status** is the starting point for understanding local changes. **git blame <path>** attributes lines to commits, but it does not prove why a line changed or who should be blamed for a defect; inspect the commit and surrounding history. **git range-diff** can help compare two versions of a series of commits, such as before and after a rebase.

The useful habit is to inspect before altering. Establish the branch, base, and relevant revisions; read the diff; then decide whether to merge, revert, cherry-pick, or rewrite history. Git history is evidence for investigation, not a substitute for understanding runtime behavior.

### Q35. How does git bisect help find the commit that introduced a regression?

**git bisect** performs a binary-search-like investigation through a range of commits. I identify one revision known to be good and another known to be bad, then let Git select an intermediate commit to test. If the regression is reproducible and each test result can be classified reliably, Git repeatedly narrows the range until it identifies a likely first bad commit.

    git bisect start
    git bisect bad <known-bad-commit>
    git bisect good <known-good-commit>
    # Build and run the regression test for each selected revision.
    git bisect good
    # or: git bisect bad
    git bisect reset

If a revision cannot be built or tested, I can mark it skipped rather than misclassify it. I should ensure the good and bad revisions correspond to comparable conditions, including relevant build configuration and dependencies. A flaky test makes bisect results less trustworthy, so I try to create a deterministic regression test first.

Bisect narrows the search efficiently, but a candidate commit still needs verification. I inspect its patch, test the parent and candidate where possible, determine the actual root cause, and choose a fix or revert. The right senior-level answer includes both the command and the quality of the evidence used to label revisions good or bad.

### Q36. What are reflogs, and how can you recover a deleted branch or a lost commit?

A reflog records recent updates to local references and HEAD, such as switching branches, committing, resetting, or rebasing. It can help recover a commit after a branch was deleted or a branch pointer was moved. Reflogs are local and are not normally pushed to the remote, and they are subject to expiration. They are a recovery aid, not a permanent backup.

I would inspect:

    git reflog --all --date=iso
    git log --all --oneline --decorate --graph

If I find the desired commit SHA, I create a new reference immediately, for example **git branch rescue/recovered-work <sha>**, so the commit is reachable through a named branch. If reflogs do not expose it, advanced recovery may involve examining unreachable objects with **git fsck --no-reflogs --unreachable**, provided the objects still exist and have not been pruned.

A commit may be recoverable even after its branch name disappears, because deleting a branch removes a reference, not necessarily the commit object at once. But uncommitted work that was never recorded by Git may not be present in the repository at all. To avoid relying on recovery, preserve work before destructive operations and push valuable branches to a remote when appropriate.

### Q37. How would you investigate and roll back a production regression introduced by a commit or merge?

I start by identifying the deployed artifact and its source commit, then compare the last known-good release with the faulty release. I confirm the symptoms and the scope of impact, inspect the relevant commits and configuration changes, and use logs, metrics, traces, and tests to determine whether the likely cause is in code, a migration, configuration, or an external dependency. If the culprit is unclear across many commits and the behavior can be reproduced, I can use git bisect as described in Q35.

For a bad change already merged to a shared branch, a revert is often safer than rewriting history. If it was an ordinary commit, revert that commit after checking dependencies. Run targeted regression tests and the relevant integration suite, then follow deployment policy for rollout and monitoring.

A Git revert only changes the repository's content going forward. It does not automatically reverse a database migration, external event, or data mutation. If the release changed persistent state, rollback may require a separate backward-compatible recovery plan. Senior incident handling joins source-control history with artifact provenance, application evidence, and a controlled deployment response.

### Q38. How do you compare a release tag with current code and identify what changed?

First verify what the release tag points to, because a tag name alone does not prove which code was deployed. Use **git show <tag>** to inspect it and compare the tag with the target branch or commit. If reviewing a feature branch relative to a base, **git diff <base>...<branch>** may be more appropriate because it compares from the merge base to the branch tip.

These comparisons answer different questions. A commit range describes commits present on one side but not the other; a diff describes the net file change. The artifact may have been built from a commit other than the one someone assumes from the branch's current tip.

A useful release comparison is reproducible and precise: identify both endpoints by immutable commit IDs, explain the command's comparison semantics, and confirm the result against what was actually built and deployed.

### Q39. What happens when you reset the wrong commit or run git reset --hard accidentally?

I first stop making additional changes and inspect **git status**, **git reflog**, and the recent commit graph. A reset may have moved the branch reference, but commits that are no longer reachable can often still be found through the reflog for a period of time. Do not assume the most recent reflog entry is automatically the desired target; correlate the entry with the branch and operation that happened.

    git reflog --date=iso
    git branch rescue/before-recovery <recovered-commit-sha>

Reset recovery has limits. A hard reset can discard tracked, uncommitted working-tree changes, and those changes may not exist as Git objects because they were never committed or otherwise captured. Also, reset is not a universal deletion command for untracked files, though untracked files can be overwritten or obstruct checkout operations.

The practical prevention is to verify the branch and target SHA, use **git status** first, and preserve important work before destructive operations. Git can often recover moved references; it cannot promise to recover every overwritten file.

---

## 8. Tags, releases, and backports

### Q40. How do branches and tags differ, and what is the difference between lightweight and annotated tags?

A branch is a movable reference that typically advances as new commits are created. A tag is intended to name a particular point or object, commonly a release commit, and is not normally moved as development continues. An **annotated tag** is a separate object with a tagger, date, message, and optionally a signature; it is usually preferable for published releases because it can carry release metadata and be verified.

For example:

    git tag -a v2.4.0 <commit-sha> -m "Release 2.4.0"
    git show v2.4.0
    git push origin v2.4.0

A tag created locally is not automatically available on the remote, so a release process must publish it intentionally. Treat published release tags as immutable identifiers whenever possible. Build systems should record both the tag and the resolved commit SHA so later investigation can identify the exact source revision.

### Q41. How does Semantic Versioning relate to Git releases?

Semantic Versioning labels a released version as MAJOR.MINOR.PATCH. In the common model, MAJOR indicates incompatible public API changes, MINOR adds backward-compatible functionality, and PATCH contains backward-compatible fixes. This is a versioning contract, not a feature built into Git. Git stores tags, commits, and branches; tools and team policy determine how version numbers are chosen and released.

Teams often use annotated tags such as v2.4.1 to identify a release commit. Automated release tooling may derive version bumps and changelogs from commit conventions or pull-request metadata, but it should not assume that a commit type alone proves API compatibility. A database migration or runtime behavior change can be breaking even when an automated classifier does not detect it.

For a reliable release, connect the version label to an immutable source commit, a reproducible build or artifact identifier, test results, and deployment metadata. If multiple release lines are maintained, the same fix may appear as different commits in different lines while the release version numbers differ. The version label describes compatibility expectations; the tag and commit identify the source revision.

### Q42. A release tag points to the wrong commit and a deployment has already consumed it. What do you do?

I would first determine exactly what consumed the tag: did a pipeline resolve it to a commit SHA at build time, or does a later deployment job look up the tag again? I would inspect the tag object or reference, the build log, the artifact digest, and deployment metadata to establish which source revision actually produced the running artifact. A tag name by itself is not sufficient evidence of the deployed code.

Then I would coordinate a controlled correction. Depending on repository policy and downstream consumers, the safest option may be to publish a corrected release tag or a new patch version rather than move an already published tag. I would not move a release tag silently because consumers may have recorded the old commit.

Finally, rebuild and test from the intended commit, record the artifact digest and source SHA, and deploy through the normal approval path. This incident highlights why release systems should resolve tags to commits at build time, retain provenance, and treat published version identifiers as immutable wherever possible.

### Q43. How do you backport a critical fix to multiple supported release branches?

I start by identifying the exact supported branches and versions, the root cause, and whether the fix depends on code or APIs introduced only on the main development line. If the fix is isolated and compatible, cherry-pick is often a practical way to apply it to each branch. I would not merge all of main into an old release branch merely to get one fix unless that is the team's intended release policy.

A successful cherry-pick means Git applied a patch; it does not prove the fix works on the older version. I resolve conflicts in the context of that version, run tests on each target branch, inspect dependencies and configuration, and ensure the fix is included in each release candidate. Record the original fix and the commit or pull request that backported it, so reviewers can trace the relationship.

After release, verify which tags and artifact digests contain the change. This matters for security fixes because an organization needs evidence that every supported line actually received the remediation, not just that a similar commit exists somewhere in the repository.

### Q44. Why should build artifacts be connected to immutable source revisions?

A branch such as main is a movable reference. At one moment it may point to commit A, then after another push it points to commit B. Therefore, reliable delivery records the resolved commit SHA and the artifact's digest, alongside build inputs and test results.

That provenance helps answer questions such as which commit introduced a production bug, whether the code running in two environments is identical, and whether a security fix reached a specific release. It also prevents confusion when a release tag was moved or a rebuild produced a different artifact from the same source due to timestamps, dependencies, or build environment changes.

At senior level, Git history is only one part of the deployment chain. CI should build a known revision, produce traceable artifacts, and retain metadata that links the artifact to its source and tests. Deployments should identify the artifact actually promoted—not simply rebuild an ambiguous branch name at deployment time.

---

## 9. Configuration, repository hygiene, and performance

### Q45. What Git configuration and line-ending settings are important in a multi-platform team?

Git configuration can be set at system, global user, or repository-local levels, with more specific settings typically overriding broader settings. Use **git config --show-origin --get-regexp** to see effective settings and where they come from. Team policy should document settings that affect consistency rather than assume every developer has the same defaults.

Line endings can cause noisy diffs when Windows and Unix-like systems use different conventions. The **core.autocrlf** setting may help some workflows, but for cross-platform repositories a deliberate **.gitattributes** policy is often more reliable because it describes how paths should be normalized. Use **git diff --check** to catch whitespace errors and inspect whether a large diff is really a line-ending conversion.

Avoid changing global settings as a first response to one repository's problem. Diagnose the effective configuration, repository attributes, and actual file content, then fix the policy at the appropriate scope. This prevents developers from masking a repository-wide consistency issue with personal machine settings.

### Q46. Why might a repository become unexpectedly large or slow, and how would you investigate?

Repository performance issues can come from large historical blobs, very many files, extensive history, expensive working-tree operations, generated files, poor ignore rules, or tooling that repeatedly scans the whole tree. Start by understanding the symptom: is clone time slow, checkout expensive, status slow, or is the object database large? A large object database calls for investigating history and blobs; slow status may relate to filesystem behavior, untracked-file volume, or repository layout.

Useful checks include **git count-objects -vH**, **git status**, and history inspection for large files. Packfiles consolidate and often delta-compress objects, while Git maintenance and garbage collection optimize object storage. If a large artifact was committed years ago, deleting it from the newest tree does not erase its historical versions.

Avoid blindly running cleanup or history rewriting before measuring. For a monorepo, sparse checkout or partial clone may reduce how much content a developer needs locally. For large binary assets that change frequently, Git LFS may be a better storage model. The right fix depends on which part of the workflow is slow and whether the data must remain in history.

### Q47. When should a team use Git LFS, sparse checkout, partial clone, or a monorepo-specific strategy?

These approaches address different costs. **Git LFS** stores pointer files in ordinary Git history while large file contents are stored through the LFS service. A **monorepo** is an architectural repository choice, not a performance feature by itself.

I would measure the bottleneck first. If developers need only one application within a large repository, sparse checkout may reduce working-tree burden. LFS is appropriate for large binary content, not a generic fix for every repository-size problem.

Choose the least complex strategy that solves the measured problem and verify that builds, releases, and offline or disaster-recovery workflows still work.

### Q48. Why doesn't deleting a large file or running garbage collection necessarily reduce repository size?

When a file is committed, its content becomes an object referenced by a tree, which is referenced by a commit. Deleting the file in a later commit only changes the latest snapshot. Older commits can still reference the original blob, so the object remains reachable and must be retained to preserve history. This is why the working directory can be small while the repository's object database remains large.

Garbage collection can pack objects, delta-compress them, and eventually prune objects that are unreachable and old enough under the repository's retention rules. It will not delete an old blob that is still reachable from a branch, tag, or another reference. If a large file was committed in historical commits and must be removed from history, the team may need to rewrite history with a tool such as git-filter-repo.

History rewriting changes commit IDs and can disrupt every clone, fork, open branch, and build reference that depends on the old history. Coordinate the rewrite, update all affected references and clones, and consider whether retaining history is more important than shrinking the repository. Measure the gain before undertaking a disruptive cleanup.

### Q49. How do you choose between submodules, subtree, and a monorepo?

A monorepo keeps multiple projects or components in one Git repository. It simplifies atomic cross-project changes and can make shared tooling consistent, but can require ownership boundaries, build tooling, and practices that keep operations efficient as the repository grows. A **subtree** brings another project's files into the parent repository's history, often with less special checkout workflow, but syncing upstream changes and understanding ownership require discipline.

I would choose based on independent versioning, access control, need for atomic changes, team ownership, tooling, and contributor experience. If a component must be pinned to a specific external revision and maintained independently, a submodule may fit. Subtree is useful when importing and synchronizing external code while keeping normal files inside the main repository.

No layout automatically solves dependency management or release coordination. Document how versions are updated, how CI checks changes across component boundaries, and who owns the dependency or directory.

---

## 10. Security

### Q50. A secret or token was committed and pushed. What should you do first?

Assume the credential is exposed, even if the repository is private. **Revoke or rotate the secret first**, then assess its permissions and available usage logs. Leaving an exposed active token in place while rewriting history leaves the most important risk unresolved.

Next, remove the secret from the active configuration and store its replacement in an approved secret manager or CI secret store. Investigate whether the credential was used unexpectedly, determine its scope and blast radius, and follow the incident-response process. Add secret scanning or prevention controls and review why the token entered source control.

If the exposure affected a production credential, involve the security or incident-response team. Make sure the replacement is not committed in another location and that dependent services have successfully adopted it. The order matters: revoke, investigate, contain, clean up history if necessary, then improve prevention. Do not promise that rewriting Git history can remove every copy or cache of the secret.

### Q51. When is git-filter-repo appropriate, and what are the risks of rewriting repository history?

A tool such as **git-filter-repo** is appropriate when the team needs to remove sensitive content or large files from many historical commits, or otherwise intentionally transform repository history. It is more appropriate for repository-wide cleanup than trying to manually edit each commit. However, it rewrites the commits affected by the change, which normally produces new commit IDs and can invalidate references to the old history.

Before a rewrite, coordinate with repository owners and collaborators, identify every relevant branch and tag, preserve a recovery copy according to policy, and plan how forks, mirrors, CI references, and developer clones will be handled. After rewriting, the cleaned history must be published under an agreed plan; collaborators may need to reclone or carefully realign their branches rather than merging old history back in and reintroducing the removed content. Hosting providers may retain caches or references that require separate handling.

For secret exposure, rotate the credential before performing this work. For large-file cleanup, measure whether the storage benefit justifies the disruption. History rewriting is a repository migration, not an ordinary cleanup command; the technical operation is only one part of the communication and recovery plan.

### Q52. How do least privilege, branch protection, and signatures improve repository security?

Least privilege means users, automation, and tokens receive only the permissions needed for their responsibilities. A developer who needs to open pull requests may not need permission to bypass branch protections; a CI job that publishes an artifact should not automatically receive broad administrative access. Use scoped credentials, rotate them according to policy, and prevent secrets from appearing in logs or source code.

Branch protection and repository rules can require reviews, status checks, signed commits, linear history, or prevent force pushes and direct pushes to protected branches, depending on the hosting platform. These controls make policy enforceable rather than dependent on every developer remembering it. A signature does not prove that code is correct or that the signer thoroughly reviewed it.

Combine controls rather than relying on a single mechanism: least-privilege access, mandatory review, CI checks, secret scanning, trusted release processes, and artifact provenance. Review bypass permissions and service-account credentials as carefully as human access. State clearly what each control verifies and what risks remain.

---

## 11. Senior-level scenarios

### Q53. Your feature branch has diverged from main, and teammates have already pulled your commits. How would you update it?

I would not immediately rebase or force-push. First fetch and inspect the graph so I know which commits are unique to each side:

    git fetch origin
    git log --oneline --graph --decorate --left-right origin/main...feature/my-feature

Because teammates have already pulled my feature commits, those commit IDs may be part of their branches. Rewriting the feature history can force them to repair their local history or create duplicate-looking commits. Then I can push normally.

If the team prefers rebasing and the branch owner coordinates with everyone consuming it, a history rewrite may still be possible. In that case I would agree on the timing, fetch immediately before the push, verify the remote tip, and use force-with-lease under repository policy. I would communicate the new base and commit IDs so teammates know what changed.

The senior decision is not merely merge versus rebase. It is to protect work that others depend on, minimize unnecessary history disruption, and validate the integrated code.

### Q54. A merge completed cleanly, but integration tests fail after deployment. How do you diagnose it?

I would start with the exact deployed artifact and its source commit, then compare it with the previous known-good version. A merge without conflicts only means Git could combine file changes; it says nothing about API compatibility, schema assumptions, transaction boundaries, or runtime behavior. I would reproduce the failure if possible, inspect relevant logs and traces, and compare both the merge commit's parents with the final result to understand what each branch contributed.

I would examine whether the failure involves a changed REST contract, database migration, configuration default, message/event schema, dependency version, concurrency behavior, or a test fixture that no longer reflects production. Then I would add or improve a regression test and determine the smallest safe correction. If the incident requires immediate rollback, I would follow release procedures and account for database or external side effects rather than assume reverting code fully restores the system.

After fixing the issue, I would rerun unit and integration tests and verify the actual artifact deployed. I would also improve the pre-merge or pre-release checks that failed to catch the incompatibility. The lesson is that textual merge success and production correctness are different gates.

### Q55. The team wants to deploy many times a day, keep history readable, and avoid developers overwriting one another. What workflow would you recommend?

I would lean toward short-lived feature branches or trunk-based development, supported by fast CI, required reviews appropriate to risk, and feature flags for work that should merge before it is user-visible. Small changes integrated frequently reduce the amount of divergence a branch can accumulate. If multiple developers share a feature branch, preserve its published commits or coordinate any rewrite explicitly.

Protect main and release branches against unreviewed pushes and unsafe force-pushes. Require appropriate tests and use deployment controls such as canary releases, health checks, and rollback procedures. Keep releases traceable to commit SHAs and artifact digests.

This workflow is not “trunk-based at any cost.” It depends on automated tests, observability, backward-compatible changes, and the ability to disable or roll back a risky feature. The aim is to reduce batch size and integration risk while protecting the shared branch and production system.

---

## Mock interview rounds

These rounds test how the concepts fit together. The model answers are intentionally conversational; explain the reasoning first, then use commands to support the decision. They complement Q1–Q55 rather than adding another numbered question bank.

### Round 1 — Fundamentals and repository state

**Interviewer:** You edited a Java service, staged one file, then made another edit before committing. What exactly will your next commit contain, and how do you check?

**Model answer:** A commit records the index, not everything currently visible in the working tree. I would run `git status`, `git diff`, and `git diff --staged`. This is why I review the staged diff rather than assuming that staging a file means every subsequent edit is staged.

### Round 2 — Divergence, merge, and rebase

**Interviewer:** Your feature branch is behind `main`, but two teammates have based their work on commits already pushed to your feature branch. How do you integrate the latest `main`?

**Model answer:** I would fetch first and inspect the graph and unique commits. Since other developers depend on the published feature commits, I would avoid rewriting them without coordination. If the team explicitly agrees to rewrite the shared branch, I would coordinate the timing, verify the remote tip, and use `--force-with-lease` only under the team's policy.

### Round 3 — Production regression

**Interviewer:** A release has a regression, and the suspected change is somewhere among several merges. How do you investigate and restore service?

**Model answer:** First I would identify the deployed commit or artifact digest and compare it with the last known-good release; I would not assume the current branch tip is what production runs. I would use `git log --graph`, `git show`, and targeted diffs to narrow the change. Then I would test the corrective change and follow the service's rollback or redeployment process; Git history operations alone do not undo database or external side effects.

### Round 4 — Hotfix and maintenance branches

**Interviewer:** A critical fix is ready on `main`, but two older release branches are still supported. How do you backport it?

**Model answer:** I would confirm that the fix is self-contained and compatible with each supported version. If the fix is one logical commit, cherry-picking it onto each maintenance branch is often practical; I would resolve branch-specific conflicts, test each target version, and verify that the resulting changes are equivalent. I would record which branches received the fix and release each through its normal validation process.

### Round 5 — Secret exposure and repository cleanup

**Interviewer:** A developer pushed an API token and then deleted it in a follow-up commit. Is the incident resolved?

**Model answer:** No. The token remains in earlier history and may already have been copied into clones, forks, logs, or artifacts. I would also add prevention such as secret scanning and improve the approved secret-management workflow.

### Round 6 — Release integrity and senior judgment

**Interviewer:** A deployment pipeline consumed a release tag that was later found to point at the wrong commit. What evidence do you need, and should you move the tag?

**Model answer:** I would establish which commit the pipeline resolved when it built the artifact, and inspect build records, commit SHA, artifact digest, and deployment metadata. A tag name alone does not prove which code is running. The long-term safeguard is to resolve tags to immutable commit SHAs at build time and treat published release identifiers as immutable.

---

## Last-minute revision checklist and command reference

Use this checklist to test whether you can explain the concept without reciting a memorized definition.

- [ ] Explain the working tree, index, local repository, and remote; see Q2–Q3.
- [ ] Describe blob, tree, commit, branch, HEAD, and detached HEAD; see Q5–Q7.
- [ ] Explain merge-base, fast-forward, three-way merge, and semantic conflicts; see Q8–Q10.
- [ ] Choose merge or rebase based on whether the branch is shared; see Q11–Q15.
- [ ] Explain fetch, pull, push, remote-tracking refs, non-fast-forward, and force-with-lease; see Q16–Q18.
- [ ] Choose a branching strategy and explain its trade-offs; see Q19.
- [ ] Compare reset modes and distinguish reset, revert, and restore; see Q27–Q29.
- [ ] Explain stash and cherry-pick, including what they preserve or recreate; see Q30–Q33.
- [ ] Use log, diff, reflog, and bisect for debugging and recovery; see Q34–Q39.
- [ ] Connect tags, commits, artifacts, and deployment metadata; see Q40–Q44.
- [ ] Diagnose ignore rules, line endings, large files, and repository performance; see Q26 and Q45–Q49.
- [ ] Explain the correct order for responding to a leaked credential; see Q50–Q52.
- [ ] For every proposed command, state what it changes, what risk it carries, and how you will verify the outcome.

### Quick command-state reference

| Goal | Command or approach | What to remember |
|---|---|---|
| Inspect local state | **git status**, **git diff**, **git diff --staged** | Working tree and index are separate. |
| Inspect history | **git log --oneline --graph --decorate --all** | History is a graph of commits and refs. |
| See incoming remote commits | **git fetch origin**, then inspect **origin/main** | Fetch updates remote-tracking refs; it does not integrate by itself. |
| Integrate history | **git merge** or **git rebase** | Choose based on shared-history and team policy. |
| Undo local branch movement | **git reset** | Select soft/mixed/hard based on what to preserve. |
| Undo a published commit | **git revert** | Adds a new inverse commit instead of rewriting shared history. |
| Unstage a file | **git restore --staged path** | Keeps working-tree edits. |
| Find a likely regression commit | **git bisect** | Needs reliable good/bad test results. |
| Recover a moved/deleted ref | **git reflog** | Local, time-limited recovery aid; not a backup. |
| Rewrite a permitted remote feature branch | **git push --force-with-lease** | Fetch, inspect, coordinate, and verify expected remote state first. |
| Identify a release source | **git show tag**, record commit SHA and artifact digest | Branch and tag names alone are insufficient deployment provenance. |

This is a revision aid, not a substitute for checking the exact options available in the Git version and hosting environment you use. Before a destructive command, inspect **git status**, verify the branch and target commit, and preserve work you may need.

## Official references

- [Git documentation](https://git-scm.com/doc) — official manuals and reference material.
- [Git reference manual](https://git-scm.com/docs) — command options and version-sensitive behavior.
- [Pro Git book](https://git-scm.com/book/en/v2) — repository model, branching, internals, and workflows.
- [GitHub documentation](https://docs.github.com/) — GitHub-specific pull requests, protected branches, authentication, and repository policies.
- [git-filter-repo project](https://github.com/newren/git-filter-repo) — history rewriting for repository cleanup.

Use Git's manuals to confirm command semantics. Treat pull requests, required reviews, branch protection, and host-side secret scanning as hosting-provider capabilities rather than universal Git features. In an interview, explain not just which command you would run, but why it is appropriate, what it changes, what risks it carries, and how you would validate the result.
