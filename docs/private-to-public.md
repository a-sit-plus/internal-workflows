# Private development with public branch publication

Use [publish-public.yml](../.github/workflows/publish-public.yml) to publish one private integration branch to an existing public branch. Example:

```text
a-sit-plus/private-valera: feature PRs -> squash merge -> development -> CI succeeds
                                                                         |
                                                        publication App pushes
                                                                         |
                                                                         v
                                               a-sit-plus/valera: development
```

Valera's integration branch is called `development`; repositories using `develop` must change the trigger and the `source-branch` / `destination-branch` inputs accordingly.

The private repository owns integration. The public branch receives the same commits, with the same SHAs and authors. The App is the pushing identity, not a replacement commit author. Private PRs, review discussions, Actions logs and repository settings are not copied.

**Merging to the private integration branch authorizes eventual public publication of every reachable commit.** This includes commit messages, author emails, workflow files, reverted changes and earlier versions of files. Squash feature PRs before integration if their intermediate commits should remain private. Removing a file in a later commit does not remove it from published history. If some integrated work must remain private permanently, use a separate publication branch or a selective export tool instead.

## 1. Create the initial private copy

This is an independent repository, not a GitHub fork. Public GitHub forks cannot have private visibility. Keep the public repository in place.

Start with a clean temporary directory, authenticated GitHub CLI (`gh`) and Git credentials that can create and populate the private repository. These are maintainer bootstrap credentials, separate from the publication App. Do not copy a dirty developer worktree or local signing files.

```bash
# Create an EMPTY private repository: no generated README, license or .gitignore.
gh repo create a-sit-plus/private-valera --private

# Disable Actions BEFORE importing workflow files to avoid premature builds/deployments.
gh api --method PUT repos/a-sit-plus/private-valera/actions/permissions -F enabled=false

git clone --bare https://github.com/a-sit-plus/valera.git valera-bootstrap.git
cd valera-bootstrap.git
git remote add private https://github.com/a-sit-plus/private-valera.git

# Copy only the initial branches you need, preserving their complete history.
git push private refs/heads/development:refs/heads/development refs/heads/main:refs/heads/main
# Optional: copy existing PUBLIC release tags as well.
git push private --tags

cd ..
gh repo edit a-sit-plus/private-valera --default-branch development
```

For repositories without `main`, omit that refspec. Keep Actions disabled until the runner, secret, branch protection and caller setup below is ready. Repository creation can also be done in the web UI; ensure the resulting repository is private and empty before copying commits.

Git does not copy issues, PRs, releases/release assets, environments, rulesets, secrets, variables, runner access, installed Apps, webhooks or team permissions. Recreate the needed settings explicitly. Preserve licensing/notice files and contributor attribution. Grant private access only to the intended teams and review organization base permissions and private-fork policy.

If the repository uses Git LFS, Git refs alone do not transfer LFS objects: run `git lfs fetch --all` against the public origin and `git lfs push --all private` during bootstrap. **The reusable publication workflow does not upload new LFS objects**; extend it with a deliberate LFS transfer before using it for new LFS content. Submodule repositories also need their own visibility, access and publication policy; their content is not copied by pushing the parent repository.

For normal development:

```bash
git clone git@github.com:a-sit-plus/private-valera.git
cd private-valera
git remote add public https://github.com/a-sit-plus/valera.git
```

Develop and open PRs against the private repository. Keep the public branch bot-managed. Import public contributions into private integration before publishing again.

## 2. Register an organization-owned publication App

In organization **Settings -> Developer settings -> GitHub Apps**, register an App, for example `a-sit-plus-publication` (the App slug must be available).

- Set a descriptive homepage URL, such as the organization URL.
- Disable webhooks: Actions will request tokens directly; no webhook server, OAuth callback service or separately deployed App backend is needed.
- Grant repository **Contents: Read and write** and **Workflows: Read and write**. Metadata read access is implicit. Do not grant Actions administration, repository administration or PR permissions for this direct-push workflow.
- If both repositories are in the App-owning organization, installation can be limited to that account. Cross-organization installations require an App registration that permits those installations.
- Install the App on **selected public destination repositories**, starting with `a-sit-plus/valera`. Do not select every repository by default. The App does not need installation on `private-valera`: the caller's `GITHUB_TOKEN` reads that source.
- Record its **Client ID**, generate a private key, and retain the downloaded PEM securely.

The job creates an installation token restricted to the configured destination and Contents/Workflows write permissions. Tokens expire after one hour and are revoked by the token action at job completion. A fresh token is created every run. The PEM private key does not expire automatically: rotate it periodically, update the stored secret, verify publication, then revoke the old key. Multiple keys allow rotation without downtime.

One App can serve multiple public repositories. However, **anyone holding its private key can mint tokens for any repository accessible to the App**. Restricting one job's token does not restrict the private key. Separate Apps for separate trust boundaries, or keep the key in a central publication service/repository that enforces allowed source/destination mappings. Different private keys of the same App do not provide repository isolation.

## 3. Configure the private repository

Under **Settings -> Secrets and variables -> Actions**, add:

| Kind | Name | Example/value |
| --- | --- | --- |
| Variable | `PUBLIC_SYNC_APP_CLIENT_ID` | App Client ID |
| Secret | `PUBLIC_SYNC_APP_PRIVATE_KEY` | Entire downloaded PEM, including BEGIN/END lines |

For multiple equally trusted source repositories, organization variables/secrets can be shared with **selected private source repositories only**. Check your plan supports organization secrets for private repositories. Never make the key available to the public destinations. Do not place it in Git or print it in logs.

Configure private `development` to require PR review and the existing CI checks. Enable squash merging for ordinary feature PRs and merge commits for public-import PRs; do not require linear history on this branch. Protect changes to callers and publication code through CODEOWNERS/review rules. Restrict who can change Actions secrets and variables. The repository's default `GITHUB_TOKEN` only needs Contents read access for this publication job.

Give the private repository access to an appropriate runner. The default publication labels are:

```yaml
["self-hosted", "Linux", "X64", "docker"]
```

The runner needs Bash, Git, `base64`, outbound HTTPS to GitHub, and support for Node 24 JavaScript actions. No Docker engine is used by this job; `docker` is a runner selection label. Prefer a clean dedicated/disposable publication runner rather than a persistent runner previously used to execute PR code. The workflow does not check out or execute application code, restore build caches, or run the common build setup. A compromised host can still steal the App key.

Allow the reusable workflow and its pinned GitHub-owned actions in the organization's Actions policy. This `internal-workflows` repository is public, so private callers can reuse it if permitted by their Actions policy. If its visibility changes, configure reusable-workflow sharing and verify caller access.

## 4. Call publication after successful CI

First merge/publish the reusable workflow in `internal-workflows`. Resolve the reviewed commit SHA and use it in the caller. Do not use a moving branch reference for this privileged workflow.

In `private-valera`, append this job to the existing `.github/workflows/test-everything.yml`, alongside the Android/iOS test jobs. Its existing `push` trigger already includes `development`.

```yaml
jobs:
  # Existing build-and-test-android and build-and-test-ios jobs remain here.

  publish-public:
    needs: [build-and-test-android, build-and-test-ios]
    if: >-
      github.event_name == 'push' &&
      github.repository == 'a-sit-plus/private-valera' &&
      github.event.repository.private == true &&
      github.ref == 'refs/heads/development' &&
      github.event.deleted != true
    permissions:
      contents: read
    uses: a-sit-plus/internal-workflows/.github/workflows/publish-public.yml@REPLACE_WITH_REVIEWED_40_CHARACTER_COMMIT_SHA
    with:
      source-repository: a-sit-plus/private-valera
      source-branch: development
      destination-repository: a-sit-plus/valera
      destination-branch: development
      app-client-id: ${{ vars.PUBLIC_SYNC_APP_CLIENT_ID }}
      runner-labels: '["self-hosted", "Linux", "X64", "docker"]'
    secrets:
      app-private-key: ${{ secrets.PUBLIC_SYNC_APP_PRIVATE_KEY }}
```

Use the actual job IDs from each caller for `needs`. Publication is skipped when a dependency fails or is skipped. Protect the private branch with CI requirements too; this job is an additional post-merge check. A standalone `on: push` publication workflow without dependencies would publish before CI completes.

The reusable workflow independently checks the expected private caller, source branch, non-deletion event, and push/dispatch event type. The example caller only enables push events, so its existing manual dispatch will not publish. Re-running a successful branch-push run can retry a failed publication. For deliberate dispatch support, add a branch-restricted caller policy and retain the CI dependencies; the reusable workflow uses the selected event SHA.

The caller's hard-coded source guard means the same file can be published to `valera` safely: the publication job is skipped publicly. Supply the App key explicitly rather than forwarding all secrets. Hard-coded source/destination inputs are policy configuration; only trusted reviewers should change them.

## 5. Configure the public repository

- Install the App with access to `valera` as described above.
- Ensure public `development` already exists and shares history with private `development` (bootstrap establishes this).
- Restrict direct updates to the publication App and designated emergency maintainers. Block force pushes and branch deletion.
- Configure the App to bypass only the public integration rules it must bypass for direct publication. Merely allowing an App to push does not bypass required PRs or checks. Private checks do not satisfy the public repository's required checks.
- With rulesets, put required PRs/checks in a rule the App can bypass, and keep non-fast-forward/deletion restrictions in a separate rule without that bypass. Inspect all applicable organization/repository rules; an exception in one rule does not override another.
- Keep public `push` CI enabled. App-authenticated pushes trigger normal workflows; the conditional runner configuration should choose GitHub-hosted runners publicly. Do not grant public PR jobs access to private runners or the publication key.
- Do not configure a reverse automatic sync with the same credentials. Review and import public contributions privately first.

Releases remain authoritative in the public repository: release branches, `main`, release tags, GitHub Releases, packages and app-store deployment are managed there. The publication job synchronizes only `development`. Bring public release changes back privately using the manual procedure below. Additional published branches require explicit caller jobs, matching triggers and branch rules. Do not publish all refs as a shortcut.

After configuration, re-enable Actions in the private repository:

```bash
gh api --method PUT repos/a-sit-plus/private-valera/actions/permissions -F enabled=true
```

Keep the existing private/public runner selection in build and publish jobs; this reusable job's runner labels only select the publication runner. For Valera, ensure Linux build runners have emulator/KVM support and macOS runners provide the required Xcode and Apple Silicon simulators. Control sibling VC-K/Signum checkouts so persistent runners do not silently resolve unpublished composite-build dependencies.

## 6. Manually import public changes and releases

Use automatic private-to-public publication and manual public-to-private imports while public changes are infrequent. No reverse-sync App or scheduled import workflow is needed. Maintainers use their own normal Git credentials and private repository access for the import PR.

### Import public development

From a clean private checkout, `origin` must point to `a-sit-plus/private-valera`, and `public` to `a-sit-plus/valera`. Add the public remote once if it is missing:

```bash
git remote add public https://github.com/a-sit-plus/valera.git
```

Fetch both integration branches and create a dedicated private import branch:

```bash
git fetch --no-tags public
git switch development
git pull --ff-only origin development
# Choose a fresh branch name for each import if this name already exists.
git switch -c sync/public-development
git merge --no-ff public/development
git push -u origin sync/public-development

gh pr create --repo a-sit-plus/private-valera \
  --base development --head sync/public-development \
  --title "Import public development" \
  --body "Merge public development while preserving public commit ancestry. Use a merge commit for this PR."
```

If Git reports that the public history is already integrated, no import PR is needed. If `git pull --ff-only` fails because local `development` has diverged, reconcile your local work before continuing; do not reset or force-push over it. If the merge conflicts, resolve and review the files, stage those resolutions and run `git merge --continue` before pushing. To abandon an in-progress merge, use `git merge --abort`.

Review the private PR and let its normal CI run. **Choose “Create a merge commit” when merging this import PR. Do not squash-merge, rebase-merge or cherry-pick the public commits.** Their original SHAs must remain ancestors of private `development`. Ordinary private feature PRs can continue using squash merges. Review imported workflow/build changes before executing them with private infrastructure or credentials.

Once the import PR merges, the normal private `development` push runs CI and publication. The public branch can then fast-forward to the combined history. A failed publication caused by public divergence is expected until the import is complete; rerunning the old failed commit does not include the new merge. Let the new integration commit's run publish it. If public changes arrive again meanwhile, repeat the import.

### Keep release ownership public

Use the existing public release process:

1. Create and prepare public `release/x.y.z` from the intended public commit.
2. Merge the release into public `main`, then create the public release tag, GitHub Release and release artifacts through the public release process.
3. Merge public `main` back into public `development` using a merge commit. Include version bumps, changelog edits, release fixes and hotfixes. Preserve shared ancestry when integrating release branches; do not squash or rebase these cross-branch integrations.
4. Import public `development` into private `development` using the procedure above.

A hotfix made on public `main` follows the same return path. Private development can continue during public release preparation; resolve any conflicts during the import PR. Assign the public release maintainer responsibility for the merge-back and private import so these changes are not forgotten.

There is no need to maintain private `main`, copy `release/*` branches, or duplicate GitHub Releases and binary assets. If bootstrap copied `main`, treat that private branch as an initial snapshot unless you explicitly maintain it; do not use it as the authoritative release branch or publish it back publicly.

Gate release/deployment jobs on the exact public repository identity, for example `github.repository == 'a-sit-plus/valera'`, and on the intended release event/ref. Runner selection alone does not enforce public-only release ownership. Keep signing/store credentials in the public release environment and do not share them with private development just for synchronization. Importing workflow files or pushing a copied tag must not launch a private release or duplicate a store deployment.

### Optionally copy a specific public release tag

Tags are optional for private development. To inspect an exact public release locally, fetch its tag without moving existing tags. Replace `v5.9.3` below with an actual published release tag:

```bash
git fetch --no-tags public refs/tags/v5.9.3:refs/tags/v5.9.3
```

If the tag is also needed on GitHub in the private repository, push only that tag:

```bash
git push origin refs/tags/v5.9.3:refs/tags/v5.9.3
```

These commands preserve the original tag object, including annotations/signatures. Stop and investigate if an existing local/private tag points elsewhere; do not force-update it. Copying a tag also makes its referenced commit history available privately, but does not integrate it into private `development`, copy GitHub release notes/assets, or replace the release merge-back. Avoid broad `--tags` pushes from developer checkouts, which may contain unrelated local tags.

## 7. Verify and operate

1. Confirm the initial public/private integration branch SHAs match.
2. Merge a small publishable change privately after CI/review. Confirm the post-merge test jobs pass before publication runs.
3. Check the publication summary for source SHA, destination branch and previous public SHA. Confirm public `development` matches the published SHA and public CI starts.
4. Retry the same run: publication should succeed without changes. A stale run whose commit is already an ancestor of the public tip also does nothing.
5. Verify divergence handling in disposable repositories: add different commits to both branches and confirm publication fails without overwriting public work.

The job transfers only the triggering commit and its ancestry, and pushes exactly one branch without force, deletion or automatic tags. It uses a temporary bare repository, never executes source files, keeps tokens out of remote URLs/on-disk Git configuration, disables Git hooks, and removes temporary objects on exit. A crash/process kill can prevent cleanup; use disposable runners for stronger isolation.

Concurrency serializes publication for a destination within each private caller repository. GitHub concurrency can replace pending runs; a later successful commit includes earlier integrated commits. There must be **one authoritative private source per public destination branch**. Separate callers do not share a cross-repository lock. Git's ordinary push rules reject concurrent non-fast-forward updates.

If public history diverges, import/reconcile it in the private repository, run CI, and retry. Do not repair synchronization with force pushes. A public descendant of an old event SHA is treated as already published; the next private change will detect any still-unincorporated public work. GitHub App/branch protection errors require fixing installation access, Contents/Workflows permissions, or applicable branch rules. Monitor failed publication runs: private CI success alone does not prove publication succeeded.

To pause publication, disable/remove its caller job or revoke the App installation. To undo published behavior, merge and publish a revert privately. Removing a public branch or reverting a file does not make previously published content confidential again.

## References

- [GitHub fork visibility](https://docs.github.com/en/pull-requests/reference/forks)
- [Duplicating a repository, including Git LFS](https://docs.github.com/en/repositories/creating-and-managing-repositories/duplicating-a-repository)
- [GitHub App authentication in Actions](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/making-authenticated-api-requests-with-a-github-app-in-a-github-actions-workflow)
- [Official token action and repository scoping](https://github.com/actions/create-github-app-token)
- [App private key lifecycle](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/managing-private-keys-for-github-apps)
- [Branch protections](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [Reusable workflow access](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations)
