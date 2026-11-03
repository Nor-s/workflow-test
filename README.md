# workflow-test

Automatically opens a pull request in this repository when `Nor-s/submodule-test/main` advances.

## How it works

1. A push to `submodule-test/main` runs `Notify workflow-test`, which sends a `submodule-updated` event to this repository.
2. `Update submodule` fetches the latest commit on the `main` branch configured in `.gitmodules`.
3. When the submodule reference changes, it opens a PR from `chore/update-submodule-test` into this repository's default branch.

An existing update PR is kept current as more commits arrive. Once that PR is merged, the next upstream change opens a new PR. No PR is created when the latest commit is already included.

You can also run the check manually under Actions → `Update submodule` → **Run workflow**.

## GitHub setup

### workflow-test: Allow pull request creation

Under [Settings → Actions → General](https://github.com/Nor-s/workflow-test/settings/actions) → **Workflow permissions**, enable **Allow GitHub Actions to create and approve pull requests**. The workflow explicitly requests `contents: write` and `pull-requests: write`, so you do not need to change the default token permissions.

This repository uses its built-in `GITHUB_TOKEN` to create PRs.

### submodule-test: Add the notification token

Sending an event to another repository requires a token with access to the destination repository. The built-in `GITHUB_TOKEN` from `submodule-test` cannot access `workflow-test`.

1. Create a [fine-grained personal access token](https://github.com/settings/personal-access-tokens/new).
   - Resource owner: `Nor-s`
   - Repository access: **Only select repositories** → `workflow-test`
   - Repository permissions: **Contents → Read and write**
2. Add a repository secret under [submodule-test → Settings → Secrets and variables → Actions](https://github.com/Nor-s/submodule-test/settings/secrets/actions).
   - Name: `WORKFLOW_TEST_TOKEN`
   - Secret: the token you created

Update the secret when the token expires. No notification token is needed in the `workflow-test` repository.

## Apply the changes

The workflows belong to separate repositories and must be committed and pushed separately. First, publish the receiving workflow to this repository's `main` branch.

```sh
git add .gitmodules .github/workflows/update-submodule.yml README.md
git commit -m "ci: create pull requests for submodule updates"
git push origin main
```

Then publish the notification workflow to `submodule-test/main`. This commit itself will trigger a submodule update PR.

```sh
git -C submodule-test add .github/workflows/notify-workflow-test.yml
git -C submodule-test commit -m "ci: notify workflow-test on main updates"
git -C submodule-test push origin main
```

The receiving workflow must exist on `workflow-test`'s default branch for repository dispatches to work.

## Verify

1. Push a change to `submodule-test/main`.
2. Check that `Notify workflow-test` succeeds in the upstream repository's Actions tab.
3. Check that `Update submodule` succeeds here and opens a PR with the updated submodule commit.
4. Push another upstream change while the PR is open and confirm that the same PR is updated.
5. Merge the PR, push another upstream change, and confirm that a new PR is created.

GitHub documentation: [repository dispatch events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#repository_dispatch), [notification API token permissions](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event), [allowing Actions to create pull requests](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#preventing-github-actions-from-creating-or-approving-pull-requests).
