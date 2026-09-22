# Nifty Morning Golden Zone + EMA

This repository now contains only the morning Fibonacci setup active from 09:55 IST. The previous-day early setup and triple-top/bottom detector have been removed from the current version; earlier versions remain in Git history.

The setup starts with the candle opening at **09:55 AM IST**. The measurement range is **09:15-09:55**, using the eight candles opening from 09:15 through 09:50. The range freezes at 09:55, before the first eligible touch candle opens. Existing opposite-swing replacement rules remain enabled.

Source: [nifty_morning_golden_zone.pine](nifty_morning_golden_zone.pine)

This Pine Script v6 indicator identifies morning direction, fixes a Fibonacci pullback zone, and marks BUY or SELL signals when a directional candle crosses the 9 EMA within the three candles after a zone touch. It draws signals and exposes alert conditions; it does not place orders or implement a backtest strategy.

## Chart and time requirements

- Intended chart: Nifty, standard 5-minute candles. The script rejects other timeframes and synthetic chart types. It does not restrict the ticker, so select Nifty yourself.
- All session calculations use `Asia/Kolkata` (IST), independently of the chart display timezone.
- Sessions run Monday through Friday, when chart data exists.
- Morning observation: 09:15 inclusive to 09:55 exclusive. This includes eight candles opening at 09:15, 09:20, ..., 09:50.
- The morning range and initial direction are established when the 09:50 candle closes at 09:55. The candle opening at 09:55 is excluded from the morning calculation. Later zone failures can reverse the active direction.
- Touch and signal candles must open from 09:55 inclusive to 15:30 exclusive. The earliest touch confirms at 10:00, and the earliest possible signal confirms at 10:05. The last possible signal confirms at 15:30 on the 15:25 candle.
- Morning values and the daily signal allowance reset on each new IST calendar date. The EMA is continuous across days and does not reset.

## 1. Measure the morning

Define:

| Symbol | Meaning |
| --- | --- |
| O | Open of the 09:15 candle |
| H | Highest high of the eight morning candles |
| L | Lowest low of the eight morning candles |
| C | Close of the 09:50 candle, known at 09:55 |
| R | Morning range: H - L |
| p | Momentum threshold as a fraction; default 0.25 |

The script requires the 09:15 opening candle, exactly eight morning candles, and a positive range before a direction can qualify. Incomplete morning data produces no trading setup.

## 2. Decide morning direction

| Direction | Exact conditions, all required |
| --- | --- |
| Bullish | C > O, and C >= H - R * p |
| Bearish | C < O, and C <= L + R * p |
| Neither | Neither qualifying condition is met; no golden zone or signals for that day |

With the default p = 0.25, bullish means the morning closes above its open and in the top 25% of the range. Bearish means it closes below its open and in the bottom 25%.

This is the implemented definition of initial momentum. It does not measure minimum points moved, volume, ATR, consecutive directional candles, or the order in which the morning high and low occurred. A bullish morning starts with a buy zone; a bearish morning starts with a sell zone. Each subsequent active zone failure reverses the direction.

## 3. Calculate the golden zone

Default retracement inputs are 0.50 (shallow) and 0.618 (deep).

The initial Fibonacci anchors now differ from the full morning H/L used for the momentum filter. Let FH/FL be the selected Fibonacci high/low and FR = FH - FL.

| Morning direction | Zone bottom | Zone top |
| --- | --- | --- |
| Bullish: retracement down from FH | FH - FR * 0.618 | FH - FR * 0.50 |
| Bearish: retracement up from FL | FL + FR * 0.50 | FL + FR * 0.618 |

### Initial morning anchor selection

- BUY: FL is the 09:15 candle's low; FH is a matched upper level from a green/red pair among the remaining 09:20-09:50 candles.
- SELL: FH is the 09:15 candle's high; FL is a matched lower level from a green/red pair among the remaining 09:20-09:50 candles.
- The green/red candles need not be adjacent. Green means close > open, red means close < open; dojis are excluded. The opening candle supplies only the opening anchor, not one of the matching pair.
- A match is an actual overlap between wick/body price intervals. There is no point-tolerance setting and wick tips need not be equal. Intervals touching at one exact price also qualify.
- Search all green/red pairs in the 09:20-09:50 window, including non-adjacent candles. Do not restrict the search to the two independently most extreme candles.
- For BUY, a wick interval runs from the upper body edge to the high. For SELL, it runs from the low to the lower body edge. The body interval always spans min(open, close) to max(open, close).
- Check wick-wick, both wick-body combinations, and body-body overlap. Zero-length wicks do not qualify as wick intervals; the candle may still supply its body. The opening anchor always uses its high/low even if its wick has zero length.
- For each overlap, take its highest shared price for BUY or lowest shared price for SELL. Then choose the highest candidate across all pairs for BUY, or lowest for SELL. Do not average the two prices.
- At the same extreme price, use precedence **wick-wick -> wick-body -> body-body**. Price extremity comes first: a lower wick-wick overlap cannot displace a higher body overlap for BUY (and vice versa for SELL).
- Candidates must produce a positive range against the opening anchor. If either color is absent or no valid overlap exists, no initial zone or subsequent reversal sequence starts that day. There is no fallback to an unshared morning extreme.
- Example: upper wicks spanning 100-110 and 106-115 overlap at 106-110, so the bullish candidate anchor is 110 even though their tips differ by 5 points. Lower wicks spanning 80-90 and 85-95 give a bearish candidate anchor of 85.
- The original momentum filter still uses all eight candles' highest high and lowest low. This change affects Fibonacci anchors only.

Initial levels freeze at 09:55. Replacement zones still use confirmed swing high/low wicks with the existing rules below; the green/red matching rule does not apply to them. Active zones stay fixed until invalidated.

Example: if the selected FH = 25,200 and FL = 25,000, then FR = 200. A bullish setup gives a zone of 25,076.4 to 25,100. A bearish setup gives a zone of 25,100 to 25,123.6. These illustrate two possible directions on separate days.

## 4. BUY conditions

All of the following must hold on the same completed signal candle:

1. There is an active buy zone, from either the bullish morning or a later reversal.
2. The candle opens within the allowed signal session.
3. It is candle 1, 2, or 3 after an eligible touch of the active zone. The entry candle itself does not need to touch the zone.
4. It is bullish: `close > open`.
5. Its close crosses above the 9 EMA: current `close > EMA9`, and previous candle `close <= previous EMA9`.
6. The zone has not been invalidated, and the signal candle is later than its activation candle.
7. If the one-signal-per-day option is enabled, no earlier signal has occurred that day.

The BUY label is confirmed at this candle's close.

## 5. SELL conditions

All of the following must hold on the same completed signal candle:

1. There is an active sell zone, from either the bearish morning or a later reversal.
2. The candle opens within the allowed signal session.
3. It is candle 1, 2, or 3 after an eligible touch of the active zone. The entry candle itself does not need to touch the zone.
4. It is bearish: `close < open`.
5. Its close crosses below the 9 EMA: current `close < EMA9`, and previous candle `close >= previous EMA9`.
6. The zone has not been invalidated, and the signal candle is later than its activation candle.
7. If the one-signal-per-day option is enabled, no earlier signal has occurred that day.

The SELL label is confirmed at this candle's close.

## Zone invalidation and direction reversal

Invalidation is evaluated on completed candles during the 09:55-15:30 entry session, before checking entry conditions.

| Active zone | Invalidation | Next direction and swing |
| --- | --- | --- |
| Buy | Close strictly below zone bottom | Switch to SELL; find a confirmed high-to-low swing |
| Sell | Close strictly above zone top | Switch to BUY; find a confirmed low-to-high swing |

A wick beyond the boundary, or a close exactly on it, does not invalidate the zone. Once broken, the old zone disappears from that candle onward and cannot produce new signals. Historical zone plots and previously confirmed signals remain visible. No signal is allowed from the breaking candle.

### Finding the immediate opposite swing

1. Track confirmed pivots throughout the current day's regular session, including the morning. A pivot high has two strictly lower highs on each side. A pivot low has two strictly higher lows on each side. Equal highs/lows do not qualify. All five candles must belong to the same day's regular session.
2. A pivot becomes known only at the close of the second candle to its right (10 minutes after the pivot candle closes). Nothing is backdated to the pivot candle.
3. After a buy zone breaks, wait for a pivot low whose candle is at or after the breaking candle. Pair it with the most recent confirmed pivot high strictly before that low. This forms the new high-to-low sell swing.
4. After a sell zone breaks, wait for a pivot high whose candle is at or after the breaking candle. Pair it with the most recent confirmed pivot low strictly before that high. This forms the new low-to-high buy swing.
5. The starting pivot may precede the break, but both anchors must be from the current session and define a positive price range. A candle that qualifies as both a pivot high and pivot low is ignored because its internal ordering is unknown.
6. Apply the same Fibonacci formulas to that swing's high and low. Reject a candidate buy zone if any close from its endpoint through its confirmation is below its bottom. Reject a candidate sell zone if any such close is above its top.
7. Use the first eligible confirmed swing. If a candidate is rejected or lacks a starting anchor, stay in the requested opposite direction and wait for the next eligible endpoint. No active zone means no signals.
8. Activate the replacement at the confirmation candle's close. A new touch can arm a window beginning on the following candle; the earliest entry is one candle after that touch. Keep the replacement fixed until it breaks, then reverse again using the same rules. Touch windows never carry over between zones.

Example: bullish morning -> buy zone -> close below its lower edge -> wait for confirmed high-to-low swing -> sell zone -> zone touch -> bearish candle crossing below EMA9 within the next three candles -> SELL, provided the daily signal allowance remains available.

All pivot tracking, pending reversals, and active zones reset on the next IST date. Days with no qualifying morning direction never start this reversal sequence.

## Exact entry behavior and limits

- The EMA uses closing prices with length 9.
- A touch means `low <= zoneTop` and `high >= zoneBottom`. A wick touching a boundary counts; the touch candle can be bullish, bearish, or a doji. A close invalidating the zone takes priority and cannot arm a window.
- If candle T touches the zone, only T+1, T+2, and T+3 can signal from that touch. T itself cannot signal from its own touch; T+4 is too late. For example, a touch on the 10:15 candle permits signals on the 10:20, 10:25, and 10:30 candles at their closes.
- Every eligible touch, including a retouch during an open window, restarts the countdown for the next three candles. Consecutive candles overlapping the zone each count as a new touch. For example, a touch at T and a retouch at T+3 allow entry at T+4, T+5, or T+6 from the retouch.
- Entry is checked against the previous touch before recording the current candle's touch. Thus a retouch candle can itself signal if it is within the previous touch's three-candle window and meets the entry conditions. A first touch, or a retouch after the previous window expired, cannot signal from its own touch.
- A signal consumes its window. Zone invalidation, replacement, leaving the entry session, and a new day clear any pending touch. A new signal needs a new touch, and the daily limit still applies.
- The entry candle need not overlap the zone. It must have the correct bullish/bearish body and a fresh EMA crossover on that same entry candle.
- Merely closing above the EMA for a buy, or below it for a sell, is insufficient: a fresh close-to-close crossover is required.
- There is no requirement for the candle to open on the opposite side of the EMA. The crossover compares the current and previous closes with their respective EMA values.
- A doji (`close == open`) cannot trigger either signal.
- The script does not require a separate departure from the zone before a touch, or a rejection close outside the zone.
- Closing beyond the active zone's deep edge invalidates it and switches direction, as described above. There is no separate morning-high/low invalidation rule.
- By default, only the first qualifying signal is shown each day across both directions. Zone invalidations and replacements continue after that signal, but cannot produce another entry unless this option is disabled. A reversal does not reset the daily allowance.
- Signals require completed candles. The zone is only established after the morning closes; no future bars are used in the calculations.
- No stop-loss, take-profit, exit, position sizing, order execution, or performance statistics are implemented. A signal marks a condition at candle close, not a guaranteed execution price.

## Settings

| Input | Default | Effect |
| --- | --- | --- |
| Morning close within top / bottom % of range | 25 | Allowed range: 1-50. Smaller values require a close nearer the directional extreme. |
| Shallow retracement | 0.50 | Allowed range: 0-1; must be less than the deep retracement. |
| Deep retracement | 0.618 | Allowed range: 0-1; must be greater than the shallow retracement. |
| Only one signal per day | Enabled | Limits the day to its first qualifying BUY or SELL. |
| Show frozen morning high / low | Enabled | Controls the morning high/low lines; does not change signals. |

The 5-minute timeframe, session times, timezone, EMA length, and three-candle post-touch window are fixed in the code.

## Display and alerts

- Orange line: 9 EMA.
- Green/red lines: frozen morning high/low, when enabled. A complete but directionally unqualified morning can still show these lines.
- Blue lines: selected initial Fibonacci anchors, controlled by the same high/low display option. They remain the initial anchors even if a later replacement zone forms.
- Gold lines and yellow shading: the active golden zone, initially based on the morning and later on a confirmed opposite swing. No zone is shown while waiting for a replacement.
- Green BUY label below the signal candle; red SELL label above it.
- Separate alert conditions: `Nifty golden zone BUY` and `Nifty golden zone SELL`.

To use:

1. Open Nifty with standard 5-minute candles in TradingView.
2. Copy the source file into Pine Editor, save it, and add it to the chart.
3. Create separate alerts using the BUY and SELL conditions, selecting **Once Per Bar Close**.

Adding the indicator does not itself create active alerts.

## Validation status

This description was checked against the saved script. The script has not yet been compiled inside TradingView or backtested; no profitability or execution results have been established.
