# container-checkout-private-submodules

[![Lint](https://github.com/bduncanltd/container-checkout-private-submodules/actions/workflows/lint.yml/badge.svg)](https://github.com/bduncanltd/container-checkout-private-submodules/actions/workflows/lint.yml)

A GitHub composite action that checks out a repository and initialises private submodules via SSH inside a GitHub Actions container job.

## The problem

When a GitHub Actions job runs inside a container (using `container:` in the workflow), the standard approach of using [`actions/checkout`](https://github.com/actions/checkout) with `submodules: recursive` does not work for private submodules in other repositories.
This is because `actions/checkout` uses the workflow's `GITHUB_TOKEN`, which only has access to the repository the workflow is running in.

Using [`webfactory/ssh-agent`](https://github.com/webfactory/ssh-agent) to load a deploy key also fails inside container jobs, because the SSH agent socket is created on the host runner under `/tmp`, which is not mounted into the container.

This action solves both problems.

## How it works

1. Forces [`webfactory/ssh-agent`](https://github.com/webfactory/ssh-agent) to place its socket in a directory that GitHub Actions always mounts into containers (`/home/runner/work/_temp/_github_workflow` on the host = `/github/workflow`
inside the container).
2. Runs `ssh-keyscan` to add GitHub's host key to `known_hosts`. Inside container jobs `HOME=/github/home`, so the file ends up in a non-standard location. [`GIT_SSH_COMMAND`](https://git-scm.com/docs/git-config#Documentation/git-config.txt-coresshCommand) is used to point SSH at the right file.
3. Checks out the repository using [`actions/checkout`](https://github.com/actions/checkout).
4. Configures git to [trust the workspace directory](https://git-scm.com/docs/git-config#Documentation/git-config.txt-safedirectory) regardless of ownership, as the workspace is created by the host runner user and may not match the user inside the container.
5. Initialises submodules using the SSH agent.

## Requirements

1. GitHub-hosted Linux runner (`ubuntu-latest`)
2. Container job (`container:` defined at the job level)
3. Submodules hosted on GitHub
4. An SSH deploy key with read access to all submodule repositories, stored as a repository secret

## Usage

```yml
jobs:
  my-job:
    runs-on: ubuntu-latest
    container:
      image: my-image:latest
    steps:
      - uses: bduncanltd/container-checkout-private-submodules@v1
        with:
          ssh-private-key: ${{ secrets.SSH_DEPLOY_KEY }}
          lfs: true # optional, defaults to false
      - run: # ... rest of your job
```

### Inputs

| Input | Required | Description |
| --- | --- | --- |
| `ssh-private-key` | Yes | SSH private key with read access to submodule repos. |
| `lfs` | No | Whether to download Git LFS files. Defaults to `false`. |

## Security

The `ssh-private-key` input is passed directly to `webfactory/ssh-agent`, which loads it into memory and never writes it to disk.
The public key fingerprint is suppressed from logs via `log-public-key: false`.

## Future improvements

- The `git submodule update --init` command is not configurable. There is no way to pass extra flags such as `--recursive` or `--depth`.
- Only GitHub is supported as a submodule host. Supporting other providers (e.g. GitLab, Bitbucket) would require `ssh-keyscan` to be run against additional hosts.
- Only tested on `ubuntu-latest`. Behaviour on other GitHub-hosted Linux runners is unknown.
- Does not support self-hosted runners, as the socket and `known_hosts` workarounds rely on mount paths specific to GitHub-hosted runners (`/home/runner/work/_temp/_github_workflow`, `/github/workflow`, `/github/home`).
