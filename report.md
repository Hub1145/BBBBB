# OKX Trading Bot Analysis Report

This report summarizes findings regarding the Auto-Cal Add Position logic, negative position handling, and data presentation inconsistencies across the platform.

## 1. Auto-Cal Add Position (Mode 1 & 2)

### Issues with Negative Position Reading
A critical bug was identified in `handlers/position_manager.py` within the `_map_side` method. When the bot is in `net_mode` (One-way) and the `direction` is set to `both`, the bot fails to correctly identify the side of a closing trade.

- **The Problem**: When a short position is closed (quantity becomes 0), the `_map_side` function defaults to the `direction` config value. If `direction` is `both`, it defaults to `long`.
- **The Result**: A closed short position is never correctly processed by `_handle_closure`, causing the bot to believe it is still in a short position. This prevents new orders from being placed and keeps "ghost" metrics on the dashboard.

### Broken Loop Quantity Tracking
In `net_mode` with `direction: both`, the logic to update `loop_qty` is flawed:
- A `sell` order is always mapped to the `short` side by the `orders` channel handler.
- If a user is closing a `long` position with a `sell` order, the bot incorrectly increases the `short` loop quantity instead of decreasing the `long` loop quantity.
- This leads to an ever-increasing `Used Amount (Loop)` on the dashboard that doesn't reflect actual capital usage.

## 2. Inconsistent Fee Usage

The platform currently mixes **Actual Fees** and **Estimated Fees** across different features:

| Feature | Fee Type Used | Calculation Method |
| :--- | :--- | :--- |
| **Auto-Cal Profit** | Actual Fees | Sum of `fillFee` from OKX order data. |
| **Size Auto-Cal Profit** | Estimated Fees | `Size Amount * (trade_fee_percentage / 100)` |
| **Net Profit (UI)** | Actual Fees | `UPL - Actual Fees - Realized Loss` |
| **Need Add (Mode 2)** | Estimated Fees | Uses `trade_fee_percentage` as `surplus_target`. |

**Consequence**: The "Target Auto Profit" shown on the dashboard might not align with the "Used Fee" displayed nearby because they are derived from different sources.

## 3. Data Handling & Presentation Inconsistencies

### REST vs. Socket Data
There is a significant mismatch between the data returned by the `/api/status` (REST) endpoint and the data emitted via WebSocket (`account_update`):

1. **Missing Fields in REST**: Fields like `used_fees`, `size_fees`, `raw_need_add_usdt`, and `raw_need_add_above_zero` are present in WebSocket updates but missing from the REST status endpoint.
2. **UI Flickering**: When the frontend performs a periodic status sync via REST, these missing fields cause UI elements to flicker or disappear until the next WebSocket update arrives.
3. **Calculation Mismatch**: The `net_pnl` calculation for positions in `app.py` does not always match the `position_net_pnl` logic used in `bot_engine.py`.

### Unrealized PnL vs. Net PnL in Auto-Exit
In the real-time WebSocket handler (`bot_engine.py`), the `check_auto_exit` method is called with `cached_unrealized_pnl` passed for both the `net_pnl` and `unrealized_pnl` parameters:
```python
triggered, reason = self.auto_cal_manager.check_auto_exit(net_pnl, self.cached_unrealized_pnl)
```
This causes the "Above Zero" and "Profit Target" (Mode 1 & 2) exits to trigger based on **Unrealized PnL** instead of **Net PnL** (which should subtract fees).

## 4. Negative Position Handling

- **Internal Representation**: Short positions are stored with negative quantities.
- **Frontend Display**: The frontend displays these negative quantities directly.
- **Auto-Add Logic**: In `auto_cal_manager.py`, the `price_diff` for shorts is calculated as `(mkt - last_add_price)`, which correctly handles the inverse relationship for short entries. However, the use of `abs()` in some parts of `position_manager.py` while omitting it in others leads to potential comparison errors.

## 5. Summary of Key Findings

1. **Ghost Positions**: Short positions fail to close in memory when in `both` direction mode.
2. **Budget Leaks**: Loop quantity tracking is incorrect when closing positions in One-way mode.
3. **Fee Confusion**: No single source of truth for fee calculation (Actual vs. Estimated).
4. **Exit Logic Error**: Auto-Exit triggers are ignoring fees because they use raw UPL instead of Net PnL.
5. **Sync Issues**: REST API is an incomplete version of the WebSocket API, causing dashboard instability.
