[README (1).md](https://github.com/user-attachments/files/33669285/README.1.md)
# Dealing Range Liquidity Reversal

*Formerly Session-Based Liquidity Reversals. It stopped being about the Asia,
London and New York sessions a while ago; the range is whatever window you set,
so the name now says what it trades.*

A Pine Script v6 indicator for TradingView. It waits for a dealing range to be
swept, marks the CISD that confirms the reversal, and carries the setup through
to an entry: an FVG (or the MSS candle's gap), a retest, a stop and a target.

I built it for lower-timeframe execution on index futures (MNQ/NQ, 1m to 5m). It
also runs on the 24/7 perpetuals that mirror them, held to the CME clock. Every
level comes from the dealing range and from candle bodies. There are no ATR
filters, no point thresholds and nothing tuned to make past sessions look good.

> **Read the disclaimer at the bottom before using this on a live account.**

### Status of this release

v1.02, the first public version. A working build, not a finished one.

- **Checked against my marks:** 13 Oct 2026 XYZ100 on 1h, 15m, 5m, 3m and 2m, and
  14 Oct 2026 XYZ100 on 5m. The +CISD lands on the candle I marked on all six.
- **18 Oct 2026 XYZ100, the Sunday-reopen gap sweep:** the 5m +CISD (29,181) and
  stop (29,255) match my marks on the chart. The 1h (29,270) and 3m (29,181)
  match when the exported candles are replayed through the rules.
- **Open:** on 18 Oct 15m the engine takes 29,270; I marked 29,238. Until I can
  say why, the rule stays as it is.
- **Not yet re-checked under the current rules:** my earlier sessions (1, 2, 7
  and 8 Oct), the hand-marked NQ set, the Dated DR's time zone, and the perp
  clock switched off.

---

## Installing and updating

1. Open the `.pine` file and copy everything.
2. In TradingView, open the **Pine Editor**, start a new indicator, paste the code
   over its contents, and save.
3. **Add to chart** on a chart of 4H or lower with standard candles. Execution is
   built for 1m–5m; 1H and 4H are for framing a larger dealing range.

**Updating:** remove the indicator and add it again, or reset its settings to
defaults. Then re-enter your times and recreate your alert. TradingView binds
saved settings by position, and recent versions moved inputs, so old values
would otherwise land in the wrong fields.

---

## The idea

Each chart runs one dealing range with two windows.

- The **Dealing Range (DR)** is a window you set. Its high and low are the
  liquidity pool.
- The **Execution Window (EW)** starts where the DR ends. It waits for price to
  sweep one side of the pool and turn.

The model in one line: **sweep → CISD → FVG (or MSS → FVG) → retest → execute.**

You choose the DR. Frame the swing you want: start it at the low you care about
and end it when the higher-timeframe level is taken. The indicator measures from
whatever window you give it.

It isn't a breakout tool. A sweep on its own is not a signal; the setup needs a
CISD back inside the range and an FVG that price comes back to.

If you trade CRT or turtle soup, you'll recognise the idea: price runs one side
of a range, fails, and delivers to the other side. This runs it on a dealing
range you define instead of a single higher-timeframe candle, and only arms once
the sweep, the CISD and the FVG retest have all printed.

---

## Settings, in order

Market · DR window · Dated DR · Swept level · DR / EW boxes · DR levels ·
Structure levels · Entry model · RR calculator · Fib levels · Status block · Alerts

### DR window

- **Dealing range:** start and end, in New York time. Daylight saving is handled.
- **Execution window ends:** hour : minute, two dropdowns (`06 : 00`). The EW
  always starts at the DR's end, so a 19:00–23:00 DR with an end of 10:00 gives
  an EW of 23:00 → 10:00. It may cross midnight.
- Defaults: DR 18:00–20:00, EW ends 00 : 00.
- Alerts are tagged with the DR window, e.g. `[DR 19:00–23:00]`, or `[Dated DR]`.

To watch two dealing ranges, add the indicator twice.

### Dated DR

The daily times repeat every day, so they can't describe a range that runs past
24 hours or across a weekend. **Dated DR** fixes the DR and its EW by date and
time instead: a checkbox plus **From**, **To** and **EW to**, greyed out while
it's unticked.

- From its start it is the only DR; the daily schedule is off.
- It starts on the first candle at or after From that the CME clock counts.
- The EW runs from To to EW to, which can be days later.
- Set To on the close of the candle that made the extreme you want inside. On
  11–12 Oct 2026 the low came in Monday's 22:00 hour, so the DR ran Sunday 19:00
  to Monday 23:00.

**Check the time zone once.** The daily windows are read in New York time; the
date pickers follow TradingView's own convention, which I haven't confirmed. After
setting From, check that the DR box starts on the candle you meant.

**Long windows on 1m.** Zone searches look back at most 500 candles. For a
multi-day Dated DR, use 5m or higher.

### Swept level

Sometimes the DR's own extreme is the sweep: you end the DR after it has taken a
key level (PDL, PDH, an old swing). Tick **Swept level** and type that level, not
the DR high or low itself. The DR's extreme has to trade through it.

- A level below EQ: if the DR low traded through it, the DR closes with its low
  already swept, and the DR low is the raid low. A level above EQ works the same
  for the DR high.
- The CISD is then read from the run into the DR's own extreme.
- A dotted line marks the level, and the **Swept** row in the status block says
  `armed`, `not taken` (the DR's extreme didn't trade through it) or `pending`
  (the DR is still forming).
- It applies to the latest DR only. Untick it when that DR's EW is done.

---

## The CISD

While the engine hunts, only the **run into the sweep** counts.

| Step | Rule |
|---|---|
| **Run** | The consecutive bearish candles (bullish for a short) into the raid extreme. It ends at the raid candle, or at the candle before it when the raid candle closes the other way. A doji or an opposite candle ends the run. If none of the run's candles opened inside the DR, the run keeps going back through candles sitting wholly outside the DR until it reaches the one that opened inside |
| **Sweep candles** | Every candle that traded beyond the DR level, until the CISD is accepted |
| **Acceptance** | The first body close **above the sweep** (above the highs of all the sweep candles), or above the whole run. A close still inside the sweep's wick doesn't count |
| **CISD** | The run candle that close takes: the **closest** one, the highest open still below the close (the lowest above it, for a short), at or beyond the swept DR level |
| **Block** | From that candle to the end of the run, across the bodies |
| **Re-read** | A lower low (higher high) before acceptance re-reads the run from the new extreme |

One CISD per raid extreme. After a CISD dies, the next one needs a new extreme.

| Chart | Accepted by | +CISD | Block |
|---|---|---|---|
| 14 Oct 5m | 02:10 close 30,582, above the 02:00–02:05 sweep (high 30,551) | 01:50 open, 30,575 | 30,516–30,575 |
| 13 Oct 1h | 09:00 close 30,580, above the whole run | 06:00 open, 30,545 | 30,444–30,545 |
| 13 Oct 15m | 09:30 close 30,554, above the whole run | 08:30 open, 30,464 | 30,409–30,464 |
| 13 Oct 5m | 09:30 close 30,434, above the whole run | 09:25 open, 30,430 | 30,409–30,430 |
| 13 Oct 3m | 09:36 close 30,521, above the sweep | 09:27 open, 30,435 | 30,386–30,435 |
| 13 Oct 2m | 09:36 close 30,495, above the sweep | 09:26 open, 30,436 | 30,399–30,436 |

On 16 Oct (short) the 1m and 2m runs into the 30,031 high were cut by 2-point red
candles above the DR high, and every candle left opened outside the DR. Before
v7.27 they never printed a -CISD. From v7.27 they walk back to 01:00 like the 3m and 5m:

| Chart | Accepted by | -CISD | Block |
|---|---|---|---|
| 16 Oct 1m | 01:10 close 30,004 | 01:00 open, 30,010 | 30,010–30,029 |
| 16 Oct 2m | 01:10 close 29,998 | 01:00 open, 30,010 | 30,010–30,029 |
| 16 Oct 3m | 01:09 close 29,998 | 01:00 open, 30,010 | 30,010–30,029 |
| 16 Oct 5m | 01:15 close 30,002 | 01:00 open, 30,010 | 30,010–30,023 |

On 14 Oct the 02:05 candle closed at 30,550, over the 02:00 open, but still inside
the wick the 02:00–02:05 sweep makes on a higher timeframe. The engine waits for
02:10.

### Gap sweep

Sometimes the reopen candle does the sweep: the first candle after the daily halt
or the weekend opens with a gap toward the DR extreme and trades through it. There
is no run into that sweep. The gap is the leg, and the candles before the close
are just drift into the halt.

- The CISD then comes from the **run into the DR's own extreme**, picked the same
  way: the highest open still below the accepting close (lowest above, for a
  short).
- The reopen candle can accept it with its own close. If it doesn't, later
  candles accept by the normal rule.
- The run is stored when the DR extreme forms, so it still reads on 1m–5m after a
  weekend has pushed it more than 500 candles back.

18 Oct 2026 XYZ100, CME clock on. The DR low (29,094) printed Friday 15:00. The
Sunday 18:00 reopen gapped down to 29,087 and closed back up:

| Chart | Run into the DR low | Accepted by | +CISD |
|---|---|---|---|
| 1h | Fri 11:00–14:00 | Sun 18:00 close 29,421 | Fri 14:00 open, 29,270 |
| 5m | Fri 15:00–15:05 | Sun 18:00 close 29,256 | Fri 15:00 open, 29,181 |
| 3m | Fri 15:00–15:06 | Sun 18:00 close 29,249 | Fri 15:00 open, 29,181 |

## OBs and PBs

Once a CISD is held, later blocks on its side are read from swings.

| Step | Rule |
|---|---|
| **Swing** | A high (low) strictly higher (lower) than the two candles either side. A tie is not a swing |
| **Leg** | From the previous swing to the swing it ends at, inside the EW |
| **Level** | The open of the earliest opposing candle of the leg that a close goes back through |
| **Confirmation** | A candle in the reversal direction that closes through it and advances past the highest (lowest) living block on its side |
| **Block** | The bodies of the opposing candles from that anchor on |

A **PB** is a single-candle block that trades into the most recent living OB,
doesn't close through that OB's mean threshold `(open + close) / 2`, and sits
beyond it. One PB per parent.

OBs and PBs are hidden by default (tick **OB / PB**). They're still detected;
nothing in the entry model uses them.

---

## Invalidation

Every PD array dies by itself on **two consecutive body closes beyond it**. A
wick never kills anything.

| PD array | What it is | When it dies |
|---|---|---|
| **DR high / low** | a line | That side waits for a close back inside the DR before a CISD can form on it again |
| | | A close beyond it by **a quarter of the DR's length**: the reversal on that side is off for this DR; define a new one. A CISD held on that side dies |
| **CISD** | its block | Its OBs, PBs and FVG die with it, an open trade is flattened at that close, and the engine hunts for the next CISD |
| **OB / PB** | its block | That block only |
| **FVG** | the three-candle gap, or the merged zone | That zone only; while the CISD holds, the engine looks for the next FVG |

---

## Entry model

The sweep, the CISD and an FVG that gets retested carry the trade. The MSS is
drawn when it prints but doesn't gate the entry.

1. **Path A: sweep → CISD → FVG.** The first FVG in the reversal direction after
   the raid extreme. If the MSS prints and no FVG has been presented, the **IFVG**
   is the fallback: a gap from the raid leg that the displacement inverted.
2. **Path B: sweep → CISD → MSS → FVG.** The gap whose middle candle is the MSS
   candle.

Both limits can be live together. The near edge touched first fills and the
other is dropped. A zone can't fill on the candle that armed it.

| Rule | Detail |
|---|---|
| **MSS** | The first close through the swing that built the raid leg |
| **Zones** | Volume imbalances widen the gap to the neighbouring body, and imbalances chained just before its first candle join it |
| **Stacked FVGs** | **First gap only** (default): path A trades the first gap alone. Best R, but in a fast move it may never fill. **Merge**: gaps sharing a candle become one zone; the entry moves to the new near edge and the stop stays put. Fills more often at a wider risk |
| **Entry** | Limit at the zone's near edge, which must sit inside the DR |
| **Stop** | Beyond the zone's first candle: its low for a long, its high for a short. When that first candle is the raid extreme itself, the stop goes beyond the next candle instead, not at the sweep's extreme (18 Oct 5m: 29,255, not 29,087). An IFVG keeps its far edge |
| **Target** | Two checkboxes on one row under **Trade levels** (RR calculator). **TP at EQ** (on by default): EQ if it pays 2R or more, else the DR's 0.705 (measured from the swept extreme) if it pays 2R or more. **TP at the opposite DR extreme**: the DR low for a short, the DR high for a long, if it pays 2R or more. Both on: EQ (or the 0.705) is TP1 and the trade runs to the far extreme, TP2; each label shows its own R and the stop stays where it is. No target at 2R, no trade. After a stop, price reaching TP1 before the re-entry fills ends the setup |

- **Before a fill,** the setup ends on a CISD kill, delivery completing, an armed
  target trading first, or the EW closing.
- **After a fill,** it ends at the stop, the target, flat at the EW close, or flat
  at a CISD kill. The stop and the target act on the touch, the way a platform
  closes the position: the trade comes off the chart while the candle is still
  open. The EW close and the CISD kill act at the close.
- **Same candle:** a candle that touches both the entry and the target is no
  trade; OHLC can't say which came first. Live, a candle that touches the stop
  and the target goes to whichever was touched first. On history (after a
  reload) the chart has no tick order, so that candle reads as unclear.
- **Re-entry:** after a stop, while the CISD holds, the setup arms the first
  **new** FVG formed after the stop. No IFVG on a re-entry. If EQ trades first,
  the setup ends. There's no cap on the count.
- **Rounding:** targets are rounded to a placeable price, against the trade.
  On trade.xyz markets that's the venue's real step: 1.00 on XYZ100, 0.1 on SP500
  and GOLD.
- **Delivery freeze:** once price reaches the opposite DR extreme, the EW
  stops. A ✓ marks the candle. It's on by default (**Freeze the EW once delivery
  completes**, in Entry model).

### What you see

- **CISD, OB, PB:** a box over the block with the level line inside, labelled
  `+CISD`, `-OB`, `+PB` and so on. **Block boxes** (Structure levels, on by
  default) turns the boxes off; the blocks still work underneath. **Block
  opacity %** sets their fill (default 5). Structure lines run 10 bars by default.
- **MSS:** a dashed line from the broken swing to the break.
- **Zones:** `+FVG`, `-IFVG`, `+FVG (MSS)` with a dotted C.E. line. **FVG line
  length** (Entry model, default 10 bars) sets how far a zone runs past the bar
  that armed it; a zone still armed or filled keeps running with price.
- **Retest marker:** △ / ▽ on the candle that touched the limit.
- **Trade levels:** Entry, `SL -1R` and `TP xR` labels on a dotted bracket to the
  right of price, blue for Entry and TP, red for SL.
- **After a stop,** the trade comes off the chart. The zone stays until its own
  two closes.

### RR calculator and Trade levels

Three price boxes, **Entry**, **SL** and **TP**, and the **Trade levels**
dropdown.

- **Engine** (default): the chart and the status block show the indicator's own
  trade.
- **Manual:** they show the levels you typed, with the R the target pays.

Only one trade is ever shown. In Manual mode the engine still runs; its entry,
SL and TP are just not drawn.

---

## Status block

Short rows, in order. A row only shows when it has something to say.

| Row | Shows |
|---|---|
| **Status** | Where the setup stands: `DR forming` · `Waiting for sweep` · `DR L swept` · `DR L swept on gap` · `Closed below DR L` · `Expanded below DR` · `+CISD` · `Waiting for retest` · `Filled · 3.7R / 8.5R` · `Delivery complete` · `EW closed` |
| **Next** | The one thing it's waiting for: `Close > 29,270` · `No leg into sweep` · `Waiting for FVG` · `FVG / MSS FVG` · `TP1 reached` · `No trade · +FVG risk too wide` |
| **Swept** | Your swept level, when ticked: `29,094 · armed` · `not taken` · `pending` |
| **Clock** | On clocked perps only: `CME`, `CME · 28,890 ignored` (price went beyond the DR while CME was closed), or `24/7` |
| **Trade** | `Long 2.7R` with Entry, SL and TP, or `Manual long 4.6R` in Manual mode |

---

## Alerts

Create one alert with **Any alert() function call**, then choose what reaches you
with the checkboxes.

| Alert | Default | Fires |
|---|---|---|
| CISD formed | on | bar close |
| CISD invalidated | on | bar close, with the reason |
| Setup ready | on | when a zone arms, with limit, SL, TP and R |
| Zone retest | on | intrabar, on the first touch of the limit |
| Stop hit | on | intrabar, on the touch of the stop |
| Target hit | on | intrabar, on the touch of the target; also TP1 when both TP boxes are on |
| Retest confirmed | on | bar close |
| Structure active / invalidated | off | bar close |

Recreate the alert after updating the script. TradingView doesn't fire alerts on
historical bars or in Bar Replay.

---

## Perpetual futures and the CME clock

| Symbol | Mirrors |
|---|---|
| `HIP3XYZ:*` (every trade.xyz market: XYZ100, SP500, GOLD, …) | Nasdaq, S&P, gold, … |
| `BINANCE:CLUSDT.P` | Crude oil |

**Hold perpetual futures to the CME clock** (on by default) runs these on the
CME calendar: Friday 17:00 to Sunday 18:00 New York and the daily 17:00–18:00
halt are skipped. The DR and EW carry through the halt, and swings, gaps and
every scan read only the candles CME prints. Untick it to trade the perp's 24/7
hours.

With it on, nothing the perp prints over the weekend counts: no sweep, no run,
no CISD. Only where price reopens counts. If that reopen gaps through the DR,
it's a gap sweep (see the CISD section). If weekend candles traded beyond the DR,
the Clock row says so, e.g. `CME · 28,890 ignored`.

Chart the perp itself, not the futures contract; these venues fill on their own
price. CME futures and real crypto perps (BTC, ETH, …) are never clocked.

---

## Guards

The script stops with a message if:

- the chart is above 4H, or uses Heikin Ashi, Renko or other synthetic candles;
- the EW's end equals the DR's end or falls inside the DR;
- a Dated DR's To isn't after From, or EW to isn't after To.

## Known limitations

- An OB's swing needs two candles on the right, so OBs appear two candles late.
  That's lag, not repaint. The CISD doesn't wait for a swing.
- Everything runs on confirmed candles except the zone retest alert.
- The trade levels are drawn, not tested. There's no backtest behind them.
- On 1H and 4H a candle belongs to the DR or EW only if it opens inside it. Put
  the times on candle boundaries and give the DR several candles.
- On a 4H perp, the 16:00 NY candle carries the halt inside it. 1H and lower
  separate it cleanly.
- Scans look back 500 chart candles, and on a clocked perp the skipped weekend
  candles count toward that. A weekend is 2,940 candles on 1m, 980 on 3m and 588
  on 5m. The gap-sweep run is stored, so that case reads. A normal run that
  carries on from Friday into Sunday's reopen is cut at the weekend on 1m–5m.

---

## Version history

**v1.02:**
- **Gap sweep.** When the reopen candle sweeps the DR extreme on a gap, the CISD
  comes from the run into the DR extreme, and the reopen candle can accept it
  with its own close.
- **Stop.** When the zone's first candle is the raid extreme, the stop goes
  beyond the next candle.
- **Status block.** Rebuilt as short rows: Status, Next, Swept, Clock, Trade.
- **Swept level.** Dotted line, and it says whether it armed.
- **Clock row.** Says when price beyond the DR was ignored while CME was closed.

**v1.01:** first public release, renamed **Dealing Range Liquidity Reversal**.
The engine is unchanged from v7.28. Settings: **Session** is now **DR window**,
the EW end reads HH : MM, **Session boxes** is now **DR / EW boxes**, **Sessions
drawn** is now **DRs drawn**. Settings need a reset.

### Pre-release, as Session-Based Liquidity Reversals

**v7.28:** stop and target act on the touch, not the bar close: trade off the
chart, status updated, **Stop hit** / **Target hit** alerts. EW close and kills
are unchanged.

**v7.27:** **TP at EQ** and **TP at the opposite DR extreme** checkboxes. With
both on, the bracket shows TP1 at EQ and TP2 at the DR's far end, each with its R,
and the trade runs to TP2. The run walk-back from v7.26 is rewritten; on
TradingView the old one never ran, so 1m/2m still missed the 16 Oct -CISD. The
status line shows what the CISD hunt is reading. Settings need a reset (new inputs).

**v7.26:** a run into the raid with no candle opened inside the DR keeps walking
back through candles that sit wholly outside the DR, to the one that opened
inside it. Lower timeframes no longer lose the CISD to tiny counter candles above
the DR high (below the DR low).

**v7.25:** the CISD is accepted by a body close above the sweep (or the whole
run) and takes the closest run candle below that close. Every PD array dies by itself on two consecutive body closes beyond
it. CISD, OB and PB are blocks across their candles' bodies, drawn as boxes. A
DR side waits for a return after two closes beyond it, and turns off after a
quarter-DR expansion.

**v7.24:** the CISD comes from the run into the sweep: the open of the run's
first candle, accepted on the first close back through it. Older legs can't
print as the CISD.

**v7.23:** the EW starts at the DR's end; you set only its end time. The Session
dropdown is gone. Alerts are tagged with the DR window.

**v7.22:** **Stacked FVGs** setting, with First gap only as the default.

**v7.21:** **Trade levels** is Engine or Manual; one trade on screen at a time.

**v7.20:** Dated DR in its own section with an EW end; range stats, the range
filter and -1 / +1 removed; the CME clock reaches swings, gaps and scans; entry
and target on one candle is no trade; runs on charts up to 4H.

**v7.10–v7.19:** two-path entry model, re-entry after a stop, swept level, RR
calculator, the 2R minimum, the stop beyond the zone's first candle, the CME
clock on perps.

**v7.0–v7.9 and earlier:** see the script's history.

---

## Disclaimer

**This is an educational and analytical tool. It is not financial advice, and
nothing it draws is a recommendation to buy or sell anything.**

- **No performance claim.** What it drew on past charts says nothing about future
  results.
- **The trade levels are illustrative.** They show what the rules produce. They
  don't account for slippage, commissions or fills, and a touch on the chart is
  not a fill in your account.
- **Futures and perpetuals carry a high risk of loss.** Leverage cuts both ways,
  and you can lose more than you put in. Trade only money you can afford to lose,
  and check your broker's and prop firm's rules yourself.
- **You are responsible for your trades.** Test on replay or a simulator first,
  and don't hand your decisions to an indicator.
- **Software can be wrong.** If a level looks wrong, assume it might be, and check
  it against the chart.
- **Terminology.** CISD, order block, propulsion block, FVG, IFVG and MSS come
  from ICT (Inner Circle Trader) teaching. This project isn't affiliated with or
  endorsed by ICT, TradingView, CME Group, Hyperliquid or trade.xyz. My readings
  of these concepts are my own.
