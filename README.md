# cloud-builder

Cloud's builder. Cloud (the homelab's self-hosted cloud) dispatches
`.github/workflows/build.yml` to turn one commit of an App repo into an image
on ghcr. Nothing else starts it. The source of truth is `builder/` in the
cloud repo; `scripts/publish-builder.sh` there publishes it here.

This repo is public, so GitHub Actions minutes cost nothing. Anyone can read
its run logs, run titles and artifacts, so none of them holds a User's data
in plain form:

| What anyone can read | What it holds |
|---|---|
| Dispatch inputs | `build_id`, a UUID, and `job`, the build sealed with [age](https://age-encryption.org) to this repo's secret `CLOUD_JOB_KEY` |
| Run title | `build <build_id>` |
| Run log | step names, and one `cloud-failure: <code>` line when a step fails |
| Artifacts | `tree-<build_id>`: the source tree, sealed to `CLOUD_JOB_KEY`'s recipient, kept 1 day. `log-<build_id>-fetch` and `log-<build_id>-build`: each job's output, sealed to Cloud's log recipient, kept 3 days |

Every step that touches the build writes its output and its errors to a
file, never to the run log. Values read from the job are masked
(`::add-mask::`, with `%`, CR and LF escaped) before any later step runs,
except values under 4 characters and values that are part of what the
workflow prints itself. No expression puts a value into a step's `env`,
which the log prints.

## What a run does

| Job | What it does | What it can see |
|---|---|---|
| `fetch` | opens and checks the job, checks out exactly `sha` from `repo_url`, deletes `.git`, seals the tree and uploads it | its first step gets `CLOUD_JOB_KEY`. Only its fetch step gets a repository token: `FORGEJO_READ_TOKEN` for `source: forgejo`, `GH_OWNER_READ_TOKEN` for `source: github-owner`, none for `public`. A token is sent only to forgejo.viktorbarzin.me or github.com, as a header scoped to that host, with no system or user git config (so no Git LFS filter runs) and none at all for a checkout that needs no network |
| `build` | opens the job and the tree (with git's modes, 644 and 755, so an image that copies it in as root can still read it as its App's user), builds it with the repo's `Dockerfile`, else Railpack 0.43.0, and pushes `<image>:<sha>` to ghcr, labelled with this repo as `org.opencontainers.image.source` | its first step gets `CLOUD_JOB_KEY`; no repository token; `GITHUB_TOKEN` with `packages: write` only |

Each job stops after 20 minutes. Cloud cancels a run that a newer push
overtook; the workflow has no concurrency group, since one named after the
image would name the App in the clear.

## The job

The `job` input is base64 of an age file sealed to `CLOUD_JOB_KEY`'s
recipient. Inside, JSON:

| Field | Form |
|---|---|
| `v` | `1` |
| `build_id` | the same UUID as the `build_id` input |
| `repo_url` | a plain https clone URL: no port, credentials, query or fragment |
| `sha` | the full 40-character commit sha |
| `image` | `ghcr.io/viktorbarzin/cloud-<project>_<app>`, without a tag |
| `source` | `public`, `forgejo` or `github-owner` |
| `build_env` | the App's plain env from the Project File, `{NAME: value}`; never secrets |
| `log_recipient` | Cloud's age recipient for the build output |

## Build env

Plain env reaches the build; secrets never do. Cloud leaves out names that
would steer this host or its tools instead of the App's build: `PATH`,
`LD_*`, `DOCKER_*`, `BUILDX_*`, `BUILDKIT_*`, `HTTP_PROXY`, `HTTPS_PROXY`,
`NO_PROXY`, `ALL_PROXY`, `NODE_OPTIONS`, `BUN_*` and
`RAILPACK_CONFIG_FILE`, in any case, and the
job check refuses a job that has one. No value goes into the environment
of a process on the runner. With a `Dockerfile`, each
variable is passed as `--build-arg NAME=value`, which a Dockerfile uses
where it declares `ARG NAME`, and all of them as `CLOUD_BUILD_ENV`, base64
of `export NAME='value'` lines. The template's Dockerfile runs
`eval "$(printf '%s' "$CLOUD_BUILD_ENV" | base64 -d)"` before its build.
A tree with a link that resolves outside it is refused (`build_failed`;
the sealed log names the link). Without a `Dockerfile`, `railpack prepare`
runs in a container (`alpine` by digest, read-only, no capabilities, as the
runner's user) that mounts only the tree read-only, Railpack and an empty
plan directory, so nothing of the host is in reach; it has the network,
which Railpack needs to fetch mise. Railpack gets the env by `--env`, and the build reads
them as BuildKit secrets from mode-600 files under `$RUNNER_TEMP`, deleted
after the build. The image is built and loaded into the runner's Docker
before it signs in to ghcr, so a `FROM` line cannot pull another App's
private image with this repo's token; then it signs in and pushes.

## When a run fails

The failing step prints one line `cloud-failure: <code>`, which Cloud reads
from the job log, and only there, to tell the User what to do next. Every
`cloud-failure:` line is dropped from the output before it is sealed, and
values that this workflow prints itself are never masked, so neither the
User's output nor a mask can change the code. A failed push prints no code:
Cloud then says the builder stopped and to push again.

| Code | Meaning |
|---|---|
| `invalid_input` | the job could not be opened, or a field did not pass the checks above, or the env has a name Cloud leaves out |
| `fetch_failed` | the repo or the commit could not be read |
| `no_build_plan` | no Dockerfile, and Railpack could not tell the language |
| `build_failed` | the build itself failed |
| `timed_out` | a time limit stopped it: 3 minutes for Railpack's plan, 15 for the build, 20 for each job |

## Secrets

| Secret | What |
|---|---|
| `CLOUD_JOB_KEY` | the builder's age identity (`AGE-SECRET-KEY-1...`). Cloud holds only its recipient (`BUILD_JOB_RECIPIENT`) |
| `FORGEJO_READ_TOKEN` | reads repos on forgejo.viktorbarzin.me; scope `read:repository` only, never an admin scope |
| `GH_OWNER_READ_TOKEN` | reads the owner's private github.com repos |

Cloud dispatches with its own token (Vault `secret/cloud/system`, key
`github_dispatch_token`), which needs Actions read and write on this repo.
It also reads this repo's visibility with that token: while the repo is
public its minutes are free, and Cloud does not stop builds when the
account's private minutes run low.
Cloud's log identity is `build_log_key` in the same Vault path; it never
comes here.

## Settings

`scripts/publish-builder.sh --settings` sets them: only actions pinned by
sha, GitHub's own and `docker/setup-buildx-action`; a read-only default
`GITHUB_TOKEN`; while private, no workflows from fork pull requests; once
public, approval for every outside contributor. The workflow starts on
`workflow_dispatch` only. The workflow's first comment carries
`cloud-builder-format: 2`; `scripts/publish-builder.sh --delete-old-runs
--yes` deletes every run of the first, plain workflow, whose logs showed
repos, commits, images and build output. `scripts/publish-builder.sh --check-run <id>`
reads a finished run and fails if any of this does not hold, a run of the
plain workflow is left, or any given
text shows in its logs or artifacts.
