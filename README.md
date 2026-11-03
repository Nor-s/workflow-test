# Automatic submodule update PRs

A push to a submodule's `main` branch opens or updates a pull request in its parent repository.

A GitHub App sends the notification between repositories; the parent uses its own `GITHUB_TOKEN` to create the commit and PR.

**Push to source → GitHub App notification → target workflow → update PR**

| Role | Example | What belongs here |
| --- | --- | --- |
| **Source / submodule** | `Nor-s/submodule-test` | App credentials and the notification workflow |
| **Target / parent** | `Nor-s/workflow-test` | `.gitmodules` and the workflow that updates the submodule reference |
| **GitHub App** | `submodule-update2` | Permission to send an event to the target repository |

The examples use `Nor-s`, `submodule-test`, and `workflow-test`.

For your own setup, replace these names with your account or organization, source repository, and target repository.


Screenshots show the personal-account example. The same repository setup works in an organization; the organization-specific settings are called out below.

## Setup

1. [Create a GitHub App](#1-create-a-github-app)
2. [Configure credentials in the submodule repository](#2-configure-credentials-in-the-submodule-repository)
3. [Install the app on the target](#3-install-the-app-on-the-target)
4. [Add the submodule workflow](#4-add-the-submodule-workflow)
5. [Add the target workflow](#5-add-the-target-workflow)

## 1. Create a GitHub App

Choose the account that will own the app, then open **Developer settings → GitHub Apps → New GitHub App**.

| App owner | Where to start |
| --- | --- |
| Personal account | Profile menu → Settings, or [create an app](https://github.com/settings/apps/new) |
| Organization | Your organization → Settings; use an organization owner or an account granted permission to manage its apps |

Give the app a unique name, such as `your-org-submodule-updater`, and use the target repository URL as its homepage. [GitHub's registration guide](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app) covers both ownership options.

Configure these fields:

| Setting | Value for this setup |
| --- | --- |
| Webhook | off **Active** |
| Callback, OAuth, device flow, and setup URL | **Default** |
| Repository **Contents** permission | **Read and write** |
| Repository **Metadata** permission | **Read-only** |
| Repository **Pull requests** permission | **No access** |
| Installation availability | **Only on this account** / **Any account** |

The app sends notifications through the [repository dispatch API](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event), which requires `Contents: write`.

Metadata access is added automatically. 
 
> [!IMPORTANT]
> Choose **Only on this account** when the app owner also owns the target

<img src="docs/images/app-contents-permission.png" alt="GitHub App repository permissions with Contents set to Read and write" width="760">

<details>
<summary>Webhook and installation availability screenshots</summary>

Clear **Active** in the webhook section. GitHub supports disabling webhooks for apps used only for authentication. See [registration settings](https://docs.github.com/en/apps/maintaining-github-apps/modifying-a-github-app-registration#activating-or-deactivating-the-github-app-webhook).

<img src="docs/images/app-webhook-disabled.png" alt="GitHub App webhook section with Active unchecked" width="760">

Choose the installation availability for the account that owns the target. A personal app intended for an organization target must allow installation on that organization. See [app installation availability](https://docs.github.com/en/apps/using-github-apps/installing-your-own-github-app).

<img src="docs/images/app-installation-scope.png" alt="GitHub App installation availability choices: Only on this account or Any account" width="760">

</details>

Save the app and copy its **Client ID** from its settings. The workflow reads that value as `APP_CLIENT_ID`.

## 2. Configure credentials in the submodule repository

In the **source repository**, `submodule-test`, open **Settings → Secrets and variables → Actions**. The notification workflow reads the two values below from there.

### 2.1 APP_CLIENT_ID

Copy the **Client ID** from the app's **General** settings page.

<img src="docs/images/app-client-id.png" alt="GitHub App General settings page with the Client ID field highlighted" width="900">

In the **Variables** tab, choose **New repository variable**. Name it `APP_CLIENT_ID` and paste the Client ID as its value. The workflow reads it as `${{ vars.APP_CLIENT_ID }}`.

<img src="docs/images/repository-variable-app-client-id.png" alt="Repository variables list in submodule-test showing APP_CLIENT_ID" width="900">

### 2.2 APP_PRIVATE_KEY

Open the app's settings, then choose **Credentials → Key pairs → New key**. GitHub downloads a `.pem` file. Keep this original file: GitHub stores only the public key, so the private key cannot be downloaded again. See [managing app private keys](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/managing-private-keys-for-github-apps#generating-private-keys).

<img src="docs/images/app-credentials-key-pairs.png" alt="GitHub App Credentials page with the Key pairs tab selected and the New key button" width="900">

<img src="docs/images/app-credentials-new-key.png" alt="GitHub App Credentials page with Credentials, Key pairs, and New key highlighted" width="900">

In the **Secrets** tab, choose **New repository secret** under **Repository secrets**. Name it `APP_PRIVATE_KEY` and paste the **entire PEM file** as its value, including the `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` lines and the original line breaks. The workflow reads it as `${{ secrets.APP_PRIVATE_KEY }}`. Keep the PEM outside the repository's tracked files.

<img src="docs/images/new-repository-secret.png" alt="New repository secret form in Nor-s/submodule-test with the name APP_PRIVATE_KEY" width="900">

Use **Repository secrets** for this workflow. Environment secrets require a job that selects the matching `environment`, which these examples do not do. See [environment secret availability](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments#environment-secrets).

## 3. Install the app on the target

Open the app's **Install App** page, or use:

```text
https://github.com/apps/YOUR_APP_SLUG/installations/new
```

1. Choose the account or organization that owns **the target repository**.
2. Choose **Only select repositories**.
3. Select **`workflow-test`**.
4. Review the permissions and click **Install**.

Install the app on the parent repository (`workflow-test`), not the submodule. If GitHub shows **Request**, an organization owner must approve it.

## 4. Add the submodule workflow

In the **source repository**, create `.github/workflows/notify-workflow-test.yml`:

```yaml
name: Notify workflow-test

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions: {}

jobs:
  notify:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - name: Create a GitHub App token for workflow-test
        id: app-token
        uses: actions/create-github-app-token@64ab729a6294000f5aa4a4bc0b73d1f32bc45510 # v3.3.0
        with:
          client-id: ${{ vars.APP_CLIENT_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: Nor-s
          repositories: workflow-test
          permission-contents: write

      - name: Notify workflow-test of an upstream update
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
        run: |
          gh api --method POST repos/Nor-s/workflow-test/dispatches \
            -f event_type=submodule-updated
```

For your target, change both `owner: Nor-s` and `repos/Nor-s/workflow-test/dispatches`. The `repositories` input is the target repository name. Change the `push.branches` filter if your source branch is different.

This action creates a short-lived installation token, uses it for the notification, and revokes it during job cleanup. The app credentials remain in repository settings. See [Create GitHub App Token](https://github.com/actions/create-github-app-token/tree/v3.3.0#usage).

The source and target both use the event type `submodule-updated`; keep those values aligned.

## 5. Add the target workflow

### Register the submodule

In the **target repository**, add the source if it is not already a submodule:

```sh
git submodule add -b main https://github.com/Nor-s/submodule-test.git submodule-test
```

For an existing submodule, check that the target tracks a `.gitmodules` entry with the correct URL, path, and branch:

```ini
[submodule "submodule-test"]
    path = submodule-test
    url = https://github.com/Nor-s/submodule-test.git
    branch = main
```

The workflow updates the branch named in `.gitmodules`. Keep the Git submodule entry and this file committed in the target.

### Allow PR creation

In the target's **Settings → Actions → General → Workflow permissions**, enable **Allow GitHub Actions to create and approve pull requests** and save.

The workflow below requests `contents: write` and `pull-requests: write` explicitly. An organization or enterprise policy can restrict the repository setting; ask its owner to enable the required policy if it is locked. See [Actions permissions](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#preventing-github-actions-from-creating-or-approving-pull-requests).

### Create the workflow file

In the target, create `.github/workflows/update-submodule.yml`:

```yaml
name: Update submodule

on:
  repository_dispatch:
    types: [submodule-updated]
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: update-submodule-test
  cancel-in-progress: false

jobs:
  update:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Check out the default branch
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          ref: ${{ github.event.repository.default_branch }}
          submodules: true

      - name: Update submodule-test to the latest main commit
        run: git submodule update --remote --checkout -- submodule-test

      - name: Create or update the pull request
        uses: peter-evans/create-pull-request@5f6978faf089d4d20b00c7766989d076bb2fc7f1 # v8.1.1
        with:
          token: ${{ github.token }}
          base: ${{ github.event.repository.default_branch }}
          branch: chore/update-submodule-test
          delete-branch: true
          add-paths: submodule-test
          commit-message: 'chore: update submodule'
          author: 'efaa98 <email.for.ai.automation@gmail.com>'
          committer: 'efaa98 <email.for.ai.automation@gmail.com>'
          title: 'chore: update submodule'
          body: 'update submodule'
```

The tracked implementation is [update-submodule.yml](.github/workflows/update-submodule.yml). For another submodule path, update the `git submodule update` command, `add-paths`, and PR branch name together.

### Customize the commit and PR

| What you want to change | Action input | Value used here |
| --- | --- | --- |
| Commit message | `commit-message` | `chore: update submodule` |
| Commit author | `author` | `efaa98 <email.for.ai.automation@gmail.com>` |
| Commit committer | `committer` | `efaa98 <email.for.ai.automation@gmail.com>` |
| PR title | `title` | `chore: update submodule` |
| PR body | `body` | `update submodule` |

Replace both identity fields with your intended commit name and email. Matching `author` and `committer` produces one commit identity instead of the additional “and github-actions[bot]” committer attribution.

The **PR creator** is determined by the authentication token. This workflow uses `GITHUB_TOKEN`, so the PR is opened by `github-actions[bot]` even when the commit metadata names `efaa98`. The [Create Pull Request inputs](https://github.com/peter-evans/create-pull-request#action-inputs) configure these values independently.

The workflow uses a fixed PR branch, so later source pushes update the same open PR. After that PR is merged, the next source change creates a new PR. If the latest source commit is already included in the target, there is no update to propose.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Empty `client-id` input | The **source** has a repository **variable** named `APP_CLIENT_ID` with the app's Client ID |
| Empty `private-key` input | The **source** has a repository **secret** named `APP_PRIVATE_KEY` |
| `Invalid keyData` or `Failed to read private key` | Replace the secret with the complete PEM file, including its BEGIN/END lines and original line breaks |
| `404 Not Found` while looking up the app installation | Install the app for the **target owner**, include the target repository, and check the source workflow's `owner` |
| `403` when sending the notification | The app installation must grant **Contents: write** on the target; approve any pending permission changes |
| GitHub Actions is not permitted to create a PR | Enable PR creation in the target's Actions settings and check organization policies |
| `no submodule mapping found in .gitmodules` | Commit the matching `.gitmodules` entry in the target |
| An action is not allowed to run | Allow `actions/create-github-app-token`, `actions/checkout`, and `peter-evans/create-pull-request` in the organization's Actions policy |
| Notification succeeds but no target run appears | Publish the receiver on the target's default branch and match the `submodule-updated` event type |
| No new PR appears | Check whether an update PR is already open or the target already references the latest source commit |

The screenshots were captured while setting up this example. GitHub's navigation may change; the linked official guides describe the current settings.
