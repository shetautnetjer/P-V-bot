# agents.md — Codex Road‑Map (CPU‑First Build)

> **Purpose:** This file is **not** a runtime spec.  It is a curated list of **codex prompts, design notes, and TODOs** that guide ChatGPT‑Codex toward generating a CPU‑based prototype of our **single‑pair price + volume trading engine**.  Treat every code block here as a *draft prompt* you can paste into Codex to scaffold real Rust files.

---

## 0 · High‑Level Goal

* Build a Rust binary that:

  1. Accepts `--pair BASE/QUOTE` on the CLI.
  2. Discovers every relevant pool/orderbook via **Jupiter**.
  3. Subscribes to **price ticks** (Pyth or synthetic) **and** **signed volume** (reserve deltas or CLOB fills).
  4. Computes **price indicators** *and* **volume indicators** on CPU ring‑buffers.
  5. Blends them with a **dynamic weight vector** that updates after each trade as a function of **PnL, slippage, fee, and latency**.
  6. If the blended score ≥ cutoff, fetches a route from Jupiter, runs safety checks, simulates swap, and—if safe—executes through **Jito**.

No GPU, no async DAG complexity—just a clean, testable CPU baseline.

---

## 1 · Suggested Folder Layout (Codex Prompt)

```text
src/
  ├─ main.rs            # CLI + runtime supervisor
  ├─ discover.rs        # pool discovery (Jupiter)
  ├─ streams.rs         # price + volume async streams
  ├─ indicators.rs      # price & volume math
  ├─ scoring.rs         # weight blend + adaptive update
  ├─ execution.rs       # route fetch, safety, simulate, send
  └─ types.rs           # common structs / enums
```

Paste the layout above into Codex and ask it to generate empty module stubs with `#[tokio::main]` in `main.rs`.

---

## 2 · Key Data Structures

```rust
/// one price tick
authored
struct PriceTick { slot: u64, ts: i64, price: f64 }

/// buy vs sell volume in lamports or token atoms
struct VolTick { slot: u64, ts: i64, buy: f64, sell: f64 }

/// indicator output (z‑scores)
struct Indicators {
    momentum: f64,
    hilbert_phase: f64,
    fractal_atr: f64,
    vwap_z: f64,
    cvd_slope: f64,
    ofi: f64,
    kyle_lambda: f64,
}

/// adaptive weights (mutable)
struct Weights { price: [f64; 5], volume: [f64; 3] }
```

**Prompt:** “Codex, generate `types.rs` implementing these structs with `serde::Serialize` for logging.”

---

## 3 · Price + Volume Streams

### 3.1 Pool Discovery

```
Generate an async fn `discover_pools(base: &str, quote: &str)`:
  • hit `https://quote-api.jup.ag/v6/pools` with query params
  • deserialize JSON into `PoolMeta { program_id, lp_token_mint, reserve_a, reserve_b, oracle_symbol }`
  • filter by known program_ids (Raydium, Orca, Phoenix, Meteora)
  • return Vec<PoolMeta>
```

### 3.2 Price Feed

* **If Pyth symbol exists** → subscribe to WS and forward `PriceTick`.
* **Else** → compute mid‑price per slot:
  `mid = (reserve_b / reserve_a)` for AMM or `(best_bid+best_ask)/2` for CLOB.
  Prompt: “Codex, implement `stream_price(pool: &PoolMeta, tx: Sender<PriceTick>)` using `tokio_tungstenite`.”

### 3.3 Volume Feed

* **AMM:** subscribe `accountSubscribe` on both token accounts; per slot compute Δreserve → sign by direction.
* **CLOB:** subscribe to event queue; classify fills as buy (maker on ask) or sell (maker on bid).
  Prompt template included above.

---

## 4 · Indicator Math (CPU‑Friendly)

Use `ndarray` or pure loops—must run <1 ms per tick.

| Category  | Name          | Rust Sketch                                  |    |                                    |
| --------- | ------------- | -------------------------------------------- | -- | ---------------------------------- |
| Price     | Momentum      | `(p[t] - p[t-k]) / p[t-k]`                   |    |                                    |
| Price     | Hilbert Phase | analytic signal via 4‑tap FIR Hilbert kernel |    |                                    |
| Price     | Fractal ATR   | \`atr = α\*                                  | ΔP | + (1-α)\*atr\_prev\`, α = f(Hurst) |
| Volume    | VWAP‑Z        | `(p - vwap)/std_vwap`, vwap = Σ(p\*v)/Σv     |    |                                    |
| Volume    | CVD Slope     | OLS slope on last 60 CVD points              |    |                                    |
| Volume    | OFI           | Σ(ΔBidSize - ΔAskSize) per slot              |    |                                    |
| Liquidity | Kyle λ        | `(ΔP) / signed_volume` over 30 s             |    |                                    |

Prompt Codex: “Generate `fn compute_indicators(buffer: &RingBuf) -> Indicators` filling the struct above.”

---

## 5 · Dynamic Scoring & Weight Update

### 5.1 Forward Pass

```rust
let price_vec   = [ind.momentum, ind.hilbert_phase, ind.fractal_atr, ind.fama_gap, ind.perm_entropy];
let volume_vec  = [ind.vwap_z, ind.cvd_slope, tanh(ind.ofi / DEPTH_K)];
let raw_score   = dot(weights.price, price_vec) + dot(weights.volume, volume_vec);
let risk_adj    = 1.0 + 0.25 * tanh(ind.kyle_lambda / LAMBDA_BASE);
let final_score = raw_score * risk_adj - slippage_penalty;
```

*Trigger* when `final_score ≥ CUTOFF`.

### 5.2 Back‑prop Weight Update

After each `TradeReceipt`:

```math
r = (profit_pct - slippage_pct - fee_pct) / risk
w_new = w_old + η * r * z
w_new = clip(w_new, -1.0, 1.0);
```

Prompt: “Codex, implement `update_weights(weights: &mut Weights, ind: &Indicators, receipt: &TradeReceipt)`.”

---

## 6 · Execution & Safety Checklist

The engine supports **two execution modes** selectable via `--exec-mode` (`public`, `jito`, **`light`**).

| Mode               | Privacy                         | MEV‑Safe   | Dependencies              |
| ------------------ | ------------------------------- | ---------- | ------------------------- |
| `public` (default) | None                            | ❌          | QuickNode RPC only        |
| `jito`             | Medium (bundle until inclusion) | ✅          | Jito relayer key          |
| **`light`**        | Full (zk‑shielded)              | ✅ (hidden) | Light SDK + Steel program |

If `--exec-mode light` is chosen, the pipeline detours into a private workflow described below.

### 6A · Light Protocol Execution Path

> **Goal:** execute the Jupiter‑routed swap *inside* a **Light Protocol shielded pool**, then withdraw to the public wallet once confirmed—obscuring route, amounts, and side.

1. **Shield Funds**

   * Prompt Codex:

   ```rust
   // using light_sdk 0.4
   let ctx = LightContext::new(rpc_url, keypair);
   ctx.deposit(base_mint, amount).await?;
   ```
2. **Pull Route** (same Jupiter `/quote`).  Record only *final out\_mint* and *minimum\_out\_amount*; do **not** reveal the full split route on‑chain.
3. **Generate Shielded Swap Instruction**

   * Steel smart‑contract call: `steel_swap_zk(route_hash, min_out)` executed **inside Light circuit**.
   * Prompt: "Codex, generate `build_light_swap_ix()` which wraps the Jupiter route hash into a Steel instruction and serializes it for Light SDK execution."
4. **Prove + Submit**

   * `light_sdk::prove_and_submit(vec![deposit_ix, light_swap_ix])`.
5. **Withdraw (Optional)**

   * Once proof settled, call `ctx.withdraw(out_mint, amount, destination_pubkey)`.
6. **TradeReceipt** emits `shielded_sig`, `proof_hash`, `gas_used`, `pnl_est`.

### 6B · Weight Update Adjustments for Light Mode

`slippage_pct` is unknown until withdrawal.  Use **simulated slippage** from Jupiter + add penalty for `proof_cost_lamports`.

```math
r = (pnl_sim - proof_cost_lamports/Lot) / risk
```

### 6C · Codex Prompts for Light Integration

* **Prompt‑1**: “Create a `light.rs` module with `struct LightCtx` wrapping Light SDK client and helper fns deposit/withdraw/swap.”
* **Prompt‑2**: “In `execution.rs`, add `match exec_mode` branch that builds the vector of shielded instructions, calls `prove_and_submit`, and returns a `TradeReceipt`.”
* **Prompt‑3**: “Generate a minimal `steel_swap_zk` Anchor/Steel IDL snippet with inputs `(route_hash: [u8;32], min_out: u64)`.”

### Legacy Public/Jito Path (unchanged)

1. Jupiter quote.
2. Simulate swap; derive `slippage_pct`.
3. Safety checks.
4. Sign & send (public) **or** bundle (Jito).

---

1. Fetch quote via Jupiter `/quote`.
2. Simulate swap; derive `slippage_pct`.
3. Run extension / authority checks on mint.
4. Route whitelist (program\_id). 5. Price divergence vs Pyth < 5 %.
   If all pass, sign + bundle.
   Prompt: “Generate `execute_trade(route, wallet) -> TradeReceipt` with dummy signing placeholder.”

---

## 7 · Logging & Testing

* Write fills, indicators, weights every block to CSV (`csv_async`).
* Provide `cargo test` for indicator functions using fixture CSV.
* Add `./scripts/replay.rs` that replays historical slot data and asserts Sharpe > 1.

---

## 8 · Next Steps

1. Flesh out code via prompts above in Codex.
2. Integrate into a single binary (`main.rs`).
3. Dry‑run on devnet with a small test pair.
4. Profile CPU cost; refactor hot loops for SIMD.
5. Port indicator math to CUDA once stable.
