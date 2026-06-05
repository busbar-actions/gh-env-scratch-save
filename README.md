> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/gh-env-scratch-save

Create or update a **GitHub Environment** and store a scratch org's credentials
there as **environment secrets** — a secure cross-job/cross-workflow handoff that
keeps the session out of step outputs, artifacts, and caches.

Pair it with [`sf-org-create`](https://github.com/busbar-actions/sf-org-create):
that action writes the new org's credentials to a JSON file; this action persists
them into a stable named Environment slot (default `scratch`) that later jobs can
target with `environment:`.

## What it does

1. Reads the JSON written by `sf-org-create` (`credentials-path`), which contains
   the [`CredentialContext`](../../busbar-extensions/crates/busbar-auth) plus the
   org/username/login-url metadata.
2. Resolves a GitHub API token — either a passed `github-token`/`GITHUB_TOKEN`, or
   a **GitHub App installation token** minted from `github-app-id` +
   `github-app-private-key` (it finds the installation that can reach the target
   repo).
3. Creates or updates the GitHub Environment in the target repository.
4. Stores the session as encrypted environment **secrets**: `SF_ACCESS_TOKEN`,
   `SF_INSTANCE_URL`, `SF_USERNAME`, `SF_LOGIN_URL`, `SF_SCRATCH_ORG_ID`,
   `SF_SCRATCH_ORG_INFO_ID`, `BUSBAR_CREDENTIAL_CONTEXT` (full context JSON), and
   `SF_REFRESH_URL` when a one-time refresh URL is present.
5. Emits step outputs (ids/names only — **never the secrets**), a job-summary
   table, and a notice annotation.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `credentials-path` | no | `.busbar/scratch-credentials.json` | Path to the JSON written by `sf-org-create`. |
| `environment-name` | no | `scratch` | GitHub Environment name (the stable named slot). |
| `repository` | no | `` (current repo) | Target repository in `owner/repo` form. Empty = `github.repository`. |
| `github-token` | no | `` | Token with rights to create environments and write environment secrets. Optional when using App creds. |
| `github-app-id` | no | `` | GitHub App id used to mint an installation token for the target repo. |
| `github-app-private-key` | no | `` | GitHub App private key PEM used to mint the installation token. |
| `version` | no | `latest` | Release tag of the `gh-env-scratch-save` binary. |
| `binary-repo` | no | `busbar-actions/actions-dist` | Repo hosting the prebuilt binary releases. |

## Outputs

| Output | Description |
|---|---|
| `environment-name` | The Environment name that was created/populated. |
| `repository` | The repository that received the environment. |
| `scratch-org-id` | The scratch org id persisted into the environment. |
| `scratch-org-info-id` | The DevHub `ScratchOrgInfo` id persisted into the environment. |

The stored Salesforce session never appears in outputs — outputs carry only ids
and names. The session lives only in the Environment's encrypted secret store.

## Auth / permissions model

This action **does not** mint or use a Salesforce OIDC token — it is a pure
producer/handoff for credentials that `sf-org-create` already minted. It needs
**GitHub API** write access to manage environments and environment secrets:

```yaml
permissions:
  contents: read
```

Provide that access one of two ways:

- **`github-token`** (or the default `github.token`) — must be able to create
  environments and write environment secrets. The default `GITHUB_TOKEN` only
  works for the current repo and needs sufficient scope; cross-repo writes
  generally require a PAT or App token.
- **GitHub App** — set `github-app-id` + `github-app-private-key`; the binary
  lists installations, finds the one that can reach the target repo, and mints a
  short-lived installation token. Preferred for writing into another repository.

> The Salesforce `SF_ACCESS_TOKEN` it stores is **handed off**, not owned by this
> action — so this action must NOT revoke it. Revocation/teardown is the
> consuming job's responsibility (see Cleanup).

## Example

```yaml
jobs:
  provision:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write          # for sf-org-create's OIDC exchange
    environment: devhub-pbo-scratch
    steps:
      - uses: busbar-actions/sf-org-create@v1
        id: org
        with:
          org-name: "PR-${{ github.event.number }}"
          admin-email: ci@example.com

      - uses: busbar-actions/gh-env-scratch-save@v1
        with:
          credentials-path: ${{ steps.org.outputs.credentials-path }}
          environment-name: scratch          # stable slot downstream jobs target
          # cross-repo / preferred: use App creds instead of github-token
          github-app-id: ${{ vars.BUSBAR_GH_APP_ID }}
          github-app-private-key: ${{ secrets.BUSBAR_GH_APP_PRIVATE_KEY }}

  # Later job consumes the saved session
  use-org:
    needs: provision
    runs-on: ubuntu-latest
    environment: scratch        # secrets: SF_ACCESS_TOKEN, SF_INSTANCE_URL, ...
    steps:
      - run: echo "instance ${{ '$SF_INSTANCE_URL' }}"
        env:
          SF_ACCESS_TOKEN: ${{ secrets.SF_ACCESS_TOKEN }}
          SF_INSTANCE_URL: ${{ secrets.SF_INSTANCE_URL }}
```

## Observability

The binary uses the shared
[`github-actions-ux`](../../busbar-extensions/crates/github-actions-ux) crate for
inputs, the `$GITHUB_OUTPUT` / `$GITHUB_STEP_SUMMARY` writers, the summary table,
and the notice annotation. On error it calls `github_actions_ux::fail`, which
emits a `::error` workflow command and exits 1.

## Cleanup

This action only **saves** the session. To tear down, delete the scratch org with
[`sf-org-delete`](https://github.com/busbar-actions/sf-org-delete) (using the
saved `scratch-org-info-id`) in a later job or cleanup workflow, and remove the
Environment / revoke the stored session there — this action does not (and must
not) revoke the handed-off token itself.
