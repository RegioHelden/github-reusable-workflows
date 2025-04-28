# GitHub reusable workflows

The workflows in this repository are the base for building and publishing our open source Python libraries.

They require read and write access to the repositories that they should manage. \
The easiest way to do so is to create a personal access token on a user account where the resource owner is set to your organization. \
Read and write access to code, issues and pull requests is needed.

## Protect your main branch

To enforce that all changes are only coming in through pull requests, you need to protect the main branch of our repository

As a repository admin

* Go to your repository
* Select tab `Settings`
* In the `Code and automation` section, go to `Branches`
* Click `Add branch ruleset`
* Use `Protect main` for `Ruleset Name`
* Select `Active` for `Enforcement status`
* Under `Target branches`, click `Add target` and select `Include default branch`
* As `Branch rules`, select `Require a pull request before merging`, keep all other settings default
* Check `Block force pushes`

## Create a new token

As the user that should execute the workflows, create a new token

* Click on your profile picture in the top right corner
* Go to `Settings`
* Go to `Developer Settings`
* Open `Personal access tokens`
* Go to `Fine-grained tokens`
* Click `Generate new token`
* Select your organization as `Resource owner`
* Select that the token never expires
* Select `All repositories` for `Repository access`
* Select the following `Repository permissions`
  * `Metadata` read only
  * `Contents` read and write
  * `Issues` read and write
  * `Pull requests` read and write

As an organization admin, approve the token request

* Go to your organization
* Select tab `Settings`
* In the `Third-party Access` section open `Personal access tokens`
* Go to `Pending requests`
* Approve the request

As an organization admin, set the token as the `COMMIT_KEY` secret on your organization

* Go to your organization
* Select tab `Settings`
* In the `Security` section open `Secrets and variables`
* Go to `Actions`
* Create a `New organization secret` called `COMMIT_KEY` with the token

## Set up trusted publishing

To deploy your packages to PyPI, you first need to set up trusted publishing there.

As a PyPI project admin

* Go to the project management view on PyPI
* Select menu `Publishing`
* Add a new entry for GitHub using the following settings
  * Owner: The GitHub organization owning the repository
  * Repository name: The name of the repository
  * Workflow name: If you follow the naming guidelines below, use `build-and-publish.yaml`
  * Environment name: leave empty

The project that should be managed by the workflows must have at least one tag in the form of `vX.X.X` and a release for that tag on GitHub. The release version (without `v`) must match the current version in the `version` file in the repository root.

# Usage

Using these workflows to publish a release follows specific rules.

* All pull requests must contain exactly one label describing their use case (e.g. `bug`, `feature` or `backwards-incompatible`)
  Based on these labels, we decide what kind of [semantic version](https://semver.org/) to release.
* When all pull requests for the release are merged, we need to manually trigger the `Open release PR` action
* This will then open a PR with the version change and an automatically generated changelog
* Once the version update is merged to the main branch, we will automatically create a git tag
* After the tag creation, we will automatically build the package, upload it to PyPI and make a GitHub release

## Integration

We will need to create five workflow files in our target repository that will be triggered by different events.

The easiest way is to integrate your project with our [modulesync setup for FOSS Django libraries](https://github.com/RegioHelden/modulesync-config-django). This will automatically push all actions to your target repository. You will then just need to create a `workflow.yaml` with your project specific test setup and the `ruff` integration. See `Check code` below.

### Sync labels

We're syncing labels from our central definition at https://github.com/RegioHelden/.github/blob/main/labels.yaml to be sure that all projects are handled in the same way.

#### Parameters

* `delete-other-labels`, default `true`, decide if labels not listed in the central definition should be removed

#### Example

Usually called `cron.yaml`

```yaml
---

name: Cron actions

on:
  workflow_dispatch:
  schedule:
    - cron:  '0 0 * * *'

jobs:
  # synchronize labels from central definition at https://github.com/RegioHelden/.github/blob/main/labels.yaml
  # see https://github.com/RegioHelden/github-reusable-workflows/blob/main/.github/workflows/sync-labels.yaml
  update-labels:
    name: Update labels
    permissions:
      issues: write
    uses: RegioHelden/github-reusable-workflows/.github/workflows/sync-labels.yaml@main

```

### Check code

Run ruff linter and formatting on the submitted code. \
May/should be extended by custom validation logic.

#### Parameters

* `python-version`
* `ruff-version`

#### Example

Usually called `workflow.yaml`

```yaml
---

name: Check code

on:
  # code pushed to pull request branch
  push:
    branches-ignore:
      - main
  # when draft state is removed (needed as automatically created PRs are not triggering this action)
  pull_request:
    types: [ready_for_review]

jobs:
  # lint code for errors
  # see https://github.com/RegioHelden/github-reusable-workflows/blob/main/.github/workflows/python-ruff.yaml
  lint:
    name: Lint
    permissions:
      contents: read
    uses: RegioHelden/github-reusable-workflows/.github/workflows/python-ruff.yaml@main
    with:
      ruff-version: "0.11.0"

```

### Check pull request

To be able to generate a proper changelog and determine what [semantic version](https://semver.org/) change is needed, we require all PRs to have exactly one label attached (e.g. `bug` or `feature`).

#### Example

Usually called `check_pull_request.yaml`

```yaml
---

name: Check pull request

on:
  # when labels are added/removed or draft status gets changed to ready for review
  pull_request:
    types: [opened, synchronize, reopened, labeled, unlabeled, ready_for_review]

jobs:
  # make sure the pull request matches our guidelines like having at least one label assigned
  # see https://github.com/RegioHelden/github-reusable-workflows/blob/main/.github/workflows/check-pull-request.yaml
  check-pull-request:
    name: Check pull request
    permissions:
      issues: write
      pull-requests: write
    uses: RegioHelden/github-reusable-workflows/.github/workflows/check-pull-request.yaml@main

```

### Trigger a new release

When all changes for your release are merged, you can start the release process. \
Go to `Actions`, then `Open release PR`. Click `Run workflow` and run it on the `main` branch. \
This action will then create the version update PR.

#### Secrets

* `personal-access-token`, required, the personal access token created above

#### Example

Usually called `open-release-pr.yaml`

```yaml
---

name: Open release PR

on:
  workflow_dispatch:

jobs:
  # trigger creation of a release pull request with version increase and changelog update
  # see https://github.com/RegioHelden/github-reusable-workflows/blob/main/.github/workflows/release-pull-request.yaml
  open-release-pr:
    name: Open release PR
    permissions:
      contents: write
      pull-requests: write
    uses: RegioHelden/github-reusable-workflows/.github/workflows/release-pull-request.yaml@main
    secrets:
      personal-access-token: "${{ secrets.COMMIT_KEY }}"

```

### Create git tag

Once the release PR is reviewed and merged, we will create a new git tag for the new version.

#### Secrets

* `personal-access-token`, required, the personal access token created above

#### Parameters

* `python-version`

#### Example

Usually called `tag-release.yaml`

```yaml
---

name: Tag release

on:
  push:
    branches:
      - main

jobs:
  # create a new git tag when a version update was merged to main branch
  # see https://github.com/RegioHelden/github-reusable-workflows/blob/main/.github/workflows/tag-release.yaml
  tag-release:
    name: Create tag
    permissions:
      contents: write
    uses: RegioHelden/github-reusable-workflows/.github/workflows/tag-release.yaml@main
    secrets:
      personal-access-token: "${{ secrets.COMMIT_KEY }}"

```

### Build and publish

Once the tag is pushed, we will

* Build Python packages (with wheel and source tarball) using `uv build`
* Sign the package files and publish a new release on GitHub
* Publishing to PyPI must be part of the local repo for now as trusted publishing does not work with resuable workflows

#### Parameters

* `python-version`

#### Example

Usually called `build-and-publish.yaml`

```yaml
---

name: Build and publish

on:
  push:
    tags:        
      - '**'   

jobs:
  # build package, make release on github and upload to pypi when a new tag is pushed
  # see https://github.com/RegioHelden/github-reusable-workflows/blob/main/.github/workflows/build-and-publish.yaml
  build-and-release:
    name: Build and publish
    permissions:
      contents: write
      id-token: write
    uses: regiohelden/github-reusable-workflows/.github/workflows/build-and-publish.yaml@main

  # must be defined in the repo as trusted publishing does not work with reusable workflows yet
  # see https://github.com/pypi/warehouse/issues/11096
  publish-pypi:
    name: Publish on PyPI
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    needs:
      - build-and-release
    steps:
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install the latest version of uv
        uses: astral-sh/setup-uv@v5

      - name: Download the distribution packages
        uses: actions/download-artifact@v4
        with:
          name: python-package-distributions
          path: dist/

      - name: Publish
        run: uv publish --trusted-publishing always

```
