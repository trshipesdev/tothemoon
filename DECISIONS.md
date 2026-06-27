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
Per-token cap at 12% of deployable keeps any single bet from exceeding ~$90.

---

## Live Trading Pre-Blockers (DO NOT GO LIVE WITHOUT THESE)

1. **Per-chain wallet balance**: `per_chain_room()` must check actual wallet balance, not just % of total vault.
   SOL wallet and EVM wallet are separate. A $1000 vault with $800 on SOL can't spend $300 on Base.

2. **Solana tx confirmation**: `_sol_submit_tx` fires and forgets. If tx fails to land, bot thinks it sold.
   Need to poll for confirmation or mark position as "sell_pending".

3. **EVM tx blocking**: `wait_for_transaction_receipt(timeout=120)` blocks the engine loop for up to 2 min.
   All other positions unmonitored during that time. Fix: run EVM sells in background thread.

---

## What "every revision helps ALL" means

User instruction: each fix should improve behavior across ALL modes/vault sizes, not just the specific
case that triggered the fix. Cooldown fix helps $100 vault and $1000 vault equally.
Dollar stop helps both. Seed sizing is only active when dynamic_sizing=True (small vaults).
