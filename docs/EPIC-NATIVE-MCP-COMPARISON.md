# BlueprintMCP vs. Epic's native UE 5.8 MCP

Measured against the installed **UE 5.8.1** engine on 2026-07-30, by enumerating declarations in
`Engine/Plugins/Experimental/`. Not from documentation or marketing — from the shipped source tree.

---

## 1. What Epic actually ships in 5.8

UE 5.8 ships a first-party MCP server. It is **experimental and disabled by default**, so most users
will never have seen it.

| Plugin | Role |
|---|---|
| `ModelContextProtocol` | The MCP server itself — HTTP, spec-compliant |
| `ToolsetRegistry` | Registration/reflection layer that turns engine functions into tools |
| `AIAssistant` | In-editor chat UI that consumes the toolsets |
| `MCPClientToolset` | Lets the editor act as an MCP *client* to other servers |
| 26 × `*Toolset` plugins | The actual capabilities |

**Server details** (`ModelContextProtocol.h`, `ModelContextProtocolEditor.cpp`):

- HTTP on **port 8000**, path **`/mcp`** — no collision with our 9847.
- Implements the MCP spec directly (`2025-06-18`, referencing `2025-11-25` for tool naming).
- Auto-start is governed by `UModelContextProtocolSettings`.
- Tool names are namespaced `toolset.tool_name`, split at the last dot.
- `IModelContextProtocolTool` is a **public interface** — third-party code can register tools into
  Epic's server.
- There is a `bEnableToolSearch` setting and a `SemanticSearchToolset`, plus a tool-hash-mapping
  commandlet. Epic needed *semantic search over their own tools* — see §5, this matters.

## 2. How tools are declared

Two mechanisms, both feeding `ToolsetRegistry`:

**Python** — `@toolset_registry.tool_call` on static methods of an `unreal.ToolsetDefinition`:

```python
@unreal.uclass()
class ActorTools(unreal.ToolsetDefinition):
    @toolset_registry.tool_call
    @staticmethod
    def get_label(actor: unreal.Actor) -> str:
        """Returns the actor's human friendly name as it appears in the editor."""
        return actor.get_actor_label()
```

**C++** — `UFUNCTION(meta = (AICallable))` on statics of a `UToolsetDefinition`:

```cpp
UFUNCTION(meta = (AICallable), Category = "PCG|Graph")
static UPCGExecuteGraphInstanceAsyncResult* RunPCGInstantGraph(UPCGGraph* Graph, ...);
```

Schema, description and parameter types are **derived by reflection** from the signature and
docstring. This is the single biggest architectural difference from us — see §4.

## 3. Coverage counts

| | Tools |
|---|---|
| Epic — Python (`@tool_call`) | **596** |
| Epic — C++ (`AICallable`) | **294** |
| **Epic total** | **~890** |
| **BlueprintMCP** | **252** |

Epic's largest toolsets:

| Toolset | Tools | Notes |
|---|---|---|
| AnimationAssistantToolset | 319 py | Sequencer, Control Rig, anim layers, keyframing, FBX |
| EditorToolset | 224 py + 25 c++ | actor, asset, blueprint, material, mesh, texture, tables |
| NiagaraToolsets | 56 c++ | |
| PCGToolset | 31 c++ | |
| UMGToolSet | 23 c++ | |
| DataflowAgent | 22 c++ | |
| SequencerAnimMixerToolset | 21 py | |
| PhysicsToolsets / PluginToolset | 17 each | |
| ToolsetRegistry | 16 c++ | meta-tools |
| GASToolsets / SlateInspectorToolset | 14 each | |
| MVVMToolset, StateTree, MetaHumanGenerator, ConfigSettings, AutomationTest, DataRegistry, GameFeatures, AIModule (BehaviorTree), Conversation, GameplayTags, ChaosCloth, SemanticSearch, WorldConditions, LiveCoding | 1–9 each | |

**Raw name overlap is only 30 exact matches — ignore that number.** It is an artefact of namespacing:
Epic has `BlueprintTools.create`, we have `create_blueprint`. Concept overlap is far higher than name
overlap. Compare capabilities, not identifiers.

## 4. Where each side is genuinely ahead

### Epic covers, we have nothing (verified: zero matching tools on our side)

Ranked by how much work they'd be to replicate and how often they'd matter:

1. **Sequencer / cinematics — the biggest gap by a wide margin.** ~319 tools: tracks, bindings,
   sections, keyframing, curve editor, outliner, marked frames, playback, spawnables/possessables,
   sub-sequences, FBX import/export. We have *zero*.
2. **Control Rig** — full rig graph authoring, bones/controls/nulls, forward+backward solve graphs,
   space switching, anim layers, tweening, snapping.
3. **Gameplay Ability System** — attribute sets, gameplay cues, ability system inspection.
4. **AI authoring** — BehaviorTree and StateTree inspection/editing.
5. **Physics assets and Chaos Cloth.**
6. **Asset-type tools we lack** — texture, static mesh, skeletal mesh, primitives, string tables.
7. **Project-level tooling** — GameplayTags, DataRegistry, GameFeatures, plugin management, config
   settings, Live Coding trigger, automation-test execution.
8. **MVVM view-model binding and Slate runtime inspection.**
9. **A Blueprint graph text DSL** — `read_graph_dsl` / `write_graph_dsl` / `get_graph_dsl_docs`.
   Round-trips a whole graph as text. Genuinely clever, and closer to how an LLM wants to work than
   node-at-a-time calls. Our `build_graph` is batch-create, not round-trip.
10. **Blueprint details we lack** — `add_node_pin`/`remove_node_pin` (dynamic pins),
    `retarget_node_class`, `get_connected_subgraph`, `set_variable_replication`,
    `set_variable_instance_editable`, `add_component_bound_event`, `get_default_object` (CDO).

### We cover, Epic does not

- **Standalone/headless operation.** Epic's server lives in an open editor. We also run as a
  commandlet with no editor — that is what makes CI and scripted use possible.
- **Graph snapshot / diff / restore** (`snapshot_graph`, `diff_graph`, `restore_graph`,
  `analyze_rebuild_impact`, `find_disconnected_pins`) — built after a real data-loss incident.
- **Material graph editing at expression level** — Epic has material *tools*; we have
  add/connect/disconnect/move on the expression graph plus snapshot/diff/restore of it.
- **Anim Blueprint state machine authoring** — states, transitions, transition rules, blend spaces.
- **Groom binding management.**
- **Lighting validation and post-process configuration.**
- **Transactions** (`begin_transaction`/`end_transaction`) and undo/redo control.
- **Riot Crowd** — bespoke, not comparable.
- Note: Epic *does* have capture tools (`CaptureViewport`, `CaptureAssetImage`, `CaptureEditorImage`,
  `Screenshot` in EditorToolset/DataflowAgent), so our vision layer is **not** unique. Ours renders
  from an arbitrary camera in any world; theirs is viewport/asset-centric. Overlapping, not identical.

## 5. On "just add their coverage into ours"

Three problems with the direct approach, then three better options.

**Problem 1 — licensing.** These plugins are Unreal Engine source under the **UE EULA, not an open
licence**, and 14 of the 27 are explicitly `"NoRedist": true` (including `ToolsetRegistry` itself).
Copying their implementations into a public repo is not available to us. Clean-room reimplementation
from the public API surface is fine; copying code is not.

**Problem 2 — volume.** ~640 net-new tools. Our architecture needs, per tool, a C++ handler + route +
dispatch entry + a TypeScript `server.tool` + tests. Epic gets theirs free by reflection. Hand-writing
640 of those is not a realistic amount of work, and it would roughly quadruple the maintenance surface.

**Problem 3 — it may make the server worse.** 252 + 640 ≈ 890 tools all listed at once would swamp a
client's context. **Epic hit this themselves** — that is why 5.8 ships `bEnableToolSearch`, a
`SemanticSearchToolset` and a tool-hash-mapping commandlet. Adding everything without solving
discovery reproduces the problem they had to build extra machinery to escape.

### Better options

**A. Run both servers.** Zero engineering. Epic's is HTTP MCP on `localhost:8000/mcp`, ours is on
9847; no conflict. Add both to `.mcp.json` and the client gets the union today. This is the honest
first move, and it also tells us empirically which of Epic's tools actually get used before we invest
in reimplementing any of them.

**B. Proxy/aggregate.** Our TypeScript server connects to Epic's endpoint, pulls its tool list, and
re-exposes it namespaced (`epic.sequencer.play`). Hundreds of lines, not hundreds of tools. Ships no
Epic code — it calls what is already installed on the user's machine, so the EULA problem disappears.
Also lets us filter *which* of their 890 we surface, which addresses Problem 3.

**C. Reimplement selectively.** Pick the gaps that matter for the work actually being done here and
build those properly. On current evidence that is **Sequencer first** (largest gap, and directly
relevant to cinematic work), then the **Blueprint graph DSL** idea, then **dynamic node pins** and
**CDO access** — the last two are small and close gaps in our existing core rather than opening new
domains.

Note options A/B depend on Epic's plugins being enabled, and they are experimental and off by
default. That is a real caveat, not a footnote: an experimental plugin's API can change between
engine versions without deprecation.

## 6. Reproducing these numbers

Python tools:

```
grep -rn "@toolset_registry.tool_call" "C:/Program Files/Epic Games/UE_5.8/Engine/Plugins/Experimental/Toolsets"
```

C++ tools:

```
grep -rnE "UFUNCTION\(.*AICallable" "C:/Program Files/Epic Games/UE_5.8/Engine/Plugins"
```

Ours:

```
grep -rhoE 'server\.tool\(\s*"[a-z_]+"' Tools/src/tools/
```
