# Wiki Lookup Plus — Design Spec

**Date:** 2026-08-31
**Status:** Approved (design); pending spec review
**Target repo:** `runelite-wiki-lookup-plus` (new standalone repo)
**Reference:** [`safmailas/runelite-ping-plugin`](https://github.com/safmailas/runelite-ping-plugin) — used as the AI-SDLC methodology template and Gradle/CI scaffold source.

---

## 1. Summary

A RuneLite Plugin Hub plugin that extends the click-to-lookup behaviour of the
built-in Wiki plugin. The user toggles a "lookup tool" from a wiki icon near the
minimap; the next left-click on anything — an in-world entity **or an interface
widget** (skill-guide row, monster examine text, game-option label) — is consumed
and resolved to an OSRS Wiki page, which opens in the system browser.

Resolution order:

1. **Cache hit** → open the stored page immediately.
2. **Known name** (real item / NPC / object) → open `/w/<Title>` directly.
3. **Unknown / ambiguous / free widget text** → local Ollama returns the best
   page title plus a one-paragraph summary → open that page and persist the
   result. If Ollama is unavailable → open `Special:Search?search=<text>`.

Every newly resolved target is written to a single on-disk cache so the next
lookup of the same thing lands directly without AI or search.

### Novelty vs. the built-in Wiki plugin

- Coverage extends to arbitrary interface widgets, not just NPCs/objects/items.
- Ambiguous or unnamed targets are disambiguated by a local model.
- Confirmed resolutions (and their AI summaries) are remembered across sessions.

### Non-goals (v1)

- No hover tooltip card.
- No sidebar panel, no infobox, no in-game rendering of summaries.
- No right-click submenu, no chat command.
- No cloud AI, no bundled model, no non-Ollama local backends.
- No new runtime dependencies beyond what RuneLite already bundles.

---

## 2. Behaviour detail

### Activation

- An `IconOverlay` draws a clickable wiki icon adjacent to the minimap.
- Clicking it flips `active`. While `active`, the cursor context is "lookup mode".
- Mode persists until the icon is clicked again. Config toggle
  `deactivateAfterLookup` (default `false`) makes it one-shot.
- Plugin disable, logout, or client shutdown forces `active = false`.

### Lookup click

- When `active`, `LookupClickHandler` consumes the next **left**-click
  (`MouseEvent` is marked consumed so the game does not act on it).
- Right-click, camera drag, and UI clicks that are not on a resolvable target
  are ignored and leave `active` unchanged.
- The click context is passed to `TargetExtractor`.

### Resolution & open

- `Resolver.resolve(target)` runs on RuneLite's scheduled executor (never the
  client thread).
- On a cache hit the browser opens effectively immediately; on a miss the
  resolve step is expected to complete in under ~2s (Ollama timeout) before the
  browser opens.
- The page opens via `net.runelite.client.util.LinkBrowser.browse(url)`.

---

## 3. Components

Each unit has one purpose, a narrow interface, and is testable in isolation.

| Component | Responsibility | Depends on |
|---|---|---|
| `WikiLookupPlusPlugin` | Lifecycle; register `IconOverlay` + mouse listener; owns the `active` flag; wires dependencies. | RuneLite `Plugin`, `OverlayManager`, `MouseManager`, `ScheduledExecutorService` |
| `WikiLookupConfig` | `enableAi` (default false), `ollamaHost`, `ollamaPort`, `ollamaModel`, `deactivateAfterLookup`, `maxCacheEntries`. | RuneLite `Config` |
| `IconOverlay` | Draw the minimap-adjacent icon; expose its screen bounds; report clicks on it. | `Client`, `OverlayManager` |
| `LookupClickHandler` | `MouseListener`; when `active`, consume the next left-click and build a click context (screen point + hovered `MenuEntry` + hovered `Widget`). | `Client`, `MouseManager` |
| `TargetExtractor` | Click context → `LookupTarget { rawText, type, id? }`. Classifies NPC / GAME_OBJECT / GROUND_ITEM / ITEM / WIDGET_TEXT. Strips level/quantity/colour tags and menu prefixes. | `Client` (read-only), `ItemManager` for item names |
| `Resolver` | `LookupTarget → ResolvedPage { url, pageTitle, source }`. Pipeline: `ResolutionCache` → direct-name → `OllamaClient` → `Special:Search`. Stateless beyond its injected collaborators. | `ResolutionCache`, `OllamaClient`, `WikiUrls` |
| `OllamaClient` | `POST http://{host}:{port}/api/generate` with a fixed prompt; parse `{ title, summary, confidence }`; ~2s connect+read timeout; any failure returns `null`. Also a cheap `isAvailable()` probe used to gate the AI path. | Injected `OkHttpClient`, Gson |
| `WikiUrls` | Build and URL-encode `/w/<Title>` (spaces → underscores) and `Special:Search?search=<term>`. Pure functions. | — |
| `ResolutionCache` | Single JSON file `RUNELITE_DIR/wiki-lookup-plus/resolutions.json`: `normalizedKey → { pageTitle, summary, source, savedAt }`. Bounded LRU (`maxCacheEntries`, default 1000). Load once on startup; debounced async save on write. All disk I/O lives here. | `RUNELITE_DIR`, Gson |

### `LookupTarget` types

- `NPC`, `GAME_OBJECT`, `GROUND_ITEM`, `ITEM` — carry a game id where available;
  a resolved canonical name means the direct-name path applies.
- `WIDGET_TEXT` — raw visible text of the clicked widget, no id semantics; always
  goes cache → AI → search (never direct-name).

### `ResolvedPage.source`

`DIRECT` | `AI` | `SEARCH` — recorded in the cache entry for later inspection
and for the methodology write-up's telemetry section.

---

## 4. State management

Deliberately minimal, decided up front:

- **Transient state:** exactly one field — `boolean active` on
  `WikiLookupPlusPlugin`. Flipped only by an icon click or a lifecycle event.
  No other class holds lookup-mode state.
- **Persistent state:** exactly one owner — `ResolutionCache`. Mutated only
  through `get(key)` / `put(key, entry)`. No other class touches the file or
  holds cached entries.
- **Resolution logic is stateless:** `TargetExtractor` and `Resolver` are pure
  transformations over their inputs plus injected collaborators; they can be
  reasoned about and tested without any setup beyond constructing fakes.
- **Threading:** the client thread only reads `active` and consumes the click.
  All resolution, network, and disk work happens on the scheduled executor.
  `active` is `volatile`; `ResolutionCache` guards its map with a single lock.

---

## 5. Data flow

```
left-click (active == true)
  └─ LookupClickHandler consumes event
       └─ TargetExtractor → LookupTarget
            └─ executor: Resolver.resolve(target)
                 ├─ ResolutionCache.get(key)         → hit  → ResolvedPage(source=cache's source)
                 ├─ direct-name (named entity)       → ResolvedPage(DIRECT)  + cache.put
                 ├─ OllamaClient.generate(...)       → ResolvedPage(AI)      + cache.put(title+summary)
                 └─ WikiUrls.search(rawText)         → ResolvedPage(SEARCH)  + cache.put
            └─ LinkBrowser.browse(page.url)
  └─ if deactivateAfterLookup: active = false
```

Normalized cache key: `lowercase(trim(collapse_whitespace(rawText)))` +
`"|" + type`. Keeping `type` in the key avoids a widget-text "Attack" colliding
with an NPC "Attack".

---

## 6. Error handling

| Situation | Behaviour |
|---|---|
| Click resolves to empty/blank text or no target | Ignore the click; leave `active` unchanged; no browser open. |
| `OllamaClient` unreachable, times out, or returns unparseable body | Treat as `null`; fall through to `Special:Search`. Logged at `debug`. |
| `enableAi == false` | AI path skipped entirely; unknown targets go straight to `Special:Search`. |
| `LinkBrowser.browse` fails | Logged at `debug`; no user-facing error. |
| `resolutions.json` missing | Start with an empty cache. |
| `resolutions.json` corrupt / unparseable | Discard, log at `warn`, back it up to `resolutions.json.bak`, start empty. |
| Cache write fails | Log at `debug`; in-memory entry retained; retry on next debounced flush. |
| LRU over capacity | Evict least-recently-used entries on `put` until within `maxCacheEntries`. |

---

## 7. Testing

JUnit 4 + Mockito, consistent with RuneLite conventions. No live network, no
live client.

| Test class | Cases |
|---|---|
| `TargetExtractorTest` | NPC menu entry with combat level suffix; item menu entry with quantity; game object; ground item; plain widget text; widget text with colour tags; empty/whitespace target → no `LookupTarget`. |
| `ResolverTest` | Cache hit returns stored source; named entity → `DIRECT` URL + cache write; unknown + AI enabled → uses `OllamaClient` stub, writes title+summary; AI disabled → `SEARCH`; AI returns `null` → `SEARCH`; verifies executor usage (no client-thread calls). |
| `OllamaClientTest` | Well-formed JSON → parsed struct; HTTP 500 → `null`; socket timeout → `null`; truncated/garbage body → `null`; `isAvailable()` true/false. Uses `MockWebServer` or a stubbed `OkHttpClient`. |
| `ResolutionCacheTest` | Put then reload from disk; LRU eviction order; corrupt file recovery + `.bak` creation; concurrent `put` under lock; debounced save fires once for a burst. |
| `WikiUrlsTest` | Spaces → underscores; apostrophes, ampersands, unicode encoded correctly; `Special:Search` term encoding; leading/trailing whitespace trimmed. |

Manual test matrix (dev client, recorded in `docs/REPO_READINESS.md`):
NPC, object, ground item, inventory item, equipped item, skill-guide entry,
monster examine text, a Settings/Options label — each with AI on and off, and a
second lookup of the same target confirming a direct cache hit.

---

## 8. Repository & build

- **New repo** `runelite-wiki-lookup-plus`.
- Gradle scaffold copied from `runelite-ping-plugin`: `build.gradle`,
  wrapper, `example` run task / dev client launcher, `checkstyle` config.
- `sourceCompatibility` / `targetCompatibility` = **JDK 11**; CI also builds on
  21 to confirm forward compatibility.
- GitHub Actions: `build`, `test`, `checkstyleMain` on push and PR.
- License: **BSD 2-Clause**; header on every `.java` file.
- Assets: `icon.png` (Plugin Hub icon) + at least two screenshots (icon in
  place; a resolved lookup) committed under `docs/`.
- **Zero new runtime dependencies.** Uses RuneLite-provided `OkHttpClient`,
  Gson, Guice, Lombok only.

---

## 9. Non-code deliverables

### `docs/METHODOLOGY.md`

The AI-SDLC process, generalized from how the ping plugin was built:

1. Brainstorm → approved design (this document's lineage).
2. Written spec committed before code.
3. Implementation plan with discrete, independently reviewable tasks.
4. Subagent-driven implementation, one task per unit from §3.
5. Review gates: spec self-review, per-task code review, pre-submission
   security/scope review.
6. Artifacts retained under `docs/superpowers/` (spec, plan, review notes).

### `docs/PLUGIN_HUB_SUBMISSION.md`

Runbook:

- Fork `runelite/plugin-hub`, branch per plugin.
- Add `plugins/wiki-lookup-plus` manifest: `repository=<https clone url>`,
  `commit=<40-char hash>`.
- PR description template, including the review-risk framing:
  *localhost-only optional AI, plugin fully functional without it, only outbound
  network is to `oldschool.runescape.wiki` and `localhost`, no telemetry, no
  bundled secrets.*
- CI expectations and how to read failures.
- Dependency-verification note (nothing new to add).

### `docs/REPO_READINESS.md`

Checklist that gates opening the submission PR: CI green on 11 and 21, all §7
tests passing, checkstyle clean, license headers present, icon + screenshots
committed, manual test matrix executed, `docs/METHODOLOGY.md` and
`docs/PLUGIN_HUB_SUBMISSION.md` complete.

---

## 10. Deferred to v2

- In-game viewable summaries (re-click shows the stored summary in chat).
- Hover tooltip card.
- Right-click "Wiki" submenu with sub-options.
- Chat command (`::wikilookup <text>`).
- Additional local backends (llama.cpp, LM Studio) via a configurable endpoint.
- Wiki content prefetch / infobox stat extraction.
