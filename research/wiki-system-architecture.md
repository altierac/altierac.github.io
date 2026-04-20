# OpenClaw Wiki System: Deep Research Report

**Date:** 2026-04-19
**Models:** Opus 4.6, GPT-5.4, Gemini 3.1 Pro
**Freshness:** Current as of April 19, 2026
**Sources:** 85+ sources across 3 models (68 Opus findings from source code deep-dive, 14 GPT-5.4 document/community analyses, 8 Gemini comparative analyses)

---

## Executive Summary

1. **The wiki is a deterministic compile layer, not a knowledge engine.** It indexes, cross-references, and surfaces knowledge that agents and humans create - it does NOT use LLMs to generate content. All synthesis creation is agent-initiated via `wiki_apply`. (Opus: confirmed at source level; GPT-5.4: confirmed via docs; Gemini: confirmed via code analysis)

2. **Self-healing is indirect, not autonomous.** Freshness tracking, contradiction surfacing, search quality scoring, and opt-in prompt injection create *pressure* for agents to fix issues - but no daemon or background process actually performs repairs. (All three models agree)

3. **Bridge mode is broken in our version (2026.4.14).** A known bug (#63092, #65698) causes `publicArtifacts.listArtifacts` to be lost during cached plugin state serialization - function references don't survive JSON round-trips. This affects ALL users, not just CLI. Our bridge shows 0 artifacts despite correct config. (Opus: root-caused in source; GPT-5.4: confirmed via GitHub issues)

4. **Dreaming and the wiki are parallel systems with one-way import.** Dreaming writes to `MEMORY.md` and `DREAMS.md`; the wiki can import dream reports via bridge mode (when it works), but dreaming does NOT trigger wiki syntheses. Zero references to "wiki" or "synthesis" exist in dreaming code. (Opus: grep-confirmed; GPT-5.4: confirmed via dreaming docs)

5. **Freshness is timestamp-based only.** Fresh (<30 days), aging (30-89 days), stale (≥90 days). No semantic staleness detection - updating `updatedAt` makes a page "fresh" regardless of content quality. (Opus: extracted exact thresholds from source)

6. **Search is text-matching with quality weights, not embedding-based.** Fresh/confident/non-contested claims rank higher. QMD separately provides semantic search over wiki files. Three search paths exist into wiki content. (Opus: full scoring formula extracted; Gemini: confirmed no embeddings)

7. **Human blocks are sacred - never overwritten by any operation.** The `<!-- openclaw:human:start/end -->` markers are preserved across all updates, including `wiki_apply`, compile, and bridge sync. (All three models confirm)

8. **Our `includeCompiledDigestPrompt` is OFF.** This means stale/contradicted pages are NOT being surfaced in agent prompts. Enabling it would create self-healing pressure by injecting the top 4 problematic pages into every prompt. (Opus: confirmed from config analysis)

9. **Compile runs after every `wiki_apply` but NOT after direct file edits.** There is no file watcher. Direct markdown edits require manual `openclaw wiki compile`. Bridge sync happens lazily before every wiki tool call. (Opus: confirmed from code tracing)

10. **The architecture is ready for self-healing but doesn't implement it.** All primitives exist: structured claims, evidence chains, freshness tracking, contradiction detection, lint, quality-scored search, prompt injection, and narrow mutation tools. The missing piece is orchestration - a scheduled agent loop that reads lint reports and applies fixes. (All three models agree)

---

## Architecture Overview

### Data Flow: Ingest → Source → Synthesis → Compile → Digest

```
[Raw Files/URLs]                 [Active Memory Plugin (memory-core)]
      |                                      |
      v                                      v (publicArtifacts via SDK)
[openclaw wiki ingest]           [Bridge Sync (syncImportedSourcesIfNeeded)]
      |                                      |
      v                                      v
[sources/] ──────────┬──────────> [sources/] (bridge-backed)
                     |                       |
                     v                       v
              [Agent calls wiki_apply op=create_synthesis]
                     |
                     v
[syntheses/] ← synthesis body + claims[] + sourceIds[] + confidence
                     |
                     v
[compileMemoryWikiVault()] ── deterministic, NO LLM ──┐
      |         |         |         |                  |
      v         v         v         v                  v
  indexes   related    dashboards  digest.json    claims.jsonl
             blocks     (5 types)
                     |
                     v
[Prompt Injection (opt-in)] → top 4 pages + top 2 claims per page
```

### Five Page Kinds
| Kind | Directory | Created By | Purpose |
|------|-----------|------------|---------|
| source | `sources/` | `wiki ingest`, bridge sync, ChatGPT import | Raw evidence with provenance |
| entity | `entities/` | Agent via `wiki_apply` | Named things (people, tools, orgs) |
| concept | `concepts/` | Agent via `wiki_apply` | Abstract ideas, patterns |
| synthesis | `syntheses/` | Agent via `wiki_apply` | Curated knowledge with claims |
| report | `reports/` | Compile pipeline | Auto-generated dashboards |

### Three Vault Modes
| Mode | How Sources Arrive | Dependencies |
|------|--------------------|-------------|
| `isolated` | Manual ingest only | None |
| `bridge` | Imports public artifacts from active memory plugin via SDK | Requires `publicArtifacts.listArtifacts` (currently broken) |
| `unsafe-local` | Direct filesystem reads from configured paths | Same-machine only |

### State Files (`.openclaw-wiki/`)
| File | Purpose |
|------|---------|
| `state.json` | Vault metadata (version, createdAt, renderMode) |
| `log.jsonl` | Append-only operation log (init, compile, ingest, lint) |
| `source-sync.json` | Bridge/unsafe-local sync state (tracks imported files) |
| `cache/agent-digest.json` | Compiled page/claim digest for agents |
| `cache/claims.jsonl` | All claims indexed as JSONL |
| `locks/` | Lock directory (created but unused in current code) |
| `import-runs/` | ChatGPT import run records with snapshots |

---

## What's Already Shipped

These features work TODAY and we can use them:

### 1. Vault Initialization & Structure
- 5 page directories (entities, concepts, syntheses, sources, reports)
- `inbox.md` placeholder (not processed by any code, just a scratchpad)
- `_attachments/` and `_views/` directories (created but unused - future)

### 2. Source Page Ingest
- `openclaw wiki ingest <path>` imports local files as source pages
- Auto-compile after ingest when `autoCompile: true` (our config)
- Provenance metadata preserved (sourcePath, sourceType)

### 3. ChatGPT Export Import (Enterprise-Grade)
- Full `conversations.json` parsing with active branch extraction
- Auto-triage: risk level detection (health, legal, finance = high risk)
- Auto-labeling: topic inference (language-learning, travel, cooking, etc.)
- Preference signal extraction from user messages
- **Full rollback support** with pre-update snapshots per import run
- High-risk conversations have digest "withheld" from durable candidates

### 4. Synthesis Creation & Mutation
- `wiki_apply op=create_synthesis` with required `sourceIds[]`
- Structured claims with evidence chains, confidence (0-1), status
- `wiki_apply op=update_metadata` for updating existing page frontmatter
- **Idempotent**: re-applying same synthesis produces no changes
- **Provenance-first**: syntheses REQUIRE at least one sourceId (hard validation)

### 5. Compile Pipeline (Deterministic, No LLM)
Triggered by: `wiki_apply`, `wiki ingest`, `wiki compile`, ChatGPT import/rollback, bridge sync with changes, `wiki lint`

Produces:
- **Related blocks** on every non-report page (backlinks, shared sourceIds, wikilinks)
- **Five dashboard reports**: `stale-pages.md`, `contradictions.md`, `low-confidence.md`, `claim-health.md`, `open-questions.md`
- **Agent digest** (`agent-digest.json`): pageCounts, claimCount, claimHealth, contradictionClusters, top pages with claims
- **Claims index** (`claims.jsonl`): all claims indexed for fast lookup
- **Root and directory indexes** with page listings

### 6. Freshness Tracking
```javascript
// Thresholds from source:
fresh:   < 30 days since updatedAt
aging:   30-89 days since updatedAt  
stale:   ≥ 90 days since updatedAt
unknown: missing updatedAt
```
- Claim freshness uses LATEST of: claim.updatedAt, page.updatedAt, evidence[].updatedAt
- Freshness affects search ranking: fresh +8, aging +4, stale -2, unknown -4

### 7. Contradiction Detection
- **Page-note clusters**: from `contradictions[]` array in frontmatter (agent-written)
- **Claim clusters**: from claims with contested status sharing IDs across pages
- Four contested statuses: `contested`, `contradicted`, `refuted`, `superseded`
- Contested claims get -6 in search scoring (vs +4 for normal)

### 8. Lint System (6 Categories)
| Category | Checks |
|----------|--------|
| structure | missing id, missing pageType, type mismatch, missing title, duplicate id |
| provenance | missing sourceIds on non-source pages, missing import provenance, claims without evidence |
| links | broken wikilinks |
| contradictions | contradiction notes present, claim conflicts across pages |
| open-questions | pages with unresolved questions |
| quality | low confidence pages (<0.5), stale pages/claims |

### 9. Search System (Two Tiers)
**Tier 1 - Digest-based (fast)**: Searches `agent-digest.json` and `claims.jsonl` first
**Tier 2 - Full page scan (fallback)**: Reads all markdown files when digests unavailable

Scoring formula:
```
Title exact match:    +50    |  Claim text match:   +25
Title contains:       +20    |  Claim ID match:     +10
Path contains:        +10    |  Confidence:         +0-10
ID contains:          +20    |  Freshness:          -4 to +8
SourceId match:       +12    |  Contested:          -6 vs +4
Body occurrences:     +1 each (up to +10)
Additional claims:    +2 each (up to +10)
```

### 10. Three Search Paths Into Wiki Content
| Path | Method | Type | Works Now? |
|------|--------|------|-----------|
| `wiki_search` | Wiki's quality-weighted text matching | Structured | ✅ |
| `memory_search corpus=wiki` | Routes to wiki corpus supplement | Text | ✅ |
| `memory_search` (default) | QMD semantic search (indexes wiki dir) | Vector/semantic | ✅ |

### 11. Prompt Injection (Opt-in)
- Ranking: contradictions×6 + questions×4 + claims×2
- Top 4 pages, top 2 claims per page injected into agent prompts
- Claims include status, confidence, and freshness qualifiers
- **Currently OFF in our config** (`includeCompiledDigestPrompt: false`)

### 12. Managed Blocks Architecture
| Block Type | Markers | Behavior |
|------------|---------|----------|
| Generated | `<!-- openclaw:wiki:generated:start/end -->` | Replaced on every `wiki_apply` |
| Human | `<!-- openclaw:human:start/end -->` | NEVER touched by any operation |
| Related | `<!-- openclaw:wiki:related:start/end -->` | Auto-generated by compile |
| Index | `<!-- openclaw:wiki:index:start/end -->` | Auto-generated page listings |
| Dashboard | `<!-- openclaw:wiki:<report>:start/end -->` | Per-report managed blocks |

### 13. Obsidian Support
- Wikilink rendering: `[[path|title]]` in obsidian mode vs `[title](path.md)` in native mode
- CLI helpers: `obsidian status`, `search`, `open`, `command`, `daily` (require Obsidian app + CLI)
- Frontmatter compatible with Dataview-style queries

### 14. Gateway API (14+ Methods)
| Method | Scope | Notable |
|--------|-------|---------|
| `wiki.status` | read | Vault health overview |
| `wiki.palace` | read | Memory Palace - structured overview for Control UI |
| `wiki.importInsights` | read | ChatGPT import analysis |
| `wiki.doctor` | read | Health check (bridge issues, invalid layout) |
| `wiki.compile` | write | Rebuild all indexes and dashboards |
| `wiki.lint` | write | Run all lint checks |
| `wiki.apply` | write | Apply mutations |
| `wiki.bridge.import` | write | Force bridge sync |
| `wiki.search` | read | Search with provenance labels |
| `wiki.get` | read | Read page (supports `claim:<id>` lookup) |

### 15. Lazy Bridge Sync
- `syncImportedSourcesIfNeeded()` runs before EVERY wiki tool call (search, get, apply, lint, status)
- Incremental: only rewrites pages when source file changed (mtime or size)
- Prunes pages for deleted source files
- Auto-compiles if changes detected or indexes missing
- **Would work IF publicArtifacts wasn't broken**

---

## What's NOT Shipped

Features we might have assumed exist but DON'T:

### 1. No Automatic Synthesis Generation
The wiki registers NO event listeners, NO scheduled tasks, NO automatic synthesis triggers. Zero code paths create syntheses without an agent explicitly calling `wiki_apply`. (Opus: confirmed via full `index.js` analysis; Gemini: confirmed via code review)

### 2. No LLM-Powered Claim Verification
Claims are manually set by agents, not auto-verified against sources. No pipeline exists to check whether a claim is actually supported by its referenced evidence.

### 3. No Semantic Staleness Detection
Freshness is purely timestamp-based. A page with completely outdated content but a recent `updatedAt` is considered "fresh." No content-comparison or source-drift detection exists.

### 4. No Automatic Contradiction Resolution
Contradictions are surfaced in dashboards but never auto-resolved. No code proposes merges, picks winners, or suggests resolutions.

### 5. No Automatic Entity/Concept Extraction
Entity and concept pages must be manually created via `wiki_apply`. No NER or topic extraction runs over ingested sources.

### 6. No Dreaming → Wiki Automatic Flow
Dream reports can be imported as sources (when bridge works), but dreaming NEVER triggers `wiki_apply` or creates syntheses. Zero mentions of "wiki", "synthesis", or "create_synthesis" in dreaming code. (Opus: grep-confirmed across dreaming-narrative-CeE1UhA_.js)

### 7. No Scheduled Wiki Maintenance
No cron jobs for compile/lint/bridge sync. The wiki is purely reactive - work only happens when a tool is called.

### 8. No Embedding-Based Wiki Search
Wiki search is text-matching with quality scoring. Semantic similarity comes from QMD indexing the wiki directory separately, not from the wiki plugin itself.

### 9. No Wiki Page Deletion Tool
Pages can only be removed during bridge sync (pruning deleted sources). No `wiki_apply op=delete` or similar exists.

### 10. No URL Auto-Ingest
`allowUrlIngest` config exists but ingest only accepts local file paths. No URL fetch logic exists in the ingest pipeline.

### 11. No Confidence Decay
Claims don't lose confidence over time. A claim set at 0.95 stays at 0.95 forever unless manually updated.

### 12. No Entity Merging/Dedup
No detection for near-duplicate pages (e.g., "Apple" and "Apple Inc"). The wiki-maintainer skill tells agents to avoid duplicates, but no automated dedup exists.

### 13. No Background File Watcher
Direct file edits to wiki pages don't trigger anything. You must run `openclaw wiki compile` manually.

---

## Self-Healing Capabilities

### What Actually Exists

The wiki has **observability and structured maintenance primitives** rather than autonomous repair:

| Mechanism | Type | How It Works | Limitations |
|-----------|------|-------------|-------------|
| Stale page reporting | Detection | Compile generates `reports/stale-pages.md` listing pages with `updatedAt` > 90 days | Timestamp-only, no content analysis |
| Contradiction surfacing | Detection | Compile clusters contested claims and page `contradictions[]` arrays | Agent-authored metadata, not AI-detected |
| Quality-scored search | Incentive | Fresh/confident claims rank higher, contested rank lower | Passive - doesn't fix anything |
| Prompt injection | Pressure | Opt-in digest surfaces problematic pages to agents (top 4 by: contradictions×6 + questions×4 + claims×2) | **Currently OFF** in our config |
| Lint reports | Diagnosis | 6-category health check covering structure, provenance, links, contradictions, questions, quality | Diagnostic only - doesn't auto-fix |
| Bridge pruning | Cleanup | Deleted source files get their wiki pages removed during sync | Only works for bridge-imported pages |
| Missing index recovery | Repair | Auto-compiles when indexes are missing (on any tool call) | Only fixes missing files, not stale content |
| `wiki doctor` | Diagnosis | Detects bridge issues, invalid vault layout, missing dependencies | Diagnostic only |

### The Self-Healing Gap

As GPT-5.4 and Gemini both identified: the **loop is missing**. The primitives exist but nobody orchestrates them:

```
[WHAT EXISTS]                          [WHAT'S MISSING]
wiki_lint → surfaces issues      →     Scheduled agent reads lint output
reports/stale-pages.md           →     Agent identifies which pages to refresh  
reports/contradictions.md        →     Agent proposes resolution
prompt injection (opt-in)        →     Agent acts on surfaced problems
wiki_apply                       →     Agent writes fixes
```

The system is designed for **agent-assisted repair**, not autonomous repair. The agent must:
1. Run `wiki_lint`
2. Read the reports
3. Decide what to fix
4. Call `wiki_apply` with corrections
5. Compile updates

No code automates steps 1-5 as a loop.

---

## Bridge Mode

### How It Works (Architecture)

Bridge mode imports "public artifacts" from the active memory plugin through SDK seams:

```
[memory-core] → api.registerMemoryCapability({
    publicArtifacts: { listArtifacts: collectWorkspaceArtifacts }
})
    ↓
[memory-wiki bridge sync] → listActiveMemoryPublicArtifacts()
    ↓
Imports 4 artifact types:
├── memory-root (MEMORY.md)         ← bridge.indexMemoryRoot
├── daily-note (memory/*.md)        ← bridge.indexDailyNotes  
├── dream-report (memory/dreaming/) ← bridge.indexDreamReports
└── event-log (memory event journal) ← bridge.followMemoryEvents
    ↓
Creates source pages with:
├── pageType: source
├── sourceType: "memory-bridge" / "memory-bridge-events"
├── sourcePath, bridgeRelativePath, bridgeWorkspaceDir
└── Content as fenced markdown/json block
```

Bridge pages are **read-only copies** - they don't interpret or transform content.

### Sync Behavior
- **Incremental**: Tracks state in `source-sync.json` (syncKey, mtime, size, renderFingerprint)
- **Skip logic**: If source unchanged AND page file exists, skip writing
- **Pruning**: Removes pages for sources that no longer exist
- **Lazy trigger**: `syncImportedSourcesIfNeeded()` runs before every wiki tool call
- **Auto-compile**: If any changes detected, triggers full compile

### The Known Bug (Critical)

**Issue #63092, #65698, #65976, #67327** - Bridge reports 0 artifacts across multiple versions (4.8 through 4.14).

**Root cause** (Opus Finding 31/46 - confirmed via source code):

```javascript
// memory-core/cli-metadata.js (CLI mode) - NO capability registration
register(api) {
  api.registerCli(...); // Only registers CLI commands
}

// memory-core/index.js (gateway mode) - HAS capability registration  
register(api) {
  api.registerMemoryCapability({
    publicArtifacts: { listArtifacts: listMemoryCorePublicArtifacts }  // ← Function reference
  });
}
```

The `listArtifacts` is a **function reference**. When the plugin registry caches and restores state, function references are lost during JSON serialization. `memoryPluginState.capability` gets restored as an object but `publicArtifacts.listArtifacts` becomes `undefined`.

**Evidence this is the cause**:
- `restoreMemoryPluginState()` spreads `state.capability.capability` - functions don't survive JSON round-trips
- Both CLI and gateway contexts show 0 artifacts (confirmed via our `wiki_status` tool call)
- Issue #65976 reports `removedCount: 305` - destructive false-negative reconciliation
- Issue #67327 reports import succeeds via `gateway call` but `status`/`doctor` still show 0

**Destructive behavior**: When bridge falsely sees 0 artifacts, it can **delete existing bridge pages** during reconciliation. Issue #65976 proposes a guardrail for zero-artifact anomaly detection.

### Workaround
Switch to `unsafe-local` mode with explicit paths:
```json
{
  "vaultMode": "unsafe-local",
  "unsafeLocal": {
    "allowPrivateMemoryCoreAccess": true,
    "paths": ["~/.openclaw/workspace/MEMORY.md", "~/.openclaw/workspace/memory"]
  }
}
```
This bypasses `publicArtifacts` entirely and reads files directly. (Opus: architecturally sound but untested)

---

## Dreaming Integration

### What Dreaming Does (in memory-core, NOT wiki)

Three phases, scheduled via cron (default: `0 3 * * *`):
1. **Light**: Ingests recent daily signals, dedupes, stages candidates. Never writes to MEMORY.md.
2. **Deep**: Ranks candidates using 6 weighted signals (frequency 0.24, relevance 0.30, query diversity 0.15, recency 0.15, consolidation 0.10, conceptual richness 0.06). Promotes qualified items to MEMORY.md.
3. **REM**: Extracts patterns and reflective signals. Never writes to MEMORY.md.

Outputs: `DREAMS.md` (diary), `memory/dreaming/<phase>/YYYY-MM-DD.md` (phase reports)

### What the Wiki Can Import (When Bridge Works)
- Dream report files get kind `dream-report` in bridge import
- Imported as read-only source pages with provenance metadata
- No analysis, no synthesis, no reverse flow

### What Does NOT Happen
- Dreaming does NOT trigger `wiki_apply`
- Dreaming does NOT create syntheses
- Wiki does NOT feed back into dreaming
- Zero cross-references in code between dreaming narrative and wiki

### Dreaming-Wiki UI Integration (v2026.4.11 Feature)
Release v2026.4.11 shipped: "Dreaming can inspect imported source chats, compiled wiki pages, and full source pages directly from the UI" (PR #64505). However, Issue #65091 reports this **regressed** - compiled wiki pages no longer load in Dreaming sessions after upgrade. The feature exists in product intent but may not be stable.

### The Theoretical Full Pipeline
```
daily notes → Dreaming Light → Deep promotion → MEMORY.md
                                                    ↓
                                        Bridge import (when working)
                                                    ↓
                                           Wiki source page
                                                    ↓
                                    Agent reads + creates synthesis
                                                    ↓
                                        wiki_apply + compile
```

Steps 3-5 are manual/agent-initiated. No automation connects dreaming outputs to wiki syntheses.

---

## Evidence Map

| Feature | Opus (Source Code) | GPT-5.4 (Docs/Community) | Gemini (Comparison) |
|---------|-------------------|--------------------------|---------------------|
| Deterministic compile (no LLM) | ✅ Confirmed in `compileMemoryWikiVault()` | ✅ Docs say "deterministic pages" | ✅ "compile just builds indexes" |
| 5 dashboard reports | ✅ Code generates all 5 | ✅ Docs list all 5 | ✅ Confirmed |
| Freshness tracking (30/90 day) | ✅ Exact thresholds from `buildFreshnessFromTimestamp()` | ✅ Docs mention freshness | ✅ Confirmed |
| Contradiction detection (manual) | ✅ `buildPageContradictionClusters()` - from frontmatter | ✅ Docs say contradiction dashboards | ✅ "lint just surfaces them" |
| Claim evidence chains | ✅ Full schema extracted | ✅ Docs describe structured claims | ✅ Confirmed |
| Bridge mode (broken) | ✅ Root-caused: function serialization | ✅ Issues #65698, #65976, #67327 | — |
| No auto-synthesis | ✅ Zero event listeners or scheduled tasks | ✅ Docs never claim it | ✅ "wiki does not background-synthesize" |
| Prompt injection (opt-in) | ✅ `buildDigestPromptSection()` code analyzed | ✅ Docs describe it | — |
| Lazy bridge sync | ✅ `syncImportedSourcesIfNeeded()` before all tools | ✅ Issue discussion confirms | — |
| Human blocks preserved | ✅ `preserveHumanBlocks` in `replaceManagedMarkdownBlock()` | ✅ Skills warn "don't overwrite" | ✅ Confirmed |
| ChatGPT import + rollback | ✅ Full pipeline with snapshots | ✅ Release note v2026.4.11 | — |
| QMD indexes wiki separately | ✅ QMD `index.yml` has wiki collection | — | — |
| Search quality scoring | ✅ Full formula extracted | ✅ Docs mention freshness-weighted | — |
| Dreaming → wiki one-way only | ✅ Zero wiki refs in dreaming code | ✅ Docs say dreaming owns promotion | ✅ Confirmed |
| No entity/concept auto-extraction | ✅ No NER code found | ✅ Not mentioned in docs | ✅ "No background analysis" |
| No URL ingest (config exists) | ✅ `allowUrlIngest` flag but no fetch code | — | — |
| No file watcher | ✅ Purely reactive, no fs.watch | — | ✅ "No background daemon" |
| Unicode slug collisions | — | ✅ Issue #65965 | — |
| Dreaming UI regression | — | ✅ Issue #65091 | — |
| Perplexity import requested | — | ✅ Issue #65546 | — |
| Community adoption positive | — | ✅ Reddit, Blink blog | — |
| Comparison to Obsidian/Notion/Mem | — | — | ✅ "None truly self-healing" |

---

## Configuration Recommendations

### Our Current Config (from `openclaw.json`)
```json
{
  "plugins.entries.memory-wiki": {
    "vaultMode": "bridge",
    "bridge": { "enabled": true, "readMemoryArtifacts": true, "indexDreamReports": true, "indexDailyNotes": true, "indexMemoryRoot": true, "followMemoryEvents": true },
    "search": { "backend": "shared", "corpus": "all" },
    "context": { "includeCompiledDigestPrompt": false },
    "vault.renderMode": "obsidian"
  },
  "plugins.entries.memory-core": {
    "dreaming": { "enabled": true, "frequency": "0 3 * * *", "storage.mode": "both" }
  }
}
```

### What's Suboptimal and What to Change

| Setting | Current | Recommended | Why |
|---------|---------|-------------|-----|
| `vaultMode` | `"bridge"` | Keep, but consider `"unsafe-local"` as workaround | Bridge is broken (#65698). Unsafe-local would actually import files. |
| `includeCompiledDigestPrompt` | `false` | **`true`** | This is the main self-healing lever. Surfaces stale/contradicted pages in agent prompts. Currently OFF means zero prompt pressure. |
| `search.backend` | `"shared"` | Keep `"shared"` | ✅ Correct. Enables cross-corpus search (wiki + QMD). |
| `search.corpus` | `"all"` | Keep `"all"` | ✅ Correct. Searches both wiki and memory. |
| `ingest.autoCompile` | `true` (default) | Keep `true` | ✅ Correct. Auto-recompiles after ingest. |
| `render.preserveHumanBlocks` | `true` (default) | Keep `true` | ✅ Correct. Never lose human notes. |
| `render.createBacklinks` | `true` (default) | Keep `true` | ✅ Correct. Auto cross-linking. |
| `render.createDashboards` | `true` (default) | Keep `true` | ✅ Correct. Health reports. |
| `vault.renderMode` | `"obsidian"` | Keep if browsing in Obsidian, otherwise `"native"` | Obsidian mode uses `[[wikilinks]]`. We're on a headless VM so this only matters if we ever open the vault locally. |
| Obsidian CLI settings | all `false` (default) | Keep `false` | ✅ Correct. No Obsidian app on headless VM. |

### Immediate Action Items
1. **Enable `includeCompiledDigestPrompt: true`** - single biggest improvement for self-healing pressure
2. **Test `unsafe-local` mode** as bridge workaround - would actually import MEMORY.md and daily notes
3. **Set up a cron job** for `openclaw wiki compile && openclaw wiki lint` (daily, e.g., after dreaming at 4 AM)
4. **Create entity and concept pages** - our vault has 0 entities and 0 concepts (only sources and syntheses)

### Recommended Config Change
```json
{
  "plugins.entries.memory-wiki": {
    "vaultMode": "bridge",
    "bridge": { "enabled": true, "readMemoryArtifacts": true, "indexDreamReports": true, "indexDailyNotes": true, "indexMemoryRoot": true, "followMemoryEvents": true },
    "search": { "backend": "shared", "corpus": "all" },
    "context": { "includeCompiledDigestPrompt": true },
    "vault.renderMode": "obsidian"
  }
}
```

---

## What We Should Build

### 1. Nightshift Wiki Maintenance Cron (HIGH PRIORITY)
A scheduled cron (e.g., 4 AM daily, after dreaming at 3 AM) that:
1. Runs `openclaw wiki compile`
2. Runs `openclaw wiki lint`
3. Reads `reports/stale-pages.md`, `reports/contradictions.md`, `reports/low-confidence.md`
4. For each issue, decides whether to fix via `wiki_apply` or flag for human review
5. Logs actions to `memory/YYYY-MM-DD.md`

This closes the self-healing loop that the architecture is designed for but doesn't implement.

### 2. Bridge Mode Workaround (HIGH PRIORITY)
Until the bridge bug is fixed:
- Either switch to `unsafe-local` mode with explicit paths
- Or build a simple script that manually ingests key memory files:
  ```bash
  openclaw wiki ingest ~/.openclaw/workspace/MEMORY.md
  openclaw wiki ingest ~/.openclaw/workspace/memory/$(date +%Y-%m-%d).md
  ```

### 3. Heartbeat Wiki Check (MEDIUM PRIORITY)
Add to HEARTBEAT.md: periodically (every few heartbeats) run `wiki_lint` and check if any lint issues need attention. Log check timestamp to `heartbeat-state.json`.

### 4. Entity/Concept Page Bootstrap (MEDIUM PRIORITY)
Our vault has 0 entities and 0 concepts. We should:
- Create entity pages for recurring subjects (people, tools, projects)
- Create concept pages for recurring themes (research topics, architectural patterns)
- These improve cross-linking and make the vault more navigable

### 5. Synthesis-from-Sources Pipeline (MEDIUM PRIORITY)
When new sources are ingested, a background check should:
1. Read the new source page
2. Check if any existing synthesis pages reference related topics
3. Suggest or auto-create syntheses for uncovered topics
4. Update existing syntheses with new evidence

### 6. Confidence Decay Automation (LOW PRIORITY)
Build a periodic job that:
- Reduces confidence of claims older than 60 days without evidence refresh
- Surfaces decayed claims in lint reports
- Triggers re-verification searches

### 7. Auto-Contradiction Detection (LOW PRIORITY)
Build a periodic agent task that:
- Reads all synthesis claims
- Uses LLM to identify semantic contradictions between claims
- Tags contradicted claims via `wiki_apply op=update_metadata`
- Currently contradictions are only detected if manually flagged

### 8. Slug Collision Guard (LOW PRIORITY)
Issue #65965 shows CJK titles can collide to `sources/page.md`. Until fixed upstream:
- Always provide explicit source IDs when ingesting non-ASCII content
- Or pre-check for slug collisions before ingesting

---

## Our Vault Health Snapshot

From `agent-digest.json` (as of 2026-04-19):

```json
{
  "pageCounts": { "entity": 0, "concept": 0, "source": 99, "synthesis": 27, "report": 6 },
  "claimCount": 36,
  "claimHealth": {
    "freshness": { "fresh": 36, "aging": 0, "stale": 0, "unknown": 0 },
    "contested": 0,
    "lowConfidence": 0,
    "missingEvidence": 0
  },
  "contradictionClusters": []
}
```

**Assessment:**
- ✅ All 36 claims fresh (vault <30 days old)
- ✅ Zero contested, low-confidence, or missing-evidence claims
- ⚠️ Zero entities or concepts (vault is source/synthesis-heavy)
- ⚠️ Bridge importing nothing (0 exported artifacts - known bug)
- ⚠️ Prompt digest injection OFF (no self-healing pressure)
- ⚠️ Average 1.4 claims per synthesis (thin coverage)
- ℹ️ Claim statuses: confirmed (26), active (5), proven (3), speculative (1), planned (1)

---

## Appendix: Key Code References

### Compile Pipeline Entry
```javascript
// cli-tAKZ-1z5.js line 1008
async function compileMemoryWikiVault(config) {
  await initializeMemoryWikiVault(config);
  let pages = await readPageSummaries(rootDir);
  const updatedFiles = await refreshPageRelatedBlocks({config, pages});
  if (updatedFiles.length > 0) pages = await readPageSummaries(rootDir);
  const dashboardUpdatedFiles = await refreshDashboardPages({config, rootDir, pages});
  const digestUpdatedFiles = await writeAgentDigestArtifacts({rootDir, pages, pageCounts});
}
```

### Freshness Assessment
```javascript
// cli-tAKZ-1z5.js line 39
function buildFreshnessFromTimestamp(params) {
  const daysSinceTouch = Math.floor((now - timestampMs) / DAY_MS);
  if (daysSinceTouch >= 90) return { level: "stale" };
  if (daysSinceTouch >= 30) return { level: "aging" };
  return { level: "fresh" };
}
```

### Claim Freshness Resolution
```javascript
// cli-tAKZ-1z5.js line 89
function assessClaimFreshness(params) {
  return buildFreshnessFromTimestamp({
    timestamp: resolveLatestTimestamp([
      params.claim.updatedAt,
      params.page.updatedAt,
      ...params.claim.evidence.map((e) => e.updatedAt)
    ])
  });
}
```

### Bridge Artifact Discovery (The Broken Path)
```javascript
// memory-state-CKpinHhR.js
async function listActiveMemoryPublicArtifacts(params) {
  // This returns [] because capability.publicArtifacts.listArtifacts
  // is a function reference lost during JSON serialization of cached plugin state
  return (await memoryPluginState.capability?.capability.publicArtifacts?.listArtifacts(params) ?? []);
}
```

### Prompt Digest Ranking
```javascript
// index.js - buildDigestPromptSection()
// Ranking formula for which pages get injected into prompts:
score = contradictions * 6 + questions * 4 + claims * 2
// Top 4 pages by score, top 2 claims per page
```

### Managed Block Replacement (Core Primitive)
```javascript
// memory-host-markdown-CtdshNaS.js
function replaceManagedMarkdownBlock(params) {
  const managedBlock = `${heading}\n${startMarker}\n${body}\n${endMarker}`;
  const existingPattern = new RegExp(`${heading}\n${startMarker}[\\s\\S]*?${endMarker}`, "m");
  if (existingPattern.test(original)) return original.replace(existingPattern, managedBlock);
  return `${trimmed}\n\n${managedBlock}\n`;  // Append if new
}
```

---

## Summary of Model Contributions

| Model | Focus Area | Key Unique Contributions |
|-------|-----------|-------------------------|
| **Opus 4.6** | Source code deep-dive (~4500 lines analyzed) | Root-caused bridge bug (function serialization), extracted full search scoring formula, confirmed zero event listeners, found unused config flags, discovered QMD wiki collection, traced complete wiki_apply flow |
| **GPT-5.4** | Docs, community, issues, release notes | Found 4 GitHub issues on bridge bug, analyzed release notes v2026.4.7/4.11, found community adoption signals (Reddit, Blink blog), identified Dreaming UI regression, surfaced slug collision issue |
| **Gemini 3.1 Pro** | Contrarian/comparison analysis | Compared against Obsidian/Notion/Mem/Roam, identified the missing "healing loop" clearly, enumerated 5 requirements for truly self-healing wiki, confirmed no system is truly self-healing out-of-box |

---

*This report should guide our wiki maintenance decisions. The TL;DR: enable `includeCompiledDigestPrompt`, build a nightshift cron for compile+lint, work around the bridge bug, and create entity/concept pages. The architecture is preem - the orchestration layer is what we need to build.*
