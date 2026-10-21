//@version=6
indicator("LeangHuot Indicaor", overlay=true, max_labels_count=500)

// ============================================================================
// 1. USER INPUTS
// ============================================================================
grp_ui = "Visual & UI Settings"
showLines      = input.bool(false, "Show Underlying EMAs, VWAP & POC Lines?", group=grp_ui, tooltip="Keep OFF for a clean chart that only shows BUY and SELL labels.")
showDashboard  = input.bool(true,  "Show Status Dashboard Table?", group=grp_ui)
showTradeInfo  = input.bool(true,  "Include SL & TP1 inside Buy/Sell Label?", group=grp_ui)

grp_htf = "Pillar 2A: Higher-Timeframe (HTF) Trend Filter"
useHtfFilter   = input.bool(true,  "Require 1H Trend Alignment?", group=grp_htf)
htfTf          = input.timeframe("60", "HTF Timeframe (Default: 1H)", group=grp_htf)
htfEmaLen      = input.int(50,     "HTF EMA Length", minval=1, group=grp_htf)
allowVwapCt    = input.bool(true,  "Allow Counter-Trend Signal Directly at Session VWAP?", group=grp_htf)

grp_zones = "Pillar 2B & 3: Institutional Zones (VWAP, EMAs, Volume POC)"
zoneTolDollars = input.float(2.50, "Zone Touch Tolerance ($ on Gold)", minval=0.1, step=0.25, group=grp_zones, tooltip="How close the candle wick must get to EMA100/200, Session VWAP, or POC to count as testing the zone.")
vpLookback     = input.int(288,    "24h Volume Profile Lookback (Bars, 288 = 24h on 5m)", minval=20, maxval=1000, group=grp_zones)
vpBins         = input.int(24,     "Volume Profile Price Bins", minval=10, maxval=100, group=grp_zones)

grp_of = "Pillar 1: Order Flow Trigger (BuAx / BeAx Logic)"
useExternalSrc = input.bool(false, "Link External Sn1P3r Signal Plot? (Optional)", group=grp_of, tooltip="Turn ON if you want to feed an external plot from another indicator on your chart.")
extBuAxSrc     = input.source(close, "External BuAx Source (triggers when > 0)", group=grp_of)
extBeAxSrc     = input.source(close, "External BeAx Source (triggers when > 0)", group=grp_of)
volMult        = input.float(1.15, "Internal Order Flow Volume Multiplier (vs 20 SMA)", minval=0.5, step=0.05, group=grp_of)
wickRatioMin   = input.float(0.25, "Minimum Rejection Wick Ratio (0.0 to 1.0)", minval=0.05, maxval=0.80, step=0.05, group=grp_of)

grp_sess = "Session Guard (Phnom Penh Time, UTC+7)"
skipDeadZone   = input.bool(true,  "Block Signals in Dead Zone (23:30 - 06:30 UTC+7)?", group=grp_sess)
skipLateAsian  = input.bool(false, "Block Signals in Late Asian Handover (11:30 - 14:00 UTC+7)?", group=grp_sess)

grp_risk = "Risk Management (TP1 -> Breakeven Rule)"
slBuffer       = input.float(1.20, "Stop Loss Buffer Beyond Wick ($)", minval=0.1, step=0.1, group=grp_risk)
tp1Distance    = input.float(4.50, "TP1 Target Distance ($) [Move SL to BE]", minval=0.5, step=0.5, group=grp_risk)
cooldownBars   = input.int(3,      "Minimum Cooldown Bars Between Signals", minval=1, group=grp_risk)

// ============================================================================
// 2. PILLAR 2: MOVING AVERAGES & VWAP CALCULATIONS
// ============================================================================
// Intraday EMAs & VWMA (5m / Chart Timeframe)
ema20  = ta.ema(close, 20)
ema50  = ta.ema(close, 50)
ema100 = ta.ema(close, 100)
ema200 = ta.ema(close, 200)
vwma20 = ta.vwma(close, 20)

// Session Anchored VWAP (Anchors at Daily 00:00 UTC Session Open)
isNewDay    = timeframe.change("1D")
sessionVwap = ta.vwap(hlc3, isNewDay)

// 24-Hour Rolling VWAP Approximation
rollingVwap = ta.vwma(hlc3, vpLookback)

// Higher-Timeframe (1H) EMA50 (Non-Repainting)
htfEma50 = request.security(syminfo.tickerid, htfTf, ta.ema(close, htfEmaLen)[1], lookahead=barmerge.lookahead_off)

// Macro Trend Bias (1H EMA50 + 24h Rolling VWAP)
isMacroBull = close > htfEma50 and close > rollingVwap
isMacroBear = close < htfEma50 and close < rollingVwap

// ============================================================================
// 3. PILLAR 3: 24-HOUR VOLUME PROFILE POC CALCULATION
// ============================================================================
f_calcVolumePoc(int lookback, int numBins) =>
    float highestHigh = ta.highest(high, lookback)
    float lowestLow   = ta.lowest(low, lookback)
    float rangeSize   = highestHigh - lowestLow
    float pocLevel    = hlc3
    if rangeSize > 0 and bar_index > lookback
        float binStep = rangeSize / numBins
        array<float> volBins = array.new_float(numBins, 0.0)
        for i = 0 to lookback - 1
            float bPrice = hlc3[i]
            float bVol   = nz(volume[i], 1.0)
            int binIdx   = math.min(numBins - 1, math.max(0, int(math.floor((bPrice - lowestLow) / binStep))))
            array.set(volBins, binIdx, array.get(volBins, binIdx) + bVol)
        float maxV = -1.0
        int maxIdx = 0
        for j = 0 to numBins - 1
            float v = array.get(volBins, j)
            if v > maxV
                maxV   := v
                maxIdx := j
        pocLevel := lowestLow + (maxIdx + 0.5) * binStep
    pocLevel

poc24h = f_calcVolumePoc(vpLookback, vpBins)

// ============================================================================
// 4. INSTITUTIONAL CONFLUENCE ZONE CHECKS
// ============================================================================
// Check if candle high/low swept into or tested a key institutional level
nearEma100   = (low - zoneTolDollars <= ema100)      and (high + zoneTolDollars >= ema100)
nearEma200   = (low - zoneTolDollars <= ema200)      and (high + zoneTolDollars >= ema200)
nearSessVwap = (low - zoneTolDollars <= sessionVwap) and (high + zoneTolDollars >= sessionVwap)
nearPoc24h   = (low - zoneTolDollars <= poc24h)      and (high + zoneTolDollars >= poc24h)

inInstitutionalZone = nearEma100 or nearEma200 or nearSessVwap or nearPoc24h

// Identify which zone was tapped for the label/alert tooltip
string activeZoneText = nearSessVwap ? "Session VWAP" : nearEma200 ? "EMA 200" : nearEma100 ? "EMA 100" : nearPoc24h ? "24h Vol POC" : "Zone"

// ============================================================================
// 5. PILLAR 1: ORDER FLOW ACCELERATOR TRIGGER (BuAx / BeAx)
// ============================================================================
volAvg      = ta.sma(volume, 20)
hasVolSpike = volume >= (volAvg * volMult)
candleRange = math.max(high - low, syminfo.mintick)
upperWick   = high - math.max(open, close)
lowerWick   = math.min(open, close) - low

// Internal Bullish Accelerator (BuAx): Sweeps support zone, strong lower rejection wick or bullish momentum close + expanding volume
internalBuAx = hasVolSpike and ((close > open and (lowerWick / candleRange >= wickRatioMin or close > high[1])) or (lowerWick / candleRange >= 0.45 and close >= low + candleRange * 0.6)) and (close >= ema20 or close >= sessionVwap or low <= ema100 + zoneTolDollars)

// Internal Bearish Accelerator (BeAx): Sweeps resistance zone, strong upper rejection wick or bearish momentum close + expanding volume
internalBeAx = hasVolSpike and ((close < open and (upperWick / candleRange >= wickRatioMin or close < low[1])) or (upperWick / candleRange >= 0.45 and close <= high - candleRange * 0.6)) and (close <= ema20 or close <= sessionVwap or high >= ema100 - zoneTolDollars)

buAxTrigger = useExternalSrc ? (extBuAxSrc > 0) : internalBuAx
beAxTrigger = useExternalSrc ? (extBeAxSrc > 0) : internalBeAx

// ============================================================================
// 6. SESSION FILTER (PHNOM PENH TIME, UTC+7)
// ============================================================================
inDeadZone     = not na(time(timeframe.period, "2330-0630", "Asia/Phnom_Penh"))
inLateAsian    = not na(time(timeframe.period, "1130-1400", "Asia/Phnom_Penh"))
inAsianMorning = not na(time(timeframe.period, "0700-1130", "Asia/Phnom_Penh"))
inLondonOpen   = not na(time(timeframe.period, "1400-1900", "Asia/Phnom_Penh"))
inNyOpen       = not na(time(timeframe.period, "1900-2300", "Asia/Phnom_Penh"))

sessionAllowed = (not skipDeadZone or not inDeadZone) and (not skipLateAsian or not inLateAsian)

// ============================================================================
// 7. FINAL COMBINED BUY / SELL SIGNAL ENGINE
// ============================================================================
// Trend alignment check (with optional exception if rejecting directly off Session Anchored VWAP)
validBullTrend = not useHtfFilter or isMacroBull or (allowVwapCt and nearSessVwap and close > sessionVwap)
validBearTrend = not useHtfFilter or isMacroBear or (allowVwapCt and nearSessVwap and close < sessionVwap)

rawBuy  = sessionAllowed and validBullTrend and inInstitutionalZone and buAxTrigger
rawSell = sessionAllowed and validBearTrend and inInstitutionalZone and beAxTrigger

// Cooldown lock so consecutive bars don't spam duplicate labels
var int lastSignalBar = -999
canFire = (bar_index - lastSignalBar) >= cooldownBars

buySignal  = rawBuy  and canFire
sellSignal = rawSell and not buySignal and canFire

if buySignal or sellSignal
    lastSignalBar := bar_index

// ============================================================================
// 8. CLEAN CHART VISUALS (ONLY BUY OR SELL LABELS)
// ============================================================================
// Calculate Risk Levels based on your back-tested rules
float buySL   = low  - slBuffer
float buyTP1  = close + tp1Distance
float sellSL  = high + slBuffer
float sellTP1 = close - tp1Distance

if buySignal
    string buyTxt = showTradeInfo ? "BUY\n@" + str.tostring(close, format.mintick) + "\nSL: " + str.tostring(buySL, format.mintick) + "\nTP1 (BE): " + str.tostring(buyTP1, format.mintick) : "BUY"
    label.new(bar_index, low, text=buyTxt, style=label.style_label_up, color=color.new(#00E676, 0), textcolor=color.black, size=size.normal, tooltip="Sn1P3r BUY Confluence\nZone: " + activeZoneText)

if sellSignal
    string sellTxt = showTradeInfo ? "SELL\n@" + str.tostring(close, format.mintick) + "\nSL: " + str.tostring(sellSL, format.mintick) + "\nTP1 (BE): " + str.tostring(sellTP1, format.mintick) : "SELL"
    label.new(bar_index, high, text=sellTxt, style=label.style_label_down, color=color.new(#FF1744, 0), textcolor=color.white, size=size.normal, tooltip="Sn1P3r SELL Confluence\nZone: " + activeZoneText)

// Optional Plots (Hidden by default so only BUY/SELL labels show)
plot(showLines ? sessionVwap : na, "Session VWAP (00:00 UTC)", color=color.new(color.yellow, 0), linewidth=2)
plot(showLines ? poc24h      : na, "24h Volume Profile POC",   color=color.new(color.orange, 0), linewidth=2, style=plot.style_stepline)
plot(showLines ? ema100      : na, "EMA 100",                  color=color.new(color.aqua, 20),  linewidth=1)
plot(showLines ? ema200      : na, "EMA 200",                  color=color.new(color.purple, 0), linewidth=2)
plot(showLines ? htfEma50    : na, "1H EMA 50",                color=color.new(color.white, 30), linewidth=2, style=plot.style_cross)

// ============================================================================
// 9. COMPACT STATUS DASHBOARD (TOP RIGHT)
// ============================================================================
var table dash = table.new(position.top_right, 2, 5, bgcolor=color.new(#131722, 15), border_color=color.new(color.gray, 60), border_width=1)

if barstate.islast and showDashboard
    string sessName = inDeadZone ? "DEAD ZONE (SKIP)" : inLateAsian ? "LATE ASIAN (CAUTION)" : inLondonOpen ? "LONDON (A+ EDGE)" : inAsianMorning ? "ASIAN MORNING" : inNyOpen ? "NY OPEN" : "ACTIVE"
    color  sessCol  = (inDeadZone or (skipLateAsian and inLateAsian)) ? color.new(#FF1744, 20) : inLondonOpen ? color.new(#00E676, 20) : color.new(color.blue, 30)
    string biasTxt  = isMacroBull ? "BULLISH (Look for BUY)" : isMacroBear ? "BEARISH (Look for SELL)" : "NEUTRAL / RANGE"
    color  biasCol  = isMacroBull ? color.new(#00E676, 20) : isMacroBear ? color.new(#FF1744, 20) : color.new(color.gray, 30)

    table.cell(dash, 0, 0, "Sn1P3r Engine v6", text_color=color.white, bgcolor=color.new(#2A2E39, 0), text_size=size.small)
    table.cell(dash, 1, 0, sessionAllowed ? "ARMED" : "MUTED (SKIP)", text_color=color.white, bgcolor=sessCol, text_size=size.small)
    
    table.cell(dash, 0, 1, "1H Macro Bias", text_color=color.silver, text_size=size.small)
    table.cell(dash, 1, 1, biasTxt, text_color=color.white, bgcolor=biasCol, text_size=size.small)
    
    table.cell(dash, 0, 2, "PP Session (UTC+7)", text_color=color.silver, text_size=size.small)
    table.cell(dash, 1, 2, sessName, text_color=color.white, bgcolor=sessCol, text_size=size.small)
    
    table.cell(dash, 0, 3, "Session VWAP", text_color=color.silver, text_size=size.small)
    table.cell(dash, 1, 3, str.tostring(sessionVwap, format.mintick), text_color=color.yellow, text_size=size.small)
    
    table.cell(dash, 0, 4, "24h Vol POC", text_color=color.silver, text_size=size.small)
    table.cell(dash, 1, 4, str.tostring(poc24h, format.mintick), text_color=color.orange, text_size=size.small)

// ============================================================================
// 10. UNIFIED ALERTS (FOR TELEGRAM BOT / WEBHOOKS)
// ============================================================================
alertcondition(buySignal,  title="Sn1P3r Combined BUY",  message="🟢 Sn1P3r A+ BUY on {{ticker}} ({{interval}}) at {{close}} | Secure 50% at TP1 & Move SL to BE!")
alertcondition(sellSignal, title="Sn1P3r Combined SELL", message="🔴 Sn1P3r A+ SELL on {{ticker}} ({{interval}}) at {{close}} | Secure 50% at TP1 & Move SL to BE!")

if buySignal
    alert("🟢 Sn1P3r A+ BUY [" + syminfo.ticker + " " + timeframe.period + "m] @ " + str.tostring(close, format.mintick) + " | Zone: " + activeZoneText + " | SL: " + str.tostring(buySL, format.mintick) + " | TP1 (Move to BE): " + str.tostring(buyTP1, format.mintick), alert.freq_once_per_bar_close)

if sellSignal
    alert("🔴 Sn1P3r A+ SELL [" + syminfo.ticker + " " + timeframe.period + "m] @ " + str.tostring(close, format.mintick) + " | Zone: " + activeZoneText + " | SL: " + str.tostring(sellSL, format.mintick) + " | TP1 (Move to BE): " + str.tostring(sellTP1, format.mintick), alert.freq_once_per_bar_close)