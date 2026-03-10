# Project Audit and Fixes Report: Auto-Cal and Data Handling

This report summarizes the findings and improvements made to the trading bot's core logic, specifically focusing on the "Auto-Cal Add Position" features and the consistency of data across the platform.

## 1. Position Handling and Negative Value Accuracy

### Findings:
* **One-Way Mode Mapping:** In "One-way" (`net_mode`) account configurations, OKX returns position data where the `side` can be `long` or `short`, but the internal logic was not correctly reconciling these against the bot's `in_position` maps when a position was closed or reversed. This led to "ghost" positions appearing on the dashboard or incorrect logic triggers.
* **Loop Quantity Tracking:** The `update_loop_qty` function was failing to decrement the active loop quantity when a position was reduced by an opposing trade in One-way mode.
* **Negative Position Logic:** Logic that checked for "negative" positions (short positions) was inconsistent between the Backend (which uses `posSide` and `sz`) and the Frontend (which sometimes relied on signed `sz`).

### Fixes:
* **Robust Side Mapping:** Updated `PositionManager._map_side` to strictly validate position closure. If OKX reports `idx=0` (net mode) and `sz=0`, both 'long' and 'short' internal states are now explicitly cleared.
* **Cross-Side Reduction:** Fixed `PositionManager.update_loop_qty` to correctly identify when a 'buy' trade reduces a 'short' position (and vice versa) in One-way mode, ensuring the `loop_qty` remains accurate.

## 2. Auto-Cal Add Position (Modes 1 & 2)

### Findings:
* **Gap Trigger Logic:** The "Gap" trigger was calculated relative to a local `last_add_price` variable. This meant if the market moved significantly and the position's average entry price shifted, the gap check remained "stuck" to the old price.
* **Profit Target (Mode 2) Formula:** The trigger for Mode 2 was using `net_profit` (which subtracts fees). However, the recovery calculation formula was designed around `unrealized_pnl` (raw market difference). This mismatch caused the bot to over-add or fail to trigger when expected.
* **TP/SL Desync:** After an Auto-Cal addition filled, the position's average entry price would change, but the existing Take-Profit and Stop-Loss orders on the exchange remained at their old prices. This prevented the "Exit as a whole" strategy from working correctly.

### Fixes:
* **Average Entry Anchoring:** The gap trigger in `AutoCalManager.check_auto_add` now uses the actual `avgPx` (Average Entry Price) from the position data. This ensures the gap is always relative to the current breakeven point.
* **Standardized Trigger Metric:** Mode 2 (Profit Target) now consistently uses `unrealized_pnl` for the trigger, matching the user's requested formula: `Unrealized PnL >= Notional * Fee% * Multiplier`.
* **Real-time TP/SL Refresh:** Implemented a detection mechanism in `bot_engine.py` that flags whenever an "autocal" order fills. The bot now immediately calls `batch_modify_tpsl` to update the TP/SL for the *entire* position based on the new average entry price.

## 3. Fee Consistency and Data Presentation

### Findings:
* **Inconsistent Definitions:** The terms "Profit", "Net PnL", and "Unrealized PnL" were used interchangeably in several modules, leading to different values appearing on the Dashboard versus those used for safety triggers.
* **Trade Fee Sync:** The `trade_fee_percentage` set in the UI was not consistently propagated to the `AutoCalManager`, leading to incorrect "Need Add" USDT calculations.

### Fixes:
* **Standardized Metrics:**
    * **Net Profit:** Now defined as `Unrealized PnL - Fees - Realized Loss`. This is the primary metric for the Dashboard and "Above Zero" (Mode 1) triggers.
    * **Unrealized PnL:** Used for Mode 2 triggers and recovery notional calculations to maintain parity with OKX's interface.
* **UI/Backend Parity:** Verified that `trade_fee_percentage` is correctly handled by `app.py` and synchronized to the engine on every config update.

## 4. Stability and State Management

### Findings:
* **Rapid-Fire Orders:** A race condition existed where the bot could send multiple market addition orders in a single second because the position state hadn't updated yet.
* **Account Switching Security:** Previously, switching between 'Developer' and 'User' API keys did not explicitly stop the bot. This could lead to the bot automatically opening trades on a newly selected account if it was already "Running" on the previous one.
* **Passive Mode Trading:** The Auto-Cal recovery features were not strictly gated by the `is_running` flag, potentially allowing them to trade while the bot was supposed to be in "Passive" monitoring mode.

### Fixes:
* **Concurrency Protection:** Introduced `_is_adding` flags and optimistic state updates to ensure only one addition is processed per side at a time.
* **Reliable Account Switching:** The bot snapshot critical credentials (API keys, Symbol, Mode) and compares them on every update. If a switch is detected, it **automatically stops all trading** and resets the "Master Activation" flag. This prevents the bot from opening trades on a new account automatically.
* **Master Activation Flag:** Introduced `persistent_mode_active`. Persistent recovery features only activate after you click **"Start"** at least once for the current account/session.
* **Persistent Recovery (The "No-Stop" Auto-Cal):** Once activated, Auto-Cal recovery features (Additions, Margin adjustments, and Real-time exits) become **persistent**. They will continue to manage and protect your existing positions even if the strategy loop is "Stopped". This ensures that recovery trades remain active while the engine is running.
* **The "Stop All" Fail-safe:** To completely halt every process (including Auto-Cal and background monitoring), use the new **"Stop All"** button. This performs a hard shutdown of the engine and all WebSocket connections.
* **OCO Order Support:** Standardized position-level TP/SL to use `oco` order types for exchange-side safety.

## 5. Budget and Capacity Enforcement

### Findings:
* **Order Amount Explosion:** A bug in the Auto-Cal recovery math allowed the `gain_on_rec` denominator to approach zero, leading to extremely large "Need Add" calculations (e.g., $382,000$ when the budget was $10,000$).
* **Batch Overlap:** In One-way mode, pending orders returned by OKX were marked as `posSide: net`. The bot's batch logic was looking for `posSide: long/short`, failed to see the existing orders, and kept placing new ones until it hit 13 orders instead of the requested 3.
* **Limit Clarity:** The "Max Amount" displayed on the dashboard was calculated simply as `Max Allowed * Leverage`, ignoring the `Rate Divisor`, which led to confusion about the actual available capacity.

### Fixes:
* **Unrestricted Auto-Cal:** Auto-Cal additions are now **fully decoupled** from all budget settings (`Max Allowed`, `Remaining Amount`, and `Rate Divisor`). The `AutoCalManager` ignores these loop-specific limits and uses funds directly from the available account balance to execute recovery trades. This ensures that recovery logic can always fire when mathematically required, regardless of current loop budget exhaustion.
* **Mathematical Enforcement:** The recovery formula `Add Amount = (-UPL / (Rec% - Target%)) - Notional` is followed exactly as defined. The bot calculates and executes the precise addition amount needed to reach your recovery targets, without truncation or budget capping.
* **Aggressive Continuous Entry:** Removed the restriction that prevented new entry batches while previous orders were still pending. The bot now supports overlapping batches, placing new orders every loop cycle as long as signals are active and budget remains.
* **Normalized Side Tracking:** Pending orders are now normalized to `long` or `short` side based on their trade direction to ensure accurate real-time capacity management during these continuous entries.
* **Accurate Capacity Display:** The "Max Notional Cap" on the dashboard now correctly accounts for the `Rate Divisor` using the formula: `(Max Equity Limit / Rate) * Leverage`.

## 6. Understanding Auto-Cal Logs & Execution

You may see logs like this:
`Auto-Cal (SHORT): UPL=$-13.22, Raw Need-Add=$16119.86. (Target: $0.49, Safety Threshold: -$0.33)`

### What these values mean:
*   **UPL ($-13.22$):** Your current Unrealized PnL on OKX.
*   **Safety Threshold ($-0.33$):** The point where recovery logic starts. If your UPL is "more negative" than this, the bot calculates how much you need to add.
*   **Raw Need-Add ($16,119.86$):** The mathematically required amount (Notional) to add to your position to bring your average entry price to a level where you can reach your profit target.
*   **Target ($0.49$):** Your profit goal (Size * Fee% * Multiplier). If UPL hits this, the position closes.

### When does the order actually place?
The bot calculates these numbers every few seconds so that the **Dashboard "Need Add" badges** are always accurate. However, an order is **NOT** placed every time the calculation runs. An order is only sent to the exchange when:

1.  **Authorization:** You have clicked **"Start"** at least once (Master Activation is on).
2.  **Trigger Condition:**
    *   **The Gap Trigger:** The market price moves far enough away from your Average Entry price (based on your "Gap Threshold" setting).
    *   **The PnL Trigger:** The "Need Add" amount is positive and Mode 2 is enabled.
3.  **Safety Check:** The bot is not already waiting for an order to fill (`_is_adding` protection).

**Summary:** The logs show the bot's "thinking" process. It calculates that you need $16k to recover, but it will wait for the price to hit your specified **Gap Threshold** before executing that order to ensure it enters at the best possible price.

## Conclusion
The platform now handles data consistently by distinguishing between **Net Profit** (realizable cash) and **Unrealized PnL** (market movement). The Auto-Cal recovery features are now robust, anchored to the actual position average entry price, and properly synchronized with exchange-side TP/SL orders. By decoupling Auto-Cal from loop-specific budgets and implementing a "Master Activation" flag, the bot ensures secure account transitions while providing persistent, mathematically accurate position protection.
