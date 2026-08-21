# agentic.md — ctpbee fork: architecture, conventions, and local changes

Working notes for AI agents and developers on this local editable checkout of
[ctpbee](https://github.com/ctpbee/ctpbee). **Violating the conventions below
silently corrupts tick data or breaks hot-path throughput.**

## What this fork feeds

This checkout is the market-data engine of the local "hive" stack:

- `hive_recorder` (in the `hive` repo) embeds a `CtpbeeApi` extension and
  streams every tick to the hive gRPC server;
- `ctpbee.date.trade_dates` is the source of the futures trading calendar
  (via `hive/extral_update/update_trading_date.py` → hft DB `trading_date`
  table, and as the recorder's in-process gate `_is_trading_time`);
- `.ctpbee/contract/*.json` feeds `hive_tool`'s `contracts` table
  (pricetick / multiplier for slippage and PnL).

Changes here are **upstreamable by design**: minimal diffs, behavior-preserving,
English comments explaining motivation and equivalence.

## Architecture map

```
C++ CTP callbacks (interface/<broker>/md_api.py, td_api.py)
  └─ onRtnDepthMarketData → build TickData → Event → blinker signal
       common_signals.tick_signal            (module-level, tick/bar/timer)
       app.app_signal.<event>_signal         (per-app: order/trade/...)
  └─ Recorder (record.py, the data center)
       process_tick_event (@call decorator, helpers.py)
       ├─ app.tools[*].on_tick(tick)
       └─ app._extensions → CtpbeeApi subclasses (e.g. hive_recorder)
```

| Module | Role |
|---|---|
| `app.py` | `CtpBee` container: config, extensions, tools, engine lifecycle |
| `interface/ctp`, `ctp_rohon`, `ctp_mini` | Broker adapters (MD = market data, TD = trading). The three `md_api.py` copies share the same tick-parsing shape — keep them in sync |
| `constant.py` | `Entity` data objects (`TickData`/`OrderData`/...) under `@frozen` attribute protection; `Event` |
| `record.py` | `Recorder`: latest tick/order/trade per symbol, local position manager, fan-out to tools + extensions |
| `signals.py` | blinker signals; common (tick/bar/timer) vs per-app |
| `helpers.py` | `call` decorator (event → extension dispatch), shared utilities, `build_tick_datetime` |
| `func.py` | `Hickey` 7×24 process manager (`auth_time` session gating; hive_recorder shifts open times by −300 s), legacy `get_current_trade_day` (hive uses its own calendar) |
| `date.py` | Hardcoded `trade_dates` list (1990 → **2026-12-31**, see risks) |

## Hot-path optimization (2026-08-21, this branch)

Per-tick Python-side fixed cost: **~412 µs → ~3.4 µs** (~120×). Two changes:

1. **`build_tick_datetime()` in `helpers.py`** replaces the per-tick
   `datetime.strptime` in all three interfaces:
   - caches the `(year, month, day)` tuple per `ActionDay` (constant within a
     session; cache grows a few entries per day, keyed by `date.today()` for
     the fallback so midnight rolls over automatically);
   - direct `datetime(...)` construction (~0.3 µs vs ~3.7 µs);
   - **millisecond quantization to 100 ms is preserved deliberately**
     (`int(UpdateMillisec / 100)` legacy semantics: 550 ms → `".5"` → 500 ms).
     Changing to full ms precision is a downstream contract decision
     (dedup keys, minute-bar boundaries), not a free fix;
   - DCE branch (`use_today=True`): ActionDay is the trading day on DCE, so
     local today + UpdateTime is used (upstream issue #165);
   - empty/invalid ActionDay falls back to today — the old code raised
     `ValueError` here, the only intentional behavior delta.
2. **`constant.py`**:
   - `__set_attr__` (the `@frozen` guard) now reads the caller function name
     via `sys._getframe(1).f_code.co_name` instead of
     `inspect.getframeinfo(...)`: identical semantics (same frame's function
     name, same underscore-prefix allow rule, same error message),
     ~14 µs → <1 µs per attribute write;
   - `Entity.__init__` writes constructor kwargs with one
     `self.__dict__.update(mapping)` instead of ~40 `setattr` calls —
     equivalent because the guard always allowed `__init__` writes straight
     into `__dict__`; protection after construction is unchanged.

Benchmarks (Windows, Python 3.13): datetime parse 3.7 → 0.3 µs (11×);
`TickData` construction 408 → 3.1 µs (130×).

### Verified equivalence

`tests/test_hotpath_optimization.py` (run `python tests/test_hotpath_optimization.py`,
no DB / no broker needed): 10/10 checks. The legacy `strptime` implementation
is inlined as the oracle: 100 ms quantization matrix (0/49/50/99/100/550/999),
DCE branch, cache isolation across ActionDays, `@frozen` allow/deny by caller
name, TickData 40-field construction + `__post_init__` `local_symbol` +
`_to_dict`. Import smoke covers all three interfaces (no circular imports).

## Conventions and known edges (read before touching)

- **`@frozen` rule**: only callers whose function name starts with `_` may
  write `Entity` attributes. Do not weaken the guard; bulk `__init__` writes
  are equivalent precisely because `__init__` was always allowed.
- **DCE timestamps use the local machine date** (`date.today()`) — assumes a
  Beijing-time host. Same class of assumption exists in hive-side code.
- **CTP "no price" sentinels pass through raw** (bid/ask ≈ DBL_MAX on
  limit-locked markets) — normalize downstream if a consumer cares.
- **`date.py` calendar expires 2026-12-31**: after that date the recorder's
  `_is_trading_time`, `update_trading_date.py`, and the `TradingCalendar`
  fallback silently stop recognizing trading days. Replace the calendar source
  (hft `trading_date` table is the intended single source of truth) or bump
  the list before 2027.
- **Keep the three `md_api.py` copies in sync**; shared helpers
  (`build_tick_datetime`) prevent drift for the parsing path.

## Changelog

| Date | Change |
|---|---|
| 2026-08-21 | Hot-path optimization: `build_tick_datetime` (+ActionDay cache) shared by ctp/ctp_rohon/ctp_mini; `@frozen` guard via `sys._getframe`; `Entity.__init__` bulk init. Equivalence suite `tests/test_hotpath_optimization.py` (10 checks) + benchmarks. No public API change; only behavior delta: empty ActionDay now falls back to today instead of raising. |
