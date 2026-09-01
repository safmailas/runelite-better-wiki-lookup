# runelite-better-wiki-lookup

A RuneLite Plugin Hub plugin that extends the built-in Wiki lookup: toggle a
lookup tool from a wiki icon near the minimap, then left-click **anything** —
in-world entity or interface widget (skill guides, monster examine, game
options) — to open the matching OSRS Wiki page. Known targets land directly;
unknown or ambiguous targets are resolved by a local model (Ollama) and the
result is remembered for next time. Falls back to `Special:Search` when no local
model is available.

**Status:** design approved, implementation not started.

See [`docs/superpowers/specs/2026-08-31-wiki-lookup-plus-design.md`](docs/superpowers/specs/2026-08-31-wiki-lookup-plus-design.md)
for the full design spec.

## License

BSD 2-Clause (to be added with the initial code scaffold).
