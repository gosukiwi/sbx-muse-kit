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
you quit, so idle VMs don't hold resources:

```bash
# musex: sandboxed `muse --yolo` scoped to $PWD (one VM per directory); halts the sandbox when you quit
musex() {
  local name="muse-$(basename "$PWD")"
  sbx run --kit "$MUSE_KIT" muse --name "$name"

  echo "Stopping '$name' sandbox..."
  sbx stop "$name"
}
```

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
  `lookaside.facebook.com` (all `:443`). If Muse hits a blocked host, watch
  `sbx policy log` and extend `permissions.network.allow` in the kit.
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
