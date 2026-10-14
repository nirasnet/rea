# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Repository invariants (product authority, architecture rules, performance priorities, generated-file ownership, commit conventions) live in AGENTS.md and apply in full:

@AGENTS.md

## What REA is

`rea-agents` is a local reverse-engineering tool exposed as both a one-shot CLI and an MCP stdio server. Both adapters drive the same application workflows. Agents call it to inspect binaries, apps, browser/Electron runtimes, captures, firmware, APKs, and so on. Results come back with Evidence provenance and explicit unknowns. Deep engines (Hopper, Ghidra, IDA, JADX, Frida, and others) are external and bring-your-own. REA supervises them as owned processes.

## Commands

The toolchain is pinned in `.nvmrc` (Node) and `package.json#packageManager` (npm). Build, check and generation tasks run through Turbo, and `scripts/run-exclusive.mjs` serializes writers with checkout-local SQLite locks in `.cache/rea-command-locks/`. Never delete those lock files while commands are running.

```bash
npm ci                                   # install; also sets up Husky hooks (pre-commit: format+lint staged; pre-push: check:fast)
npm run build:cached                     # compile runtime to dist/ + generate packaged skill (npm run build forces rerun)
npm run check:fast                       # cached typecheck + lint (lint includes verify:module-boundaries)
npm run check                            # pre-review gate: typecheck, lint, format:check, knip, metadata:check, architecture guards
npm run format                           # oxfmt (linter is oxlint)

npm run test:local -- src/config.test.ts            # run one source test without building
npm run test:focused -- tests/acceptance/x.test.ts  # exact files; builds first when a compiled boundary needs it
npm run test:changed                     # tests affected since merge base with origin/main (import graph; may miss bridges/registration)
npm run test:fast                        # all source-level projects, no build
npm run test:boundary | test:mcp | test:acceptance
npm run docs:check                       # when contracts, generated metadata, or README onboarding change
npm run verify:test-discovery            # after adding or moving test files
npm run metadata:generate                # after package, lockfile, or managed-skill version changes
```

After a package/lockfile change, run `npm run metadata:generate` before building. Packaging/setup changes also need `npm run verify:package` and `npm pack --dry-run`. Provider changes need the matching real-provider `verify:*` lane, which requires the engine installed on the host (see `docs/testing.md#real-toolchain-verification-lanes`). Simulated providers do not establish real behavior. `REA_LOG_LEVEL` controls stderr logging. stdout carries only MCP protocol messages or command results.

### Test layout (Vitest projects in `vitest.config.ts`)

- `domain`: `src/{contracts,domain}/**/*.test.ts`
- `services`: `src/application/**/*.test.ts`
- `adapters`: `src/*.test.ts` and provider directories under `src/`
- `composition`, `conformance`, `evaluation`: `tests/<name>/`
- `boundary`, `cli-boundary`, `mcp-boundary`, `process-boundary`: `tests/boundary/{,cli,mcp,process}/`. These use the compiled runtime. `process-boundary` runs files serially.
- `acceptance`: `tests/acceptance/`. Complete compiled CLI/MCP workflows.
- `process-global`: `tests/process-global/`. Per-file isolation; CLI module mocks belong here.

Use `createTestTempDirectory` from `tests/fixtures/temporaryDirectory.ts` for temp files. Never glob-clean `/tmp/rea-*`. Fake launchers, bridges and CDP seams live in `tests/fixtures/`.

## Architecture

`docs/architecture.mermaid` is the authoritative map. Dependencies point inward. `scripts/verify-module-boundaries.mjs` (run by lint) and `scripts/verify/architecture-guards.mjs` enforce this.

- **Entry points.** `scripts/rea.mjs` is the bin dispatcher: `rea mcp` starts the server, and anything else runs the one-shot CLI. `src/main.ts` + `src/main/` handle MCP config, session and process lifetime. `src/cli.ts` + `src/cli/` (plus the `src/cli*.ts` helpers) are the CLI adapter.
- **`src/server/`**: MCP request translation. `createServer.ts` plus one `register*Tools.ts` per capability, and `contractToolHandler.ts` binds handlers to named contracts.
- **`src/composition/`**: production factories, one per capability. Each builds fresh typed providers without starting engines or acquiring targets. Optional adapters load independently, so one failing adapter must not take down its peers.
- **`src/application/`**: shared CLI/MCP workflows (analysis, setup/update/doctor, investigations, web/JS services, reconstruction ledgers). Key pieces:
  - `SessionProviderRouter` / `BinarySession`: one immutable deep-provider binding per target.
  - `AnalysisProviderRegistry`: deterministic provider selection with no fallback.
  - `src/application/investigation/`: InvestigationRecords and EvidenceLedger ownership.
  - Ports such as `*Port.ts` are implemented by provider layers.
- **`src/contracts/`**: caller-visible Zod schemas for every tool, plus shared error schemas (`errorSchemas.ts` → generated `docs/error-contract.schema.json`).
- **`src/domain/`**: pure, side-effect-free semantics (evidence, comparisons, application graphs).
- **Provider layers**, one directory each: `hopper/`, `ghidra/`, `ida/`, `android/`, `browser/`, `inspector/`, `javascript/`, `dotnet/`, `firmware/`, `wasm/`, `evm/`, `native/`, `artifacts/`, `pcap/`, `sqlite/`, `frida/`, and others. Engine protocols stay here.
- **In-engine bridge code** lives in `bridge/` and is shipped in the package: the Hopper Python bridge, Ghidra headless Java scripts, the JADX bridge, the LLDB tracer, and Swift helpers. The Windows Node-API addon lives in `native/windows/`.
- **`src/process/`**: owned process groups, private roots, deadlines, bounded reads, and cleanup. All provider supervision reuses it. `src/windows/` is the Windows host boundary.
- **`src/config/`**: Zod-validated environment parsing.
- **`src/evaluation/`**: agent release evaluation and transcript scoring.

### Generated vs authored

- Edit the agent skill in `.agents/skills/reverse-engineer-anything/`. `skills/` is generated output.
- `src/generatedPackageMetadata.ts` and `docs/error-contract.schema.json` are tracked but generated. Regenerate them rather than hand-editing.
- `docs/public/product-catalog.json`, `docs/verification/managed-conformance-*.json`, and `.cache/mcp-tool-catalog.json` are ignored outputs, produced by `test:prepare` / `docs:generate`.
- Docs are a VitePress site (`docs/`, navigation in `docs/.vitepress/config.ts`). The public site is `website/`.
- When adding or changing an MCP tool, follow `docs/tool-design.md` and refresh the catalog with `npm run mcp-catalog:generate`.
