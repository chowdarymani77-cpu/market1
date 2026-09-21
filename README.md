# Nifty Golden Zone + 9 EMA

Source: [nifty_morning_golden_zone.pine](nifty_morning_golden_zone.pine)

Pine Script v6 indicator for standard Nifty 5-minute candles. It draws two session-based setups and provides BUY/SELL alerts. It does not place orders or implement stops, targets, exits, position sizing, or a backtest.

## Schedule (India time)

All times use Asia/Kolkata, Monday through Friday, independently of chart display timezone. Session membership is based on candle opening time.

| Setup | Candle opening times | Zone source |
| --- | --- | --- |
| First hour | 09:15 through 10:10 | Previous trading session's last completed confirmed swing |
| Later session | 10:15 through 15:25 | Today's 09:15-10:00 range, followed by opposite swings after invalidation |

The 10:10 candle can confirm an early entry at 10:15. The 10:15 candle belongs to the later setup. Touch windows never transfer between setups. The chart must contain previous-session history to calculate an early zone. The indicator does not restrict the ticker; select Nifty yourself.

## 1. Previous-day zone: first hour only

Use the last completed swing confirmed during the previous trading session available in the chart, rather than the previous calendar date. This handles weekends and holidays with no chart bars.

- A confirmed swing high has two strictly lower highs on each side; a swing low has two strictly higher lows on each side.
- All five pivot-window candles must belong to the same regular session, 09:15-15:30. Equal highs/lows do not qualify. A candle qualifying as both a high and low is ignored because its internal ordering is unknown.
- Confirmation occurs at the close of the second candle to the pivot's right, 10 minutes after the pivot candle closes. Unconfirmed end-of-day swings are excluded.
- Each confirmed high pairs with the most recent earlier confirmed low to form a low-to-high swing. Each confirmed low pairs with the most recent earlier confirmed high to form a high-to-low swing. Both anchors must define a positive range.
- The last such completed pair supplies the next session's zone. Low-to-high means BUY; high-to-low means SELL. There is no morning momentum filter for this early setup.
- On the 09:15 candle, initialize one fixed zone. Its first touch can be recorded at that candle's close. Without a prior completed pair or the 09:15 candle, no early setup is available. The script uses available chart history and does not certify a complete prior-day dataset.
- Evaluate invalidation from today's first candle close; prior-day price movement after swing confirmation is not an additional validity filter.
- Retire this zone after its first entry, a close beyond its deep edge, or the end of the first hour. Do not replace it or reverse its direction. This single-use rule applies even if the daily signal limit is disabled.

The first possible early entry confirms at 09:25, following a touch on the 09:15 candle. The last possible early entry confirms at 10:15.

## 2. Morning-range setup: from 10:15

Observe nine candles opening at 09:15, 09:20, ..., 09:55. At 10:00, freeze:

| Symbol | Definition |
| --- | --- |
| O | Open of the 09:15 candle |
| H | Highest morning high |
| L | Lowest morning low |
| C | Close of the 09:55 candle |
| R | H - L |
| p | Momentum threshold, default 0.25 |

Require the 09:15 candle, exactly nine morning candles, and R > 0.

- Bullish: C > O and C >= H - R * p. Begin with a BUY zone.
- Bearish: C < O and C <= L + R * p. Begin with a SELL zone.
- Neither: no later setup or later reversal sequence for that day. The early setup is independent.

This momentum definition does not require minimum points, volume, ATR, directional candle counts, or chronological high/low ordering.

The initial later zone is calculated at 10:00 but is displayed and processed for touches and invalidation only from the candle opening at 10:15. Price action from 10:00 through 10:15 does not arm or invalidate this later zone. Its first possible entry confirms at 10:25, following a touch on the 10:15 candle.

### Later-zone failure and replacement

- Buy-zone failure: a completed candle closes strictly below the zone bottom. Switch the search direction to SELL.
- Sell-zone failure: a completed candle closes strictly above the zone top. Switch the search direction to BUY.
- Invalidation precedes entry evaluation; the breaking candle cannot signal from the failed zone. Pending touches are cleared.
- For a replacement SELL, wait for a confirmed pivot low whose candle is at or after the breaking candle. Pair it with the most recent confirmed high strictly before that low.
- For a replacement BUY, wait for a confirmed pivot high whose candle is at or after the breaking candle. Pair it with the most recent confirmed low strictly before that high.
- Use the same strict two-candle pivot rule described above. Both anchors must be in today's session and have positive range. The starting pivot may precede the break.
- Reject a candidate buy zone if any close from the endpoint through its confirmation is below the candidate bottom. Reject a candidate sell zone if any such close is above its top.
- Use the first eligible confirmed pair. Otherwise wait for the next eligible endpoint in the requested direction. No active zone means no entry.
- A replacement activates at the confirmation close. Touch eligibility begins with the following candle; an entry can occur on a subsequent candle.
- Keep each active zone fixed until it fails, then repeat the opposite-direction search. The later setup retains this replacement behavior even after an entry, subject to the daily signal limit.

## 3. Golden-zone calculations

Use the same formulas for the previous-day swing, the initial morning range, and later replacement swings. Let H and L be the applicable anchors and R = H - L.

| Direction | Zone bottom | Zone top |
| --- | --- | --- |
| BUY | H - R * deep | H - R * shallow |
| SELL | L + R * shallow | L + R * deep |

Defaults: shallow = 0.50, deep = 0.618. For H = 25,200 and L = 25,000, the buy zone is 25,076.4-25,100; the sell zone is 25,100-25,123.6.

A wick beyond a boundary does not invalidate a zone. A close exactly on the deep boundary also does not invalidate it. There is no separate stop rule based on the morning high/low.

## 4. Touch and three-candle entry window

A touch means low <= zoneTop and high >= zoneBottom. A wick touching the boundary counts. The touch candle may have either body direction or be a doji, but must not invalidate the zone.

- Every eligible touch restarts a window for the NEXT three candles. If T touches, T+1, T+2, and T+3 may signal; T cannot signal from its own touch and T+4 is too late without a retouch.
- Consecutive candles overlapping the zone each count as a new touch. For example, a retouch at T+3 permits entries through T+6.
- Check entry against the previous touch before recording the current candle's touch. Thus a retouch candle may itself signal within the previous touch's window.
- The entry candle does not need to touch the zone.
- A signal consumes the window. Invalidation, zone replacement, session changes, and a new day clear pending windows.
- Windows cannot extend beyond the applicable setup's session.

## 5. Exact BUY and SELL conditions

Every entry requires a confirmed candle, an active non-invalidated zone, a valid prior touch within three candles, the matching setup session, and an available daily signal allowance.

| Entry | Candle body | EMA crossover |
| --- | --- | --- |
| BUY | close > open | Current close > current EMA9 and previous close <= previous EMA9 |
| SELL | close < open | Current close < current EMA9 and previous close >= previous EMA9 |

A doji cannot enter. Merely being above/below the EMA is insufficient; a fresh crossover is required. No requirement forces the candle to open on the opposite side of the EMA. EMA9 uses closing prices and runs continuously across days.

## 6. Daily allowance and settings

| Setting | Default | Meaning |
| --- | --- | --- |
| Morning close within top / bottom % of range | 25 | Later setup's momentum threshold; range 1-50 |
| Shallow retracement | 0.50 | Range 0-1; must be less than deep |
| Deep retracement | 0.618 | Range 0-1; must be greater than shallow |
| Only one signal per day | Enabled | Shared across early and later setups, both directions |
| Show frozen morning high / low | Enabled | Display only; no effect on entry conditions |

IMPORTANT: With the daily limit enabled, an early entry prevents a later entry that day. Disable it to permit entries from both setups. The early zone still allows only its first entry. Later zones can continue producing entries from fresh touches when the limit is disabled.

Active zones, pivot anchors, pending windows, and the daily allowance reset each IST day. The last completed session swing is retained separately for the next trading session's early setup. A session with no confirmed swing provides no early zone for the following session; the script does not fall back to an older swing.

## Display and use

- Orange: 9 EMA.
- Aqua zone: previous-day swing during the first hour; removed from the entry/invalidation candle onward when retired.
- Gold boundaries/yellow shading: later setup's active zone. No zone while waiting for replacement.
- Green/red lines: frozen morning high/low when enabled.
- BUY/SELL labels remain on historical signal candles. Zones are not backdated to their swing endpoints.

Copy the entire source file into TradingView Pine Editor, save, and add it to a standard 5-minute Nifty chart. Create alerts for `Nifty golden zone BUY` and `Nifty golden zone SELL`, selecting Once Per Bar Close. Adding the indicator alone does not create alerts. Recreate existing alerts after updating the code so they use the new version.

## Validation status

The implementation and documentation have been reviewed locally. Pine compilation and execution in TradingView are still unverified, and no backtest or profitability results have been established. Local scenario checks are not a substitute for TradingView validation.
