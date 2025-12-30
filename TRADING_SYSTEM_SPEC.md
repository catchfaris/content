# Trading System Specification

## Spec

### Definitions
- **Symbol**: A tradable instrument (e.g., XAUUSD, EURUSD). Rules apply independently per symbol.
- **Pair**: Two simultaneous trades on the same symbol: exactly one Buy and one Sell opened at the same time.
- **Active pair**: The currently open Buy/Sell trades for a symbol. Only one active pair is allowed per symbol at any time.
- **Equity snapshot at open**: Account equity in USD captured at the moment each trade of a pair is opened. Used to compute that trade's profit/loss thresholds and lot size.
- **TP threshold**: +15% of the equity snapshot, denominated in USD, per trade. Example: equity snapshot USD 10,000 → TP = +USD 1,500 for that trade.
- **SL threshold**: −5% of the equity snapshot, denominated in USD, per trade. Example: equity snapshot USD 10,000 → SL = −USD 500 for that trade.
- **Lot size rule**: 1 lot corresponds to 1% of current equity balance at the time of opening. Both Buy and Sell in a pair use the same lot size.

### State / Flow per Symbol
1. **Idle**: No active pair exists for the symbol.
2. **Open pair**: Exactly one Buy and one Sell trade are open for the symbol. Both share the same opening timestamp for concurrency purposes, but each retains its own equity snapshot and thresholds.
3. **Partial closure**: One side of the pair is closed (manually or automatically); the other remains open until its own closure condition is met.
4. **Reset / Reopen eligibility**: When both sides of a pair are closed, the symbol returns to Idle and becomes eligible to open a new pair.

### Entry Rules
- When a symbol is Idle, open exactly two trades simultaneously: one Buy and one Sell.
- Do not open a new pair for a symbol if any trade for that symbol is already open.
- Multiple symbols may each maintain their own active pair concurrently without restriction.
- Lot size for both Buy and Sell is computed once at the moment of opening using the lot size rule (1 lot = 1% of current equity balance).

### Exit Rules
#### Manual Closure
- Operator may manually close the Buy, the Sell, or both at any time.
- Manual closure of both trades immediately transitions the symbol to Idle and triggers eligibility for automatic reopening.
- Manual closure of only one side leaves the remaining side active until it closes automatically or is manually closed.

#### Automatic Closure
- Each trade closes independently when either condition is reached relative to its equity snapshot:
  - Profit ≥ TP threshold (+15% of snapshot equity) → close trade.
  - Loss ≤ SL threshold (−5% of snapshot equity) → close trade.
- Thresholds remain fixed per trade and do not update after opening.

### Reopen Rules
- A new pair (Buy + Sell) for a symbol may be opened only after both trades of the previous pair are closed (manually or automatically).
- If only one side closes, no new trades for that symbol open until the remaining side also closes.
- Manual closure of both sides triggers immediate eligibility to reopen; manual closure of one side does not.

### Lot / Volume Calculation Rule
- Determine the account's current equity balance in USD at trade entry time.
- Calculate 1% of that equity balance; this defines “1 lot.”
- Use the same lot size for both Buy and Sell in the pair, proportional to the 1% rule. (Exact broker volume mapping must follow platform/broker constraints.)

### Data to Store Per Trade
- Symbol.
- Trade direction (Buy or Sell).
- Equity snapshot at trade open (USD).
- TP threshold value (USD) computed as +15% of equity snapshot.
- SL threshold value (USD) computed as −5% of equity snapshot.
- Lot size applied at open.
- Timestamps: open time; close time when applicable.
- Closure reason: manual, TP hit, SL hit.

## Advantages
- Predictable risk/reward: Fixed TP/SL percentages anchored to equity snapshot give consistent per-trade risk in USD terms.
- Symmetric hedging: Simultaneous Buy/Sell per symbol balances directional exposure while waiting for one side to resolve.
- Simple state control: Single active pair per symbol avoids overlapping logic and simplifies reopen gating.
- Equity-proportional sizing: Lot size scales with current equity, keeping position sizing consistent through growth or drawdown.
- Automatic recovery: Auto-reopen after both sides close maintains continuous operation without manual intervention.

## Open Questions
- If equity changes between the Buy and Sell submission timestamps, should both trades use the same equity snapshot (the first trade's value) or each use its own snapshot at its exact open timestamp?
- How to map the “1 lot = 1% of equity” rule to broker-specific volume increments (e.g., minimum lot size, step size) when 1% translates to a non-standard volume?
- Are partial closes (reducing volume without closing the entire position) allowed, and if so, how do they affect reopen eligibility and threshold tracking?
- Should the system attempt immediate reopening if a manual closure happens outside trading hours or during maintenance windows?
- Does the TP/SL evaluation use gross profit/loss only, or net of commissions/swaps, when checking the USD thresholds?

## Test Cases
1. **Auto TP Buy / Auto SL Sell**: Open pair on a symbol; price moves favorably for Buy to hit +15% threshold while Sell hits −5%; verify Buy closes for profit, Sell for loss, and reopen happens only after both are closed.
2. **Manual Close One Side**: Manually close the Buy while Sell remains open; confirm no new trades open until Sell also closes.
3. **Manual Close Both**: Manually close both Buy and Sell; confirm the symbol immediately becomes eligible to open a new pair.
4. **Auto SL Both**: Both sides hit −5% thresholds; ensure closures occur independently and reopening waits until both are closed.
5. **Auto TP Both**: Both sides reach +15% thresholds (e.g., via volatility spikes); confirm both close and a new pair can open afterward.
6. **Multi-Symbol Independence**: Run pairs on XAUUSD and EURUSD concurrently; closing conditions on one symbol must not affect the other symbol's state or reopen timing.
7. **Equity Snapshot Validation**: With equity changing between the Buy and Sell opens, verify each trade uses the intended snapshot to compute its TP/SL per the decided rule.
8. **Restart/Recovery**: After platform restart or reconnection, ensure stored data (equity snapshot, thresholds, lot size, state) persist so that open trades continue with correct TP/SL logic and reopen gating.
9. **Lot Size Scaling**: Equity increases from USD 10,000 to USD 12,000; next pair should size at 1% of USD 12,000 for both Buy and Sell.
