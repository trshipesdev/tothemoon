# PENDING — tothemoon bot

Everything left to build, fix, or wire up. Roughly priority order within each section.

---

## 🚀 Strategy overhaul — 2026-06-26 (deployed, 373 passing tests)

End-to-end review + max-profit tuning. All shipped to the server in shadow mode.

### Entry quality
- [x] **Momentum hype** — hype scores recent (h1) volume velocity, not a 24h average (`_pair_to_candidate`). Falls back to h24/24 when h1 missing.
- [x] **Buyer/seller gate** — reject tokens being dumped (<45% of h1 trades are buys, `moonshot.buy_ratio_min`). Degrades gracefully when txn data absent.
- [x] **Mode liquidity floor actually applied** — was a flat $15k regardless of mode; now uses the active mode's `liq_min` (degen $10k → more candidates, safe $50k → fewer/safer). This was a real bug.
- [x] **Per-mode `max_age_min`** — safe accepts older/established, degen wants fresher (replaced dead `age_min` config).
- [x] **Per-mode `max_entries_per_token_day`** — degen 6 / hype 4 / default 3 / safe 2 (was a flat global that strangled the bot once it cycled the few liquid tokens).

### Exit / risk realism
- [x] **Exit price impact** — `shadow_sell` now craters the fill when dumping a big bag into a thin pool (uncapped at the 5% entry ceiling). The moon-bag's +500% gains were fake-optimistic before this.
- [x] **Mode-lock bug fix** — positions store `entry_mode` at buy; exits read the mode the position was ENTERED under, so an AI mode-switch can't retroactively change a held position's SL/TP.
- [x] **Adaptive trailing stop** — tightens as unrealized gain grows (>+800% → 22%, >+300% → 28%, pre-ladder >+100% → 10%). A flat 45% trail gave back 45% of a +500% move.
- [x] **Stall exit** — bank a position up ≥20% that stopped climbing for 10 min and slipped ≥6% off peak (the #1 leak: winners round-tripping to breakeven). Skips the moon bag. `CONFIG["stall_exit"]`, dashboard-tunable.

### Capital / sizing
- [x] **Conviction sizing** — ticket scales with setup quality (hype + buy pressure), bounded [0.8,1.5]× so it nudges without breaching caps (`_conviction_mult`).
- [x] **Dynamic reserve** — hold back 40% (vs 25%) while the drawdown brake is on.
- [x] **Max open positions** cap (12) — memecoins dump together; don't over-concentrate.
- [x] **Manual blacklist** — never buy listed symbols/addresses (dashboard-editable).

### AI auto-pilot
- [x] **Faster cadence** — 30 → 12 min, AND event-triggered: consults immediately when the drawdown brake first trips.

### Realism / data
- [x] **Live Solana priority-fee gas** — `getRecentPrioritizationFees`-based estimate (fees spike during launch congestion). Falls back to static without a SOL_RPC_URL.

### Observability
- [x] **Per-mode P&L tracker** — `STATE["mode_perf"]` attributes realized P&L to the entry mode; dashboard "Realized P&L by mode" card.
- [x] **Dashboard chart fix + zoom** — fixed the broken-when-hidden canvas; added scroll/drag/pinch zoom + reset button.
- [x] **Strategy & risk Settings card** — stall exit, buyer/seller gate, max open, blacklist, all wired to `/api/config`.

### Backtester (replay the past through the real strategy)
- [x] **`src/bot/backtest.py`** — replays real historical price paths (GeckoTerminal OHLCV, free) through the LIVE entry/exit code under each mode. Reuses production functions via a simulated clock + mocked price feed. Shipped in the Docker image. Run: `docker compose exec cryptobot python backtest.py --days 3`.
- [x] **Week-of-data sourcing from online scanners** — pull a broad token universe (GeckoTerminal trending + DexScreener boosts) so the backtest isn't limited to what the bot already traded; runs the whole week as a fast sim.
- [ ] **Forward recorder** — snapshot the live trending feed + per-tick liquidity so backtests get real entry timing + rug/liq-drain modeling (closes the two backtester caveats: entry-at-window-start, constant-liquidity).

### Multi-instance
- [x] **`docker-compose.multi.yml`** — run all 4 modes side-by-side on the same live market (ports 8801-8804, Telegram off, AI off) + `scripts/compare-modes.sh`. Best on a ≥4GB box (1GB Oracle can't hold 4).

### Not done (honest scope — needs keys / real wallets)
- [ ] **Wallet rotation** for live trading (single wallet now — traceable/targetable). Needs real wallets, can't test in shadow.
- [ ] **Jito bundles / private RPC** (anti-sandwich on Solana), gas-balance management, tx-failure retry. Live-only.
- [ ] **More launch sources** (pump.fun, GeckoTerminal new-pools) for earlier detection. Need keys/integration.

---

## 🔬 Reanalysis follow-ups — 2026-06-26 (overnight: net +$50, +5%)

Bot owner wants NUANCE, not black-and-white blocks. Theme: **win off rugs by exiting
seconds-before (not minutes-before); micro-sell to bleed them first.** Each item below
is "make it smarter," not "add a hard gate."

### Already shipped from the reanalysis
- [x] **Phantom-position jam fixed** — closed shells counted against max-open, froze entries for hours.
- [x] **Mode shown per trade** in History; **sortable columns + last-week/all window**.
- [x] **Capital-at-risk metric** — peak $ on the table at once, starting bet, positions-at-once, "could've run this with ~$X".

### HIGH priority — research/design then build
- [ ] **Anti-rug exit ("bleed them first")** — micro-sell scale-out ladder + pre-rug tripwires (liq tick-down, single whale dump, sell-pressure flip, dev-wallet move). Goal: ride the pump, bail seconds before the dump, NOT minutes early. Intelligent on false positives (most shitcoins wobble). Applies to ALL modes.
- [ ] **Pre-rug signal (item #8)** — was supposed to exist; high priority. Get out seconds/minute before, very intelligent on false positives.
- [ ] **Faster stall/exit (item #2)** — current 10-min/6% never fired. Research micro-sell vs hard exit; how to bleed them before they bleed me.

### Research first (does it help or hurt? quantify)
- [ ] **hype vs degen vs all (item #3)** — owner's terminal tests (300 entries / 30 days) showed HYPE consistently wins; my 3-token sample was underpowered (degen won by catching one +206% runner). Need a LARGE-sample comparison (recording-based once data accumulates, or many historical tokens). Likely answer: degen=variance/runner-catching, hype=consistency → per-token mode (#68) is the real fix.
- [ ] **PISSBOT smarter re-entry (item #4)** — do NOT block after any prior loss (incl. 80¢ losses). Research: block only after a HARD-exit loss, or net-negative beyond a threshold, on that symbol.
- [ ] **MINE −$25.40 deep-dive (item #4-data)** — investigate exactly what happened tick-by-tick; what exit would've capped it.
- [ ] **Hard dollar stop (item #5)** — research how much it would've helped/hurt vs the % stop before adding.
- [ ] **Halve size on re-entries/thin (item #6)** — research how much upside it would've cost.
- [ ] **first-N-minutes freshness (entry #18)** — research hurt/help.
- [ ] **holder-growth signal (entry #20)** — research how much I'd miss out on.
- [ ] **time-of-day weighting (entry #21-ish)** — research if it would've hurt.

### Build with nuance (owner approved, wants smart not blunt)
- [ ] **Moon-bag fine-tuning (item #11)** — study what would've ridden the +206% / 凪ちゃん runners to MAX. Tune trail tiers + when the wide leash kicks in. HIGH interest.
- [ ] **Quick-scalp TP rung (item #7)** — add a low first rung; on ALL modes, smartest way.
- [ ] **Per-symbol P&L memory (item #9)** — track but unblock if the coin genuinely rebounds enough "to restart the fun."
- [ ] **Re-entry win-rate tracking (item #10)** — measure; smarter than a flat cap.
- [ ] **Holder concentration / "one person holds everything" (entry #21/#26)** — owner wants to detect + still sometimes vigilante-trade the fraudy pumps. Surface it, don't always block.
- [ ] **LP-lock / mint-renounced + dev-wallet rep (entry #22/#23)** — yes, but VERY intelligent on false positives.
- [ ] **buy/sell ratio scored not gated (entry #16)** — explained; build.
- [ ] **2-consecutive-rising-candles entry (entry #17)** — flesh out fully, fail-proof.
- [ ] **degen more at-bats (item #10/#11-data)**, **velocity exit tick-to-tick (item #14)** — improve.

### Lower / explain-and-defer
- [ ] Reject-if-already-up-X% (entry #19), per-chain thresholds (#25), single-dominant-pool penalty (#26), volume-acceleration (#24) — explained; revisit. Owner: "I love rugs, want to outsmart them — don't just avoid, beat them."

---

## 🌱 Seed mode — dynamic sizing for small vaults (planned 2026-06-26)

**Goal:** set `vault_usd = 100`, enable seed mode, and the bot manages itself — betting
what it can afford, growing naturally, never running dry.

### Architecture

`cur_deployed_usd` is already live in `shadow_buy()` (line 1153). Seed mode reuses it
to compute `avail_cash = vault_usd − cur_deployed_usd` at entry time.

### Changes needed

**Change 1 — Add `"seed"` to `CONFIG["modes"]` (after line 93)**
```python
"seed": {"tp": [0.50, 1.00], "sl": 0.14, "slip_bps": 120, "liq_min": 15000,
         "max_age_min": 180, "max_entries_per_token_day": 3, "size_mult": 1.0},
```
Tighter SL (0.14) than degen (0.28) — a $100 vault can't absorb a $28 loss.

**Change 2 — Add `"dynamic_sizing": False` flag to `CONFIG["moonshot"]` (after line 174)**
```python
"dynamic_sizing": False,  # True → bet_size = min(base_size_usd, avail_cash * 0.40)
```
`False` by default — existing configs unaffected. Flip to `True` for seed runs.

**Change 3 — Extend `size_ticket_usd()` (lines 989–1004)**

After the boost block and before the existing `min(base, per_chain_room...)` line, insert:
```python
if CONFIG["moonshot"].get("dynamic_sizing"):
    avail_cash = max(0.0, STATE["vault_usd"] - STATE.get("cur_deployed_usd", 0.0))
    base = min(base, avail_cash * 0.40)
```

**Change 4 — Entry guard: $10 floor + 1.5× buffer gate (lines 3393–3398)**

Replace the `if usd >= CONFIG["moonshot"]["min_ticket_usd"]` check:
```python
_min_ticket = 10.0 if CONFIG["moonshot"].get("dynamic_sizing") else CONFIG["moonshot"]["min_ticket_usd"]
if usd < _min_ticket:
    _scout(symbol, chain, "rejected",
           f"bet_size ${usd:.2f} below floor ${_min_ticket:.2f} (not enough cash)", sc, addr)
    continue
if CONFIG["moonshot"].get("dynamic_sizing"):
    _avail = max(0.0, STATE["vault_usd"] - STATE.get("cur_deployed_usd", 0.0))
    if _avail < usd * 1.5:
        _scout(symbol, chain, "rejected",
               f"cash buffer too thin (avail ${_avail:.2f} < 1.5x bet ${usd:.2f})", sc, addr)
        continue
```
The 1.5× gate keeps at least one more minimum bet in reserve.

**Change 5 — Verify `STATE` initialization seeds `"cur_deployed_usd": 0.0`**

`size_ticket_usd` uses `.get("cur_deployed_usd", 0.0)` as a fallback, so safe either way,
but check the `STATE = {` block (~line 250–350) and add the key if missing.

### Expected behavior on $100 vault (reserve_pct = 0.25)

| Open positions | Deployed | avail_cash | bet_size | Action |
|---|---|---|---|---|
| 0 | $0 | $100 | min($30, $40) = $30 | Enter |
| 1 | $30 | $70 | min($30, $28) = $28 | Enter |
| 2 | $58 | $42 | min($30, $16.80) = $16.80 | Enter |
| 3 | $74 | $26 | $10.40, buffer check $10.40*1.5=$15.60 < $26 | Enter |
| 4 | $84 | $16 | $6.40 < $10 floor | **Blocked** |

Max ~4 concurrent positions; self-limiting, grows as wins compound.

### How to activate
```python
CONFIG["mode"] = "seed"
CONFIG["vault_usd"] = 100.0
CONFIG["moonshot"]["dynamic_sizing"] = True
CONFIG["moonshot"]["mode"] = "enter"
```

### Tasks
- [ ] Change 1: add `"seed"` to `CONFIG["modes"]` (1 line after line 93)
- [ ] Change 2: add `"dynamic_sizing": False` to `CONFIG["moonshot"]` (1 line after line 174)
- [ ] Change 3: dynamic bet cap in `size_ticket_usd()` (lines 989–1004, ~3 new lines)
- [ ] Change 4: $10 floor + 1.5× buffer gate in scanner entry loop (lines 3393–3398, ~10 lines)
- [ ] Change 5: confirm `STATE` init has `"cur_deployed_usd": 0.0`
- [ ] Add tests: seed mode bet sizing, floor enforcement, buffer gate, mode definition

---

## 🔒 Go-live security hardening (DO BEFORE PUTTING REAL FUNDS ON THE SERVER)

Currently shadow mode only — no wallet keys, no crypto anywhere, nothing to steal yet.
These MUST be done before flipping `SHADOW_MODE=false` with a funded wallet:

- [ ] **Dedicated hot wallet only** — fund a brand-new wallet with ONLY what you're willing to trade/lose. NEVER put your main wallet's private key on the server.
- [ ] **Firewall: SSH only** — only port 22 open. Dashboard (8787) stays bound to localhost; reach it via SSH tunnel (`ssh -L 8787:localhost:8787 ...`), never expose it to the internet. On Oracle: do NOT add an ingress rule for 8787.
- [ ] **Key-only SSH** — confirm password login is disabled (Oracle default). Optionally add fail2ban.
- [ ] **Lock .env perms** — `chmod 600 .env` on the server so only the owner can read the keys.
- [ ] **Set DASHBOARD_TOKEN** — random string, so even via tunnel the API requires auth.
- [ ] **Keep deps patched** — periodic `docker compose pull` / rebuild.

NOTE: Oracle "Shielded Instance" and "Confidential Computing" toggles do NOT address
these threats (they're boot-firmware / RAM-encryption features). Leave them OFF.

---

## 🐛 Bugs

- [x] **docker-compose state never persists** — bot now reads `STATE_PATH` env; Dockerfile sets `STATE_PATH=/app/data/state_default.json`; docker-compose volume `./data:/app/data` maps correctly.
- [x] **Dashboard `deployable_usd` stale** — `/api/state` now calls `deployable_now()` on every request.
- [x] **Dashboard TP/SL columns blank** — `/api/state` now returns `mode_tp` and `mode_sl` for the current mode.
- [x] **`live_buy_sol` wrong unit math** — added `_sol_token_decimals(mint)` helper (queries Solana RPC `getAccountInfo`); `live_buy_sol` now converts `outAmount` using the actual token's decimals.
- [x] **`live_buy_evm` units not from actual swap** — both `live_buy_evm` and `live_sell_evm` now use a before/after `balanceOf` diff to compute actual units received / USDC proceeds.
- [x] **`/skim` toggle does nothing** — `shadow_sell` now skims `CONFIG["skim_pct"]` (default 10%) of realized profit into `STATE["income_usd"]` when skim is enabled.
- [x] **`cmd_shadow` sends stale message** — message now says "live execution active (Jupiter / Uniswap + 1inch)".
- [x] **`engine_loop` crashes on malformed position** — `p.get("time", now_utc().isoformat())` guards the `datetime.fromisoformat` call.
- [x] **`telectl.py` incompatible with `bot_full.py`** — deleted. Used wrong state path (`./data/state.json` vs `state_default.json`) and wrong position keys (`qty`/`entry`/`stop` vs `units`/`avg`).
- [x] **`ec2-user-data.sh` `ProtectSystem=full` blocks state writes** — changed to `ProtectSystem=strict` + `ReadWritePaths=/home/ubuntu/cryptobot/data` + `Environment=STATE_PATH=...`.

---

## 🔴 Critical (bot can't trade without these)

### Live execution
- [x] **Jupiter adapter** — `live_buy_sol` / `live_sell_sol` use Jupiter v6 `/quote` + `/swap`, sign with `solders`, submit via Solana RPC. Token decimals fetched from Solana RPC.
- [x] **Uniswap / 1inch adapter** — `live_buy_evm` / `live_sell_evm` try 1inch aggregator first (if `ONEINCH_API_KEY` set) for best routing; fall back to direct Uniswap v3 / PancakeSwap v3. Balance diff used for actual units. Fee tier per chain in `EVM_FEE_TIER`.
- [x] **Wallet balance fetch** — `refresh_vault_balance()` called on startup and every hour in engine loop (live mode only).

### Live position price feed
- [x] **Fetch live prices for open positions** — `fetch_positions_prices()` batch-fetches all open positions from DexScreener. SOL positions fall back to Birdeye (if `BIRDEYE_API_KEY` set) if DexScreener misses them. Falls back to `avg*1.02` if both fail.

---

## 🟡 High (configured but not actually running)

### Telegram — command handlers
- [x] All 20+ Telegram commands implemented and registered.

### Proactive alerts
- [x] `send_alert(msg, critical)` — wired, respects quiet hours, feeds ALERT_LOG deque.
- [x] Daily digest — fires at `daily_digest_utc` hour.

### RPC health monitoring
- [x] `check_rpc_health()` — pings each chain every 5 min.

---

## 🟠 Medium (features that exist as stubs)

### Old-coin pump detector
- [x] **Social/mentions gate via LunarCrush** — `fetch_social_volume(symbol)` uses LunarCrush v4 API (`LUNARCRUSH_API_KEY`). `detect_oldcoin_pump` applies the `mentions_x` gate if the key is set; skips the gate gracefully if not.
- [x] **Fallback social source** — `fetch_social_volume()` now falls back to CoinGecko trending rank when no `LUNARCRUSH_API_KEY` is set. Rank 1 → 3.0x, rank 10 → 1.65x, not trending → 1.0x. Cache refreshes every 10 min. `detect_oldcoin_pump` social gate is now always active.

### Presale assistant
- [x] **Presale gate** — `presale_score()` is now called for every new token candidate. Social volume (LunarCrush or CoinGecko fallback) is used for the mentions proxy. Tokens scoring below `presale_min_score` (default 10, tunable in CONFIG) are downgraded to suggest-only with a Telegram warning. Score is shown in all entry/suggest alerts. Set `presale_min_score: 0` to disable.

### Autoscale grace window
- [x] `autoscale_maybe()` — add_count / last_add_ts / cooldown_min enforced.

### Rug / LP drain guard + Momentum exit strategy
- [x] Rug detection — instant exit if liq drops >40% in one tick (`rug_liq_drop`).
- [x] **Trailing stop** — exit if price falls >15% from peak (`trailing_stop_pct`). Peak tracked per-position from first buy.
- [x] **Velocity exit** — exit if m5 price change < -8% in a single tick (`velocity_exit_pct`). Catches rugs before they fully play out.
- [x] **Liq drain detector** — exit if liquidity declines >5% for 3+ consecutive ticks (`liq_drain_ticks` / `liq_drain_pct`). Detects slow LP pulls.
- [x] **Volume dry-up alert** — Telegram alert if h1 vol drops to <20% of entry vol (`vol_dry_pct`). Soft signal, doesn't exit alone.
- [x] **Re-entry watch** — after any exit, adds token to `STATE["reentry_watch"]`; bot checks every tick for price stability (3 ticks within 4% range) + liq floor (≥60% of entry liq) + vol alive (h1 > $3k). Auto-re-enters when conditions met.

### Trusted coins module (DOGE)
- [x] `manage_trusted_coins()` — dip-buy below floor, trim into exit_band.

---

## 🟣 Marketplace APIs

### Scanner sources
- [x] **DexScreener** — boosts + profiles; batch position price fetch.
- [x] **Birdeye** (`BIRDEYE_API_KEY`) — Solana trending token list added to `fetch_new_candidates`. SOL position price fallback in `fetch_positions_prices`.
- [x] **CoinGecko trending** — `fetch_coingecko_trending()` wired into `fetch_new_candidates`; resolves chain + address from platforms dict.
- [ ] **More Birdeye signals** — OHLCV, holder count, top-wallet concentration. Useful for moonshot filter quality. Low priority until scanner hit rate is measured.

### Social signals
- [x] **LunarCrush** (`LUNARCRUSH_API_KEY`) — `fetch_social_volume(symbol)` returns today vs 7-day avg ratio; gates `detect_oldcoin_pump` when `mentions_x > 1` and key is set.

### Execution routing
- [x] **Jupiter v6** — Solana swaps with actual token decimals from RPC.
- [x] **1inch Aggregation Protocol v6** (`ONEINCH_API_KEY`) — EVM swaps; auto-routes through best liquidity across all DEXes; falls back to Uniswap v3 if key not set.
- [x] **Uniswap v3 / PancakeSwap v3** — direct DEX fallback; fee tier now per-chain in `EVM_FEE_TIER`.
- [ ] **Raydium / Orca direct** — Jupiter already routes through these, so direct integration adds complexity with no fill-quality benefit. Skip unless Jupiter is down.
- [ ] **0x Protocol** — alternate EVM aggregator. Low priority while 1inch covers the main chains (ETH/Base/BSC/Polygon).

---

## 🟣 Dashboard / app (personal operator UI)

- [x] All ~20 Flask API endpoints.
- [x] `src/dashboard/index.html` — Alpine.js + Tailwind + Chart.js, no build step.
- [x] `CORS(app)` + `DASHBOARD_TOKEN` Bearer auth.
- [x] `PYTHONUNBUFFERED=1` in Dockerfile.
- [x] `HEALTHCHECK` in Dockerfile (polls `/status`).
- [x] `STATE_PATH` env wired through Dockerfile + ec2 systemd unit.
- [x] Positions table now shows `m5 change` (color-coded), `trail_pct` (how far from peak, yellow/red warning), TP formatted as `+28%, +60%`, SL as `-12%`.
- [x] Re-entry watch section appears under positions when tokens are being monitored.
- [x] `fmtTp`/`fmtSl` formatters handle both array TP and single-value SL correctly.

---

## 🔵 Lower priority (cleanup / nice-to-have)

### Module split
- [ ] `src/bot/config.py`, `data.py`, `exchange.py`, `indicators.py`, `portfolio.py`, `risk.py`, `strategy.py`, `altseason.py`, `main.py` — currently monolith; split when file gets unwieldy (currently ~3300 lines).

### Tests
- [x] 373 unit tests in `src/bot/tests.py` — run with `python3 src/bot/tests.py -v`. Covers:
  - `shadow_buy` / `shadow_sell` round-trip: weighted avg, vault balance, open_today_usd, trade_log
  - Capital sizing: reserve, per-chain cap, per-token cap, daily deploy cap, drawdown brake scaling
  - Drawdown brake activation and ticket-size reduction
  - TP cascade / `tp_index` lifecycle (start=0, increment, reset on close)
  - Skim logic: enabled/disabled, pct, loss=no-skim
  - Burst guard: count tracking, expiry after 30s
  - Moonshot filters: age, hype, liq bounds, positive flag
  - `presale_score`: all scoring paths, caps
  - `_pair_to_candidate`: age calc, hype formula, missing fields, zero price/liq
  - `compute_heat`, `est_price_impact`, `_px_dict`, objective validation
  - Re-entry watch: address guard, disabled guard
  - Slippage jitter: non-negative, within bounds
- [ ] Live execution adapters against Solana devnet + EVM testnets (requires funded test wallets)

### Misc
- [x] **Watchlist auto-join missing address** — `shadow_buy` for watchlist pumps now correctly passes `addr` so live exits can find the token.
- [x] **TP cascade** — positions now track `tp_index`; each TP level fires exactly once per position across engine ticks. Resets to 0 when position fully closes.
- [x] **`start_objective` no validation** — now raises `ValueError` if `target_usd ≤ 0` or `weeks ≤ 0`.
- [x] **Daily reset timing** — reset of `open_today_usd` moved from post-sleep to top of `engine_once()` using `STATE["last_daily_reset"]`; fires correctly even if loop is slow.
- [x] **`/api/close` hardcoded liq** — now fetches live price and liq from `fetch_positions_prices()`, falls back to `entry_liq` from position state.
- [x] **`/api/sell` stale price** — now uses live price/liq from `fetch_positions_prices()` unless caller explicitly passes them.
- [x] **`/api/buy` silent bad price** — now auto-fetches price and liq from DexScreener when address is provided and price is missing. Returns 400 if price cannot be determined.
- [x] **`/export_state` missing CONFIG** — now exports mode, moonshot mode, objective, and shadow_mode alongside STATE.
- [x] **`/status` missing build info** — now returns `build` (git sha), `shadow_mode`, and open position count.
- [x] **`save_state`/`load_state` silent failures** — now log errors. `load_state` handles `FileNotFoundError` separately and seeds missing STATE keys on upgrade.
- [x] **`_get()` no retry** — now retries up to 2× on timeout or 429/503; logs failures with URL.
- [x] **Startup wallet key validation** — `main()` now raises `RuntimeError` immediately if live mode is active but wallet env vars are missing.
- [x] **`cmd_mode` unhelpful error** — now shows valid modes list and current mode when called with bad arg.
- [ ] **Multi-tenant** — CONFIG/STATE are globals; no isolation if two processes share a host.
- [x] **RPC failover** — EVM: `_evm_w3()` iterates list and returns first connected Web3. SOL: `_sol_rpc()` health-checks each URL. Both implemented.
- [x] **`/export_state`** — now sends full state as a downloadable `.json` file attachment (no truncation).
- [x] **`stealth` config wired** — `burst_per_30s` (default 4): max new position opens per 30s window, skips with alert if hit. `slip_bps_jitter` (default 30bps): randomizes fill slippage slightly to avoid MEV pattern detection. `candidate_shuffle` was already live. `split_parts` remains unimplemented (low priority — Jupiter/1inch handle order splitting internally).
- [x] **Presale scanner source** — `presale_score()` now gates every new candidate using social volume as the mentions proxy. No external presale feed needed for basic operation; Pinksale/DexScreener launchpad integration remains a stretch goal for audit/KYC signals.
- [x] **CloudWatch heartbeat** — `emit_heartbeat()` fires every 5 min from engine loop.
- [x] **`/restart now` safety** — `save_state()` before `os._exit(0)`.
- [x] **`should_exit_no_pump` hardcoded mode** — was always reading `modes.degen.no_pump.hurdle` regardless of active mode; now uses current mode's no_pump config with degen as fallback.
- [x] **`engine_loop` double `load_state()`** — state was being loaded twice (once in `main()`, once at top of `engine_loop`). Removed the redundant second load.
- [x] **`cmd_status` showed closed positions** — now filters to `units > 0` only, adds live PnL per position, shows brake/income/re-entry-watch count.
- [x] **Engine crash silent** — `engine_once` exceptions now trigger a Telegram alert on first crash and every 10th after (throttled to once per 5 min).
