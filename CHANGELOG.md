# Changelog

All notable changes to **Mean Reversion T (MR-T)** are documented here. Versioning follows [Semantic Versioning](https://semver.org/).

## [2.0.0] — Production rewrite

The "9 分版" (ninth tuning iteration, previously internal) is now released as a production-grade, open-source, bilingual indicator.

### Added
- **Direction-parameterized state machine** — all position/setup/trade state collapsed into a single `MRState` object (`type`), with `+1/-1` direction parameters driving one shared `f_enter` / `f_exit` / `f_manage`. Removes ~200 lines of mirror-image long/short duplication (was the top structural risk: any fix had to be applied twice).
- **Single setup slot** — long/short setup state merged (they were mutually exclusive by construction); same-direction replacement blocked, opposite-direction flip preserved.
- **Cost-aware virtual P&L** — new `costBps` input (round-trip cost per side). Panel now shows last-trade and cumulative net P&L (ATR & %). Breakeven after trim is offset by round-trip cost, so "T-BE" is genuinely flat after fees.
- **Bilingual runtime UI** — new `lang` input (`zh` / `en`) switches the panel, chart labels, and runtime error messages via `f_tr()`. Alert messages and input labels are **bilingual-static** — Pine pins their text to compile-time `const string`, so they cannot switch at runtime (see the CE10123 fix below).
- **Versioning** — `SCRIPT_VERSION = "2.0.0"` displayed in the panel; `CHANGELOG.md`; bilingual `README` (EN + 中文); Mozilla Public License 2.0.
- **TradingView publishing kit** — `docs/tradingview-publish.md` with an English title, a standalone originality/usage description, and a House-Rules compliance checklist.

### Fixed
- **Trim/close lifecycle jump** — previously a single bar reaching both the trim and the close target closed the trade without ever recording the trim (final was checked before partial). Close is now gated on `partialTaken`, so the lifecycle always runs `T-trim → T-close`.
- **Dead breakeven condition** — removed the always-true clause in the BE classification.
- **Duplicated higher-timeframe helpers** — `f_prevER` / `f_prevSlopeATR` replaced by thin `[1]` wrappers over the shared `f_er` / `f_slopeATR` (identical HTF repaint-safety semantics, one source of truth).
- **Identical long/short deceleration expressions** — merged into a single `shockDecelerating`.
- **CE10088 event routing** — one-bar display events are no longer written from inside functions (Pine forbids reassigning *any* global scalar in a function, `var` or not; only object fields are writable). `f_enter` / `f_exit` / `f_manage` now record events onto `MRState` capture fields (`eventKind` / `eventDir` / `eventReason`), which the global scope derives back into the event series.
- **CE10123 const-string bounds** — `plotshape` `text` and `alertcondition` `message` require compile-time `const string`, so dynamic language switching is impossible there. Chart labels now use `label.new` (its `text` is `series string`, so `f_tr()` works); alert messages are bilingual literals; `f_tr` parameters are `simple string` so it stays usable in `simple`/`series` contexts.
- **`max_labels_count` stays at the 500 ceiling** (Pine rejects higher values, CE10178).

### Changed
- Setup/entry/exit **events consolidated** from 20 booleans into typed one-bar events (`setupEvent*`, `entryEvent*`, `partialEvent*`, `exitEvent*` + `exitReason`), so every plot label and alert derives from one source.
- `activeStop` (incl. breakeven offset) computed once in `f_activeStop` and shared by management logic and the risk-line plot.
- Internal variable renames to match the object model (no behavioral intent change).

### Notes
- All 60 prior input parameters are **preserved** — the parameter surface is unchanged.
- Input *labels* remain Chinese (Pine input titles are compile-time `const string` and cannot be localized by a runtime toggle). Runtime UI is fully bilingual.

## [1.0.0] — "9 分版" (9th tuning iteration, pre-release)

Final pre-release tuning of the dual-engine design on 15m. This version is superseded by the 2.0.0 production rewrite; no changelog was kept for earlier tuning iterations.
