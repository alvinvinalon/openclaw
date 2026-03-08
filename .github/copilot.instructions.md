# OpenClaw Codebase Patterns

## User Safety Mandate (Personal Desktop / Hobby Use)

- This repository is being used by a single trusted operator on a personal desktop for hobby testing.
- Before proposing, executing, or asking the user to execute any command that can change system state, network exposure, credentials, files, services, or security posture, provide a detailed explanation first.
- That explanation must include: what the command does, why it is needed, potential risks, and a safer/minimal alternative when possible.
- Require explicit user confirmation before proceeding with potentially unsafe or destructive actions.
- If a command appears unsafe, over-privileged, or unnecessary for the stated goal, do not proceed; suggest a safer approach.
- Default to least privilege and minimal scope changes.

**Always reuse existing code - no redundancy!**

## Tech Stack

- **Runtime**: Node 22+ (Bun also supported for dev/scripts)
- **Language**: TypeScript (ESM, strict mode)
- **Package Manager**: pnpm (keep `pnpm-lock.yaml` in sync)
- **Lint/Format**: Oxlint, Oxfmt (`pnpm check`)
- **Tests**: Vitest with V8 coverage
- **CLI Framework**: Commander + clack/prompts
- **Build**: tsdown (outputs to `dist/`)

## Anti-Redundancy Rules

- Avoid files that just re-export from another file. Import directly from the original source.
- If a function already exists, import it - do NOT create a duplicate in another file.
- Before creating any formatter, utility, or helper, search for existing implementations first.

## Source of Truth Locations

### Formatting Utilities (`src/infra/`)

- **Time formatting**: `src\infra\format-time`

**NEVER create local `formatAge`, `formatDuration`, `formatElapsedTime` functions - import from centralized modules.**

### Terminal Output (`src/terminal/`)

- Tables: `src/terminal/table.ts` (`renderTable`)
- Themes/colors: `src/terminal/theme.ts` (`theme.success`, `theme.muted`, etc.)
- Progress: `src/cli/progress.ts` (spinners, progress bars)

### CLI Patterns

- CLI option wiring: `src/cli/`
- Commands: `src/commands/`
- Dependency injection via `createDefaultDeps`

## Import Conventions

- Use `.js` extension for cross-package imports (ESM)
- Direct imports only - no re-export wrapper files
- Types: `import type { X }` for type-only imports

## Code Quality

- TypeScript (ESM), strict typing, avoid `any`
- Keep files under ~700 LOC - extract helpers when larger
- Colocated tests: `*.test.ts` next to source files
- Run `pnpm check` before commits (lint + format)
- Run `pnpm tsgo` for type checking

## Stack & Commands

- **Package manager**: pnpm (`pnpm install`)
- **Dev**: `pnpm openclaw ...` or `pnpm dev`
- **Type-check**: `pnpm tsgo`
- **Lint/format**: `pnpm check`
- **Tests**: `pnpm test`
- **Build**: `pnpm build`

If you are coding together with a human, do NOT use scripts/committer, but git directly and run the above commands manually to ensure quality.
