# Bridge Zero Artifacts: Technical Investigation Report

**Date:** 2026-04-20
**Type:** Technical Investigation (source code depth)
**Models:** Opus 4.6, GPT-5.4, Gemini 3.1 Pro
**Grounding:** Wiki system deep research from 2026-04-19

---

## 1. Executive Summary

The memory-wiki bridge mode reporting zero artifacts is **not a local misconfiguration**. It is an **upstream regression family** with two distinct root causes, confirmed across source code analysis, GitHub issue trails, and changelog history.

**Root Cause #1 (Primary): CLI Loader Gap.** The file `extensions/memory-core/cli-metadata.ts` never calls `registerMemoryCapability()`. All standalone CLI commands (`openclaw wiki bridge import`, `openclaw wiki status`, `openclaw wiki doctor`) load plugins via `loadOpenClawPluginCliRegistry()`, which uses `cli-metadata.ts` as the entry point. This entry point only registers CLI commands - it does not register the memory capability that exposes `publicArtifacts.listArtifacts`. The result: `memoryPluginState.capability` is `undefined`, `listActiveMemoryPublicArtifacts()` returns `[]`, and bridge import sees zero artifacts.

**Root Cause #2 (Secondary): Snapshot Restore Bug.** In `src/plugins/loader.ts`, the non-activating snapshot restore path and error-handler rollback both call `restoreMemoryPluginState()` without including the `capability` field. Since `restoreMemoryPluginState` explicitly sets `memoryPluginState.capability = undefined` when `state.capability` is absent, this wipes any previously registered capability. This is a race-condition risk in the gateway process.

**Critical side effect: Destructive Pruning.** When bridge import sees zero artifacts, it still runs reconciliation with an empty `activeKeys` set. `pruneImportedSourceEntries()` treats ALL existing bridge pages as stale and **deletes them**. Issue #65976 reports `removedCount: 305` from a single false-negative import.

**Correction from prior research:** The 2026-04-19 deep research report hypothesized that function references were lost during "JSON serialization of cached plugin state." This is **incorrect**. The plugin registry cache is an in-memory `Map<string, CachedPluginState>` - no JSON serialization occurs. Function references survive fine within the same process. The cache-hit restore path correctly includes `capability`. The real problems are the CLI loader gap and the snapshot restore omission.

**Current status:** Six upstream issues document this family (#63092, #65698, #65976, #66082, #67190, #67327). One open PR (#67208) proposes routing `wiki bridge import` through the gateway, but does not fix `status` or `doctor`. No released fix exists through v2026.4.15.

---

## 2. Root Cause #1: CLI Loader Gap

### The Architecture Split

OpenClaw has **two distinct plugin loading paths**:

| Path | Function | Entry Point | Context |
|------|----------|-------------|---------|
| **CLI** | `loadOpenClawPluginCliRegistry()` | `cli-metadata.ts` | `openclaw wiki bridge import`, `openclaw wiki status` |
| **Runtime** | `loadOpenClawPlugins()` | `index.ts` | Gateway daemon, agent tools, `openclaw gateway call` |

### The Smoking Gun

**File:** `extensions/memory-core/cli-metadata.ts` (full file, 24 lines)
```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/core";

export default definePluginEntry({
  id: "memory-core",
  name: "Memory (Core)",
  description: "File-backed memory search tools and CLI",
  register(api) {
    api.registerCli(
      async ({ program }) => {
        const { registerMemoryCli } = await import("./src/cli.js");
        registerMemoryCli(program);
      },
      {
        descriptors: [
          {
            name: "memory",
            description: "Search, inspect, and reindex memory files",
            hasSubcommands: true,
          },
        ],
      },
    );
  },
});
```

**Compare with** `extensions/memory-core/index.ts` (runtime mode):
```typescript
export default definePluginEntry({
  id: "memory-core",
  register(api) {
    registerBuiltInMemoryEmbeddingProviders(api);
    registerShortTermPromotionDreaming(api);
    registerDreamingCommand(api);
    api.registerMemoryCapability({        // ← THIS LINE IS MISSING IN CLI-METADATA
      promptBuilder: buildPromptSection,
      flushPlanResolver: buildMemoryFlushPlan,
      runtime: memoryRuntime,
      publicArtifacts: {                  // ← THIS IS THE CRITICAL REGISTRATION
        listArtifacts: listMemoryCorePublicArtifacts,
      },
    });
    // ... tool registrations
  },
});
```

### Why Fixing cli-metadata.ts Alone Won't Work

Even if `cli-metadata.ts` called `api.registerMemoryCapability(...)`, it would be silently swallowed. The CLI loader provides **noop handlers** for all non-CLI registrations:

**File:** `src/plugins/api-builder.ts`, line 103:
```typescript
const noopRegisterMemoryCapability: OpenClawPluginApi["registerMemoryCapability"] = () => {};
```

**File:** `src/plugins/api-builder.ts`, line 169:
```typescript
registerMemoryCapability: handlers.registerMemoryCapability ?? noopRegisterMemoryCapability,
```

**File:** `src/plugins/loader.ts`, line 2639 (CLI loader):
```typescript
const api = buildPluginApi({
  registrationMode: "cli-metadata",
  handlers: {
    registerCli: (registrar, opts) => registerCli(record, registrar, opts),
    // No registerMemoryCapability handler provided → falls through to noop
  },
});
```

### The Call Chain That Fails (CLI)

```
openclaw wiki bridge import
  → loadOpenClawPluginCliRegistry()
    → memory-core/cli-metadata.ts: register(api) { api.registerCli(...) }  // NO registerMemoryCapability
    → memory-wiki/cli-metadata.ts: register(api) { api.registerCli(...) }
  → registerWikiCli(program, config, appConfig)
  → runWikiBridgeImport({ config, appConfig })
  → syncMemoryWikiImportedSources({ config, appConfig })
  → syncMemoryWikiBridgeSources(params)
    → listActiveMemoryPublicArtifacts({ cfg: params.appConfig })   // bridge.ts:224
      → memoryPluginState.capability?.capability.publicArtifacts?.listArtifacts(params)
      → undefined?.capability.publicArtifacts?.listArtifacts(params)
      → undefined
      → ?? []                                                       // Returns empty array!
    → collectBridgeArtifacts(bridgeConfig, [])
    → artifacts = [] → artifactCount = 0
```

### The Call Chain That Works (Gateway)

```
openclaw gateway call wiki.bridge.import
  → gateway process already has loadOpenClawPlugins() result
    → memory-core/index.ts: register(api) {
        api.registerMemoryCapability({
          publicArtifacts: { listArtifacts: listMemoryCorePublicArtifacts }
        })
      }
    → memoryPluginState.capability = populated with function reference
  → wiki.bridge.import gateway method handler
  → syncMemoryWikiImportedSources({ config, appConfig })
  → syncMemoryWikiBridgeSources(params)
    → listActiveMemoryPublicArtifacts({ cfg: params.appConfig })
      → listMemoryCorePublicArtifacts(params)
      → resolveMemoryDreamingWorkspaces(params.cfg) → workspaces
      → collectWorkspaceArtifacts() for each workspace
      → returns [MEMORY.md, daily-notes, dream-reports, event-log]  // WORKS!
```

### Agent Tool Path (Also Works)

Wiki tools (`wiki_status`, `wiki_search`, etc.) execute in the gateway process where the full plugin load already happened. `syncImportedSourcesIfNeeded()` in `tool.ts` (line 77-82) calls the same chain successfully because `memoryPluginState.capability` is populated.

### Why Status/Doctor Are Also Broken

`status.ts` computes bridge artifact count via the same `listActiveMemoryPublicArtifacts()`:

**File:** `extensions/memory-wiki/src/status.ts`, line 219-225:
```typescript
const bridgePublicArtifactCount =
  deps?.appConfig && config.vaultMode === "bridge" && config.bridge.enabled
    ? (
        await (deps.listPublicArtifacts ?? listActiveMemoryPublicArtifacts)({
          cfg: deps.appConfig,
        })
      ).length
    : null;
```

In CLI mode, this always returns 0 → triggers the `bridge-artifacts-missing` warning → even when the gateway import successfully created pages.

---

## 3. Root Cause #2: Snapshot Restore Bug

### The State Capture Omission

**File:** `src/plugins/loader.ts`, line ~2229 (state capture before snapshot load):
```typescript
const previousMemoryFlushPlanResolver = getMemoryFlushPlanResolver();
const previousMemoryPromptBuilder = getMemoryPromptSectionBuilder();
const previousMemoryCorpusSupplements = listMemoryCorpusSupplements();
const previousMemoryPromptSupplements = listMemoryPromptSupplements();
const previousMemoryRuntime = getMemoryRuntime();
// ⚠️ getMemoryCapabilityRegistration() is NOT called!
```

### The Restore That Drops Capability

**File:** `src/plugins/loader.ts`, line ~2240 (non-activating snapshot restore):
```typescript
if (!shouldActivate) {
  restoreRegisteredAgentHarnesses(previousAgentHarnesses);
  restoreRegisteredCompactionProviders(previousCompactionProviders);
  restoreDetachedTaskLifecycleRuntimeRegistration(previousDetachedTaskRuntimeRegistration);
  restoreRegisteredMemoryEmbeddingProviders(previousMemoryEmbeddingProviders);
  restoreMemoryPluginState({
    corpusSupplements: previousMemoryCorpusSupplements,
    promptBuilder: previousMemoryPromptBuilder,
    promptSupplements: previousMemoryPromptSupplements,
    flushPlanResolver: previousMemoryFlushPlanResolver,
    runtime: previousMemoryRuntime,
    // ⚠️ capability is MISSING!
  });
}
```

The same omission appears in the error-handler rollback (line ~2258).

### Why This Wipes Capability

**File:** `src/plugins/memory-state.ts`, line 295-305:
```typescript
export function restoreMemoryPluginState(state: MemoryPluginState): void {
  memoryPluginState.capability = state.capability
    ? {
        pluginId: state.capability.pluginId,
        capability: { ...state.capability.capability },
      }
    : undefined;                          // ← Sets to undefined when capability not provided!
  // ...
}
```

When `state.capability` is `undefined` (not provided in the restore object), `memoryPluginState.capability` is explicitly set to `undefined`, wiping any previously registered capability.

### Contrast with the Cache-Hit Path (Correct)

**File:** `src/plugins/loader.ts`, line ~1453:
```typescript
restoreMemoryPluginState({
  capability: cached.memoryCapability,      // ✅ INCLUDED
  corpusSupplements: cached.memoryCorpusSupplements,
  promptBuilder: cached.memoryPromptBuilder,
  promptSupplements: cached.memoryPromptSupplements,
  flushPlanResolver: cached.memoryFlushPlanResolver,
  runtime: cached.memoryRuntime,
});
```

### Impact Assessment

This is a **secondary** root cause. The snapshot path is designed for non-activating loads and self-recovers on the next cache-hit restore. However, it creates a **race condition**: if a snapshot load occurs WHILE a bridge sync is in progress on another async path, the capability could be temporarily wiped, producing a false zero-artifact result.

### Proposed Fix (Gemini)

Capture `previousMemoryCapability` and include it in the restore:
```typescript
const previousMemoryCapability = getMemoryCapabilityRegistration();
// ...
restoreMemoryPluginState({
  capability: previousMemoryCapability,    // ← ADD THIS
  corpusSupplements: previousMemoryCorpusSupplements,
  promptBuilder: previousMemoryPromptBuilder,
  promptSupplements: previousMemoryPromptSupplements,
  flushPlanResolver: previousMemoryFlushPlanResolver,
  runtime: previousMemoryRuntime,
});
```

---

## 4. The Destructive Pruning Risk

When bridge import sees zero artifacts, the reconciliation logic becomes destructive.

### The Pruning Code

**File:** `extensions/memory-wiki/src/bridge.ts`, line 251-256:
```typescript
const removedCount = await pruneImportedSourceEntries({
  vaultRoot: params.config.vault.path,
  group: "bridge",
  activeKeys,          // Empty Set when 0 artifacts!
  state,
});
```

**File:** `extensions/memory-wiki/src/source-sync-state.ts`, line 89-100:
```typescript
export async function pruneImportedSourceEntries(params: {
  vaultRoot: string;
  group: MemoryWikiImportedSourceGroup;
  activeKeys: Set<string>;
  state: MemoryWikiImportedSourceState;
}): Promise<number> {
  let removedCount = 0;
  for (const [syncKey, entry] of Object.entries(params.state.entries)) {
    if (entry.group !== params.group || params.activeKeys.has(syncKey)) {
      continue;
    }
    const pageAbsPath = path.join(params.vaultRoot, entry.pagePath);
    await fs.rm(pageAbsPath, { force: true });     // ← DELETES the page file!
    delete params.state.entries[syncKey];
    removedCount += 1;
  }
  return removedCount;
}
```

### The Destructive Scenario

1. Bridge import succeeds via gateway → pages created, `source-sync.json` populated
2. A CLI invocation (e.g., `openclaw wiki bridge import` or lazy sync via CLI tools) runs
3. CLI path sees 0 artifacts (Root Cause #1)
4. `activeKeys` is empty
5. `pruneImportedSourceEntries` iterates ALL entries, finds none in `activeKeys`
6. **Deletes every bridge page file and removes all state entries**

Issue #65976 documents exactly this:
```json
{
  "importedCount": 0,
  "updatedCount": 0,
  "skippedCount": 0,
  "removedCount": 305,
  "artifactCount": 0,
  "workspaces": 0
}
```

### The Gate Checks That Don't Help

**File:** `extensions/memory-wiki/src/bridge.ts`, line 206-220:
```typescript
if (
  params.config.vaultMode !== "bridge" ||
  !params.config.bridge.enabled ||
  !params.config.bridge.readMemoryArtifacts ||
  !params.appConfig
) {
  return { importedCount: 0, ..., artifactCount: 0, ... };
}
```

Our config passes all checks. The early return is NOT the problem - the function proceeds to the broken `listActiveMemoryPublicArtifacts` call.

### Missing Guard

There is no guard for the "artifact count dropped from nonzero to zero" scenario. A defensive check would prevent catastrophic data loss:

```typescript
if (artifacts.length === 0 && existingBridgeEntries > 0) {
  logger.warn("Bridge detected 0 artifacts but existing pages exist. Skipping destructive reconciliation.");
  return { importedCount: 0, removedCount: 0, ... };
}
```

---

## 5. Evidence Map

### Which Model Found What

| Finding | Opus 4.6 | GPT-5.4 | Gemini 3.1 Pro |
|---------|----------|---------|----------------|
| CLI never registers `registerMemoryCapability` | ✅ Primary finder | Referenced | — |
| `cli-metadata.ts` vs `index.ts` gap | ✅ Full code trace | — | — |
| Noop handlers in CLI loader (`api-builder.ts:103`) | ✅ Found | — | — |
| Full CLI call chain trace | ✅ End-to-end | — | — |
| Full gateway call chain trace | ✅ End-to-end | — | — |
| Snapshot restore drops `capability` | ✅ Confirmed | — | ✅ Primary finder |
| Error-handler rollback also drops `capability` | ✅ Found | — | — |
| State capture omits `getMemoryCapabilityRegistration()` | — | — | ✅ Found |
| Proposed snapshot fix (add `previousMemoryCapability`) | — | — | ✅ Authored |
| Destructive pruning via `fs.rm` in `source-sync-state.ts` | ✅ Code trace | ✅ Via issue #65976 | — |
| Debunked JSON serialization theory | ✅ Proved incorrect | — | — |
| In-memory Map cache (no JSON) (`loader.ts:208`) | ✅ Found | — | — |
| Cache-hit restore correctly includes capability (`loader.ts:~1453`) | ✅ Found | — | — |
| Test gap: bridge tests bypass plugin loader | ✅ Analyzed all 5 tests | — | — |
| Test gap: no CLI integration test | ✅ Identified | — | — |
| `restoreMemoryPluginState` sets capability to `undefined` (`memory-state.ts:295`) | ✅ Traced | — | ✅ Traced |
| Issue #63092 (earliest report, v4.8) | — | ✅ Found | — |
| Issue #65698 (cache/capability restore, v4.11) | — | ✅ Found & analyzed | — |
| Issue #65976 (destructive pruning, v4.12) | — | ✅ Found & analyzed | — |
| Issue #66082 (another repro, v4.12) | — | ✅ Found | — |
| Issue #67190 (CLI vs gateway, v4.12) | — | ✅ Detailed read | — |
| Issue #67327 (status/doctor still broken, v4.14) | — | ✅ Detailed read | — |
| PR #67208 (route CLI import through gateway) | — | ✅ Detailed read | — |
| Changelog timeline (4.8 through 4.15) | — | ✅ Full analysis | — |
| No `bridge.ts` commit fixes zero-artifact | — | ✅ Git history check | — |
| `status.ts` uses same broken path (`status.ts:219`) | ✅ Code reference | ✅ Via issue #67327 | — |
| `shouldImportArtifact` filter works correctly (`bridge.ts:44`) | ✅ Verified | — | — |
| `collectWorkspaceArtifacts` function is correct | ✅ Verified | — | — |
| No `callGatewayFromCli` in `cli.ts` | — | ✅ Grep confirmed | — |
| Docs recommend trusting broken diagnostics | — | ✅ Identified mismatch | — |

### Source File References

| File | Key Lines | Role in Bug |
|------|-----------|-------------|
| `extensions/memory-core/cli-metadata.ts` | 7-23 | **ROOT CAUSE #1**: Only registers CLI, not memory capability |
| `extensions/memory-core/index.ts` | 37 | Correct runtime registration (has `registerMemoryCapability`) |
| `extensions/memory-core/src/public-artifacts.ts` | Full file | The artifact discovery function that never gets called in CLI |
| `src/plugins/loader.ts` | 2530-2532 | Resolves `cli-metadata.ts` as CLI entry source |
| `src/plugins/loader.ts` | 2639 | CLI handler map (only `registerCli`, no capability) |
| `src/plugins/loader.ts` | ~2229 | **ROOT CAUSE #2**: Previous state capture omits capability |
| `src/plugins/loader.ts` | ~2240 | Snapshot restore omits capability |
| `src/plugins/loader.ts` | ~2258 | Error rollback also omits capability |
| `src/plugins/loader.ts` | ~1453 | Cache-hit restore (correct - includes capability) |
| `src/plugins/loader.ts` | 208 | In-memory Map cache (no JSON serialization) |
| `src/plugins/api-builder.ts` | 103 | Noop handler for `registerMemoryCapability` |
| `src/plugins/api-builder.ts` | 169 | Falls through to noop when handler not provided |
| `src/plugins/memory-state.ts` | 278 | `listActiveMemoryPublicArtifacts` returns `?? []` |
| `src/plugins/memory-state.ts` | 295-305 | `restoreMemoryPluginState` sets capability to `undefined` |
| `src/plugins/memory-state.ts` | 197-203 | `getMemoryCapabilityRegistration()` preserves functions |
| `extensions/memory-wiki/src/bridge.ts` | 224 | **FAILURE POINT**: calls `listActiveMemoryPublicArtifacts` |
| `extensions/memory-wiki/src/bridge.ts` | 251-256 | Pruning with empty `activeKeys` |
| `extensions/memory-wiki/src/bridge.ts` | 206-220 | Gate checks (pass correctly, not the problem) |
| `extensions/memory-wiki/src/bridge.ts` | 44-56 | `shouldImportArtifact` (correct when artifacts exist) |
| `extensions/memory-wiki/src/source-sync-state.ts` | 89-100 | `pruneImportedSourceEntries` with `fs.rm` |
| `extensions/memory-wiki/src/status.ts` | 219-225 | Status uses same broken path |
| `extensions/memory-wiki/src/cli.ts` | Full file | No `callGatewayFromCli` usage |
| `extensions/memory-wiki/src/gateway.ts` | Full file | Gateway methods work correctly |
| `extensions/memory-wiki/src/bridge.test.ts` | 51-55, 165 | Tests bypass plugin loader |
| `src/plugins/loader.test.ts` | 2204 | Cache test (runtime only, proves functions survive) |

---

## 6. Upstream Issues & PRs

### Issue Timeline

| Issue | Version | Title | Status | Key Finding |
|-------|---------|-------|--------|-------------|
| **#63092** | 4.8 | Bridge imports 0 artifacts from memory-core | — | Earliest report in current cluster |
| **#65698** | 4.11 | publicArtifacts capability not restored on cached plugin registry loads | Fixed in 4.12 | Cache/capability restore bug (loader now includes `memoryCapability` in `CachedPluginState`) |
| **#65976** | 4.11/4.12 | Zero public artifacts can remove existing bridge pages | Open | Destructive false-negative reconciliation (`removedCount: 305`) |
| **#66082** | 4.12 | Bridge reports 0 exported artifacts despite valid files | — | Independent reproduction after 4.12 release |
| **#67190** | 4.12 | CLI bridge import returns 0 while gateway call works | Open | Clearest articulation of CLI vs gateway split |
| **#67327** | 4.14 | Status/doctor report 0 bridge artifacts despite populated memory | Open | Even after gateway workaround, diagnostics still broken |

### PR Status

| PR | Title | Status | Scope |
|----|-------|--------|-------|
| **#63165** | Add QMD bridge recipe | Merged (4.12) | Docs + troubleshooting guidance |
| **#64505** | Surface memory wiki imports and palace | Merged (4.11) | Dreaming UI subtabs |
| **#64779** | Resolve CLI command aliases against parent plugin | Merged (4.12) | CLI plugin resolution fix |
| **#65012** | Pass app config into CLI metadata registrar | Merged (4.12) | CLI config + cache/capability restore fix |
| **#67208** | Route wiki bridge import CLI through gateway RPC | **Open** | Fixes `wiki bridge import` only; does NOT fix `status`/`doctor` |

### What's Fixed vs What's Not

| Bug Class | Status | Release | Notes |
|-----------|--------|---------|-------|
| Cache/capability restore (warm loads) | **Fixed** | 4.12 | #65698 / #65012 - `memoryCapability` now in `CachedPluginState` |
| CLI process never has capability | **Open** | — | #67190 / PR #67208 (open, partial) |
| Status/doctor false-negative in CLI | **Open** | — | #67327 - not addressed by PR #67208 |
| Destructive pruning on false zero | **Open** | — | #65976 - no guardrail proposed in any PR |
| Snapshot restore drops capability | **Open** | — | Not filed as standalone issue |

### Changelog Evidence

| Version | Bridge-Related Change |
|---------|----------------------|
| **4.8** | Memory-wiki stack restored (bridge feature ships) |
| **4.10/4.11** | Dreaming/memory-wiki UI features (#64505) |
| **4.12** | Docs + troubleshooting (#63165), CLI config plumbing (#64779, #65012), cached capability restore fix |
| **4.14** | Dreaming UI cleanup (#66140), no bridge zero-artifact fix |
| **4.15** | No bridge-specific fix in changelog |

---

## 7. Immediate Workaround

### Gateway RPC (Works Now)

```bash
# Import via gateway (runs in daemon where capability exists)
openclaw gateway call wiki.bridge.import

# Status via gateway (also runs in daemon)
openclaw gateway call wiki.status
```

These work because the gateway process loads plugins via `loadOpenClawPlugins()` → `index.ts` → `registerMemoryCapability()` is called → `memoryPluginState.capability` is populated.

### Agent Tools (Also Work)

Wiki tools called by agents (`wiki_search`, `wiki_get`, `wiki_apply`, `wiki_status`, `wiki_lint`) execute in the gateway process. `syncImportedSourcesIfNeeded()` runs before every tool call and uses the populated capability.

### What Does NOT Work

```bash
# These all run in CLI process where capability is undefined:
openclaw wiki bridge import      # ← Returns 0 artifacts
openclaw wiki status             # ← Shows bridge-artifacts-missing warning  
openclaw wiki doctor             # ← Same false-negative
```

### Alternative: unsafe-local Mode

If gateway workaround is insufficient, switching to `unsafe-local` mode bypasses the `publicArtifacts` mechanism entirely:

```json
{
  "plugins.entries.memory-wiki": {
    "vaultMode": "unsafe-local",
    "unsafeLocal": {
      "allowPrivateMemoryCoreAccess": true,
      "paths": ["~/.openclaw/workspace/MEMORY.md", "~/.openclaw/workspace/memory"]
    }
  }
}
```

This reads files directly from the filesystem instead of going through the plugin capability chain.

---

## 8. Proposed Upstream Fix

### Fix A: Route ALL Wiki CLI Commands Through Gateway (Recommended)

Extend PR #67208's approach to ALL bridge-dependent CLI commands, not just `bridge import`:

**File:** `extensions/memory-wiki/src/cli.ts`

```typescript
// Current (broken):
bridge
  .command("import")
  .action(async (opts) => {
    await runWikiBridgeImport({ config, appConfig, json: opts.json });
  });

// Fixed:
bridge
  .command("import")
  .action(async (opts) => {
    const result = await callGatewayFromCli("wiki.bridge.import", {});
    // ... format and display result
  });
```

Same treatment for `status` and `doctor`:
```typescript
// Route status through gateway
statusCmd.action(async () => {
  const result = await callGatewayFromCli("wiki.status", {});
  // ... format and display
});

// Route doctor through gateway
doctorCmd.action(async () => {
  const result = await callGatewayFromCli("wiki.doctor", {});
  // ... format and display
});
```

### Fix B: Snapshot Restore - Include Capability

**File:** `src/plugins/loader.ts`, line ~2229:

```typescript
// Add to state capture:
const previousMemoryCapability = getMemoryCapabilityRegistration();

// Include in both restore paths:
restoreMemoryPluginState({
  capability: previousMemoryCapability,    // ← ADD THIS
  corpusSupplements: previousMemoryCorpusSupplements,
  promptBuilder: previousMemoryPromptBuilder,
  promptSupplements: previousMemoryPromptSupplements,
  flushPlanResolver: previousMemoryFlushPlanResolver,
  runtime: previousMemoryRuntime,
});
```

Apply to both the non-activating restore (line ~2240) and the error-handler rollback (line ~2258).

### Fix C: Zero-Artifact Pruning Guard (Defensive, Complementary)

**File:** `extensions/memory-wiki/src/bridge.ts`, after artifact discovery:

```typescript
const publicArtifacts = await listActiveMemoryPublicArtifacts({ cfg: params.appConfig });
const artifacts = await collectBridgeArtifacts(params.config.bridge, publicArtifacts);

// Guard against false-negative pruning
const existingEntryCount = Object.values(state.entries)
  .filter(e => e.group === "bridge").length;

if (artifacts.length === 0 && existingEntryCount > 0) {
  logger.warn(
    `Bridge detected 0 artifacts but ${existingEntryCount} existing bridge pages. ` +
    `Skipping reconciliation to prevent data loss. ` +
    `Run via gateway to verify: openclaw gateway call wiki.bridge.import`
  );
  return {
    importedCount: 0,
    updatedCount: 0,
    skippedCount: existingEntryCount,
    removedCount: 0,
    artifactCount: 0,
    workspaces: 0,
  };
}
```

### Fix D: Test Coverage

Add missing integration tests:

1. **CLI bridge import end-to-end**: Test `loadOpenClawPluginCliRegistry()` → `listActiveMemoryPublicArtifacts()` to catch the CLI gap
2. **False-zero pruning**: Test that bridge sync with 0 artifacts but existing pages does NOT delete pages
3. **Cross-context consistency**: Verify CLI and gateway produce the same artifact count
4. **Snapshot restore with capability**: Test that non-activating snapshot loads preserve the capability

---

## 9. Our Config Recommendations

### Current Config (Correct but Impacted by Bug)

```json
{
  "plugins.entries.memory-wiki": {
    "vaultMode": "bridge",
    "bridge": {
      "enabled": true,
      "readMemoryArtifacts": true,
      "indexDreamReports": true,
      "indexDailyNotes": true,
      "indexMemoryRoot": true,
      "followMemoryEvents": true
    },
    "search": { "backend": "shared", "corpus": "all" },
    "context": { "includeCompiledDigestPrompt": false },
    "vault.renderMode": "obsidian"
  }
}
```

### Recommended Changes

| Setting | Current | Recommended | Rationale |
|---------|---------|-------------|-----------|
| `vaultMode` | `"bridge"` | **Keep** (with gateway workaround) | Bridge is architecturally correct; use `gateway call` until fix ships |
| `includeCompiledDigestPrompt` | `false` | **`true`** | Enables self-healing pressure by surfacing stale/contradicted pages in agent prompts |
| `search.backend` | `"shared"` | Keep | Correct - enables cross-corpus search |
| `search.corpus` | `"all"` | Keep | Correct - searches wiki + memory |
| `vault.renderMode` | `"obsidian"` | Keep | Correct for Obsidian compatibility |

### Critical Operational Rules

1. **NEVER run `openclaw wiki bridge import` from CLI** - use `openclaw gateway call wiki.bridge.import` instead
2. **NEVER trust `openclaw wiki status` bridge artifact count** - use `openclaw gateway call wiki.status` instead
3. **Agent wiki tools are safe** - they run in the gateway process
4. **If bridge pages disappear** - re-import via `openclaw gateway call wiki.bridge.import`

---

## 10. Action Plan

### Immediate (Today)

| # | Action | Priority | Risk |
|---|--------|----------|------|
| 1 | Switch bridge operations to `openclaw gateway call wiki.bridge.import` | **Critical** | None - already confirmed working |
| 2 | Stop using `openclaw wiki bridge import` / `openclaw wiki status` from CLI | **Critical** | Prevents destructive pruning |
| 3 | Enable `includeCompiledDigestPrompt: true` in config | **High** | None - pure improvement |
| 4 | Add gateway-based bridge import to nightshift/heartbeat cron | **High** | None |

### Short-Term (This Week)

| # | Action | Priority | Notes |
|---|--------|----------|-------|
| 5 | Monitor PR #67208 for merge | **Medium** | Fixes CLI `bridge import` only |
| 6 | File issue for snapshot restore capability omission | **Medium** | Root Cause #2 has no standalone issue |
| 7 | File issue / comment on #65976 for zero-artifact pruning guard | **Medium** | Prevents destructive reconciliation |
| 8 | Verify our bridge pages are intact; re-import if needed | **High** | Check `source-sync.json` for entry count |

### Medium-Term (Track Upstream)

| # | Action | Priority | Notes |
|---|--------|----------|-------|
| 9 | Watch for status/doctor CLI fix (post-#67208) | **Medium** | #67327 remains open |
| 10 | Consider `unsafe-local` as fallback if bridge remains unreliable | **Low** | Bypasses the entire capability chain |
| 11 | Re-evaluate after next OpenClaw release | **Medium** | Check changelog for bridge fixes |

### Bug Chain Summary for Upstream

```
#63092 (4.8) → #65698 (4.11, FIXED in 4.12) → #65976 (4.12, OPEN)
                                               → #66082 (4.12)
                                               → #67190 (4.12, PR #67208 OPEN)
                                                   → #67327 (4.14, OPEN)
```

The strongest upstream chain is: **#63092 → #65698 → #65976/#66082 → #67190 → #67327**

One fix landed (cache/capability restore in 4.12). One fix proposed (PR #67208 for CLI import). Two defects remain unfixed (status/doctor, destructive pruning guard).

---

*Investigation complete. The bridge zero-artifact bug has been root-caused to source-level precision across two distinct code paths. The gateway workaround is reliable and should be used exclusively until upstream fixes ship.*
