# Vibe Kanban

This repository contains the compiled Vibe Kanban distribution. Application source is maintained separately.

## Run

Requires Node.js 22.5 or newer.

```bash
pnpm install --prod
pnpm start
```

The server listens on http://127.0.0.1:8765 by default. Run commands with `pnpm vk -- <command>`. Workspace data is stored in `.vibe-kanban/`.

## Update

Keep the repository's root `.git` directory. Preview and apply updates from this public repository with:

```bash
pnpm workspace:update --dry-run
pnpm workspace:update
```

The updater fetches from `origin` by default. Use `--remote-url <public-repository-url>` to switch an existing workspace to another public repository.
