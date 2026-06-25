# PENDING — tothemoon bot

Everything left to build, fix, or wire up. Roughly priority order within each section.

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
- [ ] `src/bot/config.py`, `data.py`, `exchange.py`, `indicators.py`, `portfolio.py`, `risk.py`, `strategy.py`, `altseason.py`, `main.py` — currently monolith; split when file gets unwieldy (currently 2115 lines).

### Tests
- [x] 68 unit tests in `src/bot/tests.py` — run with `python3 src/bot/tests.py -v`. Covers:
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
