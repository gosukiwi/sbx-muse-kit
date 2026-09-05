# Muse Code in Docker Sandboxes

Muse Code is not a built-in `sbx` agent. This repo defines it as a
[sandbox kit](https://docs.docker.com/ai/sandboxes/customize):
[`muse-kit/spec.yaml`](muse-kit/spec.yaml).
Verified with `sbx v0.39.0` on macOS arm64.

Agents: read [AGENTS.md](AGENTS.md) before you change files in this repo.

## Prerequisites

- Install sbx and start the daemon:
  `brew install docker/tap/sbx` and `sbx daemon start`.
- Sign in with `sbx login`. Then set a global network policy:
  `sbx policy init balanced`.
- One-time step: allow this kit publisher:
  `sbx settings set kit.allowedSources '["docker.io/","github.com/gosukiwi/"]'`.
- No clone needed. sbx fetches the kit from git. Set once in `~/.zshrc`:

```bash
export MUSE_KIT="git+https://github.com/gosukiwi/sbx-muse-kit.git#dir=muse-kit"
```

## Daily use

Add this wrapper to `~/.zshrc`. It starts one sandbox per directory. It
names the sandbox `muse-<directory>`. It stops the sandbox when you quit:

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

  echo "Stopping '$name' sandbox..."
  sbx stop "$name"
}
```

Run `musex` from any project directory. The first run pulls the image
and installs tools. This takes one or two minutes. Then sign in inside
the sandbox with `muse login`. Approve the code in your browser. The
token stays in that sandbox. Each sandbox needs its own login. Only
`sbx rm` deletes a sandbox and its login.

## Configuration: general and per-project

Two layers exist. The published kit holds generic items only. Host items
live outside this repo:

- General defaults live in `musex`. Each project gets them: a
  read-only `~/.agents` mount for user skills, and `GH_TOKEN` from host
  auth.
- Per-project flags live in `~/.config/musex/<project>.sh`. The file
  name must match the directory name. The wrapper sources the file. The
  file appends flags to `args`. Example for a project that serves web
  ports:

```bash
args+=(-p 5173:5173 -p 4000:4000)
```

Project repos stay clean. No config files go in the repos.

`-p` works only at sandbox creation. For a sandbox that exists, publish
ports without recreation. The login stays:

```console
$ sbx ports muse-my-project --publish 5173:5173 --publish 4000:4000
```

Publish ports only for host-to-sandbox traffic. Example: a Mac browser
that opens sandbox services. If you develop on the host and the sandbox
only runs tests, omit `-p`. Each side has its own `127.0.0.1`.

On re-attach, sbx rejects workspace flags. The wrapper therefore sends
kit and workspaces only at creation. `-e` flags go in both cases. They
apply to the agent session also on re-attach.

## User skills

Muse reads user skills from `$HOME/.agents/skills`. In the sandbox,
`HOME` is `/home/agent`. The wrapper mounts host `~/.agents` read-only,
but the mount lands at the host path. Muse does not see it. A link does
not work: Muse resolves it outside `HOME` and rejects each skill.

The kit copies real files at each start. It copies from
`$MUSEX_SKILLS_SRC/skills` to `/home/agent/.agents/skills`. The wrapper
sets `MUSEX_SKILLS_SRC`. The copy is a cache. Manage skills on the host.
Check from inside the sandbox:

```console
$ muse skills list --source user --enabled-only
```

Project skills are a separate scope. Muse skips them until you trust the
workspace. Use `--trust-workspace` for one run.

## GitHub CLI

The kit installs `gh` at creation. It allows `github.com:443` and
`api.github.com:443`. Two auth options exist:

1. Run `gh auth login` in the sandbox. The login stays in that sandbox.
2. Reuse host auth. The token needs `repo` scope. The wrapper does this
   form:

```console
$ sbx run --kit "$MUSE_KIT" muse --name my-sandbox \
    -e GH_TOKEN=$(gh auth token)
```

If `gh` hits a blocked host, run `sbx policy log <sandbox>`. Add the
host to `permissions.network.allow` in the kit.

## Network

The kit allows Meta hosts and GitHub hosts, all on port 443. No
organization governance is active here. The sandbox can therefore reach
the union of kit rules and global `sbx policy` rules. Deny wins on
conflict. All else is blocked.

`sbx policy init balanced` sets 193 global rules. They cover AI
services, package managers, code hosts, cloud hosts, OS packages, and
certificates. That is why npm, Maven, and GitHub work from the sandbox
although the kit does not list them. Do not loosen or tighten layers as
a side change. Edit allow-lists with this union in mind.

## YOLO mode

`--yolo` is the kit default. The entrypoint runs `muse --yolo`. No
approval prompts appear. Muse sandboxing stays off. The sbx microVM
isolation still applies.

To restore approvals, remove the flag in `muse-kit/spec.yaml`. Then
re-validate and recreate the sandbox. Run args after `--` only append.
They cannot unset a baked-in flag.

## Change the kit

Clone the repo to change the kit. Validate before you commit:

```console
$ sbx kit validate ./muse-kit
```

Validate the published kit without a clone:

```console
$ sbx kit validate "git+https://github.com/gosukiwi/sbx-muse-kit.git#dir=muse-kit"
```

Kit changes apply at sandbox creation only. Recreate the sandbox to use
them. Recreation deletes the in-sandbox `muse login`.
