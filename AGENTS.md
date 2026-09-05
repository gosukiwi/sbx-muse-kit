# AGENTS.md — sbx-muse-kit

Read this file before you change files in this repo. README.md is for
humans. This file is for agents.

## Layout

- `muse-kit/spec.yaml`: the full kit. Sandbox image, entrypoint,
  network rules, install steps, startup steps.
- `README.md`: human docs. Keep it brief. Use Simplified Technical
  English (ASD-STE100): short sentences, one command per sentence, no
  jargon.
- No tests exist in this repo. No build exists. No CI exists.

## Rules

- Keep the published kit generic. No secrets. No user names. No host
  paths. No project items.
- Host items live outside this repo: `musex()` in `~/.zshrc`, and
  per-project files in `~/.config/musex/<project>.sh`.
- Mounts are run-time items. Do not put mounts in the kit spec.
- The kit reads `$MUSEX_SKILLS_SRC` from the wrapper. Do not hard-code
  a skills path. Startup steps must tolerate a missing variable.
- Validate after each edit from the repo root:
  `sbx kit validate ./muse-kit`.
- Kit changes apply at sandbox creation only. Never recreate, stop, or
  delete a live sandbox without explicit user approval. Recreation
  deletes the in-sandbox `muse login`.
- Do not add per-project files to project repos.

## Before you push

- `sbx kit validate ./muse-kit` must pass on the local copy.
- `sbx kit validate` must pass on the published git URL after push.
