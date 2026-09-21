# Triple-top and triple-bottom detector

Source: [triple_top_bottom.pine](triple_top_bottom.pine). This is a separate indicator; the existing Fibonacci/EMA script is unchanged. Add it in a separate Pine Editor script. Intended starting chart: standard Nifty 5-minute candles, although the code does not enforce a ticker or timeframe.

## Detection rules

1. Find confirmed high/low pivots using TradingView's `ta.pivothigh` and `ta.pivotlow`. Default: two candles on each side. On a 5-minute chart a pivot becomes known 10 minutes after its candle closes.
2. Evaluate the last three consecutive confirmed highs for a top, or lows for a bottom. The detector does not skip an intervening mismatched pivot to assemble a better match.
3. Require at least four bars between each pair of touches and at most 100 bars from first to third, by default.
4. Require the highest minus lowest touch price to be no more than 0.25 times the current 14-period ATR. Alternatively select Points mode (default point value: 10). Tolerance and pullback thresholds are measured on the detection candle.
5. Require a pullback between each pair of touches. For a top, the intervening low must be at least 0.5 ATR below the lowest of the three peaks. For a bottom, the intervening high must be at least 0.5 ATR above the highest of the three troughs. The pivot candles themselves are excluded from these intervening ranges.
6. By default, all pivots and their confirmation windows must fall within one IST calendar day. This is a date filter, not a regular-hours filter. Disable the day option for multi-day patterns.

There is no preceding-trend, EMA, volume, Fibonacci, or morning-session requirement. These are candidate chart patterns, not automatic trade entries.

## Zones and confirmation

- Triple top: orange zone from the lowest to highest of the three swing highs.
- Triple bottom: teal zone from the lowest to highest of the three swing lows.
- Exact equal prices give a zero-height zone, visible as a horizontal level.
- The top neckline is the lower of the two intervening lows; the bottom neckline is the higher of the two intervening highs. It is drawn as a horizontal dashed line, rather than a sloping line between pivots.
- Detection labels appear on the actual confirmation candle. Boxes extend back to the first touch for visual context, but were not available for trading at that earlier time. They are created only once the third pivot is confirmed.
- Skip new candidates if the detection close is already beyond their neckline or invalidation edge. Do not emit a historical breakout retroactively.
- A separate BREAK marker requires a later close crossing below the top neckline or above the bottom neckline, comparing the previous close with that fixed level.
- A top retires if a candle closes above the highest peak; a bottom retires if a candle closes below the lowest trough. Wick breaches alone do not retire them.
- Patterns also retire after breakout, more than 50 bars after detection (adjustable), or a new day when the day filter is enabled.
- Keep at most one active top and one active bottom. New same-type candidates are ignored while one is active. After retirement, later consecutive-pivot candidates can overlap an older pattern.
- Retired drawings stop extending but remain historical. TradingView retains up to 100 boxes and 100 neckline lines; older objects are removed automatically.

## Alerts and use

Paste the complete script into a new TradingView Pine Editor script, save, and add to the chart. Four alert conditions are available: top detected, bottom detected, top neckline broken, and bottom neckline broken. Use Once Per Bar Close. Alerts and pattern decisions use completed candles.

If visually similar touches are missed, inspect the price-spread tolerance, spacing, pullback depth, and pivot strength. These adjustable defaults are a starting specification, not a calibration proven to match the supplied screenshot.

## Validation

Code reviewed locally; TradingView compilation and runtime behavior remain unverified. No performance or profitability testing has been done.

Implementation references: [TradingView pivot/drawing examples](https://www.tradingview.com/pine-script-docs/faq/visuals/) and [lines and boxes](https://www.tradingview.com/pine-script-docs/visuals/lines-and-boxes/).
