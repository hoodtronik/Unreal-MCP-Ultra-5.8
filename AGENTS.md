# AGENTS.md — cross-agent conventions for `ue5-mcp` / BlueprintMCP

This file is the cross-tool standard read by Cursor, Codex, Gemini CLI, and most other coding
agents. It complements (does not replace) `CLAUDE.md`.

## CLAUDE-NOTE convention — source of truth

Changes authored by Claude Code are marked inline with `CLAUDE-NOTE:` (or the language's comment
equivalent, e.g. `# CLAUDE-NOTE:`, `// CLAUDE-NOTE:`, `<!-- CLAUDE-NOTE: -->`). These notes record
*why* non-obvious code is the way it is and are the **source of truth**.

**Non-Claude-Code agents must not modify, remove, or alter any code marked with a `CLAUDE-NOTE:`
without first alerting the user and receiving explicit permission.** If you encounter a
`CLAUDE-NOTE:` on code you want to change, stop and ask the user first.

## Project shape

- `Tools/` — TypeScript MCP server (stdio). Build: `cd Tools && npm run build`. Tests:
  `npm test` (Vitest; spawns a real headless UE editor, so UE 5.8 must be installed).
- `Source/BlueprintMCP/` — the compiled C++ UE5 **editor** plugin that does the actual work.
  TypeScript calls it over HTTP on `localhost:9847`.

The two halves are separate programs. A change to a tool's behaviour usually needs edits on both
sides: a handler in `Source/`, and a `server.tool(...)` definition in `Tools/src/tools/`.

## Engine version

**This repo is the UE 5.8 branch of BlueprintMCP.** It targets **UE 5.8.1** and is the only version
it is built against. For UE 5.6.1 use the original repo:
[hoodtronik/Unreal-MCP-Ultra](https://github.com/hoodtronik/Unreal-MCP-Ultra).

The two repos are otherwise near-identical. The 5.8 port is a small set of API-drift fixes, each
marked with a `CLAUDE-NOTE:` that names the 5.6 form — do not "fix" them back:

| What changed in 5.8 | Fix applied |
|---|---|
| `Engine/UserDefinedStruct.h` forwarding header removed | include `StructUtils/UserDefinedStruct.h` (also valid on 5.6) |
| `UMaterial::GetMaterialResource()` reverted to `EShaderPlatform` (5.6 briefly used `ERHIFeatureLevel::Type`) | pass `GMaxRHIShaderPlatform` — genuinely version-specific |
| `FJsonObject::Values` re-keyed `FString` → `UE::FSharedString` | rebuild the key via `FString(*Pair.Key)` (compiles on both) |
| Mass split into `Runtime/Mass/{MassCore,MassEngine,…}`; base element types moved to `MassCore` | RiotCrowd's `Build.cs` adds `MassCore` (does not exist on 5.6) |

The `.uplugin` deliberately declares no `EngineVersion` field, so the editor will not refuse to load
on a different engine — the constraint is documentation, not enforcement.

## Adding a tool — both sides required

1. **C++**: implement `HandleX` in a `BlueprintMCPHandlers_*.cpp`, declare it in
   `BlueprintMCPServer.h`, **bind the route** in `Start()`, and **add the dispatch entry** in
   `RegisterHandlers()`.
2. **TypeScript**: add the `server.tool(...)` definition in `Tools/src/tools/`.
3. **Tests**: add integration tests in `Tools/test/tools/`.

**Steps 1's last two items are the ones people forget.** A handler that is implemented and declared
but never routed produces a tool that is advertised to the agent and 404s when called. As of
2026-07-22 there are ~78 such orphan handlers in this repo (`spawn_actor`, `undo`/`redo`, viewport,
cvars, widgets, PIE runtime, and more) — implemented, declared, unroutable. Check for an existing
handler before writing a new one.

## Gotchas worth knowing before you debug something for an hour

- **`npx tsc --noEmit` does not typecheck tests.** `tsconfig.json` is `include: ['src/**/*']`. Test
  and harness code is validated only by actually running Vitest.
- **The test suite baseline is not zero failures.** As of 2026-07-22: 41 files / 402 tests pass, 18
  files / 110 tests fail. Nearly all failures are the orphan-handler problem above, plus one
  test-side bug (`batch-set-pin-default` reads `pin.default`; the C++ emits `defaultValue`).
  **Compare against this baseline before blaming your change.**
- **`SaveBlueprintPackage` compiles the Blueprint as part of saving.** Do not add a separate compile
  step around it, and be aware that per-node saving is the dominant cost in batch operations.
- **New routes need a full editor restart.** Live Coding does not re-run `Start()` /
  `RegisterHandlers()`. New TS tools need an MCP client reconnect to appear.
- **Author-written tests confirm the author's model, not reality.** For engine-facing work, drive it
  in a live editor (`localhost:9847` with the editor open) before trusting green tests.
