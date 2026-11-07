# bin

## Overview

`bin/harness` is the Bash CLI that attaches Aerosend repos as git submodules under `repos/` while someone works on them, and detaches them afterwards. It reads every repo's details from `.gitmodules` and holds no repo list of its own.

## Interface

Commands, dispatched from `main` in `harness`:

| Command | Alias | Function | Behaviour |
|---|---|---|---|
| `list` | `ls` | `cmd_list` | Prints repo, category, churn, attached state and stack |
| `attach <repo...>`, `--category <cat>`, `--all` | `add` | `cmd_attach` | Checks repos out on the tip of their branch, then regenerates the workspace file |
| `detach <repo...>`, `--all`, `--category <cat>`, `[--force]` | `rm` | `cmd_detach` | Removes working trees unless work would be lost, then regenerates the workspace file |
| `status` | `st` | `cmd_status` | Prints branch, ahead/behind and changed-file count per attached repo |
| `sync` | | `cmd_sync` | Fetches attached repos and fast-forwards clean default branches |
| `exec <cmd...>` | | `cmd_exec` | Runs a command in each attached repo |
| `workspace` | | `cmd_workspace` | Rewrites `aerosend.code-workspace` |
| `set-remote-base <url>` | | `cmd_set_remote_base` | Rewrites every submodule URL in `.gitmodules` |
| `help`, `-h`, `--help` | | `usage` | Prints usage |

Internal helpers: `all_names`, `meta`, `path_of`, `branch_of`, `is_known`, `is_attached`, `attached_names`, `resolve_targets`, `unsafe_to_detach`.

## Invariants

- Stay compatible with the Bash 3.2 that ships with macOS: no associative arrays, no `mapfile` (file header comment).
- `.gitmodules` is the only manifest. `meta` reads the custom keys `category`, `churn`, `stack` and `description`, which git ignores.
- The script runs under `set -euo pipefail` and always changes to the harness root (`ROOT`) first, so it works from any directory.
- Git commands that clone or update submodules go through the `GIT` array, which sets `protocol.file.allow=always`. Without it, git refuses the local-path dummy remotes.
- A repo counts as attached only when `<path>/.git` exists (`is_attached`).
- `cmd_attach` uses `submodule update --checkout --remote`, which overrides `update = none` for that path and takes the branch tip, not the recorded gitlink. It leaves the repo on a real branch, never a detached HEAD.
- `unsafe_to_detach` blocks a detach on any of: `git status --porcelain` output (untracked files included), commits on no remote, or stash entries. Only `--force` overrides this.
- `cmd_sync` fast-forwards a repo only when it is on its default branch with an empty `git status --porcelain`. Everything else is fetched and left alone.
- `cmd_exec` keeps going when a repo's command fails, and prints that repo's exit code.

## Side Effects

- `attach`: runs `git submodule init` and `update`, which clone from the remote and write to `.git/modules/<name>` and `repos/<name>`. Switches or creates the local branch.
- `detach`: runs `git submodule deinit -f`, which deletes the working tree. The clone stays in `.git/modules/<name>`, so local branches and commits survive a `--force` detach, but uncommitted files do not.
- `attach`, `detach`, `workspace`: overwrite the gitignored `aerosend.code-workspace`.
- `sync`: runs `git fetch` and `git merge --ff-only` inside attached repos.
- `set-remote-base`: edits `.gitmodules` and runs `git submodule sync`. The change has to be committed by hand.
- `exec`: runs arbitrary commands inside attached repos.

## Verification

- No automated tests exist for this script.
- Run: `bash -n bin/harness` to check syntax, then `bin/harness list` and `bin/harness status` as read-only smoke checks.
