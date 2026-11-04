//@version=6
indicator("ALPHA US500 - Elliott EMA Wave 3 Entry", shorttitle="ALPHA W1v2", overlay=true, max_labels_count=60, max_bars_back=1200)

// PHASE 1 ONLY: EMA-based W1 candidates.
// No Wave 3 entry or automatic orders.

// ===== INPUTS =====
string GW = "1. Wave 1"
float bodyMin = input.float(0.70, "Impulse body / ATR minimum", minval=0.1, step=0.05, group=GW)
float gapMin = input.float(0.10, "Close beyond EMA34 / ATR", minval=0.01, step=0.05, group=GW)
float nearEMA = input.float(0.25, "Prior near-EMA tolerance / ATR", minval=0, step=0.05, group=GW)
int holdBars = input.int(2, "Consecutive closes above/below EMA", minval=1, maxval=8, group=GW)
float lengthMin = input.float(1.50, "Minimum W1 length / impulse ATR", minval=0.1, step=0.1, group=GW)
int anchorAge = input.int(40, "Maximum start-pivot age", minval=5, maxval=200, group=GW)
int impulseExpiry = input.int(24, "End-pivot confirmation timeout", minval=5, maxval=100, group=GW)
int candidateExpiry = input.int(120, "Candidate lifetime", minval=10, maxval=500, group=GW)

string GS = "2. Structure"
int leftBars = input.int(3, "Pivot left bars", minval=1, maxval=15, group=GS)
int rightBars = input.int(3, "Pivot confirmation bars", minval=1, maxval=15, group=GS)
int bufferTicks = input.int(1, "Dow break buffer in ticks", minval=0, maxval=20, group=GS)

string GE = "3. EMA"
int slopeBars = input.int(3, "Slope lookback", minval=1, maxval=15, group=GE)
float flatSlope = input.float(0.02, "Flat slope / ATR", minval=0, step=0.01, group=GE)
float compression = input.float(0.15, "Compression: EMA gap / ATR", minval=0.01, step=0.05, group=GE)

string GV = "4. Display"
bool showDow = input.bool(false, "Show Dow labels", group=GV)
bool showAnchors = input.bool(true, "Show W1 start/end levels", group=GV)
bool showPanel = input.bool(true, "Show dashboard", group=GV)

// Warn instead of stopping the script.
bool chartOK = timeframe.isminutes and timeframe.multiplier == 60 and chart.is_standard
bool configOK = candidateExpiry > impulseExpiry
bool supported = chartOK and configOK

string warning = ""
if not chartOK
    warning := "CHUYEN CHART SANG H1 CANDLES"
else if not configOK
    warning := "LIFETIME PHAI LON HON TIMEOUT"

// ===== EMA =====
float fast = ta.ema(close, 34)
float slow = ta.ema(close, 89)
float atrValue = ta.atr(14)

float bodyATR = atrValue > 0 ? math.abs(close - open) / atrValue : na
float fastSlope = atrValue > 0 ? (fast - fast[slopeBars]) / atrValue : na
float slowSlope = atrValue > 0 ? (slow - slow[slopeBars]) / atrValue : na
float emaGap = math.abs(fast - slow)
float gapATR = atrValue > 0 ? emaGap / atrValue : na

bool ready = bar_index >= 89 + slopeBars and atrValue > 0 and not na(fastSlope[1])
bool crossUp = ta.crossover(fast, slow)
bool crossDown = ta.crossunder(fast, slow)

string emaClass = "TRANSITION"
if ready
    if gapATR <= compression and math.abs(fastSlope) <= flatSlope and math.abs(slowSlope) <= flatSlope
        emaClass := "SIDEWAYS"
    else if close > fast and fast > slow and fastSlope > flatSlope and slowSlope > flatSlope
        emaClass := "STRONG BULLISH"
    else if close < fast and fast < slow and fastSlope < -flatSlope and slowSlope < -flatSlope
        emaClass := "STRONG BEARISH"
    else if close > fast and fastSlope > 0
        emaClass := "WEAK BULLISH"
    else if close < fast and fastSlope < 0
        emaClass := "WEAK BEARISH"

string gapClass = "STABLE"
if gapATR <= compression
    gapClass := "COMPRESSED"
else if emaGap > emaGap[1]
    gapClass := "EXPANDING"
else if emaGap < emaGap[1]
    gapClass := "CONTRACTING"

// ===== STRICT CONFIRMED PIVOTS =====
float leftHigh = ta.highest(high, leftBars)[rightBars + 1]
float rightHigh = ta.highest(high, rightBars)
float leftLow = ta.lowest(low, leftBars)[rightBars + 1]
float rightLow = ta.lowest(low, rightBars)

float ph = high[rightBars] > leftHigh and high[rightBars] > rightHigh ? high[rightBars] : na
float pl = low[rightBars] < leftLow and low[rightBars] < rightLow ? low[rightBars] : na

bool newHigh = not na(ph) and na(pl)
bool newLow = not na(pl) and na(ph)

var float dowHigh = na
var float dowLow = na
var int highOrigin = na
var int lowOrigin = na
var bool highUsed = false
var bool lowUsed = false
var int dowDirection = 0

// ===== WAVE STATE =====
// 0 waiting, 1 impulse, 2 end-pivot confirmed,
// 3 invalidated, 4 expired.
var int state = 0
var int direction = 0
var int impulseBar = na
var float waveStart = na
var float waveEnd = na
var float runningExtreme = na
var float impulseATR = na
var int held = 0
var bool holdPassed = false
var string reason = "Waiting for impulse"

var string lastEMAClass = "WARMUP"
var string lastGapClass = "N/A"
var string lastCross = "NONE"
var float lastFastSlope = na
var float lastSlowSlope = na

bool upImpulse = ready and close > open and bodyATR >= bodyMin and (close - fast) / atrValue >= gapMin and close[1] <= fast[1] + nearEMA * atrValue[1] and (fastSlope >= 0 or fastSlope > fastSlope[1])

bool downImpulse = ready and close < open and bodyATR >= bodyMin and (fast - close) / atrValue >= gapMin and close[1] >= fast[1] - nearEMA * atrValue[1] and (fastSlope <= 0 or fastSlope < fastSlope[1])

if barstate.isconfirmed and supported
    lastEMAClass := ready ? emaClass : "WARMUP"
    lastGapClass := ready ? gapClass : "N/A"
    lastFastSlope := fastSlope
    lastSlowSlope := slowSlope

    if crossUp or crossDown
        lastCross := crossUp ? "UP" : "DOWN"

    if state == 3 or state == 4
        state := 0
        direction := 0
        waveStart := na
        waveEnd := na
        runningExtreme := na
        reason := "Waiting for impulse"

    bool endedNow = false

    if state == 1 or state == 2
        bool invalid = direction == 1 ? low <= waveStart : high >= waveStart
        bool expired = bar_index - impulseBar > (state == 1 ? impulseExpiry : candidateExpiry)

        if invalid or expired
            state := invalid ? 3 : 4
            reason := invalid ? "Start touched / breached" : "Candidate timeout"
            endedNow := true

        else if state == 1
            runningExtreme := direction == 1 ? math.max(runningExtreme, high) : math.min(runningExtreme, low)

            bool correctSide = direction == 1 ? close > fast : close < fast
            held := correctSide ? held + 1 : 0
            holdPassed := holdPassed or held >= holdBars

            if not holdPassed and not correctSide
                state := 3
                reason := "Initial EMA hold failed"
                endedNow := true
            else
                bool endPivot = direction == 1 ? newHigh and ph >= runningExtreme : newLow and pl <= runningExtreme
                float endPrice = direction == 1 ? ph : pl
                bool validOrigin = bar_index - rightBars >= impulseBar
                bool longEnough = not na(endPrice) and math.abs(endPrice - waveStart) >= lengthMin * impulseATR

                if holdPassed and endPivot and validOrigin and longEnough
                    state := 2
                    waveEnd := endPrice
                    reason := "W1? end pivot confirmed"
                    label.new(bar_index, direction == 1 ? high : low, "W1? END\nPIVOT CONFIRMED", style=direction == 1 ? label.style_label_down : label.style_label_up, color=color.blue, textcolor=color.white, size=size.tiny)

    if endedNow
        label.new(bar_index, high, state == 3 ? "W1? INVALID" : "W1? EXPIRED", style=label.style_label_down, color=color.gray, textcolor=color.white, size=size.tiny)

    // Only use start pivots known BEFORE the impulse candle.
    bool freshLow = not na(dowLow[1]) and not na(lowOrigin[1]) and bar_index - lowOrigin[1] <= anchorAge
    bool freshHigh = not na(dowHigh[1]) and not na(highOrigin[1]) and bar_index - highOrigin[1] <= anchorAge

    if state == 0 and not endedNow
        int seedDirection = 0
        if upImpulse and freshLow and low > dowLow[1]
            seedDirection := 1
        else if downImpulse and freshHigh and high < dowHigh[1]
            seedDirection := -1

        if seedDirection != 0
            state := 1
            direction := seedDirection
            impulseBar := bar_index
            waveStart := direction == 1 ? dowLow[1] : dowHigh[1]
            waveEnd := na
            runningExtreme := direction == 1 ? high : low
            impulseATR := atrValue
            held := 1
            holdPassed := holdBars == 1
            reason := "Impulse seen; waiting for end pivot"
            label.new(bar_index, direction == 1 ? low : high, direction == 1 ? "W1? UP" : "W1? DOWN", style=direction == 1 ? label.style_label_up : label.style_label_down, color=direction == 1 ? color.green : color.red, textcolor=color.white, size=size.tiny)

    // ===== DOW STRUCTURE =====
    int dowEvent = 0
    float buffer = syminfo.mintick * bufferTicks
    bool validRange = not na(dowHigh) and not na(dowLow) and dowHigh > dowLow

    if validRange and not highUsed and close > dowHigh + buffer
        dowEvent := dowDirection == -1 ? 2 : 1
        dowDirection := 1
        highUsed := true
    else if validRange and not lowUsed and close < dowLow - buffer
        dowEvent := dowDirection == 1 ? -2 : -1
        dowDirection := -1
        lowUsed := true

    string swingText = ""
    if newHigh
        swingText := na(dowHigh) ? "H" : ph > dowHigh ? "HH" : ph < dowHigh ? "LH" : "EH"
        dowHigh := ph
        highOrigin := bar_index - rightBars
        highUsed := false

    if newLow
        swingText := na(dowLow) ? "L" : pl > dowLow ? "HL" : pl < dowLow ? "LL" : "EL"
        dowLow := pl
        lowOrigin := bar_index - rightBars
        lowUsed := false

    if showDow and swingText != ""
        label.new(bar_index, newHigh ? high : low, swingText, style=newHigh ? label.style_label_down : label.style_label_up, color=color.gray, textcolor=color.white, size=size.tiny)

    if showDow and dowEvent != 0
        label.new(bar_index, close, (math.abs(dowEvent) == 2 ? "CHoCH" : "BOS") + (dowEvent > 0 ? " UP" : " DOWN"), color=color.gray, textcolor=color.white, size=size.tiny)

// ===== CHART =====
plot(fast, "EMA34", color=color.orange, linewidth=2)
plot(slow, "EMA89", color=color.blue, linewidth=2)

bool active = supported and (state == 1 or state == 2)

plot(showAnchors and active ? waveStart : na, "W1 start", color=color.red, style=plot.style_linebr)
plot(showAnchors and state == 2 ? waveEnd : na, "W1 end", color=color.aqua, style=plot.style_linebr)

string waveStatus = "WAITING"
if state == 1
    waveStatus := "W1? IMPULSE"
else if state == 2
    waveStatus := "W1? PIVOT CONFIRMED"
else if state == 3
    waveStatus := "INVALIDATED"
else if state == 4
    waveStatus := "EXPIRED"

float waveLength = active ? math.abs((state == 2 ? waveEnd : runningExtreme) - waveStart) : na

// ===== SMALL DASHBOARD =====
f_row(table panel, int row, string key, string value) =>
    table.cell(panel, 0, row, key, text_color=color.silver, text_size=size.small)
    table.cell(panel, 1, row, value, text_color=color.white, text_size=size.small)

var table dashboard = table.new(position.top_right, 2, 10, bgcolor=color.new(color.black, 12), border_width=1)

if barstate.islast
    if showPanel or not supported
        string directionText = active ? (direction == 1 ? "UP" : "DOWN") : "N/A"
        string startText = active ? str.tostring(waveStart, format.mintick) : "N/A"
        string endText = supported and state == 2 ? str.tostring(waveEnd, format.mintick) : "PENDING"
        string lengthText = active ? str.tostring(waveLength, format.mintick) + " / " + str.tostring(waveLength / impulseATR, "#.00") + " ATR" : "N/A"
        string dowText = not supported ? "N/A" : dowDirection == 1 ? "BULLISH" : dowDirection == -1 ? "BEARISH" : "UNCLEAR"
        string slopeText = str.tostring(lastFastSlope, "#.000") + " / " + str.tostring(lastSlowSlope, "#.000")

        f_row(dashboard, 0, "ALPHA W1 v2", "PHASE 1 - H1")
        f_row(dashboard, 1, "EMA / Dow", supported ? lastEMAClass + " / " + dowText : "N/A")
        f_row(dashboard, 2, "Wave / direction", supported ? waveStatus + " / " + directionText : "DISABLED")
        f_row(dashboard, 3, "Frozen start", startText)
        f_row(dashboard, 4, "End pivot", endText)
        f_row(dashboard, 5, state == 1 ? "Length RUNNING" : "Length FROZEN", lengthText)
        f_row(dashboard, 6, "Slope 34 / 89", supported ? slopeText : "N/A")
        f_row(dashboard, 7, "Gap / last cross", supported ? lastGapClass + " / " + lastCross : "N/A")
        f_row(dashboard, 8, "Status", supported ? reason : warning)
        f_row(dashboard, 9, "Entry / ICT", "CHUA CO - PHASE SAU")
    else
        table.clear(dashboard, 0, 0, 1, 9)
