# Muse Code in Docker Sandboxes

Muse Code isn't a built-in `sbx` agent, so this workspace defines it as a
[sandbox kit](https://docs.docker.com/ai/sandboxes/customize): [`muse-kit/spec.yaml`](muse-kit/spec.yaml).
Verified end-to-end with `sbx v0.39.0` on macOS arm64.

## Prerequisites

- `brew install docker/tap/sbx` and `sbx daemon start`
- `sbx login` (Docker sign-in) and a global network policy (`sbx policy init balanced`)
- `muse login` on the host (Meta account, for host use)
- One-time: allowlist this kit's publisher (sbx gates remote kits by source):

```console
$ sbx settings set kit.allowedSources '["docker.io/","github.com/gosukiwi/"]'
```

No clone needed — sbx fetches the kit straight from git (verified with
`sbx kit validate`). Define once in `~/.zshrc`:

```bash
export MUSE_KIT="git+https://github.com/gosukiwi/sbx-muse-kit.git#dir=muse-kit"
```

## Run Muse in a sandbox

From this directory:

```console
$ sbx run --kit ./muse-kit muse --name muse
```

First run pulls the image and installs the Muse launcher (a minute or two).
On first launch inside the sandbox, sign in again:

```console
$ muse login
```

Approve the code in your browser. The session token stays in that sandbox.

Re-running the same command reconnects to the existing sandbox, it does not
create a new one. Once named, re-attach from any directory without the kit
flag:

```console
$ sbx run --name muse
```

A wrapper in `~/.zshrc` avoids retyping the kit reference. It names the
sandbox after the current directory (one VM per project) and stops it when
you quit, so idle VMs don't hold resources. Generic defaults every project
gets (read-only `~/.agents` mount for user-level skills, `GH_TOKEN` from
host auth) live in the wrapper; per-project flags live in
`~/.config/musex/<project>.sh`, which the wrapper sources to append to
`args` (e.g. `-p` publishes, extra mounts, `--env-file`). Project repos
stay clean — no per-project files inside the repos themselves:

```bash
# musex: sandboxed `muse --yolo` scoped to $PWD (one VM per directory); halts the sandbox when you quit
musex() {
  local proj="$(basename "$PWD")"
  local name="muse-$proj"
  local args=()
  if sbx ls -q 2>/dev/null | grep -qx "$name"; then
    args=(--name "$name")
  else
    args=(--kit "$MUSE_KIT" muse --name "$name")
    if [ -d "$HOME/.agents" ]; then
      args+=("$PWD" "$HOME/.agents:ro")
    else
      args+=("$PWD")
    fi
  fi
  if [ -d "$HOME/.agents" ]; then
    args+=(-e "MUSEX_SKILLS_SRC=$HOME/.agents")
  fi
  local tok
  tok="$(gh auth token 2>/dev/null)" && args+=(-e "GH_TOKEN=$tok")
  local conf="$HOME/.config/musex/$proj.sh"
  [ -f "$conf" ] && source "$conf"
  sbx run "${args[@]}"

  echo "Stopping 'muse-$proj' sandbox..."
  sbx stop "$name"
}
```

```bash
# ~/.config/musex/listita.sh (per-project)
args+=(-p 5173:5173 -p 4000:4000 -p 8080:8080 -p 9099:9099)
```

`-p` applies only at sandbox creation; for an existing sandbox publish
instead (no recreation needed, login survives). Publishing is only needed
for host-to-sandbox traffic (a Mac browser viewing sandbox-hosted
services): if you develop on the host and the sandbox only runs the agent
and tests against sandbox-local services, omit `-p` entirely — each side's
`127.0.0.1` is its own loopback and the two never conflict:

```console
$ sbx ports muse-listita --publish 5173:5173 --publish 4000:4000 \
    --publish 8080:8080 --publish 9099:9099
```

The `sbx ls -q` branch exists because sbx rejects workspace args on
re-attach (`already exists and can't be given new workspaces`), so kit and
workspaces are sent only at creation. `-e` flags are safe in both paths —
they apply to the agent session even on re-attach.

## User skills

Muse resolves user skills via `$HOME` (`$HOME/.agents/skills`), but inside
the sandbox `HOME` is `/home/agent` while the wrapper's read-only mount
lands at the host absolute path (e.g. `/Users/gosukiwi/.agents`) — so the
mount alone is invisible to Muse. A symlink doesn't work either: Muse
resolves it outside `HOME` and rejects every skill (`list` shows them,
`inspect`/`enable` say "skill not found"). The kit's `setup.startup` step
therefore syncs real files — `cp -r "$MUSEX_SKILLS_SRC/skills/."` into
`/home/agent/.agents/skills/` (clearing a stale symlink first, wiping
removed skills); the wrapper passes `MUSEX_SKILLS_SRC` pointing at the
mount. The kit stays generic (no host paths baked in), the host-specific
path stays in the wrapper, and the copy is refreshed on every start, so it
self-heals after sandbox recreation. Verified with
`muse skills list --source user --enabled-only` from inside the sandbox.
Treat the sandbox copy as a cache — manage skills on the host. (Project
skills, e.g. `listita/.agents/skills`, are a separate scope — Muse skips
them until the workspace is trusted; pass `--trust-workspace` for one
run.)

## GitHub CLI

`gh` is installed at sandbox creation (no-op: the current base image
already ships it) and the kit allow-lists `github.com:443` plus
`api.github.com:443`, covering PR/issue ops and git-over-HTTPS. Two ways
to authenticate:

- Interactive per sandbox: `gh auth login` inside the sandbox.
- Reuse the host's auth (token needs `repo` scope):

```console
$ sbx run --kit "$MUSE_KIT" muse --name muse-listita \
    -e GH_TOKEN=$(gh auth token)
```

The `musex` wrapper above does the second form automatically. If `gh`
hits a blocked host, run `sbx policy log muse-<proj>` to find it and
extend `permissions.network.allow` in the kit.

## YOLO mode

`--yolo` is the kit default: the entrypoint runs `muse --yolo` (no approval
prompts, Muse's own sandboxing off). The sbx microVM isolation still applies.

To get the approval flow back for one session, the kit would need the flag
removed — run args after `--` append, they can't unset a baked-in flag. Edit
`muse-kit/spec.yaml`, re-validate, and recreate the sandbox.

## How it works

- Base image `docker/sandbox-templates:shell-docker`: full Linux env with its
  own Docker daemon, workspace bind-mounted read-write.
- Install step downloads the Muse launcher from `api.meta.ai`; first `muse`
  invocation fetches the Linux binary into the sandbox.
- Network allow-list: `api.meta.ai`, `auth.meta.com`,
  `lookaside.facebook.com`, `github.com`, `api.github.com` (all `:443`).
  If Muse hits a blocked host, watch `sbx policy log` and extend
  `permissions.network.allow` in the kit.
- Effective network policy is the union of kit allows and the global
  local policy when no organization governance is active (verified
  2026-09-05, sbx v0.39.0: `sbx policy init balanced` installs 193 global
  allows across AI services, package managers, code/container hosts,
  cloud infra, OS packages, and cert validation — that is why
  `github.com`, `registry.npmjs.org`, `repo.maven.apache.org`, and
  `storage.googleapis.com` are reachable from inside a sandbox even
  though only the Meta hosts were in the kit; hosts in neither list,
  e.g. `firebase-public.firebaseio.com`, are default-denied). Deny wins
  on conflict. Kit `allow` entries are therefore explicitness for tighter
  profiles, not what currently unblocks those hosts — make allow-list
  edits with that assumption, and don't loosen or tighten either layer
  as a drive-by.
- Auth is interactive device flow per sandbox. Alternative: if you have a
  `META_API_KEY`, pass it with `sbx run --kit ./muse-kit muse -e META_API_KEY`.

## Files

- `muse-kit/spec.yaml` — the kit. Validate the published copy without cloning:

```console
$ sbx kit validate "git+https://github.com/gosukiwi/sbx-muse-kit.git#dir=muse-kit"
```

Clone this repo only to hack on the kit itself (`sbx kit validate ./muse-kit`
from the repo root).

## Multiple projects

Recommended: one sandbox per project. From the project directory:

```console
$ sbx run --kit "$MUSE_KIT" muse --name my-project
```

Each sandbox gets its own workspace mount, agent session, and `muse login`
(one login per sandbox). Re-attach with `sbx run --name my-project`.
(Omit `--name` and sbx defaults to `muse-<workdir>`, which also gives one
sandbox per directory automatically.)

## Stopping and resuming

Detaching leaves the microVM running (`sbx ls` shows `running`), so reattach
is instant — at the cost of the VM's held CPU/memory allocation (this kit
defaults to 6 CPUs / 4 GiB). When you're done for the day:

```console
$ sbx stop muse        # halts the VM, frees resources, keeps state + login
$ sbx run --name muse  # resumes transparently
```

Only `sbx rm` deletes a sandbox (and its in-sandbox `muse login` with it).

Alternative: one sandbox with several workspaces, all fixed at creation:

```console
$ sbx run --kit "$MUSE_KIT" muse \
    --name shared ~/project-a ~/project-b:ro
```

Extra workspaces mount at their absolute host paths inside the sandbox
(append `:ro` for read-only). Downsides: all projects share one agent
session and login, and you can't add a directory later without recreating
the sandbox — so prefer one sandbox per project.
