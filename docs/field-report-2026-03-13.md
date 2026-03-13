# Nunchi Agent-CLI Field Report — 2026-03-13

Real-world testing on mainnet with $83 account. OpenClaw agent as operator (autonomous, no human in the loop for execution). Here's everything we hit.

---

## 🔴 P0 — Blocks Trading

### 1. Builder Fee Can't Be Disabled

**Problem:** `hl_adapter.py:place_order()` line 188-190 silently overrides any disabled builder config:
```python
if builder is None:
    builder = _default_builder()  # always falls back to Nunchi 10bps fee
```

When a user sets `builder_address: ""` and `fee_rate_tenths_bps: 0` in their YAML config, `to_builder_info()` correctly returns `None`. But `place_order()` treats `None` as "not provided" and re-attaches the default Nunchi fee. Every order fails with "Builder fee has not been approved."

**Fix:** Use a sentinel to distinguish "not provided" from "explicitly disabled":
```python
_NO_BUILDER = object()

def place_order(self, ..., builder=_NO_BUILDER):
    if builder is _NO_BUILDER:
        builder = _default_builder()
```

We patched this locally and it works. Also: APEX loads builder config from `TradingConfig()` (env/defaults) instead of from the YAML config file, so even with the YAML fix, APEX ignores it. Need `BUILDER_ADDRESS=""` env var as workaround.

**Impact:** 100% of orders rejected on mainnet if builder wallet is unfunded. This is the #1 blocker for anyone running on mainnet.

### 2. Price Rounding Kills Sub-Dollar Assets (kPEPE, kSHIB, kBONK, etc.)

**Problem:** `hl_adapter.py:_round_price()` uses a hardcoded tick size of 0.1:
```python
def _round_price(price: float, tick: float = 0.1) -> float:
    return round(round(price / tick) * tick, 8)
```

kPEPE trades at $0.003447. `round(0.003447 / 0.1) * 0.1` = `0.0`. Every order for sub-$0.10 assets gets price=0.0 and is rejected with "Order has invalid price."

**Fix:** Fetch the actual tick size from the HL API (`meta` endpoint has `szDecimals` per asset — need the price equivalent). Or at minimum, use `significantFigures` from the meta response.

**Impact:** APEX Pulse detects volume surges on kPEPE (72x), kBONK (100x), MEME (315x) — these are the highest-signal assets — and can't trade any of them.

---

## 🟡 P1 — Degrades Performance

### 3. Rate Limiting (429) After Pulse Scan

Pulse scans 229 assets via `allMids` + candle endpoints. By the time it finishes (~20s), the rate limit budget is exhausted. The immediate entry attempt after scan hits 429. The 50ms sleep between Pulse scan requests (line 420) isn't enough.

**Suggestion:** Add a configurable delay between scan completion and entry execution (e.g., 2-3 seconds). Or batch the entry order with the next tick instead of firing immediately.

### 4. APEX `--config` Doesn't Load Builder Settings

`cli/commands/apex.py` line 233-237 loads builder from `TradingConfig()` (which reads env/defaults), not from the YAML config passed via `--config`. So the YAML `builder:` section is ignored for APEX. This is inconsistent with `hl run` which reads builder from the config.

### 5. No `--dry-run` Flag on `hl apex run`

`hl run` has `--dry-run` but `hl apex run` doesn't. The YAML `dry_run: true` field isn't read by ApexConfig. Only workaround is to let orders fail naturally (no builder fee, or no funds).

---

## 🟢 P2 — Polish / DX

### 6. Pulse Scan History Bootstraps From Prior Runs

When APEX restarts with `--fresh`, `scan_history` still has entries from previous runs (looks like it reads from disk). This means `history=5` on tick 1, which shouldn't happen on a fresh start. Signals fire immediately without the required baseline scans.

### 7. `** NO FUNDS DETECTED **` Warning Is Misleading

The wallet has $83.22. The warning fires because the check looks at `withdrawable` or free margin, but funds are there. Confusing for operators.

### 8. Entry Order Type Defaults to ALO

`_execute_enter()` defaults to ALO (maker-only), which gets rejected if price crosses the spread, then falls back to GTC. For momentum entries (catching a move), you want IOC or at least GTC upfront. ALO → GTC fallback adds latency and an extra API call on every entry.

---

## ✅ What Works Well

- **Pulse detection is legit.** 13-17 signals per 60s tick across 229 assets. VOLUME_SURGE, OI_BREAKOUT, FUNDING_FLIP — all firing with real data.
- **IOC fix (Phase 1) is solid.** MM strategies correctly default to GTC now. This was the critical blocker from last week.
- **460 tests passing.** Test coverage is strong.
- **Autoresearch integration is well-designed.** `backtest_apex.py` + `reflect_adapter.py` + `autoresearch_program.md` is the right architecture.
- **The APEX → Pulse → Radar → Guard pipeline architecture is clean.** Zero LLM in the tick loop, all mechanical. This is the right call.

---

## Environment

- Account: $83 mainnet, 0xBea01B04...
- Platform: OpenClaw agent on Linux (Debian 12)
- Python 3.11, agent-cli @ d9a1440
- Strategies tested: avellaneda_mm (live), APEX (dry-run)
- Runtime: ~2 hours of live testing
