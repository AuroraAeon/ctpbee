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
       ├─ LocalPositionManager.update_tick (data_handle/local_position.py,
       │    one dict lookup; PositionHolding recomputes pnl only when the
       │    price / position signature changed — see changelog 2026-08-21b)
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

## Test infrastructure (hard rule: every change must land with passing tests)

All suites are standalone scripts (no pytest dependency, no CTP callback, no
real Redis): `python tests/<name>.py` exits non-zero on failure.

| Suite | Checks | Covers |
|---|---|---|
| `test_hotpath_optimization.py` | 10 | `build_tick_datetime` equivalence vs legacy strptime oracle, frozen guard semantics, TickData bulk init |
| `test_position_hotpath.py` | 4 | local-position skip-on-unchanged (3000-step randomized oracle, staleness guard, skip observability) |
| `test_dispatcher.py` | 11 | Dispatcher relay hardening (queue isolation, failure containment, LRU, overflow drop) |
| `test_tool_register.py` | 15 | standalone tool_register primitive (order/dedup/unregister, isolation, key isolation, snapshot) + Tool integration |
| `test_config_env.py` | 8 | `Config.from_envvars` (override, typed parsing, fallback, strict mode, prefix) |
| `test_upper_layers.py` | 61 | end-to-end upper-layer simulation with a FakeApp + isolated global signals: constant data objects & frozen protection, Recorder event flow (tick/order/trade/position/account/contract/last, INSTRUMENT_INDEPEND, active-order bookkeeping, init-once), local position deep cases (SHFE vs non-SHFE close priority, frozen spill, order splitting, yesterday conversion), DDDR/UDDR serialization round-trips, Hickey session windows / trade-day derivation, CtpbeeApi `__call__`/`route`/`register`/`subscribe`, Config loaders |

Behavioral quirks locked by characterization (see suite comments):
`PositionData.local_position_id` uses `str(enum)` (`ag2612.SHFE.Direction.LONG`);
`OrderData` accepts both string and enum exchanges while
`CancelRequest.__post_init__` requires the enum (calling
`create_cancel_request()` on a string-exchange OrderData raises);
`main_contract_mapping` keys strip digits from the whole local_symbol
(`ag2612.SHFE` → `AG.SHFE`). Formerly `DDDR.encode→parse` was not
self-consistent — fixed (see changelog 2026-08-21g).

## Changelog

| Date | Change |
|---|---|
| 2026-08-21 | Hot-path optimization: `build_tick_datetime` (+ActionDay cache) shared by ctp/ctp_rohon/ctp_mini; `@frozen` guard via `sys._getframe`; `Entity.__init__` bulk init. Equivalence suite `tests/test_hotpath_optimization.py` (10 checks) + benchmarks. No public API change; only behavior delta: empty ActionDay now falls back to today instead of raising. |
| 2026-08-21b | Position hot path (`data_handle/local_position.py`): `update_tick`/`update_bar` skip pnl recompute when inputs are unchanged — pnl is a pure function of `(last_price, pre_settlement, positions, avg prices, size)`; the signature is compared against **current** attributes so external mutations (trade/position callbacks, yesterday-holding conversion) always trigger a recompute on the next tick (no stale pnl). `LocalPositionManager.update_tick/update_bar` use a single dict lookup. Benchmarks (both-side positions): recompute 0.31 µs → skip 0.10 µs (3×; ~60-80% of real ticks carry an unchanged price). Equivalence suite `tests/test_position_hotpath.py` (4 checks: 3000-step random trade/tick oracle, staleness guard, skip observability, changing-price correctness). |
| 2026-08-21c | Dispatcher (`stream.py`, `Mode.DISPATCHER` Redis relay) reliability hardening — wire protocol and callback signatures unchanged: ① `on_tick` only enqueues (hot path); serialization + publish run on a background thread, bounded queue (100k) drops oldest on overflow with a running counter; ② all publications go through `_publish`, which throttles warnings and never raises — a Redis outage can no longer propagate into the CTP callback chain; ③ the upstream listen loop reconnects on a fixed 5 s backoff and isolates per-message failures (`_handle_upstream`) so one bad order no longer kills the listener thread; ④ `order_key_map` is a bounded LRU (10k; evicted ids fall back to the pre-existing default index 0); ⑤ `send_order` returning empty no longer pollutes the map; ⑥ `UDDR` parse failures are logged (throttled) instead of silently swallowed; ⑦ added `close()` for best-effort thread shutdown. Suite `tests/test_dispatcher.py` (11 checks, FakeRedis injection, no real Redis needed). |
| 2026-08-21d | `tool_register` extracted into a **standalone stdlib-only primitive** (`ctpbee/tool_register.py`) so other libraries can use it without importing the ctpbee app machinery — ctpbee's own `Tool` is now just its first consumer. The decorator accepts any hashable key (string / enum / None, not just `ToolRegisterType`); hooks receive the decorated method's **return value** (existing contract, now documented). Fixed in the process: `functools.wraps` preserved; snapshot iteration (register/unregister during a fire no longer raises `Set changed size`, takes effect next round); per-hook exception isolation with throttled logging (a bad hook can no longer propagate into the tick dispatch chain); ordered + deduplicated hook lists (sets previously gave nondeterministic order); lazy `_linked` creation (never-subscribed objects cost one `getattr`; underscore-named creation keeps frozen `Entity` hosts writable); `add_func` raises a clear `ValueError` on unknown types (was `None.add` AttributeError); `remove_func` added for symmetry; **base `Tool.on_tick/on_order/on_trade/on_position/on_account` are now decorated**, so `subscribe()` works out of the box (previously the mechanism never fired unless the user decorated their own subclass — a silent dead-end). New public helpers `register_tool_hook(obj, key, func)` / `unregister_tool_hook(obj, key, func)` exported from `ctpbee`; `from ctpbee.func import tool_register` keeps working via re-export. Suite `tests/test_tool_register.py` (15 checks: standalone plain-class usage, order/dedup/unregister, exception isolation, key isolation, snapshot semantics, zero-subscription path, wraps, Tool integration incl. base activation and explicit errors). |
| 2026-08-21e | `Config.from_envvars(prefix="CTPBEE_", silent=True)`: reads config from prefixed env vars — values JSON-parsed (bool/int/float/list/dict; `CONNECT_INFO` as a JSON string), unparseable values fall back to the raw string, keys must be uppercase after prefix stripping (consistent with `from_mapping`), `silent=False` raises on non-conforming entries. Recommended precedence: `from_json` then `from_envvars` so env overrides the file. Suite `tests/test_config_env.py` (8 checks). |
| 2026-08-21f | Comprehensive upper-layer simulation suite `tests/test_upper_layers.py` (61 checks, no CTP / no real Redis — FakeApp + isolated global signals). Covers the full Recorder event flow, local-position deep cases (close priority by exchange, frozen spill, SHFE order splitting, yesterday conversion), DDDR/UDDR serialization round-trips, Hickey session windows, trade-day derivation, CtpbeeApi dispatch (`__call__`, `route`, `register`, `subscribe`) and Config loaders. Established the standing rule: **every change lands with passing tests** (109 checks across 6 suites total). Characterization-locked upstream quirks are listed in the test-infrastructure section above. |
| 2026-08-21g | Fixed `DDDR.encode→parse` self-inconsistency (the long-standing `fixme`): `loads` restores `dumps`-produced payloads directly as entity objects, so `__parse__` now adopts the object as-is and only falls back to the key-sniffing reconstruction when the inner data is a plain dict (legacy hand-built payloads keep working). Round-trip checks for TickData/OrderData/TradeData/ContractData added to `test_upper_layers.py` (now 65 checks; 113 total across 6 suites). |
