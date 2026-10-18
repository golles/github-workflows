# github-workflows

Shared GitHub Actions, reusable workflows and Renovate presets for my repositories.

## Actions

### `setup-app`

Creates a token for a GitHub App, so that commits, comments and pull requests are made by the app instead of `github-actions[bot]`, and trigger other workflows.

```yaml
- name: Setup app
  id: app
  uses: golles/github-workflows/actions/setup-app@main
  with:
    app-id: ${{ secrets.PR2D2_APP_ID }}
    private-key: ${{ secrets.PR2D2_APP_PRIVATE_KEY }}

- name: Run release drafter
  uses: release-drafter/release-drafter@v7
  with:
    token: ${{ steps.app.outputs.token }}
```

| Input         | Description             |
| ------------- | ----------------------- |
| `app-id`      | GitHub App ID.          |
| `private-key` | GitHub App private key. |

| Output  | Description                                                               |
| ------- | ------------------------------------------------------------------------- |
| `id`    | User ID of the app.                                                       |
| `name`  | Name of the app, e.g. `my-app[bot]`.                                      |
| `email` | Commit email of the app, e.g. `123+my-app[bot]@users.noreply.github.com`. |
| `token` | Installation token of the app.                                            |

## Reusable workflows

### `python-version.yaml`

Checks that a Python project uses the same Python version as Home Assistant. It reads `requires_python` of the latest `homeassistant` release from PyPI and fails when:

- `.python-version` is missing;
- `.python-version` isn't the same minor version as Home Assistant's minimum Python version: either it's ahead of Home Assistant, or Home Assistant has moved to a newer version and it's time to update;
- `requires-python` in `pyproject.toml` doesn't allow Home Assistant's minimum Python version, so the package can't be installed on Home Assistant.

```yaml
name: Python version

on: # yamllint disable-line rule:truthy
  push:
    branches:
      - main
  pull_request: ~
  schedule:
    - cron: "0 6 * * 1"
  workflow_dispatch:

permissions:
  contents: read

jobs:
  python-version:
    name: Python version
    uses: golles/github-workflows/.github/workflows/python-version.yaml@main
```

The weekly schedule makes the check fail, and notify you, when Home Assistant moves to a newer Python version.

## Renovate presets

| Preset                                                    | Use for                            |
| --------------------------------------------------------- | ---------------------------------- |
| `github>golles/github-workflows//renovate/python`         | Python libraries                   |
| `github>golles/github-workflows//renovate/ha-integration` | Home Assistant custom integrations |

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>golles/github-workflows//renovate/ha-integration"]
}
```

Repository-specific rules can be added to the repository's own `packageRules`, below the `extends`.

### `python`

Based on `config:recommended`:

- Runs before 9am and adds the `dependencies` label.
- Lock file maintenance, automerged.
- Pre-commit hooks, automerged.
- Pins GitHub Actions, npm and dev dependencies.
- Automerges minor and patch updates of dependencies that are at `1.0.0` or later, and all dev dependency updates.
- Doesn't automerge devcontainer updates.
- Doesn't update the Python version: `requires-python`, `.python-version` and the `mcr.microsoft.com/devcontainers/python` image. The Python version follows Home Assistant and is updated manually; see [`python-version.yaml`](#python-versionyaml).

### `ha-integration`

Extends `python`, and also:

- Doesn't update `pytest`, `pytest-asyncio`, `pytest-cov` and `pytest-homeassistant-custom-component`, as their versions follow Home Assistant.
- Doesn't update the `runhass` dependency group, which contains Home Assistant's runtime dependencies.
