# Project Audit Report: Data Handling & Auto-Cal Logic

## 1. Auto-Cal & Position Handling
### Short Position Logic
- **Negative Quantity Mapping**: The `PositionManager` correctly parses negative quantities (`pos < 0`) from OKX in One-way mode and maps them to the internal `short` side.
- **Directional PnL**: UPL calculation for shorts is correctly implemented as `(Entry - Market) * Qty`, ensuring that "Gap" triggers and recovery math react correctly when the market moves against the position.
- **Gap Trigger consistency**: The "Add" trigger for shorts uses `(Market - Entry)`, ensuring that additions only occur when the price moves significantly above the short entry (into a loss).

### Recovery Math Improvements
- **Order Amount Explosion Fix**: An epsilon buffer has been verified in the `AutoCalManager` denominator to prevent massive order quantities when recovery percentages are near profit targets.
- **Unrestricted Budgeting**: Recovery trades (Mode 1 & 2) are now decoupled from the `Max Allowed` loop budget. This allows the bot to execute mathematically required recovery trades using the total available balance, preventing "Trade Leakage" where a position is left unmanaged due to budget exhaustion.

## 2. Fee Handling & Metric Standardization
- **Standardized Net Profit**: The Dashboard now consistently displays `Net Profit = Unrealized PnL - Entry Fees - Cycle Realized Losses`.
- **Targeting Logic**: Auto-Exit triggers now use `used_fees` (the actual accumulated fees from the current position's fills) rather than global session fees, making the "Mode 2" profit targets highly accurate to the current trade's breakeven point.

## 3. Platform Data Synchronization
- **Nullish Coalescing Fix**: A critical issue was identified in `static/js/app.js` where the use of `||` caused values of `0` or `false` to be overwritten by defaults. This has been refactored to use `??` (nullish coalescing).
- **Persistence Parity**:
    - `add_pos_max_count` and other Martingale offsets now correctly persist as `0` when set by the user.
    - Dashboard "Quick Sync" inputs now correctly update the global `currentConfig` object, ensuring subsequent "Save Changes" from the modal don't revert live adjustments.

## 4. Operational Safety
- **Persistent Recovery**: The `persistent_mode_active` flag ensures that once a strategy is started, its recovery logic (Auto-Cal) continues to monitor positions even if the main trading loop is paused or reconfigured, preventing positions from being left "naked" without TP/SL management.
- **OCO Refresh**: Automatic synchronization of exchange-side TP/SL orders is verified to trigger immediately after every Auto-Cal "Add" fill.
