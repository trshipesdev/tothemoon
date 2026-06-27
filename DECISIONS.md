# Bot Design Decisions — locked, do not reverse without explicit user sign-off

These are decisions made through live data analysis. Each one has a specific reason.
Future sessions: READ THIS before changing any of these.

---

## Risk Controls

### Dollar stop: max($12, 10% of deployed) — KEEP
`CONFIG["moonshot"]["dollar_stop_usd"] = 12.0`
Real data showed STAR lost -$32.89, STAR stop should have fired at -$12. BBP -$29, should have been -$12.
The stop exists. The gap risk (rug before next 2s tick) means actual fills exceed the stop — that's a live-mode
problem. In shadow mode this cap is applied retroactively on the exit. DO NOT raise this or remove it.

### Post-loss cooldown: 30 min block on same token — KEEP
`recently_exited[sym]` checked in scanner for 30 min after a loss exit.
BOWIE re-entered 6 times in 2 minutes burning -$84. YAPPR, BBP, PISSBOT same pattern.
The check was broken (inside `if is_new:`) — fixed 2026-06-27. Never move it back inside `is_new`.

### Cooldown check must be OUTSIDE `if is_new:` — NEVER REVERT
`is_new` means "is this a young token launch", NOT "is this a new position".
Older tokens bypassed the cooldown entirely when the check was inside `if is_new:`.
The fix: moved `recently_exited` check to before the `if is_new:` block in scan_candidates().

### liq_prev seeded on buy — KEEP
`STATE["liq_prev"][symbol] = liq_usd` set in shadow_buy when creating a new position.
Without this, the BirdEye fallback (which uses 100k default liq) contaminated liq_prev,
causing a false rug alarm on the next DexScreener tick (real liq ~28k < 100k × 60% = false exit).
HOOD lost -$0.73 due to this before the fix on 2026-06-27.

### BirdEye liq fallback uses entry_liq not 100000 — KEEP
`bd_liq = float(bd.get("liquidity") or 0) or p.get("entry_liq", 0) or 0`
The old `or 100000.0` default poisoned the rug detector with a fake baseline.

---

## Seed Mode (small vault / $100 start)

### Dynamic sizing: bet = min(base, avail × 40%), $10 floor, 1.5× buffer gate — KEEP
`CONFIG["moonshot"]["dynamic_sizing"] = True` when vault < ~$500 or in seed mode.
Prevents blowing a $100 vault on a single $48 degen bet. Losses stay proportional.
With $100 vault: first bets are $4-10, compounds up from wins.
sim100 showed $100 → $243 (+143%) vs real $1000 → $1132 (+13%) because cooldown + capped losses.

---

## Position Limits

### max_open_positions = 12 — intentional
Memecoins crash together in market-wide flushes. Cap prevents correlated loss across all positions.
The bot rarely fills all 12 slots — DexScreener doesn't surface 12 qualifying candidates per tick.

### base_size_usd = 30.0 (degen: ×1.6 = $48/bet) — calibrated
Was $50, reduced to $30 to spread bets across more tokens.
With $1000 vault: $48/bet = 4.8% per position. 12 slots = max $576 deployed = 57% of vault.
per_token_cap_room(symbol) subtracts existing position size — total per-token exposure capped at 12% of deployable.
Was symbol-blind (flat 12% per bet regardless of existing position). TOPDOG 8×$70=$551, ? token 4×$67=$268,
both blew past $90 because each individual buy was under the cap. Fixed 2026-06-27.

---

## Live Trading Pre-Blockers (DO NOT GO LIVE WITHOUT THESE)

1. **Per-chain wallet balance**: `per_chain_room()` must check actual wallet balance, not just % of total vault.
   SOL wallet and EVM wallet are separate. A $1000 vault with $800 on SOL can't spend $300 on Base.

2. **Solana tx confirmation**: `_sol_submit_tx` fires and forgets. If tx fails to land, bot thinks it sold.
   Need to poll for confirmation or mark position as "sell_pending".

3. **EVM tx blocking**: `wait_for_transaction_receipt(timeout=120)` blocks the engine loop for up to 2 min.
   All other positions unmonitored during that time. Fix: run EVM sells in background thread.

---

### buy_ratio_min: 0.40 (was 0.45) — DO NOT raise back without data
FRONT (+511%) and MASTERCOIN (+656%) were both blocked by the 45% threshold.
Both had buy_ratio=40% and 38%, hype=100, liq=$41k/$28k, age 87-91m / 26-79m.
Tokens are suppressed at low buy_ratio while whales accumulate, then flipped.
The filter was supposed to catch dump setups — but rugs are already capped by dollar_stop.
Lowered to 0.40 on 2026-06-27 after scout log audit confirmed the misses.
Tokens in 40-45% zone get 25% smaller bets via _conviction_mult penalty.

---

---

### EVM chains disabled — DO NOT re-enable without 50+ trade sample showing positive WR
`CONFIG["chains"] = ["sol"]`
Data from Jun 25-27: ETH 0% WR (-$84, all BOWIE), Base 25% WR (-$44), BSC 0% (-$10).
SOL: 47% WR +$206. EVM chains bled -$138 total — structural, not noise.
Re-enable only if a specific EVM chain shows ≥40% WR over 50+ trades with positive total PnL.

### Profit re-entry: 60s minimum + 8% pullback — DO NOT remove the 60s gate
TOPDOG re-bought 0 seconds after its own TP sell — chased its own exit price.
60s hard floor prevents this regardless of pullback %. 8% pullback check still applies after.

### Velocity 2-min internal check: 12% drop in 2min → exit — KEEP
`velocity_2m_pct = 0.12`
21 slow-bleed losses held 2-15 minutes losing $37. DexScreener m5 window too slow.
Internal tick history (240 ticks ≈ 2min at 0.5s) exits faster before the dump compounds.

---

## What "every revision helps ALL" means

User instruction: each fix should improve behavior across ALL modes/vault sizes, not just the specific
case that triggered the fix. Cooldown fix helps $100 vault and $1000 vault equally.
Dollar stop helps both. Seed sizing is only active when dynamic_sizing=True (small vaults).
