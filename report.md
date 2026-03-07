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

## 4. Stability and Race Conditions

### Findings:
* **Rapid-Fire Orders:** A race condition existed where the bot could send multiple market addition orders in a single second because the position state hadn't updated yet.

### Fixes:
* **Concurrency Protection:** Introduced `_is_adding` flags and optimistic state updates (incrementing step counts *before* the API call returns) to ensure only one addition is processed per side at a time.
* **OCO Order Support:** Standardized position-level TP/SL to use `oco` (One-Cancels-the-Other) order types, ensuring that if one target is hit, the other is automatically cancelled by the exchange.

## 5. Budget and Capacity Enforcement

### Findings:
* **Order Amount Explosion:** A bug in the Auto-Cal recovery math allowed the `gain_on_rec` denominator to approach zero, leading to extremely large "Need Add" calculations (e.g., $382,000$ when the budget was $10,000$).
* **Batch Overlap:** In One-way mode, pending orders returned by OKX were marked as `posSide: net`. The bot's batch logic was looking for `posSide: long/short`, failed to see the existing orders, and kept placing new ones until it hit 13 orders instead of the requested 3.
* **Limit Clarity:** The "Max Amount" displayed on the dashboard was calculated simply as `Max Allowed * Leverage`, ignoring the `Rate Divisor`, which led to confusion about the actual available capacity.

### Fixes:
* **Hard Capacity Caps:** The `AutoCalManager` now strictly enforces that any addition must fit within the `remaining_amount_notional`. It will automatically truncate recovery orders to stay within your defined budget.
* **Recovery Math Hardening:** Added a minimum 0.5% safety buffer to the recovery denominator to prevent mathematical explosions.
* **Normalized Side Tracking:** Pending orders are now normalized to `long` or `short` side based on their trade direction. This allows the batch logic to correctly identify "In-Flight" orders and stop at the exact batch size requested.
* **Accurate Capacity Display:** The "Max Notional Cap" on the dashboard now correctly accounts for the `Rate Divisor` using the formula: `(Max Equity Limit / Rate) * Leverage`.

## Conclusion
The platform now handles data consistently by distinguishing between **Net Profit** (realizable cash) and **Unrealized PnL** (market movement). The Auto-Cal additions are now more robust, anchored to the actual position average entry price, strictly budget-limited, and properly synchronized with exchange-side TP/SL orders.
