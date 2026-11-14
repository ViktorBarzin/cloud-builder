# cloud-builder

Cloud's builder. Cloud (the homelab's self-hosted cloud) dispatches
`.github/workflows/build.yml` to turn one commit of an App repo into an image
on ghcr. Nothing else starts it. The source of truth is `builder/` in the
cloud repo; `scripts/publish-builder.sh` there publishes it here.

## What a run does

| Job | What it does | What it can see |
|---|---|---|
| `fetch` | checks the inputs, checks out exactly `sha` from `repo_url`, deletes `.git`, uploads the tree as an artifact kept for 1 day | only its fetch step gets a repository token: `FORGEJO_READ_TOKEN` for `source: forgejo`, `GH_OWNER_READ_TOKEN` for `source: github-owner`, none for `public`. A token is sent only to forgejo.viktorbarzin.me or github.com, as a header scoped to that host, with no system or user git config (so no Git LFS filter runs) and none at all for a checkout that needs no network |
| `build` | downloads the tree, builds it with the repo's `Dockerfile`, else Railpack 0.43.0, and pushes `<image>:<sha>` to ghcr, labelled with this repo as `org.opencontainers.image.source` | no repository token; `GITHUB_TOKEN` with `packages: write` only |

The build prints only the last 200 lines of its output and uploads nothing
but the image. Each job stops after 20 minutes, and one run per image runs
at a time: a newer push cancels the older run.

## Inputs

| Input | Form |
|---|---|
| `build_id` | Cloud's Deploy id, a UUID; the run is titled `build <build_id>` |
| `repo_url` | a plain https clone URL: no port, credentials, query or fragment |
| `sha` | the full 40-character commit sha |
| `image` | `ghcr.io/viktorbarzin/cloud-<project>_<app>`, without a tag |
| `source` | `public`, `forgejo` or `github-owner` |

## When a run fails

The failing step prints one line `cloud-failure: <code>`, which Cloud reads
from the job log to tell the User what to do next:

| Code | Meaning |
|---|---|
| `invalid_input` | an input did not pass the checks above |
| `fetch_failed` | the repo or the commit could not be read |
| `no_build_plan` | no Dockerfile, and Railpack could not tell the language |
| `build_failed` | the build itself failed |
| `timed_out` | a time limit stopped it: 3 minutes for Railpack's plan, 15 for the build, 20 for each job |

## Secrets

| Secret | What |
|---|---|
| `FORGEJO_READ_TOKEN` | reads repos on forgejo.viktorbarzin.me; scope `read:repository` only, never an admin scope |
| `GH_OWNER_READ_TOKEN` | reads the owner's private github.com repos |

Cloud dispatches with its own token (Vault `secret/cloud/system`, key
`github_dispatch_token`), which needs Actions read and write on this repo.
