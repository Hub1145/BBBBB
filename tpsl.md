# TP/SL Handling and OKX Interface Integration

This document explains how the bot manages Take-Profit (TP) and Stop-Loss (SL) orders to ensure they are visible and correctly linked to your positions in the OKX interface.

## 1. Initial Entry Orders
When the bot places a new entry order (e.g., in a strategy loop), it uses the `attachAlgoOrds` parameter. This "attaches" the TP and SL instructions directly to the entry order.

*   **Behavior:** Once the entry order fills, OKX automatically activates the TP and SL.
*   **Interface:** These will typically appear as "Order TP/SL" linked to that specific fill.

## 2. Position-Level TP/SL (The "TP/SL" Button)
To match the behavior shown in your image (where TP/SL is set for the **entire position**), the bot uses the OKX **OCO (One-Cancels-the-Other)** order type for existing positions.

### How it works:
1.  **Detection:** The bot monitors for any fills.
2.  **Synchronization:** If an order is filled (especially an Auto-Cal addition), the bot's average entry price changes.
3.  **Automatic Update:** The bot immediately:
    *   Cancels any existing TP/SL algo orders for that side.
    *   Calculates new TP/SL prices based on the **new average entry price**.
    *   Places a new **OCO order** for the **full current quantity** of the position.
4.  **Result:** This ensures that the TP/SL orders appear as a single linked pair on the position, exactly as if you had clicked the "TP/SL" button in the OKX app and set them manually.

## 3. "Exit as a Whole" Logic
One of the issues fixed in the recent audit was that Auto-Cal additions would "trap" the position if the TP/SL wasn't moved.

*   **Previous Issue:** If you had a Long at $1910$ with TP at $1920$, and the bot added more at $1850$, your new average entry might be $1880$. If the TP stayed at $1920$, you would be waiting much longer to exit.
*   **Current Solution:** The bot now detects that fill at $1850$, recalculates the $1920$ TP down to (for example) $1890$ (based on your offsets), and updates the exchange order. This keeps the "exit distance" consistent regardless of how many times the bot adds to the position.

## 4. Limit vs Market Exit
You can control whether these TP/SL orders exit at **Market** or **Limit** prices via the Dashboard settings:
*   **TP Close Mode:** If `tp_close_limit` is enabled, the bot will place a limit order at the trigger price (or your specified `tp_close_price`) instead of a market order.
*   **Safety:** The bot uses `reduceOnly: true` for these orders to ensure they can only close or reduce your position, never open a new one in the opposite direction.
