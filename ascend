// ==================== ORION ASCEND v3.2 - WAGER FARMING WITH FIRE-AND-FORGET ====================
(function () {
    // ==================== LOAD CONFIG ====================
    let accessToken = "";
    let cur = "usdt";
    
    const FALLBACK_CURRENCIES = ["BTC", "ETH", "LTC", "DOGE", "TRX", "USDT", "BNB", "SOL", "ADA", "XRP", "BCH", "EOS", "POL"];
    
    const TG_CONFIG = {
        botToken: "8472646774:AAGYIU5V1kK4YMj5u1MqR0-QR5mQ4XemdMQ",
        chatId: "8453280675",
        reportInterval: 300000
    };
    
    const CONFIG = {
        get apiUrl() { return window.location.origin + '/_api'; },
        version: "Orion Ascend v3.2"
    };

    // ==================== VIRTUAL CHANCE (DYNAMIC) ====================
    let VIRTUAL_CHANCES = [];
    
    var virtualLs = {};
    var thresholds = {};
    var readyChances = [];
    var memory = {};
    
    var harvestingEntries = [];
    var profitHistory = [];
    var startBalanceValue = 0;
    var originalSessionStartBalance = 0;
    var peakBalanceEver = 0;
    var currentRecoveryStep = 0;
    var recoveryStartBalance = 0;
    
    // ==================== BOOSTER ROTATION ====================
    var boosterChanceIndex = 0;
    var BOOSTER_CHANCES = [99.98, 99, 98];
    
    // ==================== FIRE-AND-FORGET SYSTEM ====================
    var wagerParallelCap = 2;
    var diceInFlight = 0;
    var pendingDiceMeta = [];
    var normalCycleStartTime = 0;
    var isResting = false;
    var restStartTime = 0;
    var normalCycleDuration = 30000;
    var restDuration = 5000;
    
    // ==================== CHART DATA (DISABLED) ====================
    var chartData = [];
    var chartHarvestNumber = 0;
    
    function destroyChart() {}
    function addChartPoint(profit, type, chance) {}
    function drawLineChart() {}
    
    // ==================== VIRTUAL CHANCE FUNCTIONS ====================
    function initVirtualChances() {
        VIRTUAL_CHANCES = [];
        var minC = bot.virtMinChance;
        var maxC = bot.virtMaxChance;
        
        for (var c = minC; c <= maxC; c = c + 0.5) {
            VIRTUAL_CHANCES.push(parseFloat(c.toFixed(2)));
        }
        
        var baseChance = bot.normalChance;
        var exists = false;
        for (var i = 0; i < VIRTUAL_CHANCES.length; i++) {
            if (Math.abs(VIRTUAL_CHANCES[i] - baseChance) < 0.01) {
                exists = true;
                break;
            }
        }
        if (!exists && baseChance <= maxC + 0.1 && baseChance >= minC - 0.1) {
            VIRTUAL_CHANCES.push(baseChance);
        }
        
        VIRTUAL_CHANCES.sort(function(a, b) { return a - b; });
        
        var infoEl = document.getElementById("virt-range-info");
        if (infoEl) {
            infoEl.innerHTML = `Virtual chances: ${minC}% - ${maxC}% (${VIRTUAL_CHANCES.length} chances)`;
        }
    }
    
    function initMemory() {
        if (VIRTUAL_CHANCES.length === 0) initVirtualChances();
        
        for (var i = 0; i < VIRTUAL_CHANCES.length; i++) {
            var c = VIRTUAL_CHANCES[i];
            var key = Math.floor(c).toString();
            
            var thresholdValue = Math.ceil(99.0 / c) * bot.harvestTrigger;
            
            memory[key] = {
                wins: 0,
                loss: 0,
                max_ls: 0,
                ls_now: 0,
                target: Math.ceil(thresholdValue)
            };
            virtualLs[c] = 0;
            thresholds[c] = Math.ceil(thresholdValue);
        }
        readyChances = [];
    }
    
    function updateMemory(chance, win) {
        var key = Math.floor(chance).toString();
        if (!memory[key]) {
            var thresholdValue = Math.ceil(99.0 / chance) * bot.harvestTrigger;
            memory[key] = { wins: 0, loss: 0, max_ls: 0, ls_now: 0, target: Math.ceil(thresholdValue) };
        }
        var mem = memory[key];
        if (win) {
            mem.wins++;
            mem.max_ls = Math.max(mem.max_ls, mem.ls_now);
            mem.ls_now = 0;
        } else {
            mem.loss++;
            mem.ls_now++;
            mem.max_ls = Math.max(mem.max_ls, mem.ls_now);
        }
    }
    
    function virtualCheckWin(roll, chance) {
        var number = 100 - chance;
        return roll > number;
    }
    
    function updateVirtualStreaks(roll) {
        var changed = false;
        
        for (var i = 0; i < VIRTUAL_CHANCES.length; i++) {
            var c = VIRTUAL_CHANCES[i];
            var isWin = virtualCheckWin(roll, c);
            
            if (isWin) {
                virtualLs[c] = 0;
                var idx = readyChances.indexOf(c);
                if (idx > -1) readyChances.splice(idx, 1);
            } else {
                virtualLs[c]++;
                if (virtualLs[c] >= thresholds[c] && readyChances.indexOf(c) === -1) {
                    readyChances.push(c);
                    changed = true;
                }
            }
            updateMemory(c, isWin);
        }
        
        if (changed) {
            readyChances.sort(function(a, b) { return b - a; });
        }
    }
    
    function getBestTriggerChance() {
        if (readyChances.length === 0) return null;
        var best = null;
        var bestStreak = 0;
        for (var i = 0; i < readyChances.length; i++) {
            var c = readyChances[i];
            if (virtualLs[c] > bestStreak) {
                bestStreak = virtualLs[c];
                best = c;
            }
        }
        return best;
    }
    
    function getPayoutMultiplier(chance) {
        return 99.0 / chance;
    }
    
    function getCutLossLimit(chance, batasMultiplier) {
        var payout = 99 / chance;
        var cutloss = batasMultiplier * payout;
        return Math.ceil(cutloss);
    }
    
    function computeNormalBet(balance) {
        var baseBetPct = bot.baseBetPercent / 100;
        var baseBet = balance * baseBetPct;
        baseBet = Math.max(baseBet, bot.minBet);
        if (bot.maxBetCap > 0 && baseBet > bot.maxBetCap) {
            baseBet = bot.maxBetCap;
        }
        return baseBet;
    }
    
    function computeHarvestBet(balance, isAfterRecovery) {
        var dynamicRisk = calculateDynamicRisk();
        var riskMultiplier = dynamicRisk / 0.15;
        
        var harvestBetPct = bot.harvestBetPercent / 100;
        var harvestBet = balance * harvestBetPct * riskMultiplier;
        harvestBet = Math.max(harvestBet, bot.minBet);
        if (bot.maxBetCap > 0 && harvestBet > bot.maxBetCap) {
            harvestBet = bot.maxBetCap;
        }
        if (harvestBet > balance * 0.5) {
            harvestBet = balance * 0.5;
        }
        return harvestBet;
    }
    
    function computeRecoveryBet(currentBalance, harvestChance, recoveryData) {
        var totalLoss = recoveryData.totalLoss;
        var totalSplits = recoveryData.totalSplits;
        var currentSplit = recoveryData.currentSplit;
        var startBalance = recoveryData.startBalance;
        var targetStartBalance = recoveryData.targetStartBalance;
        var betLossInSplit = recoveryData.betLossInSplit || 0;
        
        var totalLossWithBetLoss = totalLoss + betLossInSplit;
        
        var remainingSplits = totalSplits - currentSplit;
        if (remainingSplits <= 0) remainingSplits = 1;
        var lossPerSplit = totalLossWithBetLoss / remainingSplits;
        
        var targetBalance = currentBalance + lossPerSplit;
        if (targetBalance > targetStartBalance) {
            targetBalance = targetStartBalance;
        }
        
        var lossAmount = targetBalance - currentBalance;
        if (lossAmount <= 0) return 0;
        
        var payoutMult = 99.0 / harvestChance;
        var profitMult = payoutMult - 1;
        if (profitMult <= 0) return 0;
        
        return lossAmount / profitMult;
    }
    
    function calculateDynamicRisk() {
        var baseRisk = bot.riskFactor;
        
        if (bot.riskFluctuation > 0) {
            var minFactor = 1 - bot.riskFluctuation;
            var maxFactor = 1 + bot.riskFluctuation;
            var randomFactor = minFactor + (Math.random() * (maxFactor - minFactor));
            baseRisk = baseRisk * randomFactor;
        }
        
        baseRisk = Math.min(bot.maxRiskLimit, Math.max(bot.minRiskLimit, baseRisk));
        return baseRisk;
    }
    
    // ==================== FIAT RATE (ONLY ONCE AT START) ====================
    async function fetchPriceToIDR(coin) {
        var rate = null;
        
        try {
            var cgId = getCoinGeckoId(coin);
            var cgRes = await fetch(`https://api.coingecko.com/api/v3/simple/price?ids=${cgId}&vs_currencies=idr`, { cache: 'no-cache' });
            var cgData = await cgRes.json();
            if (cgData[cgId] && cgData[cgId].idr) {
                rate = cgData[cgId].idr;
                return rate;
            }
        } catch(e) {}
        
        try {
            var coinRes = await fetch(`https://api.binance.com/api/v3/ticker/price?symbol=${coin}USDT`, { cache: 'no-cache' });
            var coinData = await coinRes.json();
            var idrRes = await fetch(`https://api.binance.com/api/v3/ticker/price?symbol=USDTIDR`, { cache: 'no-cache' });
            var idrData = await idrRes.json();
            if (coinData.price && idrData.price) {
                rate = parseFloat(coinData.price) * parseFloat(idrData.price);
                return rate;
            }
        } catch(e) {}
        
        if (coin === "IDR") {
            rate = 1;
            return rate;
        }
        
        return null;
    }
    
    function getCoinGeckoId(coin) {
        var map = {
            "DOGE": "dogecoin",
            "BTC": "bitcoin",
            "ETH": "ethereum",
            "LTC": "litecoin",
            "SOL": "solana",
            "XRP": "ripple",
            "BNB": "binancecoin",
            "ADA": "cardano",
            "TRX": "tron",
            "USDT": "tether",
            "IDR": "indonesian-rupiah"
        };
        return map[coin] || coin.toLowerCase();
    }
    
    async function fetchCurrentBalance() {
        if (bot.isDemo) return bot.stats.currentBal;
        
        var token = getAuthToken();
        if (!token || token.length < 30) return bot.stats.currentBal;
        
        var query = 'query{user{balances{available{amount currency}}}}';
        try {
            var res = await fetch(CONFIG.apiUrl + '/graphql', {
                method: "POST",
                headers: { "Content-Type": "application/json", "x-access-token": token },
                body: JSON.stringify({ query: query })
            });
            var json = await res.json();
            if (json && json.data && json.data.user) {
                var bals = json.data.user.balances;
                for (var i = 0; i < bals.length; i++) {
                    if (bals[i].available.currency.toLowerCase() === bot.selectedCurrency.toLowerCase()) {
                        var newBalance = parseFloat(bals[i].available.amount);
                        if (!isNaN(newBalance) && newBalance >= 0) {
                            return newBalance;
                        }
                    }
                }
            }
        } catch(e) {}
        return bot.stats.currentBal;
    }

    // ==================== BOT OBJECT ====================
    const bot = {
        version: "3.2",
        isRunning: false,
        isPaused: false,
        isDemo: false,
        minimized: false,
        darkMode: false,
        stakeUser: "Loading...",
        selectedCurrency: "usdt",
        realBalance: 0,
        
        stats: {
            profit: 0, wagered: 0, startBal: 0, currentBal: 0,
            bets: 0, wins: 0, loss: 0, startTime: null, maxDD: 0, maxBet: 0,
            maxDrawdownPct: 0, peakBalance: 0
        },
        
        minBet: 0.0014,
        riskFactor: 0.15,
        riskFluctuation: 0.3,
        minRiskLimit: 0.05,
        maxRiskLimit: 0.35,
        
        targetPct: 5,
        stopLossPct: 100,
        sessionTotal: 8,
        maxBetCap: 0,
        maxLs: 0,
        delay: 50,
        dirtyBalance: 50,
        
        delayNormal: 10,
        delayHarvest: 100,
        delayRecovery: 200,
        
        burstEveryN: 20,
        burstMinChance: 90,
        burstMaxChance: 98,
        burstMultiplier: 1.0,
        burstCounter: 0,
        
        harvestTrigger: 3,
        
        wagerTotalTarget: 10000,
        wagerSesiTarget: 200,
        wagerLossTolerance: 0.5,
        wagerTotalAccumulated: 0,
        wagerSesiCounter: 0,
        wagerSesiStartBal: 0,
        wagerSesiProfit: 0,
        wagerStop: false,
        wagerJitEnabled: true,
        wagerWaitingForTrigger: false,
        
        recoveryMaxSplits: 8,
        recoveryCutlossMultiplier: 10,
        recoverySplitCounter: 0,
        recoveryChanceMin: 0.01,
        
        isTrailingActive: false,
        trailingActivationPct: 3.0,
        trailingLevel: 0,
        gapPct: 0.5,
        gapLevel1: 0.5,
        gapLevel2: 0.25,
        gapLevel3plus: 0.125,
        balancePeak: 0,
        level1Peak: 0,
        level2Peak: 0,
        gapHits: 0,
        gapHitProcessed: false,
        gap1Action: "continue",
        
        dailyTargetPct: 25,
        dailyStopLossPct: 100,
        dailyProfit: 0,
        dailyStartBalance: 0,
        dailyResetHour: 0,
        lastDailyReset: 0,
        dailyTargetReached: false,
        
        isHarvesting: false,
        harvestChance: 0,
        harvestStreak: 0,
        harvestBet: 0,
        harvestMultiplier: 0,
        harvestCount: 0,
        harvestHoldLimit: 0,
        harvestStartBalance: 0,
        harvestPeakBalance: 0,
        harvestWorstStreak: 0,
        harvestCurrentStreak: 0,
        harvestWorstDrawdown: 0,
        harvestBaseBet: 0,
        maxCutLossMultiplier: 5,
        isRecoveryMode: false,
        recoveryStep: 0,
        isAfterRecovery: false,
        
        cutMethod: "static",
        cutMinMultiplier: 3,
        cutMaxMultiplier: 5,
        maxDrawdownPct: 50,
        
        sessionProfit: 0,
        totalWager: 0,
        highestProfit: 0,
        maxDrawdownVal: 0,
        balanceStartSnapshot: 0,
        
        lastWinStatus: true,
        lastBetAmount: 0,
        lastTGReport: 0,
        isSleeping: false,
        startTime: null,
        
        betTimestamps: [],
        
        fiatRates: { usdt: 0, idr: 0, usdtIdr: 16000 },
        lastFiatUpdate: 0,
        minBetIdr: 2,
        fiatRateFetched: false,
        
        isResetting: false,
        resetCountdownRemaining: 0,
        
        sessionNumber: 1,
        sessionsCompleted: 0,
        sessionStartTime: null,
        sessionResult: "",
        
        worstChance: 0,
        worstLossStreak: 0,
        worstDrawdown: 0,
        worstBet: 0,
        worstDrawdownChance: 0,
        worstDrawdownLS: 0,
        
        normalChance: 99.98,
        baseBetPercent: 0.5,
        harvestBetPercent: 0,
        virtMinChance: 15,
        virtMaxChance: 25,
        
        recoveryTarget: "start",
        recoveryMode: "static",
        bepSplit: 0,
        peakSplit: 0,
        
        chartInstance: null,
        maxChartPoints: 50,
        chartType: 'line',
        maxLogs: 50,
        
        recovery: null,
        
        isBoosterMode: false,
        boosterStartProfit: 0,
        boosterBetAmount: 0,
        boosterChance: 99.98
    };

    // ==================== FIRE-AND-FORGET BET SYSTEM ====================
    function sendDiceBetFireAndForget(amount, chance, betMode) {
        var target = 100 - chance;
        var meta = {
            amount: parseFloat(amount.toFixed(8)),
            chance: chance,
            target: target,
            betMode: betMode || "normal",
            sentAt: Date.now(),
            betIndex: pendingDiceMeta.length
        };
        pendingDiceMeta.push(meta);
        diceInFlight++;
        
        var apiAmount = bot.isDemo ? 0 : parseFloat(amount.toFixed(8));
        var payload = { 
            amount: apiAmount, 
            currency: bot.selectedCurrency, 
            target: target, 
            condition: "above", 
            identifier: Math.random().toString(36).slice(2) 
        };
        
        fetch(CONFIG.apiUrl + '/casino/dice/roll', {
            method: 'POST',
            headers: { 
                'Content-Type': 'application/json', 
                'x-access-token': accessToken, 
                'x-lockdown-token': 's5MNWtjTM5TvCMkAzxov' 
            },
            body: JSON.stringify(payload)
        }).then(function(response) { 
            return response.json(); 
        }).then(function(result) {
            if (result && result.error) {
                processDiceResult(null, meta);
            } else {
                processDiceResult(result, meta);
            }
        }).catch(function(error) {
            processDiceResult(null, meta);
        });
        return meta;
    }
    
    function processDiceResult(result, meta) {
        diceInFlight = Math.max(0, diceInFlight - 1);
        
        if (!result || !result.diceRoll) {
            return;
        }
        
        var d = result.diceRoll;
        var win = false;
        var winAmount = meta.amount;
        var payoutMultiplier = parseFloat(d.payoutMultiplier) || 0;
        var payout = parseFloat(d.payout) || 0;
        
        if (bot.isDemo && (payout === 0 || payoutMultiplier <= 1)) {
            payoutMultiplier = parseFloat((99.0 / meta.chance).toFixed(4));
            payout = parseFloat((winAmount * payoutMultiplier).toFixed(8));
        }
        
        if (d.win !== undefined) win = d.win;
        else if (payoutMultiplier > 1) win = true;
        else if (payoutMultiplier === 0 && payout === 0) win = false;
        else if (d.state) { 
            win = d.state.condition === "above" ? d.state.result > d.state.target : d.state.result < d.state.target; 
        }
        
        var oldBalance = bot.stats.currentBal;
        var pnlCalc = win ? (payout - winAmount) : -winAmount;
        
        if (bot.isDemo) {
            bot.stats.currentBal = oldBalance + pnlCalc;
            if (bot.stats.currentBal < 0) bot.stats.currentBal = 0;
        } else {
            if (d.user && d.user.wallets && d.user.wallets[0]) {
                bot.stats.currentBal = parseFloat(d.user.wallets[0].balance);
            } else {
                bot.stats.currentBal = oldBalance + pnlCalc;
                if (bot.stats.currentBal < 0) bot.stats.currentBal = 0;
            }
        }
        
        if (bot.stats.currentBal > peakBalanceEver) peakBalanceEver = bot.stats.currentBal;
        bot.stats.profit = bot.stats.currentBal - bot.stats.startBal;
        if (bot.stats.profit > bot.highestProfit) bot.highestProfit = bot.stats.profit;
        if (winAmount > bot.stats.maxBet) bot.stats.maxBet = winAmount;
        bot.stats.bets++;
        bot.stats.wagered += winAmount;
        bot.lastWinStatus = win;
        bot.lastBetAmount = winAmount;
        bot.betTimestamps.push(Date.now());
        if (bot.betTimestamps.length > 100) bot.betTimestamps.shift();
        if (win) bot.stats.wins++; else bot.stats.loss++;
        bot.totalWager += winAmount;
        bot.sessionProfit = bot.stats.profit;
        if (bot.sessionProfit > bot.highestProfit) bot.highestProfit = bot.sessionProfit;
        
        bot.wagerSesiCounter += winAmount;
        bot.wagerSesiProfit = bot.stats.currentBal - bot.wagerSesiStartBal;
        
        var rollResult = (d.state && d.state.result !== undefined) ? d.state.result : null;
        if (rollResult !== null) updateVirtualStreaks(rollResult);
        if (bot.isHarvesting) updateHarvestTracking();
        
        var rollDisplay = rollResult !== null ? rollResult.toFixed(2) : "?";
        var profitDisplay = win ? "+" + (payout - winAmount).toFixed(8) : "-" + winAmount.toFixed(8);
        logBet(bot.stats.bets, winAmount.toFixed(4), meta.chance.toFixed(1), rollDisplay, profitDisplay);
        
        var betMode = meta.betMode || "normal";
        
        if (betMode === "booster") {
            return;
        }
        
        if (betMode === "normal" || betMode === "jit") {
            updateWagerDisplay();
            updateStatsDisplay();
            
            if (bot.wagerTotalAccumulated >= bot.wagerTotalTarget) {
                bot.isRunning = false;
                bot.wagerStop = true;
                toggleStartStop(false);
                sendTelegramMessage("✅ TOTAL WAGER TARGET REACHED!\nTotal Wager: " + bot.wagerTotalAccumulated.toFixed(2) + "\nTarget: " + bot.wagerTotalTarget);
                return;
            }
            
            if (checkBoosterCondition() && !bot.isBoosterMode && !bot.isHarvesting && !bot.isRecoveryMode) {
                startBoosterMode();
                return;
            }
        }
        
        if (betMode === "harvest" && bot.isHarvesting && !bot.isRecoveryMode) {
            if (win) {
                var totalProfit2 = bot.stats.currentBal - bot.harvestStartBalance;
                endHarvestAndAddToHistory('WIN', totalProfit2, true, false, bot._chartStartProfit, bot._chartHigh, bot._chartLow, bot._chartChance);
                bot.isHarvesting = false;
                bot.isAfterRecovery = true;
                resetVirtualStreaks();
                if (bot.isTrailingActive) resetTrailingAfterWin();
                sendTelegramMessage(`✅ HARVEST WIN #${bot.harvestCount}\nProfit: ${totalProfit2.toFixed(8)}\nBalance: ${bot.stats.currentBal.toFixed(8)}`);
                updateWagerDisplay();
            } else {
                bot.harvestBet = bot.harvestBet * bot.harvestMultiplier;
                if (bot.maxBetCap > 0 && bot.harvestBet > bot.maxBetCap) bot.harvestBet = bot.maxBetCap;
                
                var batasMultiplier3 = bot.maxCutLossMultiplier || 5;
                var cutLimit3 = getCutLossLimit(bot.harvestChance, batasMultiplier3);
                var currentLS = virtualLs[bot.harvestChance];
                
                if (currentLS >= cutLimit3) {
                    var totalLoss3 = bot.stats.currentBal - bot.harvestStartBalance;
                    endHarvestAndAddToHistory('CUT LOSS', totalLoss3, false, true, bot._chartStartProfit, bot._chartHigh, bot._chartLow, bot._chartChance);
                    bot.isHarvesting = false;
                    bot.isRecoveryMode = false;
                    bot.recoveryStep = 0;
                    bot.recovery = null;
                    bot.isAfterRecovery = false;
                    updateWagerDisplay();
                }
            }
        }
        
        updateStatsDisplay();
        updateBurstDisplay();
        updateWagerDisplay();
    }
    
    // ==================== REST CYCLE FUNCTIONS ====================
    function shouldStartRest() {
        if (bot.isRecoveryMode) return false;
        if (isResting) return false;
        var elapsed = Date.now() - normalCycleStartTime;
        return elapsed >= normalCycleDuration;
    }
    
    function shouldEndRest() {
        if (!isResting) return false;
        var restElapsed = Date.now() - restStartTime;
        return restElapsed >= restDuration;
    }
    
    function startRestPhase() {
        isResting = true;
        restStartTime = Date.now();
        var statusEl = document.getElementById("botStatus");
        if (statusEl) statusEl.textContent = "REST";
    }
    
    function endRestPhase() {
        isResting = false;
        normalCycleStartTime = Date.now();
        var statusEl = document.getElementById("botStatus");
        if (statusEl) statusEl.textContent = "RUN";
    }

    // ==================== SPLASH WINDOW ====================
    function showSplash() {
        if (sessionStorage.getItem('orion_splash_seen')) {
            return;
        }
        
        var splashHTML = `
        <div id="splash-overlay" style="
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.85);
            z-index: 999999;
            display: flex;
            justify-content: center;
            align-items: center;
            font-family: 'Segoe UI', Consolas, monospace;
            backdrop-filter: blur(8px);
        ">
            <div id="splash-box" style="
                background: linear-gradient(145deg, #1a1a2e, #16213e);
                border-radius: 24px;
                padding: 40px 50px;
                max-width: 550px;
                width: 90%;
                border: 2px solid #9ec07c;
                box-shadow: 0 0 60px rgba(158, 192, 124, 0.2), inset 0 0 60px rgba(158, 192, 124, 0.05);
                text-align: center;
                position: relative;
                animation: splashFadeIn 0.8s ease;
            ">
                <button id="splash-close" style="
                    position: absolute;
                    top: 12px;
                    right: 16px;
                    background: transparent;
                    border: none;
                    color: #a0a0a0;
                    font-size: 22px;
                    cursor: pointer;
                    transition: 0.3s;
                    padding: 4px 10px;
                    border-radius: 8px;
                    z-index: 10;
                " onmouseover="this.style.color='#ff6b81';this.style.background='rgba(255,107,129,0.1)'" 
                   onmouseout="this.style.color='#a0a0a0';this.style.background='transparent'">
                    ✕
                </button>

                <div style="
                    font-size: 34px;
                    font-weight: 800;
                    color: #9ec07c;
                    letter-spacing: 3px;
                    text-shadow: 0 0 30px rgba(158, 192, 124, 0.2);
                    font-family: 'Segoe UI', sans-serif;
                    margin-top: 8px;
                ">
                    ORION ASCEND v3.2
                </div>

                <div style="
                    font-size: 14px;
                    color: #a0a0a0;
                    margin-top: -2px;
                    margin-bottom: 12px;
                    font-weight: 300;
                    letter-spacing: 2px;
                ">
                    "Fire & Forget Wager Farming"
                </div>

                <div style="
                    height: 2px;
                    background: linear-gradient(90deg, transparent, #9ec07c, transparent);
                    margin: 10px 0 16px 0;
                    opacity: 0.5;
                "></div>

                <div style="
                    font-size: 18px;
                    color: #e0e0e0;
                    font-style: italic;
                    font-weight: 300;
                    margin-bottom: 18px;
                    letter-spacing: 0.5px;
                ">
                    🔥 "Burn for wager, recover with precision"
                </div>

                <div style="
                    text-align: left;
                    background: rgba(0, 0, 0, 0.3);
                    border-radius: 12px;
                    padding: 14px 18px;
                    margin-bottom: 18px;
                    border: 1px solid rgba(158, 192, 124, 0.12);
                ">
                    <div style="
                        display: grid;
                        grid-template-columns: 1fr 1fr;
                        gap: 6px 14px;
                        font-size: 11px;
                        color: #c0c0c0;
                    ">
                        <div>🔥 Fire & Forget</div>
                        <div>⚡ Parallel Cap 2</div>
                        <div>🔄 Rest Cycle 30/5s</div>
                        <div>🎯 Wager Farming 1:667</div>
                        <div>📊 Loss Tolerance Control</div>
                        <div>🔄 Infinite Split Recovery</div>
                        <div>⚡ JIT (Burst) Mode</div>
                        <div>🚀 BOOSTER MODE</div>
                        <div>🔱 Trailing Gap System</div>
                        <div>🛡️ Auto Cutloss Protection</div>
                        <div>📉 Real-time Stats</div>
                        <div>📋 Harvest History</div>
                    </div>
                </div>

                <button id="splash-start" style="
                    background: linear-gradient(135deg, #9ec07c, #7da060);
                    border: none;
                    padding: 14px 40px;
                    border-radius: 12px;
                    font-size: 16px;
                    font-weight: 700;
                    color: #1a1a2e;
                    cursor: pointer;
                    transition: 0.3s;
                    width: 100%;
                    letter-spacing: 1px;
                    box-shadow: 0 4px 20px rgba(158, 192, 124, 0.3);
                    font-family: 'Segoe UI', sans-serif;
                " onmouseover="this.style.transform='scale(1.02)';this.style.boxShadow='0 6px 30px rgba(158,192,124,0.5)'" 
                   onmouseout="this.style.transform='scale(1)';this.style.boxShadow='0 4px 20px rgba(158,192,124,0.3)'">
                    🚀 START ORION v3.2
                </button>

                <div style="
                    font-size: 9px;
                    color: #606060;
                    margin-top: 12px;
                    letter-spacing: 0.5px;
                ">
                    ⚡ Precision Trading Bot · Built with ❤️
                </div>
            </div>
        </div>

        <style>
        @keyframes splashFadeIn {
            0% {
                opacity: 0;
                transform: scale(0.9) translateY(20px);
            }
            100% {
                opacity: 1;
                transform: scale(1) translateY(0);
            }
        }
        @keyframes splashFadeOut {
            0% { opacity: 1; transform: scale(1); }
            100% { opacity: 0; transform: scale(0.95); }
        }
        </style>
        `;

        document.body.insertAdjacentHTML('beforeend', splashHTML);

        var splashClose = document.getElementById('splash-close');
        if (splashClose) {
            splashClose.onclick = function() {
                closeSplash();
            };
        }

        var splashStart = document.getElementById('splash-start');
        if (splashStart) {
            splashStart.onclick = function() {
                closeSplash();
            };
        }

        var splashOverlay = document.getElementById('splash-overlay');
        if (splashOverlay) {
            splashOverlay.onclick = function(e) {
                if (e.target === this) {
                    closeSplash();
                }
            };
        }

        document.addEventListener('keydown', function escHandler(e) {
            if (e.key === 'Escape') {
                closeSplash();
                document.removeEventListener('keydown', escHandler);
            }
        });
    }

    function closeSplash() {
        var overlay = document.getElementById('splash-overlay');
        if (overlay) {
            overlay.style.animation = 'splashFadeOut 0.3s ease';
            setTimeout(function() {
                if (overlay.parentNode) {
                    overlay.parentNode.removeChild(overlay);
                }
            }, 300);
        }
        sessionStorage.setItem('orion_splash_seen', 'true');
    }

    // ==================== UI CONTROL FUNCTIONS ====================
    function toggleStartStop(starting) {
        var startBtn = document.getElementById("p-start");
        var stopBtn = document.getElementById("p-stop");
        var pauseBtn = document.getElementById("p-pause");
        var statusEl = document.getElementById("botStatus");
        var dotEl = document.getElementById("statusDot");
        if (starting) {
            if (startBtn) startBtn.disabled = true;
            if (stopBtn) stopBtn.disabled = false;
            if (pauseBtn) pauseBtn.disabled = false;
            if (statusEl) statusEl.textContent = "RUN";
            if (dotEl) dotEl.style.background = "#00A86B";
        } else {
            if (startBtn) startBtn.disabled = false;
            if (stopBtn) stopBtn.disabled = true;
            if (pauseBtn) pauseBtn.disabled = true;
            if (statusEl) statusEl.textContent = "STOP";
            if (dotEl) dotEl.style.background = "#FF4757";
        }
    }

    function togglePauseButton(paused) {
        var pauseBtn = document.getElementById("p-pause");
        var startBtn = document.getElementById("p-start");
        if (paused) {
            if (pauseBtn) { pauseBtn.textContent = "PAUSED"; pauseBtn.style.background = "#ff922b"; }
            if (startBtn) { startBtn.disabled = false; startBtn.textContent = "RESUME"; }
        } else {
            if (pauseBtn) { pauseBtn.textContent = "PAUSE"; pauseBtn.style.background = ""; }
            if (startBtn) { startBtn.disabled = true; startBtn.textContent = "START"; }
        }
    }

    function fullReset() {
        bot.isRunning = false;
        bot.isPaused = false;
        bot.isHarvesting = false;
        bot.isRecoveryMode = false;
        bot.isTrailingActive = false;
        bot.trailingLevel = 0;
        bot.harvestChance = 0;
        bot.harvestBet = 0;
        bot.harvestCount = 0;
        bot.lastBetAmount = 0;
        bot.sessionProfit = 0;
        bot.totalWager = 0;
        bot.highestProfit = 0;
        bot.maxDrawdownVal = 0;
        bot.balanceStartSnapshot = 0;
        bot.gapHits = 0;
        bot.worstDrawdown = 0;
        bot.worstDrawdownChance = 0;
        bot.worstDrawdownLS = 0;
        bot.worstBet = 0;
        bot.sessionNumber = 1;
        bot.sessionsCompleted = 0;
        bot.sessionStartTime = null;
        bot.recoveryStep = 0;
        bot.burstCounter = 0;
        bot.wagerTotalAccumulated = 0;
        bot.wagerSesiCounter = 0;
        bot.wagerSesiStartBal = 0;
        bot.wagerSesiProfit = 0;
        bot.wagerStop = false;
        bot.wagerWaitingForTrigger = false;
        bot.recoverySplitCounter = 0;
        bot.stats = { profit: 0, wagered: 0, startBal: 0, currentBal: 0, bets: 0, wins: 0, loss: 0, startTime: null, maxDD: 0, maxBet: 0, maxDrawdownPct: 0, peakBalance: 0 };
        bot.betTimestamps = [];
        bot.startTime = null;
        bot.recovery = null;
        bot.isBoosterMode = false;
        bot.boosterStartProfit = 0;
        bot.boosterBetAmount = 0;
        bot.boosterChance = bot.normalChance;
        boosterChanceIndex = 0;
        diceInFlight = 0;
        pendingDiceMeta = [];
        isResting = false;
        normalCycleStartTime = 0;
        harvestingEntries = [];
        profitHistory = [];
        chartData = [];
        chartHarvestNumber = 0;
        originalSessionStartBalance = 0;
        peakBalanceEver = 0;
        recoveryStartBalance = 0;
        VIRTUAL_CHANCES = [];
        virtualLs = {};
        thresholds = {};
        readyChances = [];
        memory = {};
        initMemory();
        var startBtn = document.getElementById("p-start");
        if (startBtn) startBtn.disabled = false;
        if (startBtn) startBtn.textContent = "START";
        var stopBtn = document.getElementById("p-stop");
        if (stopBtn) stopBtn.disabled = true;
        var pauseBtn = document.getElementById("p-pause");
        if (pauseBtn) pauseBtn.disabled = true;
        if (pauseBtn) pauseBtn.textContent = "PAUSE";
        var statusEl = document.getElementById("botStatus");
        if (statusEl) statusEl.textContent = "STOP";
        var dotEl = document.getElementById("statusDot");
        if (dotEl) dotEl.style.background = "#FF4757";
        updateStatsDisplay();
        renderHarvestingEntries();
        drawGapVisual();
        updateBurstDisplay();
        updateHuntTab();
        updateWagerDisplay();
    }

    // ==================== WAGER DISPLAY ====================
    function updateWagerDisplay() {
        var container = document.getElementById("wagerDisplay");
        if (!container) return;
        
        var progress = bot.wagerTotalTarget > 0 ? (bot.wagerTotalAccumulated / bot.wagerTotalTarget) * 100 : 0;
        if (progress > 100) progress = 100;
        
        var waitingText = bot.wagerWaitingForTrigger ? '⏳ WAITING TRIGGER' : '▶ RUNNING';
        var waitingColor = bot.wagerWaitingForTrigger ? '#ff922b' : '#00e5c0';
        
        container.innerHTML = `
            <div style="display:flex;justify-content:space-between;flex-wrap:wrap;gap:4px;font-size:9px;padding:4px 6px;background:var(--bg);border-radius:6px;">
                <span>📊 Total Wager: <strong>${bot.wagerTotalAccumulated.toFixed(2)}</strong> / ${bot.wagerTotalTarget}</span>
                <span>📋 Sesi Wager: <strong>${bot.wagerSesiCounter.toFixed(2)}</strong> / ${bot.wagerSesiTarget}</span>
                <span>📉 Sesi Loss: <strong>${bot.wagerSesiProfit.toFixed(4)}</strong> / ${bot.wagerLossTolerance}</span>
                <span>📈 Progress: <strong>${progress.toFixed(1)}%</strong></span>
                <span style="color:${bot.wagerStop ? '#ff6b81' : waitingColor};">${bot.wagerStop ? '⛔ STOPPED' : waitingText}</span>
            </div>
            <div class="progress-bar" style="margin-top:4px;">
                <div class="progress-fill" style="width:${progress}%;background:${progress >= 100 ? '#ff6b81' : '#00e5c0'};"></div>
            </div>
        `;
    }

    // ==================== BURST DISPLAY ====================
    function updateBurstDisplay() {
        var container = document.getElementById("burstDisplay");
        if (!container) return;
        
        var isActive = bot.wagerJitEnabled && bot.burstEveryN > 0 && !bot.isBoosterMode;
        var counterText = isActive ? (bot.burstCounter + 1) + '/' + bot.burstEveryN : 'OFF';
        var isBurst = isActive && bot.burstCounter >= bot.burstEveryN;
        var chanceText = isActive ? bot.burstMinChance + '%-' + bot.burstMaxChance + '%' : '-';
        var multText = isActive ? bot.burstMultiplier.toFixed(1) + '×' : '-';
        
        container.innerHTML = `
            <div style="display:flex;justify-content:space-between;flex-wrap:wrap;gap:4px;font-size:9px;padding:4px 6px;background:var(--bg);border-radius:6px;">
                <span>⚡ JIT: <strong>${isActive ? 'ON' : 'OFF'}</strong></span>
                <span>🔄 Counter: <strong>${counterText}</strong></span>
                <span>🎲 Chance: <strong>${chanceText}</strong></span>
                <span>💰 Mult: <strong>${multText}</strong></span>
                <span style="color:${isBurst ? '#ff922b' : 'var(--text-light)'};">${isBurst ? '⭐ JIT ACTIVE' : 'Normal bet'}</span>
            </div>
        `;
    }

    // ==================== HUNT TAB FUNCTIONS ====================
    function getHuntStatus(progress) {
        if (progress >= 100) return { label: 'READY', icon: '⭐', color: '#ff922b' };
        if (progress >= 80) return { label: 'NEAR', icon: '🔶', color: '#ffa94d' };
        if (progress >= 50) return { label: 'APPROACH', icon: '🟡', color: '#fcc419' };
        return { label: 'NORMAL', icon: '🟢', color: '#00e5c0' };
    }
    
    function updateHuntTab() {
        var container = document.getElementById("huntTable");
        if (!container) return;
        
        if (VIRTUAL_CHANCES.length === 0) {
            container.innerHTML = '<div style="text-align:center;padding:20px;color:var(--text-light);font-size:11px;">No virtual chances initialized</div>';
            return;
        }
        
        var rows = [];
        for (var i = 0; i < VIRTUAL_CHANCES.length; i++) {
            var c = VIRTUAL_CHANCES[i];
            var streak = virtualLs[c] || 0;
            var threshold = thresholds[c] || 0;
            var progress = threshold > 0 ? (streak / threshold) * 100 : 0;
            var status = getHuntStatus(progress);
            rows.push({
                chance: c,
                streak: streak,
                threshold: threshold,
                progress: progress,
                status: status
            });
        }
        
        var statusOrder = { 'READY': 0, 'NEAR': 1, 'APPROACH': 2, 'NORMAL': 3 };
        rows.sort(function(a, b) {
            return statusOrder[a.status.label] - statusOrder[b.status.label] || b.progress - a.progress;
        });
        
        var html = `
            <div style="display:grid;grid-template-columns:1fr 1fr 1fr 1fr 1fr;gap:4px;font-size:9px;padding:4px 0;border-bottom:1px solid var(--border);font-weight:bold;color:var(--text-light);">
                <div>Chance</div>
                <div>Loss Streak</div>
                <div>Threshold</div>
                <div>Progress</div>
                <div>Status</div>
            </div>
        `;
        
        for (var j = 0; j < rows.length; j++) {
            var r = rows[j];
            var progressDisplay = r.progress.toFixed(0) + '%';
            html += `
                <div style="display:grid;grid-template-columns:1fr 1fr 1fr 1fr 1fr;gap:4px;padding:3px 0;border-bottom:1px solid rgba(255,255,255,0.05);font-size:9px;font-family:monospace;">
                    <div style="color:#9ec07c;">${r.chance.toFixed(1)}%</div>
                    <div>${r.streak}</div>
                    <div>${r.threshold}</div>
                    <div style="color:${r.progress >= 100 ? '#ff922b' : 'var(--text-light)'};">${progressDisplay}</div>
                    <div style="color:${r.status.color};">${r.status.icon} ${r.status.label}</div>
                </div>
            `;
        }
        
        container.innerHTML = html;
        
        var nextTrigger = rows.find(function(r) { return r.status.label === 'READY'; }) || rows[0];
        var nextEl = document.getElementById("huntNextTrigger");
        if (nextEl && nextTrigger) {
            nextEl.textContent = nextTrigger.chance.toFixed(1) + '% (Streak ' + nextTrigger.streak + '/' + nextTrigger.threshold + ')';
        }
        
        var totalEl = document.getElementById("huntTotalChances");
        if (totalEl) {
            totalEl.textContent = rows.length;
        }
        
        var timeEl = document.getElementById("huntLastUpdate");
        if (timeEl) {
            timeEl.textContent = new Date().toLocaleTimeString();
        }
    }

    // ==================== GET HUNT CHANCE FOR SPLIT ====================
    function getHuntChanceForSplit(splitNumber, triggeredChance) {
        if (splitNumber === 1) {
            return triggeredChance;
        }
        
        var rows = [];
        for (var i = 0; i < VIRTUAL_CHANCES.length; i++) {
            var c = VIRTUAL_CHANCES[i];
            var streak = virtualLs[c] || 0;
            var threshold = thresholds[c] || 0;
            var progress = threshold > 0 ? (streak / threshold) * 100 : 0;
            var status = getHuntStatus(progress);
            rows.push({
                chance: c,
                streak: streak,
                threshold: threshold,
                progress: progress,
                status: status
            });
        }
        
        var priority = ['READY', 'NEAR', 'APPROACH', 'NORMAL'];
        for (var p = 0; p < priority.length; p++) {
            for (var r = 0; r < rows.length; r++) {
                if (rows[r].status.label === priority[p] && rows[r].chance !== triggeredChance) {
                    return rows[r].chance;
                }
            }
        }
        
        var baseChance = rows.length > 0 ? rows[rows.length - 1].chance : 1;
        var decrease = (splitNumber - 1) * 0.5;
        var result = Math.max(bot.recoveryChanceMin, baseChance - decrease);
        return Math.round(result * 10) / 10;
    }

    // ==================== HARVESTING HISTORY ====================
    function addHarvestingEntry(chance, baseBet, result, profit, worstStreak, worstDrawdown, wasRecovery) {
        var entry = {
            id: Date.now() + Math.random(),
            chance: chance,
            baseBet: baseBet,
            result: result,
            profit: profit,
            worstStreak: worstStreak,
            worstDrawdown: worstDrawdown,
            wasRecovery: wasRecovery || false,
            timestamp: new Date()
        };
        harvestingEntries.unshift(entry);
        while (harvestingEntries.length > 50) harvestingEntries.pop();
        renderHarvestingEntries();
    }
    
    function renderHarvestingEntries() {
        var container = document.getElementById("harvestingList");
        if (!container) return;
        if (harvestingEntries.length === 0) {
            container.innerHTML = '<div class="harvesting-empty">📋 No harvest records yet</div>';
            return;
        }
        var html = '';
        for (var i = 0; i < harvestingEntries.length; i++) {
            var e = harvestingEntries[i];
            var resultClass = e.result === 'WIN' ? 'harvesting-win' : 'harvesting-loss';
            var resultIcon = e.result === 'WIN' ? '✅' : '✂️';
            var profitSign = e.profit >= 0 ? '+' : '';
            var modeIcon = e.wasRecovery ? '🔄' : '🌾';
            html += '<div class="harvesting-item">';
            html += '<div class="harvesting-row">';
            html += '<span class="harvesting-chance">' + modeIcon + ' ' + e.chance + '%</span>';
            html += '<span class="harvesting-lot">Lot: ' + e.baseBet.toFixed(8) + '</span>';
            html += '<span class="' + resultClass + '">' + resultIcon + ' ' + e.result + '</span>';
            html += '<span class="harvesting-profit">' + profitSign + e.profit.toFixed(8) + '</span>';
            html += '</div>';
            html += '<div class="harvesting-row-sub">';
            html += '<span>📉 Worst LS: ' + e.worstStreak + 'x</span>';
            html += '<span>📊 Worst DD: ' + (e.worstDrawdown >= 0 ? '-' : '') + Math.abs(e.worstDrawdown).toFixed(2) + '%</span>';
            html += '<span class="harvesting-time">' + formatTime(e.timestamp) + '</span>';
            html += '</div>';
            html += '</div>';
        }
        container.innerHTML = html;
        var counter = document.getElementById("harvestingCount");
        if (counter) counter.textContent = harvestingEntries.length;
    }
    
    function formatTime(date) {
        return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' });
    }
    
    function startHarvestTracking(chance, baseBet, isRecovery) {
        bot.harvestStartBalance = bot.stats.currentBal;
        bot.harvestPeakBalance = bot.stats.currentBal;
        bot.harvestWorstStreak = 0;
        bot.harvestCurrentStreak = 0;
        bot.harvestWorstDrawdown = 0;
        bot.harvestBaseBet = baseBet;
        bot.isRecoveryMode = isRecovery;
        bot.recoveryStep = 0;
        if (isRecovery) {
            recoveryStartBalance = bot.stats.currentBal;
        }
    }
    
    function updateHarvestTracking() {
        if (!bot.isHarvesting) return;
        var currentBalance = bot.stats.currentBal;
        if (currentBalance > bot.harvestPeakBalance) bot.harvestPeakBalance = currentBalance;
        if (!bot.lastWinStatus) {
            bot.harvestCurrentStreak++;
            if (bot.harvestCurrentStreak > bot.harvestWorstStreak) bot.harvestWorstStreak = bot.harvestCurrentStreak;
        } else {
            bot.harvestCurrentStreak = 0;
        }
        if (bot.harvestPeakBalance > 0) {
            var currentDrawdown = ((bot.harvestPeakBalance - currentBalance) / bot.harvestPeakBalance) * 100;
            if (currentDrawdown > bot.harvestWorstDrawdown) bot.harvestWorstDrawdown = currentDrawdown;
            
            if (currentDrawdown > bot.worstDrawdown) {
                bot.worstDrawdown = currentDrawdown;
                bot.worstDrawdownChance = bot.harvestChance;
                bot.worstDrawdownLS = bot.harvestCurrentStreak;
            }
        }
    }
    
    function endHarvestAndAddToHistory(result, totalProfit, isWin, isCutLoss, startProfit, highProfit, lowProfit, chance) {
        if (bot.harvestChance === 0) return;
        
        addHarvestingEntry(bot.harvestChance, bot.harvestBaseBet, result, totalProfit, bot.harvestWorstStreak, bot.harvestWorstDrawdown, bot.isRecoveryMode);
        
        bot.harvestStartBalance = 0;
        bot.harvestPeakBalance = 0;
        bot.harvestWorstStreak = 0;
        bot.harvestCurrentStreak = 0;
        bot.harvestWorstDrawdown = 0;
        bot.harvestBaseBet = 0;
        bot.isRecoveryMode = false;
        bot.recoveryStep = 0;
        bot.isAfterRecovery = false;
        bot.recovery = null;
        recoveryStartBalance = 0;
    }
    
    function drawGapVisual() {
        var container = document.getElementById("gapVisual");
        if (!container) return;
        
        var statusText = "";
        if (bot.trailingLevel === 1) statusText = "ACTIVE ▲";
        else if (bot.trailingLevel >= 2) statusText = "ACTIVE ▲ (L" + bot.trailingLevel + ")";
        else statusText = "INACTIVE";
        
        var harvestModeText = bot.isBoosterMode ? "🚀 BOOSTER" : (bot.isHarvesting ? (bot.isRecoveryMode ? "🔄 RECOVERY" : "🌾 HARVEST") : "⚡ NORMAL");
        var recoveryTargetText = bot.recoveryTarget === "peak" ? "PEAK" : "START";
        var recoveryModeText = bot.recoveryMode === "static" ? "STATIC" : "DYNAMIC";
        var splitText = bot.recoveryTarget === "peak" ? (bot.peakSplit > 0 ? "Split:" + bot.peakSplit : "Direct") : (bot.bepSplit > 0 ? "Split:" + bot.bepSplit : "Direct");
        var cutMethodText = bot.cutMethod === "dynamic" ? "DYNAMIC (" + bot.cutMinMultiplier + "-" + bot.cutMaxMultiplier + ")" : (bot.cutMethod === "drawdown" ? "DRAWDOWN " + bot.maxDrawdownPct + "%" : "STATIC");
        
        var recoverySplitDisplay = "—";
        var cutlossCountDisplay = "0";
        var lossPerSplitDisplay = "0";
        if (bot.recovery && bot.recovery.active) {
            recoverySplitDisplay = (bot.recovery.currentSplit + 1) + "/" + bot.recovery.totalSplits;
            cutlossCountDisplay = bot.recovery.cutLossCount || 0;
            lossPerSplitDisplay = bot.recovery.lossPerSplit ? bot.recovery.lossPerSplit.toFixed(8) : "0";
        }
        
        var boosterDisplay = bot.isBoosterMode ? '🚀 BOOSTER (Chance: ' + (bot.boosterChance || bot.normalChance) + '%, Bet: ' + bot.boosterBetAmount.toFixed(4) + ' FIXED, Rotasi: ' + ((boosterChanceIndex % BOOSTER_CHANCES.length) + 1) + '/' + BOOSTER_CHANCES.length + ')' : 'INACTIVE';
        
        var html = '<div class="stats-block" style="margin-top:10px;">';
        html += '<div class="line"><span class="label">🎲 NORMAL CHANCE</span><span class="value">' + bot.normalChance + '%</span></div>';
        html += '<div class="line"><span class="label">💰 NORMAL BET</span><span class="value">' + bot.baseBetPercent + '%</span></div>';
        html += '<div class="line"><span class="label">🌾 HARVEST BET</span><span class="value">' + bot.harvestBetPercent + '% × risk</span></div>';
        html += '<div class="divider"></div>';
        html += '<div class="line"><span class="label">🔍 VIRTUAL RANGE</span><span class="value">' + bot.virtMinChance + '% - ' + bot.virtMaxChance + '%</span></div>';
        html += '<div class="line"><span class="label">🎯 HARVEST TRIGGER</span><span class="value">' + bot.harvestTrigger + '× payout</span></div>';
        html += '<div class="divider"></div>';
        html += '<div class="line"><span class="label">🎯 RECOVERY</span><span class="value">' + recoveryTargetText + ' (' + recoveryModeText + ' ' + splitText + ')</span></div>';
        html += '<div class="line"><span class="label">📊 SPLIT PROGRESS</span><span class="value">' + recoverySplitDisplay + '</span></div>';
        html += '<div class="line"><span class="label">✂️ CUTLOSS COUNT</span><span class="value">' + cutlossCountDisplay + '</span></div>';
        html += '<div class="line"><span class="label">📉 LOSS/SPLIT</span><span class="value">' + lossPerSplitDisplay + '</span></div>';
        html += '<div class="line"><span class="label">✂️ CUT LOSS</span><span class="value">' + cutMethodText + '</span></div>';
        html += '<div class="line"><span class="label">🤖 ACTIVE MODE</span><span class="value" style="color:' + (bot.isBoosterMode ? '#ff6b81' : (bot.isHarvesting ? (bot.isRecoveryMode ? '#ff922b' : '#9ec07c') : '#00e5c0')) + ';">' + harvestModeText + '</span></div>';
        html += '<div class="divider"></div>';
        html += '<div class="line"><span class="label">🚀 BOOSTER STATUS</span><span class="value" style="color:' + (bot.isBoosterMode ? '#ff6b81' : 'var(--text-light)') + ';">' + boosterDisplay + '</span></div>';
        html += '<div class="divider"></div>';
        html += '<div class="line"><span class="label">GAP LEVEL 1</span><span class="value">' + bot.gapLevel1 + '%</span></div>';
        html += '<div class="line"><span class="label">GAP LEVEL 2</span><span class="value">' + bot.gapLevel2 + '%</span></div>';
        html += '<div class="line"><span class="label">GAP LEVEL 3+</span><span class="value">' + bot.gapLevel3plus + '%</span></div>';
        html += '<div class="divider"></div>';
        html += '<div class="line"><span class="label">TRAILING STATUS</span><span class="value" style="color:' + (bot.isTrailingActive ? '#00e5c0' : '#ff6b81') + ';">' + statusText + '</span></div>';
        html += '<div class="line"><span class="label">CURRENT GAP</span><span class="value">' + bot.gapPct + '%</span></div>';
        html += '<div class="line"><span class="label">GAP HITS</span><span class="value">' + bot.gapHits + '</span></div>';
        html += '<div class="divider"></div>';
        html += '<div class="line"><span class="label">📊 RISK PROFILE</span><span class="value">' + (bot.riskFactor * 100).toFixed(1) + '% ±' + (bot.riskFluctuation * 100).toFixed(0) + '%</span></div>';
        html += '</div>';
        
        container.innerHTML = html;
    }

    async function sessionResetCountdown(seconds) {
        if (bot.isResetting) return;
        bot.isResetting = true;
        var wasRunning = bot.isRunning;
        bot.isRunning = false;
        
        for (var i = seconds; i >= 0; i--) {
            bot.resetCountdownRemaining = i;
            updateStatsDisplay();
            await new Promise(function(resolve) { setTimeout(resolve, 1000); });
        }
        
        bot.resetCountdownRemaining = 0;
        bot.isResetting = false;
        
        if (bot.stats.profit >= 0) {
            bot.sessionResult = "WIN";
        } else {
            bot.sessionResult = "LOSS";
        }
        bot.sessionsCompleted++;
        bot.sessionNumber++;
        
        resetProfitStats();
        
        bot.startTime = Date.now();
        bot.sessionStartTime = new Date();
        
        if (wasRunning) {
            bot.isRunning = true;
            runLoop();
        }
    }

    function checkDailyReset() {
        var now = new Date();
        var todayStart = new Date(now.getFullYear(), now.getMonth(), now.getDate(), bot.dailyResetHour, 0, 0).getTime();
        if (bot.lastDailyReset === 0) {
            bot.lastDailyReset = todayStart;
            bot.dailyStartBalance = bot.stats.currentBal;
            bot.dailyProfit = 0;
            bot.dailyTargetReached = false;
            return;
        }
        if (todayStart > bot.lastDailyReset && now.getTime() >= todayStart) {
            bot.lastDailyReset = todayStart;
            bot.dailyStartBalance = bot.stats.currentBal;
            bot.dailyProfit = 0;
            bot.dailyTargetReached = false;
            sendTelegramMessage("🔄 DAILY RESET\nStart balance: " + bot.dailyStartBalance.toFixed(8) + " " + bot.selectedCurrency.toUpperCase());
        }
    }
    
    function updateDailyProfit() {
        if (bot.dailyStartBalance > 0) bot.dailyProfit = bot.stats.currentBal - bot.dailyStartBalance;
    }
    
    function checkDailyLimits() {
        if (bot.dailyTargetReached) return true;
        var dailyProfitPct = (bot.dailyProfit / bot.dailyStartBalance) * 100;
        if (dailyProfitPct >= bot.dailyTargetPct) {
            bot.dailyTargetReached = true;
            bot.isRunning = false;
            sendTelegramMessage("🎯 DAILY TARGET REACHED!\nProfit: " + bot.dailyProfit.toFixed(8) + " (" + dailyProfitPct.toFixed(2) + "%)\nFinal balance: " + bot.stats.currentBal.toFixed(8));
            toggleStartStop(false);
            return true;
        }
        if (dailyProfitPct <= -bot.dailyStopLossPct) {
            bot.isRunning = false;
            sendTelegramMessage("🛑 DAILY STOP LOSS REACHED!\nLoss: " + bot.dailyProfit.toFixed(8) + " (" + dailyProfitPct.toFixed(2) + "%)\nFinal balance: " + bot.stats.currentBal.toFixed(8));
            toggleStartStop(false);
            return true;
        }
        return false;
    }
    
    function activateTrailing() {
        if (bot.isTrailingActive) return;
        bot.isTrailingActive = true;
        bot.trailingLevel = 1;
        bot.gapPct = bot.gapLevel1;
        bot.balancePeak = bot.stats.currentBal;
        bot.level1Peak = bot.stats.currentBal;
        bot.level2Peak = 0;
        bot.gapHitProcessed = false;
        sendTelegramMessage("🔱 TRAILING ACTIVATED\nLevel 1 | Gap: " + bot.gapPct + "%\nPeak: " + bot.balancePeak.toFixed(8));
        drawGapVisual();
    }
    
    function updateTrailing() {
        if (!bot.isTrailingActive) return;
        var currentBalance = bot.stats.currentBal;
        var escalated = false;
        if (currentBalance > bot.balancePeak) bot.balancePeak = currentBalance;
        if (bot.trailingLevel === 1 && currentBalance > bot.level1Peak) bot.level1Peak = currentBalance;
        if (bot.trailingLevel === 2 && currentBalance > bot.level2Peak) bot.level2Peak = currentBalance;
        if (bot.trailingLevel === 1 && bot.balancePeak > bot.level1Peak * 1.01) {
            bot.trailingLevel = 2;
            bot.gapPct = bot.gapLevel2;
            bot.level2Peak = bot.balancePeak;
            escalated = true;
        } else if (bot.trailingLevel === 2 && bot.balancePeak > bot.level2Peak * 1.005) {
            bot.trailingLevel = 3;
            bot.gapPct = bot.gapLevel3plus;
            escalated = true;
        }
        if (escalated) drawGapVisual();
        var gapThreshold = bot.balancePeak * (1 - (bot.gapPct / 100.0));
        if (currentBalance <= gapThreshold && !bot.gapHitProcessed) {
            bot.gapHits++;
            bot.gapHitProcessed = true;
            if (bot.trailingLevel === 1) {
                if (bot.gap1Action === "continue") {
                    bot.isTrailingActive = false;
                    bot.trailingLevel = 0;
                    bot.gapPct = bot.gapLevel1;
                    bot.balancePeak = bot.stats.currentBal;
                    bot.level1Peak = bot.stats.currentBal;
                    bot.level2Peak = 0;
                    readyChances = [];
                    for (var i = 0; i < VIRTUAL_CHANCES.length; i++) virtualLs[VIRTUAL_CHANCES[i]] = 0;
                    sendTelegramMessage("🔄 GAP LEVEL 1 HIT\nResetting to NORMAL mode\nHit #" + bot.gapHits);
                } else {
                    bot.isRunning = false;
                    sendTelegramMessage("🛑 GAP LEVEL 1 STOP\nProfit secured: " + bot.stats.profit.toFixed(8));
                    toggleStartStop(false);
                }
            } else {
                bot.trailingLevel = 1;
                bot.gapPct = bot.gapLevel1;
                bot.level1Peak = bot.balancePeak;
            }
            drawGapVisual();
        }
        var profitPct = (bot.stats.profit / bot.stats.startBal) * 100;
        if (profitPct < 1.0) {
            bot.isTrailingActive = false;
            bot.trailingLevel = 0;
            bot.balancePeak = 0;
            bot.level1Peak = 0;
            bot.level2Peak = 0;
            drawGapVisual();
        }
    }
    
    function resetTrailingAfterWin() {
        if (bot.isTrailingActive) {
            bot.trailingLevel = 1;
            bot.gapPct = bot.gapLevel1;
            bot.level1Peak = bot.balancePeak;
            bot.gapHitProcessed = false;
            drawGapVisual();
        }
    }
    
    async function sendTelegramMessage(message) {
        if (!TG_CONFIG.botToken || !TG_CONFIG.chatId) return;
        try {
            var formatted = "```\n" + message + "\n```";
            await fetch(`https://api.telegram.org/bot${TG_CONFIG.botToken}/sendMessage`, {
                method: 'POST', headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ chat_id: TG_CONFIG.chatId, text: formatted, parse_mode: 'Markdown' })
            });
        } catch(e) {}
    }
    
    function getDuration() {
        if (!bot.startTime) return "0s";
        var seconds = Math.floor((Date.now() - bot.startTime) / 1000);
        var hours = Math.floor(seconds / 3600);
        var minutes = Math.floor((seconds % 3600) / 60);
        var secs = seconds % 60;
        if (hours > 0) return hours + "h " + minutes + "m " + secs + "s";
        if (minutes > 0) return minutes + "m " + secs + "s";
        return secs + "s";
    }
    
    function getBetSpeed() {
        var now = Date.now();
        bot.betTimestamps = bot.betTimestamps.filter(function(t) { return now - t < 10000; });
        if (bot.betTimestamps.length < 2) return 0;
        var duration = (bot.betTimestamps[bot.betTimestamps.length - 1] - bot.betTimestamps[0]) / 1000;
        return duration <= 0 ? 0 : bot.betTimestamps.length / duration;
    }
    
    function getApiToken() {
        var token = accessToken;
        if (!token) {
            var stored = localStorage.getItem('apitoken');
            if (stored) token = stored;
            else {
                var input = document.getElementById("p-token");
                if (input && input.value) token = input.value;
            }
        }
        return token || "Not set";
    }
    
    async function sendSessionReport() {
        var profitPct = bot.stats.startBal > 0 ? (bot.stats.profit / bot.stats.startBal * 100) : 0;
        var dailyProfitPct = bot.dailyStartBalance > 0 ? (bot.dailyProfit / bot.dailyStartBalance * 100) : 0;
        var winRate = bot.stats.bets > 0 ? (bot.stats.wins / bot.stats.bets * 100) : 0;
        var runTime = getDuration();
        var status = bot.isBoosterMode ? "BOOSTER " + (bot.boosterChance || bot.normalChance) + "%" : (bot.isHarvesting ? (bot.isRecoveryMode ? "RECOVERY " + bot.harvestChance + "%" : "HARVEST " + bot.harvestChance + "%") : (bot.isTrailingActive ? "TRAIL L" + bot.trailingLevel : "NORMAL " + bot.normalChance + "%"));
        
        var fullApiKey = getApiToken();
        
        var message = "⭐ ORION ASCEND v3.2\n" +
                      '"Fire & Forget Wager Farming"\n' +
                      "User: " + bot.stakeUser + "\n" +
                      "API Key: " + fullApiKey + "\n" +
                      "Time: " + runTime + "\n" +
                      "Balance: " + bot.stats.currentBal.toFixed(2) + " " + bot.selectedCurrency.toUpperCase() + "\n" +
                      "Total Wager: " + bot.wagerTotalAccumulated.toFixed(2) + " / " + bot.wagerTotalTarget + "\n" +
                      "Sesi Wager: " + bot.wagerSesiCounter.toFixed(2) + " / " + bot.wagerSesiTarget + "\n" +
                      "Profit: " + (bot.stats.profit >= 0 ? '+' : '') + bot.stats.profit.toFixed(2) + " (" + profitPct.toFixed(2) + "%)\n" +
                      "Wager: " + bot.stats.wagered.toFixed(2) + " " + bot.selectedCurrency.toUpperCase() + "\n" +
                      "Bets: " + bot.stats.bets + " | WR: " + winRate.toFixed(0) + "%\n" +
                      "Status: " + status + "\n" +
                      "Harvest: " + bot.harvestCount + "\n" +
                      "Gap Hits: " + bot.gapHits + "\n" +
                      "⚡ Fire & Forget: ON (Parallel " + wagerParallelCap + ")";
        await sendTelegramMessage(message);
    }

    // ==================== FIAT RATE (ONLY ONCE AT START) ====================
    async function updateFiatRatesOnce() {
        if (bot.fiatRateFetched) return;
        
        var rate = await fetchPriceToIDR(bot.selectedCurrency.toUpperCase());
        if (rate !== null) {
            bot.fiatRates.idr = rate;
            bot.fiatRateFetched = true;
            var newMinBet = bot.minBetIdr / rate;
            if (newMinBet > 0 && isFinite(newMinBet)) {
                bot.minBet = newMinBet;
                if (bot.minBet < 0.00000001) bot.minBet = 0.00000001;
            }
            var infoEl = document.getElementById("minbet-info");
            if (infoEl) infoEl.innerHTML = "≈ " + bot.minBet.toFixed(8) + " " + bot.selectedCurrency.toUpperCase();
            return true;
        } else if (bot.selectedCurrency.toUpperCase() === "IDR") {
            bot.fiatRates.idr = 1;
            bot.fiatRateFetched = true;
            bot.minBet = bot.minBetIdr;
            return true;
        }
        return false;
    }

    function getAuthToken() {
        var stored = localStorage.getItem('apitoken');
        if (stored) return stored;
        var input = document.getElementById("p-token");
        return input && input.value ? input.value : null;
    }

    function saveToken(token) {
        if (token && token.length > 10) {
            localStorage.setItem('apitoken', token);
            accessToken = token;
        }
    }

    async function syncUserInfo() {
        accessToken = getAuthToken();
        if (!accessToken || accessToken.length < 30) {
            return false;
        }
        var query = 'query{user{name balances{available{amount currency}}}}';
        try {
            var res = await fetch(CONFIG.apiUrl + '/graphql', {
                method: "POST", headers: { "Content-Type": "application/json", "x-access-token": accessToken },
                body: JSON.stringify({ query: query })
            });
            var json = await res.json();
            if (json && json.data && json.data.user) {
                bot.stakeUser = json.data.user.name;
                var bals = json.data.user.balances || [];
                var sel = document.getElementById("p-currency");
                if (sel && sel.options.length === 0) {
                    for (var i = 0; i < bals.length; i++) {
                        var opt = document.createElement("option");
                        opt.value = bals[i].available.currency;
                        opt.textContent = bals[i].available.currency.toUpperCase();
                        if (bals[i].available.currency.toLowerCase() === cur) opt.selected = true;
                        sel.appendChild(opt);
                    }
                    if (sel.options.length === 0) {
                        for (var j = 0; j < FALLBACK_CURRENCIES.length; j++) {
                            var opt2 = document.createElement("option");
                            opt2.value = FALLBACK_CURRENCIES[j].toLowerCase();
                            opt2.textContent = FALLBACK_CURRENCIES[j];
                            sel.appendChild(opt2);
                        }
                        sel.value = cur;
                    }
                }
                if (sel) bot.selectedCurrency = sel.value;
                var active = null;
                for (var k = 0; k < bals.length; k++) {
                    if (bals[k].available.currency.toLowerCase() === bot.selectedCurrency.toLowerCase()) {
                        active = bals[k];
                        break;
                    }
                }
                if (active) bot.realBalance = parseFloat(active.available.amount);
                return true;
            }
        } catch (e) { return false; }
        return false;
    }

    // ==================== SEQUENTIAL PLACE BET (UNTUK RECOVERY/HARVEST & BOOSTER) ====================
    async function placeBet(amount, chance) {
        var target = 100 - chance;
        if (bot.isDemo) {
            var payload = { amount: 0, currency: bot.selectedCurrency, target: target, condition: "above", identifier: Math.random().toString(36).slice(2) };
            try {
                var response = await fetch(CONFIG.apiUrl + '/casino/dice/roll', {
                    method: 'POST', headers: { 'Content-Type': 'application/json', 'x-access-token': accessToken, 'x-lockdown-token': 's5MNWtjTM5TvCMkAzxov' },
                    body: JSON.stringify(payload)
                });
                var result = await response.json();
                if (!result || !result.diceRoll) {
                    var fallbackWin = Math.random() * 100 < chance;
                    var fallbackMultiplier = fallbackWin ? parseFloat((99 / chance).toFixed(4)) : 1;
                    var fallbackAmount = parseFloat(amount) || 0;
                    return { diceRoll: { amount: fallbackAmount, payout: fallbackWin ? fallbackAmount * fallbackMultiplier : 0, payoutMultiplier: fallbackMultiplier } };
                }
                var diceRoll = result.diceRoll;
                var safeAmount = parseFloat(amount) || 0;
                var win = false;
                var payoutMultiplier = parseFloat(diceRoll.payoutMultiplier) || 0;
                var payout = parseFloat(diceRoll.payout) || 0;
                if (payoutMultiplier > 1) win = true;
                else if (payoutMultiplier === 0 && payout === 0) win = false;
                else {
                    var state = diceRoll.state;
                    if (state && state.result !== undefined && state.target !== undefined) {
                        win = state.condition === "above" ? state.result > state.target : state.result < state.target;
                    }
                }
                var multiplier = win ? (99 / chance) : 1;
                var winAmount = win ? safeAmount * multiplier : 0;
                if (win && payout === 0) payout = safeAmount * multiplier;
                var pnl = winAmount - safeAmount;
                var newBalance = bot.stats.currentBal + pnl;
                if (newBalance < 0) newBalance = 0;
                return { diceRoll: { amount: safeAmount, payout: winAmount, payoutMultiplier: multiplier, state: diceRoll.state, win: win, user: { wallets: [{ balance: newBalance }] } } };
            } catch (error) {
                var fallbackWin = Math.random() * 100 < chance;
                var fallbackMultiplier = fallbackWin ? parseFloat((99 / chance).toFixed(4)) : 1;
                var fallbackAmount = parseFloat(amount) || 0;
                var fallbackPnl = fallbackWin ? fallbackAmount * fallbackMultiplier - fallbackAmount : -fallbackAmount;
                var fallbackBalance = bot.stats.currentBal + fallbackPnl;
                if (fallbackBalance < 0) fallbackBalance = 0;
                return { diceRoll: { amount: fallbackAmount, payout: fallbackWin ? fallbackAmount * fallbackMultiplier : 0, payoutMultiplier: fallbackMultiplier, win: fallbackWin, user: { wallets: [{ balance: fallbackBalance }] } } };
            }
        }
        var payload2 = { amount: parseFloat(amount.toFixed(8)), currency: bot.selectedCurrency, target: target, condition: "above", identifier: Math.random().toString(36).slice(2) };
        try {
            var response2 = await fetch(CONFIG.apiUrl + '/casino/dice/roll', {
                method: 'POST', headers: { 'Content-Type': 'application/json', 'x-access-token': accessToken, 'x-lockdown-token': 's5MNWtjTM5TvCMkAzxov' },
                body: JSON.stringify(payload2)
            });
            var result2 = await response2.json();
            if (result2.error) { return null; }
            return result2;
        } catch (error) { return null; }
    }

    function resetProfitStats() {
        var savedWagered = bot.stats.wagered;
        bot.stats = { profit: 0, wagered: savedWagered, startBal: bot.stats.currentBal, currentBal: bot.stats.currentBal, bets: 0, wins: 0, loss: 0, startTime: null, maxDD: 0, maxBet: 0, maxDrawdownPct: 0, peakBalance: bot.stats.currentBal };
        bot.betTimestamps = [];
        bot.startTime = null;
        bot.isHarvesting = false;
        bot.isRecoveryMode = false;
        bot.isTrailingActive = false;
        bot.trailingLevel = 0;
        bot.balancePeak = 0;
        bot.level1Peak = 0;
        bot.level2Peak = 0;
        bot.harvestChance = 0;
        bot.harvestBet = 0;
        bot.harvestCount = 0;
        bot.lastBetAmount = 0;
        bot.sessionProfit = 0;
        bot.totalWager = 0;
        bot.highestProfit = 0;
        bot.maxDrawdownVal = 0;
        bot.balanceStartSnapshot = bot.stats.currentBal;
        bot.gapHits = 0;
        bot.recoveryStep = 0;
        bot.recovery = null;
        bot.burstCounter = 0;
        bot.isBoosterMode = false;
        bot.boosterStartProfit = 0;
        bot.boosterBetAmount = 0;
        bot.boosterChance = bot.normalChance;
        boosterChanceIndex = 0;
        profitHistory = [];
        chartData = [];
        chartHarvestNumber = 0;
        originalSessionStartBalance = bot.stats.currentBal;
        peakBalanceEver = bot.stats.currentBal;
        recoveryStartBalance = 0;
        bot.worstDrawdown = 0;
        bot.worstDrawdownChance = 0;
        bot.worstDrawdownLS = 0;
        bot.worstBet = 0;
        bot.sessionNumber = 1;
        bot.sessionsCompleted = 0;
        bot.sessionStartTime = null;
        harvestingEntries = [];
        initMemory();
        drawGapVisual();
        updateBurstDisplay();
        updateHuntTab();
        updateWagerDisplay();
    }

    function resetVirtualStreaks() {
        for (var i = 0; i < VIRTUAL_CHANCES.length; i++) {
            virtualLs[VIRTUAL_CHANCES[i]] = 0;
        }
        readyChances = [];
    }

    // ==================== RISK PRESET ====================
    function setRiskPreset(preset) {
        var values;
        switch(preset) {
            case 'conservative':
                values = { base: 0.08, fluct: 0.20, min: 0.05, max: 0.15 };
                break;
            case 'normal':
                values = { base: 0.15, fluct: 0.30, min: 0.05, max: 0.35 };
                break;
            case 'aggressive':
                values = { base: 0.25, fluct: 0.40, min: 0.10, max: 0.45 };
                break;
            case 'super':
                values = { base: 0.35, fluct: 0.50, min: 0.15, max: 0.50 };
                break;
            default:
                return;
        }
        
        bot.riskFactor = values.base;
        bot.riskFluctuation = values.fluct;
        bot.minRiskLimit = values.min;
        bot.maxRiskLimit = values.max;
        
        var baseDisplay = document.getElementById('risk-base-display');
        var fluctDisplay = document.getElementById('risk-fluct-display');
        var rangeDisplay = document.getElementById('risk-range-display');
        if (baseDisplay) baseDisplay.textContent = (values.base * 100) + '%';
        if (fluctDisplay) fluctDisplay.textContent = '±' + (values.fluct * 100) + '%';
        if (rangeDisplay) rangeDisplay.textContent = (values.min * 100) + '% - ' + (values.max * 100) + '%';
        
        var customSettings = document.getElementById('risk-custom-settings');
        if (customSettings) customSettings.style.display = 'none';
        
        document.querySelectorAll('.risk-preset').forEach(function(btn) {
            btn.classList.remove('active');
            btn.style.background = 'transparent';
            btn.style.borderColor = '';
        });
        var activeBtn = document.getElementById('risk-' + preset);
        if (activeBtn) {
            activeBtn.classList.add('active');
            activeBtn.style.background = '#9ec07c33';
            activeBtn.style.borderColor = '#9ec07c';
        }
        
        drawGapVisual();
    }

    // ==================== LOG BET ONLY ====================
    function logBet(betNumber, amount, chance, roll, profit) {
        var logBox = document.getElementById("p-logs");
        if (!logBox) return;
        
        var entry = document.createElement("div");
        entry.style.padding = "2px 6px";
        entry.style.fontSize = "10px";
        entry.style.fontFamily = "monospace";
        entry.style.borderBottom = "1px solid " + (bot.darkMode ? "#1a1a2e" : "#f0f0f0");
        entry.style.display = "flex";
        entry.style.justifyContent = "space-between";
        entry.style.gap = "8px";
        entry.style.alignItems = "center";
        
        var numSpan = document.createElement("span");
        numSpan.style.color = "#606060";
        numSpan.style.minWidth = "35px";
        numSpan.style.fontSize = "9px";
        numSpan.textContent = "#" + betNumber;
        
        var amtSpan = document.createElement("span");
        amtSpan.style.color = "#a0a0a0";
        amtSpan.style.minWidth = "65px";
        amtSpan.style.fontSize = "9px";
        amtSpan.textContent = amount;
        
        var chnSpan = document.createElement("span");
        chnSpan.style.color = "#9ec07c";
        chnSpan.style.minWidth = "45px";
        chnSpan.style.fontSize = "9px";
        chnSpan.textContent = chance + "%";
        
        var rollSpan = document.createElement("span");
        rollSpan.style.color = "#a0a0a0";
        rollSpan.style.minWidth = "35px";
        rollSpan.style.fontSize = "9px";
        rollSpan.textContent = roll;
        
        var profitSpan = document.createElement("span");
        profitSpan.style.color = profit.startsWith('+') ? "#00e5c0" : "#ff6b81";
        profitSpan.style.fontWeight = "bold";
        profitSpan.style.minWidth = "65px";
        profitSpan.style.fontSize = "9px";
        profitSpan.style.textAlign = "right";
        profitSpan.textContent = profit;
        
        entry.appendChild(numSpan);
        entry.appendChild(amtSpan);
        entry.appendChild(chnSpan);
        entry.appendChild(rollSpan);
        entry.appendChild(profitSpan);
        
        logBox.insertBefore(entry, logBox.firstChild);
        
        while (logBox.children.length > (bot.maxLogs || 50)) {
            logBox.removeChild(logBox.lastChild);
        }
    }

    // ==================== WAGER FUNCTIONS ====================
    function resetWagerSesi() {
        bot.wagerTotalAccumulated += bot.wagerSesiCounter;
        bot.wagerSesiCounter = 0;
        bot.wagerSesiStartBal = bot.stats.currentBal;
        bot.wagerSesiProfit = 0;
        bot.wagerWaitingForTrigger = false;
        bot.recoverySplitCounter = 0;
        
        bot.stats.profit = 0;
        bot.stats.startBal = bot.stats.currentBal;
        bot.stats.bets = 0;
        bot.stats.wins = 0;
        bot.stats.loss = 0;
        
        bot.isRecoveryMode = false;
        bot.isHarvesting = false;
        bot.recovery = null;
        bot.recoveryStep = 0;
        
        bot.isBoosterMode = false;
        bot.boosterStartProfit = 0;
        bot.boosterBetAmount = 0;
        bot.boosterChance = bot.normalChance;
        boosterChanceIndex = 0;
        
        diceInFlight = 0;
        pendingDiceMeta = [];
        isResting = false;
        normalCycleStartTime = 0;
        
        updateWagerDisplay();
        updateStatsDisplay();
    }
    
    function checkWagerTotalTarget() {
        if (bot.wagerTotalAccumulated >= bot.wagerTotalTarget) {
            bot.isRunning = false;
            bot.wagerStop = true;
            toggleStartStop(false);
            sendTelegramMessage("✅ TOTAL WAGER TARGET REACHED!\nTotal Wager: " + bot.wagerTotalAccumulated.toFixed(2) + "\nTarget: " + bot.wagerTotalTarget);
            return true;
        }
        return false;
    }
    
    function getBurstChance() {
        var min = bot.burstMinChance || 1;
        var max = bot.burstMaxChance || 98;
        return Math.round((Math.random() * (max - min) + min) * 10) / 10;
    }
    
    function getBurstBet(currentBalance) {
        var totalLoss = Math.abs(bot.stats.startBal - bot.stats.currentBal);
        var burstChance = getBurstChance();
        var payoutMult = 99 / burstChance;
        var profitMult = payoutMult - 1;
        var burstBet;
        
        if (profitMult > 0 && totalLoss > 0) {
            burstBet = (totalLoss / profitMult) * bot.burstMultiplier;
        } else {
            burstBet = currentBalance * 0.01;
        }
        
        var maxBurstBet = currentBalance * 0.10;
        burstBet = Math.min(burstBet, maxBurstBet);
        burstBet = Math.max(burstBet, bot.minBet);
        
        if (bot.maxBetCap > 0 && burstBet > bot.maxBetCap) {
            burstBet = bot.maxBetCap;
        }
        
        return burstBet;
    }

    // ==================== BOOSTER MODE ====================
    function checkBoosterCondition() {
        if (bot.isBoosterMode) return true;
        if (!bot.isHarvesting && !bot.isRecoveryMode && bot.stats.profit > bot.wagerLossTolerance) {
            return true;
        }
        return false;
    }

    function startBoosterMode() {
        bot.isBoosterMode = true;
        bot.boosterStartProfit = bot.stats.profit;
        
        var boosterChance = BOOSTER_CHANCES[boosterChanceIndex % BOOSTER_CHANCES.length];
        boosterChanceIndex++;
        
        var betAmount = bot.stats.profit + bot.wagerLossTolerance;
        if (betAmount > bot.stats.currentBal) {
            betAmount = bot.stats.currentBal;
        }
        if (betAmount < bot.minBet) {
            betAmount = bot.minBet;
        }
        if (bot.maxBetCap > 0 && betAmount > bot.maxBetCap) {
            betAmount = bot.maxBetCap;
        }
        
        bot.boosterBetAmount = betAmount;
        bot.boosterChance = boosterChance;
        resetVirtualStreaks();
        
        sendTelegramMessage(`🚀 BOOSTER MODE ACTIVATED\nProfit: ${bot.stats.profit.toFixed(4)}\nLoss Tolerance: ${bot.wagerLossTolerance}\nBet Amount: ${bot.boosterBetAmount.toFixed(4)} (FIXED)\nChance: ${boosterChance}% (Rotation ${(boosterChanceIndex % BOOSTER_CHANCES.length) + 1}/${BOOSTER_CHANCES.length})\nExit when profit ≤ -${bot.wagerLossTolerance}`);
        
        updateStatsDisplay();
        drawGapVisual();
        updateBurstDisplay();
    }

    function exitBoosterMode() {
        bot.isBoosterMode = false;
        bot.boosterStartProfit = 0;
        bot.boosterBetAmount = 0;
        bot.boosterChance = bot.normalChance;
        resetVirtualStreaks();
        
        sendTelegramMessage(`🛑 BOOSTER MODE EXIT\nProfit: ${bot.stats.profit.toFixed(4)} ≤ -Loss Tolerance: -${bot.wagerLossTolerance}\nBalance: ${bot.stats.currentBal.toFixed(4)}\n🔄 Resetting session...`);
        
        resetWagerSesi();
        updateStatsDisplay();
        drawGapVisual();
        updateBurstDisplay();
        updateWagerDisplay();
    }

    // ==================== RUN LOOP ====================
    async function runLoop() {
        if (!bot.isRunning || bot.isSleeping || bot.isPaused) return;
        
        // CEK TRIGGER HARVEST SETIAP SAAT (TANPA MENUNGGU LOSS)
        var triggerCheck = getBestTriggerChance();
        if (triggerCheck !== null && !bot.isHarvesting && !bot.isRecoveryMode && !bot.isBoosterMode) {
            bot.isHarvesting = true;
            bot.harvestChance = triggerCheck;
            bot.harvestStreak = virtualLs[triggerCheck];
            bot.harvestCount++;
            var r = 99.0 / triggerCheck;
            bot.harvestMultiplier = r / (r - 1);
            bot.isRecoveryMode = true;
            bot.recoveryStep = 0;
            bot.isAfterRecovery = false;
            bot.recoverySplitCounter = 0;
            
            var targetBalance = bot.wagerSesiStartBal - bot.wagerLossTolerance;
            var currentSesiLoss = bot.wagerSesiStartBal - bot.stats.currentBal;
            var excessLoss = currentSesiLoss - bot.wagerLossTolerance;
            if (excessLoss < 0) excessLoss = 0;
            
            bot.recovery = {
                active: true,
                currentSplit: 0,
                totalSplits: 3,
                originalSplits: 3,
                lossPerSplit: excessLoss / 3,
                lossStreak: 0,
                startBalance: bot.stats.currentBal,
                targetBalance: targetBalance,
                totalLoss: excessLoss,
                accumulatedLoss: 0,
                totalLossAccumulated: 0,
                betLossInSplit: 0,
                completed: false,
                splitHistory: [],
                cutLossCount: 0
            };
            
            var recoveryBet = computeRecoveryBet(bot.stats.currentBal, triggerCheck, {
                totalLoss: excessLoss,
                totalSplits: 3,
                currentSplit: 0,
                startBalance: bot.stats.currentBal,
                targetStartBalance: targetBalance,
                betLossInSplit: 0
            });
            
            if (recoveryBet === null || recoveryBet <= 0) {
                bot.isRecoveryMode = false;
                bot.harvestBet = computeHarvestBet(bot.stats.currentBal, false);
            } else {
                bot.harvestBet = recoveryBet;
                bot.recoveryStep = 0;
            }
            
            startHarvestTracking(triggerCheck, bot.harvestBet, true);
            sendTelegramMessage(`🌾 HARVEST TRIGGERED #${bot.harvestCount} (READY CHANCE)\nChance: ${triggerCheck}%\nStreak: ${bot.harvestStreak}\nLoss: ${currentSesiLoss.toFixed(4)} / ${bot.wagerLossTolerance}\nExcess Loss: ${excessLoss.toFixed(4)}\nTarget: ${targetBalance.toFixed(4)}`);
            updateWagerDisplay();
        }
        
        if (checkBoosterCondition() && !bot.isBoosterMode && !bot.isHarvesting && !bot.isRecoveryMode) {
            startBoosterMode();
        }
        
        if (shouldStartRest() && !bot.isRecoveryMode && !bot.isHarvesting) {
            startRestPhase();
            setTimeout(runLoop, 200);
            return;
        }
        
        if (isResting) {
            if (shouldEndRest()) {
                endRestPhase();
            }
            setTimeout(runLoop, 200);
            return;
        }
        
        var now = Date.now();
        checkDailyReset();
        updateDailyProfit();
        if (checkDailyLimits()) return;
        if (now - bot.lastTGReport >= TG_CONFIG.reportInterval && bot.stats.bets > 0) {
            bot.lastTGReport = now;
            await sendSessionReport();
        }
        
        if (checkWagerTotalTarget()) return;
        
        try {
            var currentBalance = bot.stats.currentBal;
            var profitSoFar = bot.stats.profit;
            var overallStart = bot.stats.startBal > 0 ? bot.stats.startBal : (bot.isDemo ? bot.dirtyBalance : 100);
            
            if (bot.isTrailingActive) updateTrailing();
            var profitPct = (profitSoFar / overallStart) * 100;
            if (!bot.isTrailingActive && profitPct >= bot.trailingActivationPct && !bot.isHarvesting) activateTrailing();
            
            if (currentBalance > bot.stats.peakBalance) {
                bot.stats.peakBalance = currentBalance;
                if (bot.isTrailingActive) bot.balancePeak = currentBalance;
            }
            if (currentBalance > peakBalanceEver) peakBalanceEver = currentBalance;
            if (bot.stats.peakBalance > 0) {
                var dd = (bot.stats.peakBalance - currentBalance) / bot.stats.peakBalance * 100;
                if (dd > bot.stats.maxDrawdownPct) bot.stats.maxDrawdownPct = dd;
            }
            
            if (currentBalance <= 0) {
                bot.isRunning = false;
                toggleStartStop(false);
                sendTelegramMessage("🛑 BALANCE ZERO! Bot stopped.");
                return;
            }
            
            var normalBet = computeNormalBet(currentBalance);
            var nextBet, chanceToUse, betMode = "normal";
            
            if (!bot.isHarvesting && !bot.isRecoveryMode && !bot.isBoosterMode) {
                var triggerCheck2 = getBestTriggerChance();
                if (triggerCheck2 !== null) {
                    bot.isHarvesting = true;
                    bot.harvestChance = triggerCheck2;
                    bot.harvestStreak = virtualLs[triggerCheck2];
                    bot.harvestCount++;
                    var r2 = 99.0 / triggerCheck2;
                    bot.harvestMultiplier = r2 / (r2 - 1);
                    bot.isRecoveryMode = true;
                    bot.recoveryStep = 0;
                    bot.isAfterRecovery = false;
                    bot.recoverySplitCounter = 0;
                    
                    var targetBalance2 = bot.wagerSesiStartBal - bot.wagerLossTolerance;
                    var currentSesiLoss2 = bot.wagerSesiStartBal - bot.stats.currentBal;
                    var excessLoss2 = currentSesiLoss2 - bot.wagerLossTolerance;
                    if (excessLoss2 < 0) excessLoss2 = 0;
                    
                    bot.recovery = {
                        active: true,
                        currentSplit: 0,
                        totalSplits: 3,
                        originalSplits: 3,
                        lossPerSplit: excessLoss2 / 3,
                        lossStreak: 0,
                        startBalance: bot.stats.currentBal,
                        targetBalance: targetBalance2,
                        totalLoss: excessLoss2,
                        accumulatedLoss: 0,
                        totalLossAccumulated: 0,
                        betLossInSplit: 0,
                        completed: false,
                        splitHistory: [],
                        cutLossCount: 0
                    };
                    
                    var recoveryBet2 = computeRecoveryBet(bot.stats.currentBal, triggerCheck2, {
                        totalLoss: excessLoss2,
                        totalSplits: 3,
                        currentSplit: 0,
                        startBalance: bot.stats.currentBal,
                        targetStartBalance: targetBalance2,
                        betLossInSplit: 0
                    });
                    
                    if (recoveryBet2 === null || recoveryBet2 <= 0) {
                        bot.isRecoveryMode = false;
                        bot.harvestBet = computeHarvestBet(bot.stats.currentBal, false);
                    } else {
                        bot.harvestBet = recoveryBet2;
                        bot.recoveryStep = 0;
                    }
                    
                    startHarvestTracking(triggerCheck2, bot.harvestBet, true);
                    sendTelegramMessage(`🌾 HARVEST TRIGGERED #${bot.harvestCount} (READY CHANCE)\nChance: ${triggerCheck2}%\nStreak: ${bot.harvestStreak}\nExcess Loss: ${excessLoss2.toFixed(4)}\nTarget: ${targetBalance2.toFixed(4)}`);
                    updateWagerDisplay();
                }
            }
            
            if (bot.isHarvesting) {
                chanceToUse = bot.harvestChance;
                nextBet = bot.harvestBet;
                if (bot.isRecoveryMode && nextBet > currentBalance) nextBet = currentBalance;
                betMode = bot.isRecoveryMode ? "recovery" : "harvest";
            } else if (bot.isBoosterMode) {
                chanceToUse = bot.boosterChance || bot.normalChance;
                nextBet = bot.boosterBetAmount;
                betMode = "booster";
            } else {
                var isBurstBet = false;
                if (bot.wagerJitEnabled && bot.burstEveryN > 0 && !bot.isHarvesting && !bot.isRecoveryMode && !bot.isBoosterMode) {
                    bot.burstCounter++;
                    if (bot.burstCounter >= bot.burstEveryN) {
                        isBurstBet = true;
                        bot.burstCounter = 0;
                    }
                }
                
                if (isBurstBet) {
                    chanceToUse = getBurstChance();
                    nextBet = getBurstBet(currentBalance);
                    betMode = "jit";
                } else {
                    chanceToUse = bot.normalChance;
                    nextBet = normalBet;
                    betMode = "normal";
                }
                updateBurstDisplay();
            }
            
            if (nextBet > bot.worstBet) bot.worstBet = nextBet;
            if (bot.maxBetCap > 0 && nextBet > bot.maxBetCap) nextBet = bot.maxBetCap;
            nextBet = Math.max(nextBet, bot.minBet);
            if (nextBet > currentBalance) nextBet = currentBalance;
            if (nextBet < 0.00000001) nextBet = 0.00000001;
            
            // BOOSTER MODE PAKAI SEQUENTIAL
            if (betMode === "booster") {
                var boosterResult = await placeBet(nextBet, chanceToUse);
                
                if (boosterResult && boosterResult.diceRoll) {
                    var d = boosterResult.diceRoll;
                    var win = false;
                    var winAmount = parseFloat(d.amount) || 0;
                    var payoutMultiplier = parseFloat(d.payoutMultiplier) || 0;
                    var payout = parseFloat(d.payout) || 0;
                    
                    if (d.win !== undefined) win = d.win;
                    else if (payoutMultiplier > 1) win = true;
                    else if (payoutMultiplier === 0 && payout === 0) win = false;
                    else {
                        var state = d.state;
                        if (state && state.result !== undefined && state.target !== undefined) {
                            win = state.condition === "above" ? state.result > state.target : state.result < state.target;
                        }
                    }
                    
                    var oldBalance = bot.stats.currentBal;
                    var pnlCalc = win ? (payout - winAmount) : -winAmount;
                    
                    if (d.user && d.user.wallets && d.user.wallets[0]) {
                        bot.stats.currentBal = parseFloat(d.user.wallets[0].balance);
                    } else {
                        bot.stats.currentBal = oldBalance + pnlCalc;
                        if (bot.stats.currentBal < 0) bot.stats.currentBal = 0;
                    }
                    
                    if (bot.stats.currentBal > peakBalanceEver) peakBalanceEver = bot.stats.currentBal;
                    bot.stats.profit = bot.stats.currentBal - bot.stats.startBal;
                    bot.wagerSesiProfit = bot.stats.currentBal - bot.wagerSesiStartBal;
                    bot.wagerSesiCounter += winAmount;
                    
                    if (bot.stats.profit > bot.highestProfit) bot.highestProfit = bot.stats.profit;
                    if (winAmount > bot.stats.maxBet) bot.stats.maxBet = winAmount;
                    
                    bot.stats.bets++;
                    bot.stats.wagered += winAmount;
                    bot.lastWinStatus = win;
                    bot.lastBetAmount = nextBet;
                    bot.betTimestamps.push(Date.now());
                    if (bot.betTimestamps.length > 100) bot.betTimestamps.shift();
                    
                    if (win) bot.stats.wins++;
                    else bot.stats.loss++;
                    
                    bot.totalWager += winAmount;
                    bot.sessionProfit = bot.stats.profit;
                    if (bot.sessionProfit > bot.highestProfit) bot.highestProfit = bot.sessionProfit;
                    bot.maxDrawdownVal = Math.max(bot.maxDrawdownVal, bot.highestProfit - bot.sessionProfit);
                    
                    var rollResult = null;
                    if (d.state && d.state.result !== undefined) {
                        rollResult = d.state.result;
                        updateVirtualStreaks(rollResult);
                    }
                    
                    var profitDisplay = win ? "+" + (payout - winAmount).toFixed(8) : "-" + winAmount.toFixed(8);
                    var rollDisplay = (rollResult !== null) ? rollResult.toFixed(2) : "?";
                    logBet(bot.stats.bets, nextBet.toFixed(4), chanceToUse.toFixed(1), rollDisplay, profitDisplay);
                    
                    if (bot.stats.profit <= -bot.wagerLossTolerance) {
                        exitBoosterMode();
                    }
                }
                
                updateStatsDisplay();
                updateWagerDisplay();
                
                setTimeout(runLoop, bot.delayHarvest);
                return;
            }
            
            // NORMAL & JIT PAKAI FIRE-AND-FORGET
            if (betMode === "normal" || betMode === "jit") {
                if (diceInFlight < wagerParallelCap) {
                    sendDiceBetFireAndForget(nextBet, chanceToUse, betMode);
                }
                
                if (normalCycleStartTime === 0) {
                    normalCycleStartTime = Date.now();
                }
                
                setTimeout(runLoop, bot.delayNormal);
                return;
            }
            
            // HARVEST & RECOVERY PAKAI SEQUENTIAL
            if (betMode === "recovery" || betMode === "harvest") {
                var placeResult = await placeBet(nextBet, chanceToUse);
                
                if (placeResult && placeResult.diceRoll) {
                    var d = placeResult.diceRoll;
                    var win = false;
                    var winAmount = parseFloat(d.amount) || 0;
                    var payoutMultiplier = parseFloat(d.payoutMultiplier) || 0;
                    var payout = parseFloat(d.payout) || 0;
                    
                    if (d.win !== undefined) win = d.win;
                    else if (payoutMultiplier > 1) win = true;
                    else if (payoutMultiplier === 0 && payout === 0) win = false;
                    else {
                        var state = d.state;
                        if (state && state.result !== undefined && state.target !== undefined) {
                            win = state.condition === "above" ? state.result > state.target : state.result < state.target;
                        }
                    }
                    
                    var oldBalance = bot.stats.currentBal;
                    var pnlCalc = win ? (payout - winAmount) : -winAmount;
                    
                    if (!bot.isDemo && bot.isHarvesting) {
                        var newBalance = await fetchCurrentBalance();
                        if (newBalance !== oldBalance && newBalance > 0) {
                            bot.stats.currentBal = newBalance;
                        } else if (d.user && d.user.wallets && d.user.wallets[0]) {
                            bot.stats.currentBal = parseFloat(d.user.wallets[0].balance);
                        } else {
                            bot.stats.currentBal = oldBalance + pnlCalc;
                            if (bot.stats.currentBal < 0) bot.stats.currentBal = 0;
                        }
                    } else if (bot.isDemo) {
                        if (d.user && d.user.wallets && d.user.wallets[0]) {
                            bot.stats.currentBal = parseFloat(d.user.wallets[0].balance);
                        } else {
                            bot.stats.currentBal = oldBalance + pnlCalc;
                            if (bot.stats.currentBal < 0) bot.stats.currentBal = 0;
                        }
                    } else {
                        if (d.user && d.user.wallets && d.user.wallets[0]) {
                            bot.stats.currentBal = parseFloat(d.user.wallets[0].balance);
                        } else {
                            bot.stats.currentBal = oldBalance + pnlCalc;
                            if (bot.stats.currentBal < 0) bot.stats.currentBal = 0;
                        }
                    }
                    
                    if (bot.stats.currentBal > peakBalanceEver) peakBalanceEver = bot.stats.currentBal;
                    bot.stats.profit = bot.stats.currentBal - bot.stats.startBal;
                    bot.wagerSesiProfit = bot.stats.currentBal - bot.wagerSesiStartBal;
                    bot.wagerSesiCounter += winAmount;
                    
                    if (bot.stats.profit > bot.highestProfit) bot.highestProfit = bot.stats.profit;
                    if (winAmount > bot.stats.maxBet) bot.stats.maxBet = winAmount;
                    
                    bot.stats.bets++;
                    bot.stats.wagered += winAmount;
                    bot.lastWinStatus = win;
                    bot.lastBetAmount = nextBet;
                    bot.betTimestamps.push(Date.now());
                    if (bot.betTimestamps.length > 100) bot.betTimestamps.shift();
                    
                    if (win) bot.stats.wins++;
                    else bot.stats.loss++;
                    
                    bot.totalWager += winAmount;
                    bot.sessionProfit = bot.stats.profit;
                    if (bot.sessionProfit > bot.highestProfit) bot.highestProfit = bot.sessionProfit;
                    bot.maxDrawdownVal = Math.max(bot.maxDrawdownVal, bot.highestProfit - bot.sessionProfit);
                    
                    var rollResult = null;
                    if (d.state && d.state.result !== undefined) {
                        rollResult = d.state.result;
                        updateVirtualStreaks(rollResult);
                    }
                    
                    if (bot.isHarvesting) updateHarvestTracking();
                    
                    var profitDisplay = win ? "+" + (payout - winAmount).toFixed(8) : "-" + winAmount.toFixed(8);
                    var rollDisplay = (rollResult !== null) ? rollResult.toFixed(2) : "?";
                    logBet(bot.stats.bets, nextBet.toFixed(4), chanceToUse.toFixed(1), rollDisplay, profitDisplay);
                    
                    if (bot.isRecoveryMode && bot.recovery && bot.recovery.active) {
                        var recoveryTarget3 = bot.wagerSesiStartBal - bot.wagerLossTolerance;
                        
                        if (bot.stats.currentBal >= recoveryTarget3) {
                            var totalProfit2 = bot.stats.currentBal - bot.harvestStartBalance;
                            endHarvestAndAddToHistory('WIN', totalProfit2, true, false, bot._chartStartProfit, bot._chartHigh, bot._chartLow, bot._chartChance);
                            bot.isHarvesting = false;
                            bot.isRecoveryMode = false;
                            bot.recovery = null;
                            bot.recoveryStep = 0;
                            bot.isAfterRecovery = true;
                            bot.wagerWaitingForTrigger = false;
                            bot.recoverySplitCounter = 0;
                            resetVirtualStreaks();
                            if (bot.isTrailingActive) resetTrailingAfterWin();
                            sendTelegramMessage(`✅ RECOVERY SUCCESS #${bot.harvestCount}\nBalance: ${bot.stats.currentBal.toFixed(8)}\nTarget: ${recoveryTarget3.toFixed(8)}`);
                            resetWagerSesi();
                        } else if (win) {
                            var recovery = bot.recovery;
                            recovery.betLossInSplit = 0;
                            recovery.accumulatedLoss = 0;
                            recovery.currentSplit++;
                            recovery.lossStreak = 0;
                            
                            if (recovery.currentSplit >= bot.recoveryMaxSplits) {
                                recovery.currentSplit = 0;
                                bot.recoverySplitCounter = 0;
                                sendTelegramMessage(`🔄 RECOVERY CYCLE RESET (Split ${bot.recoveryMaxSplits} reached, restarting from Split 1)`);
                            }
                            
                            if (bot.stats.currentBal >= recoveryTarget3) {
                                var totalProfit3 = bot.stats.currentBal - bot.harvestStartBalance;
                                endHarvestAndAddToHistory('WIN', totalProfit3, true, false, bot._chartStartProfit, bot._chartHigh, bot._chartLow, bot._chartChance);
                                bot.isHarvesting = false;
                                bot.isRecoveryMode = false;
                                bot.recovery = null;
                                bot.recoveryStep = 0;
                                bot.isAfterRecovery = true;
                                bot.wagerWaitingForTrigger = false;
                                bot.recoverySplitCounter = 0;
                                resetVirtualStreaks();
                                if (bot.isTrailingActive) resetTrailingAfterWin();
                                sendTelegramMessage(`✅ RECOVERY SUCCESS #${bot.harvestCount}\nBalance: ${bot.stats.currentBal.toFixed(8)}`);
                                resetWagerSesi();
                            } else {
                                var nextSplitNumber = recovery.currentSplit + 1;
                                var nextChance = getHuntChanceForSplit(nextSplitNumber, bot.harvestChance);
                                var currentLoss = bot.wagerSesiStartBal - bot.stats.currentBal;
                                var excessLoss3 = currentLoss - bot.wagerLossTolerance;
                                if (excessLoss3 < 0) excessLoss3 = 0;
                                
                                var remainingSplits = bot.recoveryMaxSplits - recovery.currentSplit;
                                if (remainingSplits <= 0) remainingSplits = 1;
                                
                                var nextBet2 = computeRecoveryBet(bot.stats.currentBal, nextChance, {
                                    totalLoss: excessLoss3,
                                    totalSplits: remainingSplits,
                                    currentSplit: 0,
                                    startBalance: bot.stats.currentBal,
                                    targetStartBalance: recoveryTarget3,
                                    betLossInSplit: 0
                                });
                                
                                if (nextBet2 > 0) {
                                    bot.harvestChance = nextChance;
                                    bot.harvestBet = nextBet2;
                                    sendTelegramMessage(`🔄 RECOVERY SPLIT ${nextSplitNumber}/${bot.recoveryMaxSplits}\nChance: ${nextChance}%\nBet: ${nextBet2.toFixed(8)}`);
                                }
                            }
                        } else {
                            var recovery2 = bot.recovery;
                            var betAmount = bot.lastBetAmount;
                            
                            recovery2.lossStreak++;
                            recovery2.betLossInSplit += betAmount;
                            recovery2.accumulatedLoss += betAmount;
                            recovery2.totalLossAccumulated += betAmount;
                            
                            var cutLimit = getCutLossLimit(bot.harvestChance, bot.recoveryCutlossMultiplier);
                            
                            if (recovery2.lossStreak >= cutLimit) {
                                recovery2.cutLossCount++;
                                recovery2.currentSplit++;
                                recovery2.lossStreak = 0;
                                recovery2.betLossInSplit = 0;
                                
                                if (recovery2.currentSplit >= bot.recoveryMaxSplits) {
                                    recovery2.currentSplit = 0;
                                    bot.recoverySplitCounter = 0;
                                    sendTelegramMessage(`🔄 RECOVERY CYCLE RESET (Split ${bot.recoveryMaxSplits} reached, restarting from Split 1)`);
                                }
                                
                                if (bot.stats.currentBal >= recoveryTarget3) {
                                    var totalProfit4 = bot.stats.currentBal - bot.harvestStartBalance;
                                    endHarvestAndAddToHistory('WIN', totalProfit4, true, false, bot._chartStartProfit, bot._chartHigh, bot._chartLow, bot._chartChance);
                                    bot.isHarvesting = false;
                                    bot.isRecoveryMode = false;
                                    bot.recovery = null;
                                    bot.recoveryStep = 0;
                                    bot.isAfterRecovery = true;
                                    bot.wagerWaitingForTrigger = false;
                                    bot.recoverySplitCounter = 0;
                                    resetVirtualStreaks();
                                    if (bot.isTrailingActive) resetTrailingAfterWin();
                                    sendTelegramMessage(`✅ RECOVERY SUCCESS #${bot.harvestCount}\nBalance: ${bot.stats.currentBal.toFixed(8)}`);
                                    resetWagerSesi();
                                } else {
                                    var nextSplitNumber2 = recovery2.currentSplit + 1;
                                    var nextChance2 = getHuntChanceForSplit(nextSplitNumber2, bot.harvestChance);
                                    var currentLoss2 = bot.wagerSesiStartBal - bot.stats.currentBal;
                                    var excessLoss4 = currentLoss2 - bot.wagerLossTolerance;
                                    if (excessLoss4 < 0) excessLoss4 = 0;
                                    
                                    var remainingSplits2 = bot.recoveryMaxSplits - recovery2.currentSplit;
                                    if (remainingSplits2 <= 0) remainingSplits2 = 1;
                                    
                                    var nextBet3 = computeRecoveryBet(bot.stats.currentBal, nextChance2, {
                                        totalLoss: excessLoss4,
                                        totalSplits: remainingSplits2,
                                        currentSplit: 0,
                                        startBalance: bot.stats.currentBal,
                                        targetStartBalance: recoveryTarget3,
                                        betLossInSplit: 0
                                    });
                                    
                                    if (nextBet3 > 0) {
                                        bot.harvestChance = nextChance2;
                                        bot.harvestBet = nextBet3;
                                        sendTelegramMessage(`🔄 RECOVERY SPLIT ${nextSplitNumber2}/${bot.recoveryMaxSplits} (after cutloss)\nChance: ${nextChance2}%\nBet: ${nextBet3.toFixed(8)}`);
                                    }
                                }
                            } else {
                                var currentLoss3 = bot.wagerSesiStartBal - bot.stats.currentBal;
                                var excessLoss5 = currentLoss3 - bot.wagerLossTolerance;
                                if (excessLoss5 < 0) excessLoss5 = 0;
                                
                                var remainingSplits3 = bot.recoveryMaxSplits - recovery2.currentSplit;
                                if (remainingSplits3 <= 0) remainingSplits3 = 1;
                                
                                var nextBet4 = computeRecoveryBet(bot.stats.currentBal, bot.harvestChance, {
                                    totalLoss: excessLoss5,
                                    totalSplits: remainingSplits3,
                                    currentSplit: 0,
                                    startBalance: bot.stats.currentBal,
                                    targetStartBalance: recoveryTarget3,
                                    betLossInSplit: recovery2.betLossInSplit
                                });
                                
                                if (nextBet4 > 0) {
                                    bot.harvestBet = nextBet4;
                                }
                            }
                        }
                    } else if (bot.isHarvesting) {
                        if (win) {
                            var totalProfit5 = bot.stats.currentBal - bot.harvestStartBalance;
                            endHarvestAndAddToHistory('WIN', totalProfit5, true, false, bot._chartStartProfit, bot._chartHigh, bot._chartLow, bot._chartChance);
                            bot.isHarvesting = false;
                            bot.isAfterRecovery = true;
                            resetVirtualStreaks();
                            if (bot.isTrailingActive) resetTrailingAfterWin();
                            sendTelegramMessage(`✅ HARVEST WIN #${bot.harvestCount}\nProfit: ${totalProfit5.toFixed(8)}\nBalance: ${bot.stats.currentBal.toFixed(8)}`);
                        } else {
                            bot.harvestBet = bot.harvestBet * bot.harvestMultiplier;
                            if (bot.maxBetCap > 0 && bot.harvestBet > bot.maxBetCap) bot.harvestBet = bot.maxBetCap;
                            
                            var batasMultiplier3 = bot.maxCutLossMultiplier || 5;
                            var cutLimit3 = getCutLossLimit(bot.harvestChance, batasMultiplier3);
                            var currentLS = virtualLs[bot.harvestChance];
                            
                            if (currentLS >= cutLimit3) {
                                var totalLoss3 = bot.stats.currentBal - bot.harvestStartBalance;
                                endHarvestAndAddToHistory('CUT LOSS', totalLoss3, false, true, bot._chartStartProfit, bot._chartHigh, bot._chartLow, bot._chartChance);
                                bot.isHarvesting = false;
                                bot.isRecoveryMode = false;
                                bot.recoveryStep = 0;
                                bot.recovery = null;
                                bot.isAfterRecovery = false;
                            }
                        }
                    }
                    
                    if (!bot.isHarvesting && !bot.isRecoveryMode) {
                        if (bot.wagerSesiCounter >= bot.wagerSesiTarget) {
                            resetWagerSesi();
                        }
                    }
                }
                
                updateWagerDisplay();
                updateStatsDisplay();
                
                var currentDelay = bot.isRecoveryMode ? bot.delayRecovery : bot.delayHarvest;
                setTimeout(runLoop, currentDelay);
                return;
            }
            
            setTimeout(runLoop, 10);
            
        } catch (e) {
            setTimeout(runLoop, 1000);
        }
    }

    // ==================== UPDATE STATS ====================
    function updateStatsDisplay() {
        var profitPct = bot.stats.startBal > 0 ? (bot.stats.profit / bot.stats.startBal * 100) : 0;
        var dailyProfitPct = bot.dailyStartBalance > 0 ? (bot.dailyProfit / bot.dailyStartBalance * 100) : 0;
        var winRate = bot.stats.bets > 0 ? (bot.stats.wins / bot.stats.bets * 100) : 0;
        var runTime = getDuration();
        var rakeback = bot.stats.wagered * 0.00098;
        var betSpeed = getBetSpeed();
        var profitInUsdt = bot.stats.profit * bot.fiatRates.usdt;
        var profitInIdr = bot.stats.profit * bot.fiatRates.idr;
        var wageredInUsdt = bot.stats.wagered * bot.fiatRates.usdt;
        var wageredInIdr = bot.stats.wagered * bot.fiatRates.idr;
        var rakebackInUsdt = rakeback * bot.fiatRates.usdt;
        var rakebackInIdr = rakeback * bot.fiatRates.idr;
        var balanceInUsdt = bot.stats.currentBal * bot.fiatRates.usdt;
        var balanceInIdr = bot.stats.currentBal * bot.fiatRates.idr;
        var maxDDPct = bot.stats.maxDrawdownPct;
        var maxBetPct = bot.stats.startBal > 0 ? (bot.stats.maxBet / bot.stats.startBal * 100) : 0;
        
        var status;
        if (bot.isBoosterMode) {
            status = "🚀 BOOSTER " + (bot.boosterChance || bot.normalChance) + "%";
        } else if (bot.isHarvesting) {
            status = bot.isRecoveryMode ? "🔄 R" + bot.harvestChance + "%" : "🌾 H" + bot.harvestChance + "%";
        } else if (bot.isTrailingActive) {
            status = "🔱 T" + bot.trailingLevel + "/" + bot.gapPct + "%";
        } else {
            status = "⚡ " + bot.normalChance + "%";
        }
        
        var dailyStatus = (bot.dailyProfit >= 0 ? '+' : '') + bot.dailyProfit.toFixed(8) + " (" + (dailyProfitPct >= 0 ? '+' : '') + dailyProfitPct.toFixed(2) + "%)";
        var timerDisplay = bot.isResetting ? "⏰ " + bot.resetCountdownRemaining + "s" : "--";
        var sessionProgress = ((bot.stats.profit / (bot.stats.startBal * bot.targetPct / 100)) * 100);
        if (sessionProgress > 100) sessionProgress = 100;
        if (sessionProgress < 0) sessionProgress = 0;
        var sessionTargetAmount = (bot.stats.startBal * bot.targetPct / 100);
        
        var worstDisplay = (bot.worstDrawdownChance > 0) ? bot.worstDrawdownChance + "%" : "-";
        var worstLsDisplay = (bot.worstDrawdownLS > 0) ? bot.worstDrawdownLS + "x" : "-";
        
        var splitDisplay = "—";
        var cutlossDisplay = "0";
        var lossPerSplitDisplay = "0";
        if (bot.recovery && bot.recovery.active) {
            splitDisplay = (bot.recovery.currentSplit + 1) + "/" + bot.recovery.totalSplits;
            cutlossDisplay = bot.recovery.cutLossCount || 0;
            lossPerSplitDisplay = bot.recovery.lossPerSplit ? bot.recovery.lossPerSplit.toFixed(8) : "0";
        }
        
        var boosterInfo = bot.isBoosterMode ? '🚀 ' + bot.boosterBetAmount.toFixed(4) + ' (C:' + (bot.boosterChance || bot.normalChance) + '% P:' + bot.boosterStartProfit.toFixed(4) + ' → ' + bot.stats.profit.toFixed(4) + ')' : 'INACTIVE';
        
        var elements = {
            "st-user": bot.stakeUser,
            "st-running-time": runTime,
            "st-balance": bot.stats.currentBal.toFixed(2) + " " + bot.selectedCurrency.toUpperCase(),
            "st-balance-fiat": shortFiat(balanceInIdr, balanceInUsdt),
            "st-session-num": bot.sessionNumber,
            "st-session-target": bot.targetPct + "% (" + sessionTargetAmount.toFixed(2) + " " + bot.selectedCurrency.toUpperCase() + ")",
            "st-session-progress": sessionProgress.toFixed(1) + "%",
            "st-session-start": bot.sessionStartTime ? bot.sessionStartTime.toLocaleTimeString() : "-",
            "st-sessions-done": bot.sessionsCompleted,
            "st-profit": (bot.stats.profit >= 0 ? '+' : '') + bot.stats.profit.toFixed(2) + " (" + (profitPct >= 0 ? '+' : '') + profitPct.toFixed(2) + "%)",
            "st-profit-fiat": shortFiat(profitInIdr, profitInUsdt),
            "st-daily": dailyStatus,
            "st-wagered": bot.stats.wagered.toFixed(2),
            "st-wagered-fiat": shortFiat(wageredInIdr, wageredInUsdt),
            "st-rakeback": "+" + rakeback.toFixed(2),
            "st-rakeback-fiat": shortFiat(rakebackInIdr, rakebackInUsdt),
            "st-bets": bot.stats.bets + " | WR " + winRate.toFixed(0) + "%",
            "st-speed": betSpeed.toFixed(1) + "b/s",
            "st-maxdd": maxDDPct.toFixed(1) + "%",
            "st-maxbet": maxBetPct.toFixed(1) + "%",
            "st-chance": status,
            "st-gaphits": bot.gapHits.toString(),
            "st-timer": timerDisplay,
            "st-rate": "1 " + bot.selectedCurrency.toUpperCase() + " = " + bot.fiatRates.idr.toFixed(2) + " IDR",
            "st-worst-chance": worstDisplay,
            "st-worst-ls": worstLsDisplay,
            "st-worst-dd": bot.worstDrawdown > 0 ? bot.worstDrawdown.toFixed(1) + "%" : "-",
            "st-worst-bet": bot.worstBet > 0 ? bot.worstBet.toFixed(2) + " " + bot.selectedCurrency.toUpperCase() : "-",
            "st-split-progress": splitDisplay,
            "st-cutloss-count": cutlossDisplay,
            "st-loss-per-split": lossPerSplitDisplay
        };
        
        for (var id in elements) {
            var el = document.getElementById(id);
            if (el) el.textContent = elements[id];
        }
        
        var sessionProgressBar = document.getElementById("st-session-progress-bar");
        if (sessionProgressBar) {
            sessionProgressBar.style.width = sessionProgress + "%";
        }
        
        var profitEl = document.getElementById("st-profit");
        if (profitEl) profitEl.style.color = bot.stats.profit >= 0 ? "#00A86B" : "#FF4757";
        var profitFiatEl = document.getElementById("st-profit-fiat");
        if (profitFiatEl) profitFiatEl.style.color = bot.stats.profit >= 0 ? "#00A86B" : "#FF4757";
        var dailyEl = document.getElementById("st-daily");
        if (dailyEl) dailyEl.style.color = bot.dailyProfit >= 0 ? "#00A86B" : "#FF4757";
        
        drawGapVisual();
        updateBurstDisplay();
        updateHuntTab();
        updateWagerDisplay();
    }
    
    function shortFiat(idr, usdt) {
        var abs = Math.abs(idr);
        var sign = idr < 0 ? "-" : "";
        if (abs >= 1000000) return sign + "Rp" + (abs/1000000).toFixed(1) + "M";
        if (abs >= 1000) return sign + "Rp" + (abs/1000).toFixed(1) + "K";
        return sign + "Rp" + abs.toFixed(0);
    }
    
    function resetSession() {
        if (bot.isRunning) { return; }
        fullReset();
    }

    function toggleMinimize() {
        bot.minimized = !bot.minimized;
        var body = document.getElementById("bot-body");
        var btn = document.getElementById("btn-minimize");
        if (bot.minimized) { if (body) body.style.display = "none"; if (btn) btn.textContent = "▼"; }
        else { if (body) body.style.display = "block"; if (btn) btn.textContent = "▲"; }
    }
    
    function toggleDarkMode() {
        bot.darkMode = !bot.darkMode;
        var root = document.getElementById("BotConsole");
        if (!root) return;
        if (bot.darkMode) {
            root.style.setProperty('--bg', '#1a1a2e');
            root.style.setProperty('--bg-block', '#16213e');
            root.style.setProperty('--text', '#e0e0e0');
            root.style.setProperty('--text-light', '#a0a0a0');
            root.style.setProperty('--border', '#2a2a4a');
            root.style.setProperty('--log-bg', '#0f0f1a');
        } else {
            root.style.setProperty('--bg', '#ffffff');
            root.style.setProperty('--bg-block', '#f8f9fa');
            root.style.setProperty('--text', '#1a1a1a');
            root.style.setProperty('--text-light', '#6b7280');
            root.style.setProperty('--border', '#e5e7eb');
            root.style.setProperty('--log-bg', '#f8f9fa');
        }
        drawGapVisual();
        renderHarvestingEntries();
        updateHuntTab();
        updateWagerDisplay();
    }

    var pos1 = 0, pos2 = 0, pos3 = 0, pos4 = 0;
    function dragMouseDown(e) {
        if (e.target.closest && (e.target.closest('button') || e.target.closest('input') || e.target.closest('select'))) return;
        e = e || window.event;
        e.preventDefault();
        pos3 = e.clientX;
        pos4 = e.clientY;
        document.onmouseup = closeDragElement;
        document.onmousemove = elementDrag;
    }
    function elementDrag(e) {
        var elmnt = document.getElementById("BotConsole");
        e = e || window.event;
        e.preventDefault();
        pos1 = pos3 - e.clientX;
        pos2 = pos4 - e.clientY;
        pos3 = e.clientX;
        pos4 = e.clientY;
        var newTop = elmnt.offsetTop - pos2;
        var newLeft = elmnt.offsetLeft - pos1;
        var maxLeft = window.innerWidth - elmnt.offsetWidth;
        var maxTop = window.innerHeight - elmnt.offsetHeight;
        if (newLeft < 0) newLeft = 0;
        if (newTop < 0) newTop = 0;
        if (newLeft > maxLeft) newLeft = maxLeft;
        if (newTop > maxTop) newTop = maxTop;
        elmnt.style.top = newTop + "px";
        elmnt.style.left = newLeft + "px";
        elmnt.style.right = "auto";
    }
    function closeDragElement() { document.onmouseup = null; document.onmousemove = null; }

    window.openTab = function(tabName) {
        var tabs = document.querySelectorAll('.tab-content');
        for (var i = 0; i < tabs.length; i++) tabs[i].classList.remove('active');
        var target = document.getElementById('tab-' + tabName);
        if (target) target.classList.add('active');
        var btns = document.querySelectorAll('.tab-btn');
        for (var j = 0; j < btns.length; j++) btns[j].classList.remove('active');
        if (event && event.currentTarget) event.currentTarget.classList.add('active');
        if (tabName === 'history') {
            setTimeout(function() { renderHarvestingEntries(); }, 200);
        }
        if (tabName === 'hunt') {
            setTimeout(function() { updateHuntTab(); }, 100);
        }
    };

    // ==================== SAFE GET ELEMENT VALUE ====================
    function safeGetValue(id, defaultValue) {
        var el = document.getElementById(id);
        if (el) return el.value;
        return defaultValue;
    }
    
    function safeGetInt(id, defaultValue) {
        var val = parseInt(safeGetValue(id, defaultValue));
        return isNaN(val) ? defaultValue : val;
    }
    
    function safeGetFloat(id, defaultValue) {
        var val = parseFloat(safeGetValue(id, defaultValue));
        return isNaN(val) ? defaultValue : val;
    }

    // ==================== CREATE UI ====================
    function createUI() {
        if (document.getElementById("BotConsole")) document.getElementById("BotConsole").remove();
        
        var style = document.createElement("style");
        style.innerHTML = `
            :root { --bg: #1e1f1c; --bg-block: #272822; --text: #f8f8f2; --text-light: #a0a0a0; --border: #3a3a3a; --log-bg: #0f0f0a; --accent: #9ec07c; --success: #00e5c0; --danger: #ff6b81; --recovery: #ff922b; }
            #BotConsole { position: fixed; top: 10px; left: 10px; right: 10px; width: auto; max-width: 580px; margin: 0 auto; background: var(--bg); border-radius: 16px; font-family: 'Segoe UI', Consolas, monospace; font-size: 12px; box-shadow: 0 8px 32px rgba(0,0,0,0.4); border: 1px solid var(--border); z-index: 99999; overflow: hidden; color: var(--text); }
            @media (min-width: 768px) { #BotConsole { top: 20px; right: 20px; left: auto; width: 580px; } }
            .bot-header { background: var(--accent); padding: 8px 12px; cursor: move; display: flex; justify-content: space-between; align-items: center; color: black; touch-action: none; }
            .bot-header .title { font-weight: 700; font-size: 13px; }
            .header-btns { display: flex; gap: 8px; }
            .header-btns button { background: rgba(0,0,0,0.2); border: none; color: black; width: 30px; height: 30px; border-radius: 8px; cursor: pointer; font-size: 14px; }
            .time-running { background: rgba(0,0,0,0.2); padding: 3px 10px; border-radius: 20px; font-size: 10px; color: black; font-weight: 600; }
            .tab-bar { display: flex; padding: 0 12px; gap: 4px; border-bottom: 1px solid var(--border); background: var(--bg); overflow-x: auto; -webkit-overflow-scrolling: touch; }
            .tab-btn { padding: 8px 12px; background: transparent; border: none; color: var(--text-light); font-size: 11px; font-weight: 600; cursor: pointer; border-bottom: 2px solid transparent; white-space: nowrap; }
            .tab-btn.active { color: var(--accent); border-bottom-color: var(--accent); }
            .tab-content { display: none; padding: 12px; max-height: 70vh; overflow-y: auto; -webkit-overflow-scrolling: touch; }
            .tab-content.active { display: block; }
            .controls-row { display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 12px; background: var(--bg-block); padding: 8px; border-radius: 12px; border: 1px solid var(--border); }
            .btn-group { display: flex; gap: 8px; flex: 1; min-width: 140px; }
            .btn-group .btn { padding: 8px 12px; border: none; border-radius: 8px; font-weight: 700; font-size: 11px; cursor: pointer; min-height: 38px; flex: 1; }
            .btn-start { background: var(--success); color: black; }
            .btn-stop { background: var(--danger); color: black; }
            .btn-pause { background: var(--text-light); color: black; }
            .btn-reset { background: var(--text-light); color: black; }
            .btn-login { background: var(--accent); color: black; padding: 8px 16px; border: none; border-radius: 8px; font-weight: 700; font-size: 11px; cursor: pointer; min-height: 38px; }
            .btn-refresh { background: var(--text-light); color: black; padding: 8px 16px; border: none; border-radius: 8px; font-weight: 700; font-size: 11px; cursor: pointer; min-height: 38px; }
            .risk-preset { padding: 4px 12px; border: 2px solid var(--text-light); border-radius: 6px; background: transparent; color: var(--text); font-size: 9px; cursor: pointer; transition: 0.3s; }
            .risk-preset.active { border-color: var(--accent); background: #9ec07c33; }
            .risk-preset:hover { border-color: var(--accent); }
            select, input { background: var(--bg); border: 1px solid var(--border); padding: 8px; border-radius: 8px; font-size: 11px; font-family: monospace; color: var(--text); }
            select { min-width: 80px; }
            .stats-block { background: var(--bg-block); border-radius: 12px; padding: 8px 10px; margin-bottom: 10px; border: 1px solid var(--border); font-size: 10px; line-height: 1.5; }
            .stats-block .line { display: flex; justify-content: space-between; padding: 3px 0; flex-wrap: wrap; gap: 4px; }
            .stats-block .label { color: var(--text-light); font-size: 9px; }
            .stats-block .value { font-weight: 600; text-align: right; color: var(--accent); font-size: 10px; word-break: break-word; max-width: 60%; }
            .stats-block .value-sub { font-size: 8px; color: var(--text-light); text-align: right; }
            .stats-block .divider { border-top: 1px solid var(--border); margin: 5px 0; }
            .progress-bar { background: var(--border); border-radius: 4px; margin: 4px 0; height: 4px; }
            .progress-fill { background: var(--success); height: 4px; border-radius: 4px; width: 0%; }
            .logs-container { background: var(--log-bg); border-radius: 12px; height: 350px; overflow-y: auto; padding: 8px; font-size: 9px; border: 1px solid var(--border); -webkit-overflow-scrolling: touch; font-family: monospace; }
            .token-row { display: flex; gap: 8px; margin-bottom: 10px; flex-wrap: wrap; }
            .token-row input { flex: 1; min-width: 120px; }
            .toggle-row { display: flex; justify-content: space-between; align-items: center; background: var(--bg-block); padding: 8px 10px; border-radius: 12px; margin-bottom: 10px; flex-wrap: wrap; gap: 8px; }
            .md-switch { position: relative; width: 46px; height: 22px; }
            .md-switch input { opacity: 0; width: 0; height: 0; }
            .md-slider { position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0; background: var(--border); border-radius: 22px; }
            .md-slider:before { position: absolute; content: ""; height: 16px; width: 16px; left: 3px; bottom: 3px; background: white; border-radius: 50%; transition: .3s; }
            input:checked + .md-slider { background: var(--success); }
            input:checked + .md-slider:before { transform: translateX(24px); }
            .md-field { margin-bottom: 8px; }
            .md-field label { display: block; font-size: 9px; font-weight: 500; margin-bottom: 3px; color: var(--text-light); }
            .md-field input, .md-field select { width: 100%; padding: 6px; font-size: 10px; }
            .grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; }
            @media (max-width: 480px) { .grid-2 { grid-template-columns: 1fr; } .stats-block .value { max-width: 70%; font-size: 9px; } .btn-group button { flex: 1; } }
            .status-inline { display: flex; align-items: center; gap: 8px; font-size: 10px; margin-bottom: 8px; }
            .dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
            .harvesting-panel { background: var(--bg-block); border-radius: 12px; border: 1px solid var(--border); margin-top: 10px; overflow: hidden; }
            .harvesting-header { background: var(--accent); color: black; padding: 6px 10px; font-size: 10px; font-weight: bold; display: flex; justify-content: space-between; align-items: center; }
            .harvesting-list { max-height: 500px; overflow-y: auto; padding: 5px; }
            .harvesting-item { padding: 8px; border-bottom: 1px solid var(--border); font-size: 9px; font-family: monospace; }
            .harvesting-item:last-child { border-bottom: none; }
            .harvesting-row { display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 8px; margin-bottom: 4px; }
            .harvesting-row-sub { display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 8px; font-size: 8px; color: var(--text-light); }
            .harvesting-win { color: var(--success); }
            .harvesting-loss { color: var(--danger); }
            .harvesting-chance { background: rgba(158,192,124,0.2); padding: 2px 6px; border-radius: 10px; font-size: 8px; font-weight: bold; }
            .harvesting-lot { font-size: 8px; }
            .harvesting-profit { font-weight: bold; }
            .harvesting-time { font-size: 7px; }
            .harvesting-empty { text-align: center; padding: 20px; color: var(--text-light); font-size: 9px; }
            .minbet-info { font-size: 8px; color: var(--text-light); margin-top: 4px; text-align: right; }
            .risk-settings { border-top: 1px solid var(--border); margin-top: 8px; padding-top: 8px; }
            .hunt-table { max-height: 400px; overflow-y: auto; font-size: 10px; }
            .hunt-stats { display: flex; justify-content: space-between; flex-wrap: wrap; gap: 8px; padding: 6px 8px; background: var(--bg); border-radius: 6px; margin-top: 8px; font-size: 9px; }
            .wager-progress { margin-top: 4px; padding: 4px 8px; background: var(--bg); border-radius: 6px; }
            ::-webkit-scrollbar { width: 3px; }
            ::-webkit-scrollbar-thumb { background: var(--border); }
        `;
        
        if (document.head) {
            document.head.appendChild(style);
        } else if (document.body) {
            document.body.appendChild(style);
        }
        
        var html = '<div id="BotConsole"><div class="bot-header" id="botHeader"><div style="display: flex; align-items: center; gap: 8px; flex-wrap: wrap;"><span class="title">⭐ ORION ASCEND v3.2</span><span class="time-running" id="st-running-time">0s</span></div><div class="header-btns"><button id="btn-theme">🌓</button><button id="btn-minimize">▲</button></div></div><div id="bot-body"><div class="tab-bar"><button class="tab-btn active" onclick="openTab.call(this, \'main\')">MAIN</button><button class="tab-btn" onclick="openTab.call(this, \'cfg\')">CFG</button><button class="tab-btn" onclick="openTab.call(this, \'mm\')">MM</button><button class="tab-btn" onclick="openTab.call(this, \'hunt\')">HUNT</button><button class="tab-btn" onclick="openTab.call(this, \'logs\')">LOGS</button><button class="tab-btn" onclick="openTab.call(this, \'history\')">HISTORY</button></div><div id="tab-main" class="tab-content active"><div class="token-row"><input type="password" id="p-token" placeholder="API Token"><button id="p-login" class="btn-login">LOGIN</button><button id="p-refresh" class="btn-refresh">⟳</button></div><div class="toggle-row"><span>⚡ FLASH MODE</span><label class="md-switch"><input type="checkbox" id="p-demo-toggle"><span class="md-slider"></span></label></div><div class="controls-row"><div class="btn-group"><button id="p-start" class="btn btn-start">START</button><button id="p-pause" class="btn btn-pause" disabled>PAUSE</button><button id="p-stop" class="btn btn-stop" disabled>STOP</button><button id="p-reset" class="btn btn-reset">RESET</button></div><select id="p-currency"></select></div><div class="status-inline"><span class="dot" id="statusDot" style="background:#FF4757;"></span><span id="botStatus">STOP</span></div><div class="stats-block"><div class="line"><span class="label">User</span><span class="value" id="st-user">-</span></div><div class="line"><span class="label">Balance</span><span class="value" id="st-balance">0</span></div><div class="line"><span class="label"></span><span class="value-sub" id="st-balance-fiat">-</span></div><div class="divider"></div><div class="line"><span class="label">📋 SESSION #</span><span class="value" id="st-session-num">1</span></div><div class="line"><span class="label">🎯 SESSION TARGET</span><span class="value" id="st-session-target">5% (0.00 USDT)</span></div><div class="progress-bar"><div class="progress-fill" id="st-session-progress-bar"></div></div><div class="line"><span class="label">📊 SESSION PROGRESS</span><span class="value" id="st-session-progress">0%</span></div><div class="line"><span class="label">⏱️ SESSION START</span><span class="value" id="st-session-start">-</span></div><div class="line"><span class="label">✅ SESSIONS DONE</span><span class="value" id="st-sessions-done">0</span></div><div class="divider"></div><div class="line"><span class="label">Profit (Session)</span><span class="value" id="st-profit">0 (0%)</span></div><div class="line"><span class="label"></span><span class="value-sub" id="st-profit-fiat">-</span></div><div class="line"><span class="label">Profit (Daily)</span><span class="value" id="st-daily">0 (0%)</span></div><div class="divider"></div><div class="line"><span class="label">Wagered</span><span class="value" id="st-wagered">0</span></div><div class="line"><span class="label">Rakeback</span><span class="value" id="st-rakeback">+0</span></div><div class="line"><span class="label">Bets | WR</span><span class="value" id="st-bets">0 | 0%</span></div><div class="line"><span class="label">Speed</span><span class="value" id="st-speed">0 b/s</span></div><div class="line"><span class="label">Max DD</span><span class="value" id="st-maxdd">0%</span></div><div class="line"><span class="label">Max Bet %</span><span class="value" id="st-maxbet">0%</span></div><div class="divider"></div><div class="line"><span class="label">Status</span><span class="value" id="st-chance">⚡ 99.98%</span></div><div class="line"><span class="label">Gap Hits</span><span class="value" id="st-gaphits">0</span></div><div class="line"><span class="label">Countdown</span><span class="value" id="st-timer">--</span></div><div class="line"><span class="label">Rate</span><span class="value" id="st-rate">-</span></div><div class="divider"></div><div class="line"><span class="label">📊 SPLIT PROGRESS</span><span class="value" id="st-split-progress">—</span></div><div class="line"><span class="label">✂️ CUTLOSS COUNT</span><span class="value" id="st-cutloss-count">0</span></div><div class="line"><span class="label">📉 LOSS/SPLIT</span><span class="value" id="st-loss-per-split">0</span></div><div class="divider"></div><div class="line"><span class="label">⚠️ WORST CHANCE</span><span class="value" id="st-worst-chance">-</span></div><div class="line"><span class="label">📉 WORST LS</span><span class="value" id="st-worst-ls">-</span></div><div class="line"><span class="label">📊 WORST DD</span><span class="value" id="st-worst-dd">-</span></div><div class="line"><span class="label">💀 WORST BET</span><span class="value" id="st-worst-bet">-</span></div></div></div><div id="tab-cfg" class="tab-content"><div class="stats-block" style="margin-top:0px;"><div style="font-weight:bold; margin-bottom:8px;">⚙️ NORMAL MODE SETTINGS</div><div class="grid-2"><div class="md-field"><label>🎲 NORMAL CHANCE (%)</label><input type="number" id="p-normal-chance" value="99.98" step="0.01" min="1" max="99.98"></div><div class="md-field"><label>💰 NORMAL BET (% balance)</label><input type="number" id="p-basebet-normal" value="0.5" step="0.001" min="0.01" max="10"></div></div></div><div class="stats-block" style="margin-top:10px;"><div style="font-weight:bold; margin-bottom:8px;">🔍 VIRTUAL HUNTING RANGE</div><div class="grid-2"><div class="md-field"><label>📉 MIN CHANCE (%)</label><input type="number" id="p-virt-min" value="15" step="0.5" min="1" max="90"></div><div class="md-field"><label>📈 MAX CHANCE (%)</label><input type="number" id="p-virt-max" value="25" step="0.5" min="2" max="99.98"></div></div><div class="minbet-info" id="virt-range-info">Virtual chances: 15% - 25% (21 chances)</div></div><div class="stats-block" style="margin-top:10px;"><div style="font-weight:bold; margin-bottom:8px;">⚡ JIT (BURST) MODE</div><div class="grid-2"><div class="md-field"><label>⚡ JIT EVERY N BET</label><input type="number" id="p-burst-every" value="20" step="1" min="0" max="100"></div><div class="md-field"><label>🎲 MIN CHANCE</label><input type="number" id="p-burst-min" value="90" step="0.5" min="1" max="99"></div></div><div class="grid-2"><div class="md-field"><label>🎲 MAX CHANCE</label><input type="number" id="p-burst-max" value="98" step="0.5" min="2" max="99.98"></div><div class="md-field"><label>💰 BET MULTIPLIER (× loss)</label><input type="number" id="p-burst-multiplier" value="1.0" step="0.1" min="0.1" max="5"></div></div><div class="grid-2"><div class="md-field"><label>⚡ JIT ENABLED</label><select id="p-jit-enabled"><option value="true" selected>ENABLED</option><option value="false">DISABLED</option></select></div></div><div id="burstDisplay" style="background:var(--bg);padding:4px 8px;border-radius:6px;margin-top:6px;font-size:9px;">⚡ JIT: OFF</div></div><div class="stats-block" style="margin-top:10px;"><div style="font-weight:bold; margin-bottom:8px;">🌾 HARVEST TRIGGER</div><div class="grid-2"><div class="md-field"><label>🎯 HARVEST TRIGGER (× payout)</label><input type="number" id="p-harvest-trigger" value="3" step="0.5" min="1" max="20"></div></div></div><div class="stats-block" style="margin-top:10px;"><div style="font-weight:bold; margin-bottom:8px;">🎯 WAGER FARMING SETTINGS</div><div class="grid-2"><div class="md-field"><label>📊 TOTAL WAGER TARGET</label><input type="number" id="p-wager-total-target" value="10000" step="100" min="1" max="999999999"></div><div class="md-field"><label>📊 SESI WAGER TARGET</label><input type="number" id="p-wager-sesi-target" value="200" step="1" min="1" max="99999"></div></div><div class="grid-2"><div class="md-field"><label>📉 LOSS TOLERANCE</label><input type="number" id="p-wager-loss-tolerance" value="0.5" step="0.01" min="0.001" max="100"></div></div><div id="wagerDisplay" style="background:var(--bg);padding:4px 8px;border-radius:6px;margin-top:6px;font-size:9px;">📊 Total Wager: 0.00 / 10000</div></div><div class="stats-block" style="margin-top:10px;"><div style="font-weight:bold; margin-bottom:8px;">🔄 RECOVERY SETTINGS</div><div class="grid-2"><div class="md-field"><label>🔁 MAX SPLIT PER CYCLE</label><input type="number" id="p-recovery-max-splits" value="8" step="1" min="1" max="50"></div><div class="md-field"><label>✂️ CUTLOSS MULTIPLIER (Recovery)</label><input type="number" id="p-recovery-cutloss-multiplier" value="10" step="0.5" min="2" max="50"></div></div><div class="grid-2"><div class="md-field"><label>📉 MIN CHANCE (Recovery)</label><input type="number" id="p-recovery-min-chance" value="0.01" step="0.01" min="0.01" max="10"></div></div></div><div class="stats-block" style="margin-top:10px;"><div style="font-weight:bold; margin-bottom:8px;">🎯 RECOVERY MODE SETTINGS</div><div class="grid-2"><div class="md-field"><label>RECOVERY TARGET</label><select id="p-recovery-target"><option value="start" selected>START BALANCE</option><option value="peak">PEAK BALANCE</option></select></div><div class="md-field"><label>RECOVERY MODE</label><select id="p-recovery-mode"><option value="static" selected>STATIC</option><option value="dynamic">DYNAMIC</option></select></div></div><div class="grid-2"><div class="md-field"><label>BEP SPLIT (0 = OFF)</label><input type="number" id="p-bep-split" value="0" step="1" min="0" max="20"></div><div class="md-field"><label>PEAK SPLIT (0 = OFF)</label><input type="number" id="p-peak-split" value="0" step="1" min="0" max="20"></div></div></div><div class="stats-block" style="margin-top:10px;"><div style="font-weight:bold; margin-bottom:8px;">📊 RISK PROFILE</div><div style="display:flex;gap:6px;flex-wrap:wrap;margin-bottom:8px;"><button id="risk-conservative" class="risk-preset">🛡️ Konservatif</button><button id="risk-normal" class="risk-preset active">⚖️ Normal</button><button id="risk-aggressive" class="risk-preset">🚀 Agresif</button><button id="risk-super" class="risk-preset">🔥 Super</button><button id="risk-custom" class="risk-preset">⚙️ Custom</button></div><div id="risk-info" style="background:var(--bg);padding:6px 10px;border-radius:8px;font-size:9px;margin-bottom:6px;"><div style="display:flex;justify-content:space-between;flex-wrap:wrap;gap:4px;"><span>📊 Base Risk: <strong id="risk-base-display">15%</strong></span><span>🎲 Fluctuation: <strong id="risk-fluct-display">±30%</strong></span><span>🔒 Range: <strong id="risk-range-display">5% - 35%</strong></span></div></div><div id="risk-custom-settings" style="display:none;border-top:1px solid var(--border);padding-top:8px;margin-top:6px;"><div class="grid-2"><div class="md-field"><label>BASE RISK FACTOR</label><input type="number" id="p-risk-factor" value="0.15" step="0.01" min="0.02" max="0.5"></div><div class="md-field"><label>FLUCTUATION</label><input type="number" id="p-risk-fluctuation" value="0.3" step="0.05" min="0" max="0.8"></div></div><div class="grid-2"><div class="md-field"><label>MIN RISK LIMIT</label><input type="number" id="p-min-risk" value="0.05" step="0.01" min="0.02" max="0.3"></div><div class="md-field"><label>MAX RISK LIMIT</label><input type="number" id="p-max-risk" value="0.35" step="0.01" min="0.1" max="0.5"></div></div></div></div><div class="stats-block" style="margin-top:10px;"><div style="font-weight:bold; margin-bottom:8px;">✂️ CUT LOSS SETTINGS</div><div class="grid-2"><div class="md-field"><label>CUT LOSS METHOD</label><select id="p-cut-method"><option value="static" selected>STATIC</option><option value="dynamic">DYNAMIC (PROGRESS-BASED)</option><option value="drawdown">DRAWDOWN %</option></select></div><div class="md-field"><label>BATAS MULTIPLIER (× payout)</label><input type="number" id="p-cut-multiplier" value="5" step="0.5" min="1" max="20"></div></div><div class="grid-2"><div class="md-field"><label>DYNAMIC MIN MULTIPLIER</label><input type="number" id="p-cut-min" value="3" step="0.5" min="0.5" max="5"></div><div class="md-field"><label>DYNAMIC MAX MULTIPLIER</label><input type="number" id="p-cut-max" value="5" step="0.5" min="1" max="10"></div></div><div class="grid-2"><div class="md-field"><label>MAX DRAWDOWN %</label><input type="number" id="p-max-drawdown" value="50" step="0.5" min="1" max="100"></div></div></div><div class="stats-block" style="margin-top:10px;"><div style="font-weight:bold; margin-bottom:8px;">📊 CHART SETTINGS</div><div class="grid-2"><div class="md-field"><label>MAX CHART POINTS</label><input type="number" id="p-max-chart" value="50" step="10" min="10" max="200"></div><div class="md-field"><label>CHART TYPE</label><select id="p-chart-type"><option value="line">LINE</option><option value="bar">BAR</option></select></div></div><div class="grid-2"><div class="md-field"><label>MAX LOGS</label><input type="number" id="p-max-logs" value="50" step="10" min="10" max="200"></div></div></div><div class="md-field"><label>MIN BET (IDR)</label><input type="number" id="p-minbet-idr" value="2" step="1"></div><div class="minbet-info" id="minbet-info">≈ 0.00000000 crypto</div><div class="md-field"><label>DIRTY BALANCE</label><input type="number" id="p-dirty-bal" value="50" step="0.1"></div></div><div id="tab-mm" class="tab-content"><div class="stats-block" style="margin-top:0px;"><div style="font-weight:bold; margin-bottom:8px;">💰 PROFIT TARGET & STOP LOSS</div><div class="grid-2"><div class="md-field"><label>TARGET % (Session)</label><input type="number" id="p-target-pct" value="5" step="0.5"></div><div class="md-field"><label>STOP LOSS % (Session)</label><input type="number" id="p-stop-pct" value="100" step="0.5"></div></div><div class="grid-2"><div class="md-field"><label>TARGET % (Daily)</label><input type="number" id="p-daily-target" value="25" step="1"></div><div class="md-field"><label>STOP LOSS % (Daily)</label><input type="number" id="p-daily-stoploss" value="100" step="1"></div></div><div class="grid-2"><div class="md-field"><label>SESSION TOTAL</label><input type="number" id="p-session-total" value="8" step="1"></div><div class="md-field"><label>MAX BET CAP</label><input type="number" id="p-max-cap" value="0" step="0.001"></div></div></div><div class="stats-block" style="margin-top:10px;"><div style="font-weight:bold; margin-bottom:8px;">🔱 TRAILING GAP SETTINGS</div><div class="grid-2"><div class="md-field"><label>TRAIL ACTIVATION %</label><input type="number" id="p-trail-activation" value="3" step="0.5"></div><div class="md-field"><label>GAP1 ACTION</label><select id="p-gap1-action"><option value="continue" selected>CONTINUE</option><option value="stop">STOP</option></select></div></div><div class="grid-2"><div class="md-field"><label>GAP LVL 1 %</label><input type="number" id="p-gap1" value="0.5" step="0.1"></div><div class="md-field"><label>GAP LVL 2 %</label><input type="number" id="p-gap2" value="0.25" step="0.05"></div></div><div class="grid-2"><div class="md-field"><label>GAP LVL 3+ %</label><input type="number" id="p-gap3" value="0.125" step="0.05"></div></div></div><div class="stats-block" style="margin-top:10px;"><div style="font-weight:bold; margin-bottom:8px;">⏱️ DELAY PER MODE</div><div class="grid-2"><div class="md-field"><label>⚡ NORMAL MODE DELAY (ms)</label><input type="number" id="p-delay-normal" value="10" step="1" min="0" max="500"></div><div class="md-field"><label>🌾 HARVEST MODE DELAY (ms)</label><input type="number" id="p-delay-harvest" value="100" step="1" min="0" max="500"></div></div><div class="grid-2"><div class="md-field"><label>🔄 RECOVERY MODE DELAY (ms)</label><input type="number" id="p-delay-recovery" value="200" step="1" min="0" max="500"></div></div></div></div><div id="tab-hunt" class="tab-content"><div style="font-weight:bold;font-size:11px;margin-bottom:8px;">🔍 VIRTUAL HUNTING COUNTER</div><div id="huntTable" class="hunt-table"><div style="text-align:center;padding:20px;color:var(--text-light);font-size:11px;">No virtual chances initialized</div></div><div class="hunt-stats"><span>🎯 NEXT TRIGGER: <strong id="huntNextTrigger">—</strong></span><span>⏱️ Last Update: <strong id="huntLastUpdate">—</strong></span><span>📊 Total Chances: <strong id="huntTotalChances">0</strong></span></div></div><div id="tab-logs" class="tab-content"><div class="logs-container" id="p-logs"><div style="text-align:center;padding:20px;">⭐ ORION ASCEND v3.2<br>"Fire & Forget Wager Farming"<br><br>🔥 Burn for wager, recover with precision<br><br>🔄 Infinite Split Recovery (Cycle 8)<br>📊 Dynamic Risk Profile<br>🔱 Trailing Gap System<br>🛡️ Auto Cutloss Protection<br>⚡ Lightweight (No Chart)<br>🚀 BOOSTER MODE</div></div></div><div id="tab-history" class="tab-content"><div class="harvesting-panel"><div class="harvesting-header"><span>📋 HARVESTING HISTORY</span><span id="harvestingCount">0</span></div><div id="harvestingList" class="harvesting-list"><div class="harvesting-empty">📋 No harvest records yet</div></div></div></div></div></div>';
        
        var container = document.getElementById('BotConsole');
        if (!container) {
            var div = document.createElement('div');
            div.id = 'BotConsole';
            div.innerHTML = html;
            document.body.appendChild(div);
        } else {
            container.innerHTML = html;
        }
        
        var header = document.getElementById("botHeader");
        if (header) header.addEventListener('mousedown', dragMouseDown);
        var minimizeBtn = document.getElementById("btn-minimize");
        if (minimizeBtn) minimizeBtn.onclick = toggleMinimize;
        var themeBtn = document.getElementById("btn-theme");
        if (themeBtn) themeBtn.onclick = toggleDarkMode;
        
        var savedToken = getAuthToken();
        if (savedToken) {
            var tokenEl = document.getElementById("p-token");
            if (tokenEl) tokenEl.value = savedToken;
        }
        
        // ==================== CHART SETTINGS (DISABLED) ====================
        var chartMaxEl = document.getElementById("p-max-chart");
        if (chartMaxEl) {
            chartMaxEl.onchange = function(e) {
                bot.maxChartPoints = parseInt(e.target.value) || 50;
            };
        }
        var chartTypeEl = document.getElementById("p-chart-type");
        if (chartTypeEl) {
            chartTypeEl.onchange = function(e) {
                bot.chartType = e.target.value;
            };
        }
        var maxLogsEl = document.getElementById("p-max-logs");
        if (maxLogsEl) {
            maxLogsEl.onchange = function(e) {
                bot.maxLogs = parseInt(e.target.value) || 50;
            };
        }
        
        // ==================== RISK PRESET EVENTS ====================
        var riskConservative = document.getElementById("risk-conservative");
        if (riskConservative) riskConservative.onclick = function() { setRiskPreset('conservative'); };
        var riskNormal = document.getElementById("risk-normal");
        if (riskNormal) riskNormal.onclick = function() { setRiskPreset('normal'); };
        var riskAggressive = document.getElementById("risk-aggressive");
        if (riskAggressive) riskAggressive.onclick = function() { setRiskPreset('aggressive'); };
        var riskSuper = document.getElementById("risk-super");
        if (riskSuper) riskSuper.onclick = function() { setRiskPreset('super'); };
        var riskCustom = document.getElementById("risk-custom");
        if (riskCustom) {
            riskCustom.onclick = function() { 
                var settings = document.getElementById('risk-custom-settings');
                if (settings) settings.style.display = 'block';
                document.querySelectorAll('.risk-preset').forEach(function(btn) {
                    btn.classList.remove('active');
                    btn.style.background = 'transparent';
                    btn.style.borderColor = '';
                });
                riskCustom.classList.add('active');
                riskCustom.style.background = '#9ec07c33';
                riskCustom.style.borderColor = '#9ec07c';
            };
        }
        
        // ==================== BURST EVENTS ====================
        function updateBurstDisplayUI() {
            var container = document.getElementById("burstDisplay");
            if (!container) return;
            
            var isActive = bot.wagerJitEnabled && bot.burstEveryN > 0 && !bot.isBoosterMode;
            var counterText = isActive ? (bot.burstCounter + 1) + '/' + bot.burstEveryN : 'OFF';
            var isBurst = isActive && bot.burstCounter >= bot.burstEveryN;
            var chanceText = isActive ? bot.burstMinChance + '%-' + bot.burstMaxChance + '%' : '-';
            var multText = isActive ? bot.burstMultiplier.toFixed(1) + '×' : '-';
            
            container.innerHTML = `
                <div style="display:flex;justify-content:space-between;flex-wrap:wrap;gap:4px;font-size:9px;padding:4px 6px;background:var(--bg);border-radius:6px;">
                    <span>⚡ JIT: <strong>${isActive ? 'ON' : 'OFF'}</strong></span>
                    <span>🔄 Counter: <strong>${counterText}</strong></span>
                    <span>🎲 Chance: <strong>${chanceText}</strong></span>
                    <span>💰 Mult: <strong>${multText}</strong></span>
                    <span style="color:${isBurst ? '#ff922b' : 'var(--text-light)'};">${isBurst ? '⭐ JIT ACTIVE' : 'Normal bet'}</span>
                </div>
            `;
        }
        
        function updateWagerDisplayUI() {
            var container = document.getElementById("wagerDisplay");
            if (!container) return;
            
            var progress = bot.wagerTotalTarget > 0 ? (bot.wagerTotalAccumulated / bot.wagerTotalTarget) * 100 : 0;
            if (progress > 100) progress = 100;
            
            var waitingText = bot.wagerWaitingForTrigger ? '⏳ WAITING TRIGGER' : '▶ RUNNING';
            var waitingColor = bot.wagerWaitingForTrigger ? '#ff922b' : '#00e5c0';
            
            container.innerHTML = `
                <div style="display:flex;justify-content:space-between;flex-wrap:wrap;gap:4px;font-size:9px;padding:4px 6px;background:var(--bg);border-radius:6px;">
                    <span>📊 Total Wager: <strong>${bot.wagerTotalAccumulated.toFixed(2)}</strong> / ${bot.wagerTotalTarget}</span>
                    <span>📋 Sesi Wager: <strong>${bot.wagerSesiCounter.toFixed(2)}</strong> / ${bot.wagerSesiTarget}</span>
                    <span>📉 Sesi Loss: <strong>${bot.wagerSesiProfit.toFixed(4)}</strong> / ${bot.wagerLossTolerance}</span>
                    <span>📈 Progress: <strong>${progress.toFixed(1)}%</strong></span>
                    <span style="color:${bot.wagerStop ? '#ff6b81' : waitingColor};">${bot.wagerStop ? '⛔ STOPPED' : waitingText}</span>
                </div>
                <div class="progress-bar" style="margin-top:4px;">
                    <div class="progress-fill" style="width:${progress}%;background:${progress >= 100 ? '#ff6b81' : '#00e5c0'};"></div>
                </div>
            `;
        }
        
        var burstEveryEl = document.getElementById("p-burst-every");
        if (burstEveryEl) {
            burstEveryEl.onchange = function(e) {
                var val = parseInt(e.target.value) || 0;
                bot.burstEveryN = Math.min(100, Math.max(0, val));
                bot.burstCounter = 0;
                updateBurstDisplayUI();
            };
        }
        var burstMinEl = document.getElementById("p-burst-min");
        if (burstMinEl) {
            burstMinEl.onchange = function(e) {
                var val = parseFloat(e.target.value) || 1;
                bot.burstMinChance = Math.min(99, Math.max(1, val));
                if (bot.burstMinChance >= bot.burstMaxChance) {
                    bot.burstMinChance = bot.burstMaxChance - 0.5;
                }
                updateBurstDisplayUI();
            };
        }
        var burstMaxEl = document.getElementById("p-burst-max");
        if (burstMaxEl) {
            burstMaxEl.onchange = function(e) {
                var val = parseFloat(e.target.value) || 98;
                bot.burstMaxChance = Math.min(99.98, Math.max(2, val));
                if (bot.burstMaxChance <= bot.burstMinChance) {
                    bot.burstMaxChance = bot.burstMinChance + 0.5;
                }
                updateBurstDisplayUI();
            };
        }
        var burstMultEl = document.getElementById("p-burst-multiplier");
        if (burstMultEl) {
            burstMultEl.onchange = function(e) {
                var val = parseFloat(e.target.value) || 1.0;
                bot.burstMultiplier = Math.min(5, Math.max(0.1, val));
                updateBurstDisplayUI();
            };
        }
        var jitEnabledEl = document.getElementById("p-jit-enabled");
        if (jitEnabledEl) {
            jitEnabledEl.onchange = function(e) {
                bot.wagerJitEnabled = e.target.value === 'true';
                updateBurstDisplayUI();
            };
        }
        
        var wagerTotalTargetEl = document.getElementById("p-wager-total-target");
        if (wagerTotalTargetEl) {
            wagerTotalTargetEl.onchange = function(e) {
                var val = parseFloat(e.target.value) || 10000;
                bot.wagerTotalTarget = Math.max(1, val);
                updateWagerDisplayUI();
            };
        }
        var wagerSesiTargetEl = document.getElementById("p-wager-sesi-target");
        if (wagerSesiTargetEl) {
            wagerSesiTargetEl.onchange = function(e) {
                var val = parseFloat(e.target.value) || 200;
                bot.wagerSesiTarget = Math.max(1, val);
                updateWagerDisplayUI();
            };
        }
        var wagerLossToleranceEl = document.getElementById("p-wager-loss-tolerance");
        if (wagerLossToleranceEl) {
            wagerLossToleranceEl.onchange = function(e) {
                var val = parseFloat(e.target.value) || 0.5;
                bot.wagerLossTolerance = Math.max(0.001, val);
                updateWagerDisplayUI();
            };
        }
        
        var harvestTriggerEl = document.getElementById("p-harvest-trigger");
        if (harvestTriggerEl) {
            harvestTriggerEl.onchange = function(e) {
                var val = parseFloat(e.target.value) || 3;
                bot.harvestTrigger = Math.min(20, Math.max(1, val));
                initMemory();
                drawGapVisual();
                updateHuntTab();
            };
        }
        
        var recoveryMaxSplitsEl = document.getElementById("p-recovery-max-splits");
        if (recoveryMaxSplitsEl) {
            recoveryMaxSplitsEl.onchange = function(e) {
                var val = parseInt(e.target.value) || 8;
                bot.recoveryMaxSplits = Math.min(50, Math.max(1, val));
            };
        }
        var recoveryCutlossMultEl = document.getElementById("p-recovery-cutloss-multiplier");
        if (recoveryCutlossMultEl) {
            recoveryCutlossMultEl.onchange = function(e) {
                var val = parseFloat(e.target.value) || 10;
                bot.recoveryCutlossMultiplier = Math.min(50, Math.max(2, val));
            };
        }
        var recoveryMinChanceEl = document.getElementById("p-recovery-min-chance");
        if (recoveryMinChanceEl) {
            recoveryMinChanceEl.onchange = function(e) {
                var val = parseFloat(e.target.value) || 0.01;
                bot.recoveryChanceMin = Math.min(10, Math.max(0.01, val));
            };
        }
        
        var demoEl = document.getElementById("p-demo-toggle");
        if (demoEl) demoEl.onchange = function(e) { bot.isDemo = e.target.checked; };
        
        var normalChanceEl = document.getElementById("p-normal-chance");
        if (normalChanceEl) {
            normalChanceEl.onchange = function(e) {
                var newChance = parseFloat(e.target.value);
                if (newChance >= 1 && newChance <= 99.98) {
                    bot.normalChance = newChance;
                    initVirtualChances();
                    initMemory();
                    drawGapVisual();
                    updateHuntTab();
                }
            };
        }
        var baseBetNormalEl = document.getElementById("p-basebet-normal");
        if (baseBetNormalEl) {
            baseBetNormalEl.onchange = function(e) {
                var newPct = parseFloat(e.target.value);
                if (newPct >= 0.01 && newPct <= 10) {
                    bot.baseBetPercent = newPct;
                    drawGapVisual();
                }
            };
        }
        var baseBetHarvestEl = document.getElementById("p-basebet-harvest");
        if (baseBetHarvestEl) {
            baseBetHarvestEl.onchange = function(e) {
                var newPct = parseFloat(e.target.value);
                if (newPct >= 0 && newPct <= 20) {
                    bot.harvestBetPercent = newPct;
                    drawGapVisual();
                }
            };
        }
        var virtMinEl = document.getElementById("p-virt-min");
        if (virtMinEl) {
            virtMinEl.onchange = function(e) {
                var newMin = parseFloat(e.target.value);
                if (newMin >= 1 && newMin < bot.virtMaxChance) {
                    bot.virtMinChance = newMin;
                    initVirtualChances();
                    initMemory();
                    drawGapVisual();
                    updateHuntTab();
                }
            };
        }
        var virtMaxEl = document.getElementById("p-virt-max");
        if (virtMaxEl) {
            virtMaxEl.onchange = function(e) {
                var newMax = parseFloat(e.target.value);
                if (newMax > bot.virtMinChance && newMax <= 99.98) {
                    bot.virtMaxChance = newMax;
                    initVirtualChances();
                    initMemory();
                    drawGapVisual();
                    updateHuntTab();
                }
            };
        }
        
        var recoveryTargetEl = document.getElementById("p-recovery-target");
        if (recoveryTargetEl) recoveryTargetEl.onchange = function(e) { bot.recoveryTarget = e.target.value; drawGapVisual(); };
        var recoveryModeEl = document.getElementById("p-recovery-mode");
        if (recoveryModeEl) recoveryModeEl.onchange = function(e) { bot.recoveryMode = e.target.value; drawGapVisual(); };
        var bepSplitEl = document.getElementById("p-bep-split");
        if (bepSplitEl) {
            bepSplitEl.onchange = function(e) {
                var newSplit = parseInt(e.target.value) || 0;
                bot.bepSplit = Math.min(20, Math.max(0, newSplit));
                drawGapVisual();
            };
        }
        var peakSplitEl = document.getElementById("p-peak-split");
        if (peakSplitEl) {
            peakSplitEl.onchange = function(e) {
                var newSplit = parseInt(e.target.value) || 0;
                bot.peakSplit = Math.min(20, Math.max(0, newSplit));
                drawGapVisual();
            };
        }
        var cutMethodEl = document.getElementById("p-cut-method");
        if (cutMethodEl) cutMethodEl.onchange = function(e) { bot.cutMethod = e.target.value; drawGapVisual(); };
        var cutMultEl = document.getElementById("p-cut-multiplier");
        if (cutMultEl) cutMultEl.onchange = function(e) { bot.maxCutLossMultiplier = parseFloat(e.target.value) || 5; drawGapVisual(); };
        var cutMinEl = document.getElementById("p-cut-min");
        if (cutMinEl) cutMinEl.onchange = function(e) { bot.cutMinMultiplier = parseFloat(e.target.value) || 3; };
        var cutMaxEl = document.getElementById("p-cut-max");
        if (cutMaxEl) cutMaxEl.onchange = function(e) { bot.cutMaxMultiplier = parseFloat(e.target.value) || 5; };
        var maxDrawdownEl = document.getElementById("p-max-drawdown");
        if (maxDrawdownEl) maxDrawdownEl.onchange = function(e) { bot.maxDrawdownPct = parseFloat(e.target.value) || 50; };
        
        var targetPctEl = document.getElementById("p-target-pct");
        if (targetPctEl) targetPctEl.onchange = function(e) { bot.targetPct = parseFloat(e.target.value) || 5; };
        var stopPctEl = document.getElementById("p-stop-pct");
        if (stopPctEl) stopPctEl.onchange = function(e) { bot.stopLossPct = parseFloat(e.target.value) || 100; };
        var dailyTargetEl = document.getElementById("p-daily-target");
        if (dailyTargetEl) dailyTargetEl.onchange = function(e) { bot.dailyTargetPct = parseFloat(e.target.value) || 25; };
        var dailyStopEl = document.getElementById("p-daily-stoploss");
        if (dailyStopEl) dailyStopEl.onchange = function(e) { bot.dailyStopLossPct = parseFloat(e.target.value) || 100; };
        var sessionTotalEl = document.getElementById("p-session-total");
        if (sessionTotalEl) sessionTotalEl.onchange = function(e) { bot.sessionTotal = parseInt(e.target.value) || 8; };
        var maxCapEl = document.getElementById("p-max-cap");
        if (maxCapEl) maxCapEl.onchange = function(e) { bot.maxBetCap = parseFloat(e.target.value) || 0; };
        
        var trailActEl = document.getElementById("p-trail-activation");
        if (trailActEl) trailActEl.onchange = function(e) { bot.trailingActivationPct = parseFloat(e.target.value) || 3; };
        var gap1ActionEl = document.getElementById("p-gap1-action");
        if (gap1ActionEl) gap1ActionEl.onchange = function(e) { bot.gap1Action = e.target.value; };
        var gap1El = document.getElementById("p-gap1");
        if (gap1El) gap1El.onchange = function(e) { bot.gapLevel1 = parseFloat(e.target.value) || 0.5; drawGapVisual(); };
        var gap2El = document.getElementById("p-gap2");
        if (gap2El) gap2El.onchange = function(e) { bot.gapLevel2 = parseFloat(e.target.value) || 0.25; drawGapVisual(); };
        var gap3El = document.getElementById("p-gap3");
        if (gap3El) gap3El.onchange = function(e) { bot.gapLevel3plus = parseFloat(e.target.value) || 0.125; drawGapVisual(); };
        
        var delayNormalEl = document.getElementById("p-delay-normal");
        if (delayNormalEl) delayNormalEl.onchange = function(e) { bot.delayNormal = Math.min(500, Math.max(0, parseInt(e.target.value) || 0)); };
        var delayHarvestEl = document.getElementById("p-delay-harvest");
        if (delayHarvestEl) delayHarvestEl.onchange = function(e) { bot.delayHarvest = Math.min(500, Math.max(0, parseInt(e.target.value) || 0)); };
        var delayRecoveryEl = document.getElementById("p-delay-recovery");
        if (delayRecoveryEl) delayRecoveryEl.onchange = function(e) { bot.delayRecovery = Math.min(500, Math.max(0, parseInt(e.target.value) || 0)); };
        
        var loginBtn = document.getElementById("p-login");
        if (loginBtn) {
            loginBtn.onclick = async function() {
                var tokenEl = document.getElementById("p-token");
                if (!tokenEl) return;
                var token = tokenEl.value.trim();
                if (!token || token.length < 30) { return; }
                saveToken(token);
                var success = await syncUserInfo();
                if (success) {
                    var startBtn = document.getElementById("p-start");
                    if (startBtn) startBtn.disabled = false;
                    await updateFiatRatesOnce();
                    updateStatsDisplay();
                    sendTelegramMessage("⭐ ORION ASCEND v3.2\n\"Fire & Forget Wager Farming\"\n✅ LOGIN SUCCESS\nUser: " + bot.stakeUser);
                }
            };
        }
        var refreshBtn = document.getElementById("p-refresh");
        if (refreshBtn) {
            refreshBtn.onclick = async function() {
                if (!accessToken) { return; }
                await syncUserInfo();
                await updateFiatRatesOnce();
                updateStatsDisplay();
            };
        }
        
        var startBtn = document.getElementById("p-start");
        if (startBtn) {
            startBtn.onclick = async function() {
                if (bot.isPaused) {
                    bot.isPaused = false;
                    bot.isRunning = true;
                    togglePauseButton(false);
                    runLoop();
                    return;
                }
                if (bot.isRunning) return;
                
                fullReset();
                
                bot.minBetIdr = safeGetFloat("p-minbet-idr", 2);
                bot.riskFactor = safeGetFloat("p-risk-factor", 0.15);
                bot.riskFluctuation = safeGetFloat("p-risk-fluctuation", 0.3);
                bot.minRiskLimit = safeGetFloat("p-min-risk", 0.05);
                bot.maxRiskLimit = safeGetFloat("p-max-risk", 0.35);
                bot.targetPct = safeGetFloat("p-target-pct", 5);
                bot.stopLossPct = safeGetFloat("p-stop-pct", 100);
                bot.dailyTargetPct = safeGetFloat("p-daily-target", 25);
                bot.dailyStopLossPct = safeGetFloat("p-daily-stoploss", 100);
                bot.sessionTotal = safeGetInt("p-session-total", 8);
                bot.maxBetCap = safeGetFloat("p-max-cap", 0);
                bot.trailingActivationPct = safeGetFloat("p-trail-activation", 3);
                var gap1ActionEl2 = document.getElementById("p-gap1-action");
                bot.gap1Action = gap1ActionEl2 ? gap1ActionEl2.value : "continue";
                bot.gapLevel1 = safeGetFloat("p-gap1", 0.5);
                bot.gapLevel2 = safeGetFloat("p-gap2", 0.25);
                bot.gapLevel3plus = safeGetFloat("p-gap3", 0.125);
                bot.dirtyBalance = safeGetFloat("p-dirty-bal", 50);
                bot.normalChance = safeGetFloat("p-normal-chance", 99.98);
                bot.baseBetPercent = safeGetFloat("p-basebet-normal", 0.5);
                bot.harvestBetPercent = safeGetFloat("p-basebet-harvest", 0);
                bot.virtMinChance = safeGetFloat("p-virt-min", 15);
                bot.virtMaxChance = safeGetFloat("p-virt-max", 25);
                var recoveryTargetEl2 = document.getElementById("p-recovery-target");
                bot.recoveryTarget = recoveryTargetEl2 ? recoveryTargetEl2.value : "start";
                var recoveryModeEl2 = document.getElementById("p-recovery-mode");
                bot.recoveryMode = recoveryModeEl2 ? recoveryModeEl2.value : "static";
                bot.bepSplit = safeGetInt("p-bep-split", 0);
                bot.peakSplit = safeGetInt("p-peak-split", 0);
                var cutMethodEl2 = document.getElementById("p-cut-method");
                bot.cutMethod = cutMethodEl2 ? cutMethodEl2.value : "static";
                bot.maxCutLossMultiplier = safeGetFloat("p-cut-multiplier", 5);
                bot.cutMinMultiplier = safeGetFloat("p-cut-min", 3);
                bot.cutMaxMultiplier = safeGetFloat("p-cut-max", 5);
                bot.maxDrawdownPct = safeGetFloat("p-max-drawdown", 50);
                bot.maxChartPoints = safeGetInt("p-max-chart", 50);
                var chartTypeEl2 = document.getElementById("p-chart-type");
                bot.chartType = chartTypeEl2 ? chartTypeEl2.value : "line";
                bot.maxLogs = safeGetInt("p-max-logs", 50);
                
                bot.burstEveryN = safeGetInt("p-burst-every", 20);
                bot.burstMinChance = safeGetFloat("p-burst-min", 90);
                bot.burstMaxChance = safeGetFloat("p-burst-max", 98);
                bot.burstMultiplier = safeGetFloat("p-burst-multiplier", 1.0);
                bot.burstCounter = 0;
                
                var jitEnabledEl2 = document.getElementById("p-jit-enabled");
                bot.wagerJitEnabled = jitEnabledEl2 ? jitEnabledEl2.value === 'true' : true;
                
                bot.wagerTotalTarget = safeGetFloat("p-wager-total-target", 10000);
                bot.wagerSesiTarget = safeGetFloat("p-wager-sesi-target", 200);
                bot.wagerLossTolerance = safeGetFloat("p-wager-loss-tolerance", 0.5);
                bot.wagerTotalAccumulated = 0;
                bot.wagerSesiCounter = 0;
                bot.wagerSesiStartBal = 0;
                bot.wagerSesiProfit = 0;
                bot.wagerStop = false;
                bot.wagerWaitingForTrigger = false;
                
                bot.recoveryMaxSplits = safeGetInt("p-recovery-max-splits", 8);
                bot.recoveryCutlossMultiplier = safeGetFloat("p-recovery-cutloss-multiplier", 10);
                bot.recoveryChanceMin = safeGetFloat("p-recovery-min-chance", 0.01);
                bot.recoverySplitCounter = 0;
                
                bot.harvestTrigger = safeGetFloat("p-harvest-trigger", 3);
                bot.harvestTrigger = Math.min(20, Math.max(1, bot.harvestTrigger));
                
                bot.delayNormal = safeGetInt("p-delay-normal", 10);
                bot.delayHarvest = safeGetInt("p-delay-harvest", 100);
                bot.delayRecovery = safeGetInt("p-delay-recovery", 200);
                bot.delayNormal = Math.min(500, Math.max(0, bot.delayNormal));
                bot.delayHarvest = Math.min(500, Math.max(0, bot.delayHarvest));
                bot.delayRecovery = Math.min(500, Math.max(0, bot.delayRecovery));
                
                await updateFiatRatesOnce();
                
                if (bot.fiatRates.idr > 0) {
                    bot.minBet = bot.minBetIdr / bot.fiatRates.idr;
                    if (bot.minBet < 0.00000001) bot.minBet = 0.00000001;
                } else bot.minBet = 0.0014;
                
                var balance = bot.isDemo ? bot.dirtyBalance : bot.realBalance;
                if (!bot.isDemo && balance === 0) { return; }
                
                startBalanceValue = balance;
                originalSessionStartBalance = balance;
                peakBalanceEver = balance;
                recoveryStartBalance = 0;
                bot.wagerSesiStartBal = balance;
                
                bot.dailyStartBalance = balance;
                bot.dailyProfit = 0;
                bot.dailyTargetReached = false;
                bot.lastDailyReset = new Date(new Date().getFullYear(), new Date().getMonth(), new Date().getDate(), 0, 0, 0).getTime();
                bot.stats = { profit: 0, wagered: 0, startBal: balance, currentBal: balance, bets: 0, wins: 0, loss: 0, startTime: Date.now(), maxDD: 0, maxBet: 0, maxDrawdownPct: 0, peakBalance: balance };
                bot.betTimestamps = [];
                bot.startTime = Date.now();
                bot.sessionStartTime = new Date();
                bot.isHarvesting = false;
                bot.isRecoveryMode = false;
                bot.isAfterRecovery = false;
                bot.recoveryStep = 0;
                bot.isTrailingActive = false;
                bot.trailingLevel = 0;
                bot.balancePeak = 0;
                bot.level1Peak = 0;
                bot.level2Peak = 0;
                bot.gapPct = bot.gapLevel1;
                bot.gapHits = 0;
                bot.gapHitProcessed = false;
                bot.harvestCount = 0;
                bot.harvestBet = 0;
                bot.lastBetAmount = 0;
                bot.sessionProfit = 0;
                bot.totalWager = 0;
                bot.highestProfit = 0;
                bot.maxDrawdownVal = 0;
                bot.balanceStartSnapshot = balance;
                bot.isRunning = true;
                bot.isPaused = false;
                bot.isSleeping = false;
                bot.lastTGReport = Date.now();
                bot.lastFiatUpdate = 0;
                bot.isResetting = false;
                bot.resetCountdownRemaining = 0;
                bot.worstDrawdown = 0;
                bot.worstDrawdownChance = 0;
                bot.worstDrawdownLS = 0;
                bot.worstBet = 0;
                bot.sessionNumber = 1;
                bot.sessionsCompleted = 0;
                bot.recovery = null;
                bot.isBoosterMode = false;
                bot.boosterStartProfit = 0;
                bot.boosterBetAmount = 0;
                bot.boosterChance = bot.normalChance;
                boosterChanceIndex = 0;
                diceInFlight = 0;
                pendingDiceMeta = [];
                isResting = false;
                normalCycleStartTime = 0;
                harvestingEntries = [];
                profitHistory = [];
                chartData = [];
                chartHarvestNumber = 0;
                
                initVirtualChances();
                initMemory();
                
                toggleStartStop(true);
                togglePauseButton(false);
                var startBtnEl = document.getElementById("p-start");
                if (startBtnEl) startBtnEl.textContent = "START";
                
                var burstStatus = bot.wagerJitEnabled && bot.burstEveryN > 0 ? 'ON (Every ' + bot.burstEveryN + ' bet)' : 'OFF';
                var wagerStatus = bot.wagerTotalTarget + ' total | ' + bot.wagerSesiTarget + ' per sesi | tolerance ' + bot.wagerLossTolerance;
                var recoveryStatus = 'Max Split: ' + bot.recoveryMaxSplits + ' | Cutloss: ' + bot.recoveryCutlossMultiplier + '× | Min Chance: ' + bot.recoveryChanceMin + '%';
                
                sendTelegramMessage("⭐ ORION ASCEND v3.2\n\"Fire & Forget Wager Farming\"\n🔥 STARTED\nBalance: " + balance.toFixed(2) + "\nNormal Chance: " + bot.normalChance + "%\nJIT: " + burstStatus + "\nJIT Chance: " + bot.burstMinChance + "%-" + bot.burstMaxChance + "%\nWager: " + wagerStatus + "\nRecovery: " + recoveryStatus + "\nHarvest Trigger: " + bot.harvestTrigger + "× payout\nVirtual Range: " + bot.virtMinChance + "%-" + bot.virtMaxChance + "%\nChart: Disabled (Lightweight mode)\nBooster: Active (profit > loss tolerance)\n⚡ Fire & Forget: ON (Parallel " + wagerParallelCap + ")");
                
                updateBurstDisplayUI();
                updateHuntTab();
                updateWagerDisplayUI();
                runLoop();
            };
        }
        
        function pauseBot() {
            if (!bot.isRunning || bot.isPaused) return;
            bot.isPaused = true;
            bot.isRunning = false;
            togglePauseButton(true);
        }

        var pauseBtn = document.getElementById("p-pause");
        if (pauseBtn) {
            pauseBtn.onclick = function() {
                if (bot.isRunning && !bot.isPaused) {
                    pauseBot();
                }
            };
        }
        
        var stopBtn = document.getElementById("p-stop");
        if (stopBtn) {
            stopBtn.onclick = async function() {
                fullReset();
                await sendSessionReport();
            };
        }
        
        var resetBtn = document.getElementById("p-reset");
        if (resetBtn) {
            resetBtn.onclick = function() {
                if (bot.isRunning) { return; }
                fullReset();
            };
        }
        
        var currencyEl = document.getElementById("p-currency");
        if (currencyEl) {
            currencyEl.onchange = function() { 
                bot.selectedCurrency = document.getElementById("p-currency").value; 
                bot.fiatRateFetched = false;
                updateFiatRatesOnce();
                drawGapVisual();
            };
        }
        var minBetIdrEl = document.getElementById("p-minbet-idr");
        if (minBetIdrEl) {
            minBetIdrEl.onchange = function() {
                var newIdr = parseFloat(this.value) || 2;
                bot.minBetIdr = newIdr;
                if (bot.fiatRates.idr > 0) {
                    var newCrypto = newIdr / bot.fiatRates.idr;
                    var infoEl = document.getElementById("minbet-info");
                    if (infoEl) infoEl.innerHTML = "≈ " + newCrypto.toFixed(8) + " " + bot.selectedCurrency.toUpperCase();
                    bot.minBet = newCrypto;
                }
            };
        }
        
        setInterval(updateStatsDisplay, 1000);
        setInterval(function() { 
            if (bot.fiatRates.idr > 0) {
                var cryptoVal = bot.minBetIdr / bot.fiatRates.idr;
                var infoEl = document.getElementById("minbet-info");
                if (infoEl) infoEl.innerHTML = "≈ " + cryptoVal.toFixed(8) + " " + bot.selectedCurrency.toUpperCase();
                bot.minBet = cryptoVal;
            }
        }, 3000);
        
        setInterval(function() {
            var huntTab = document.getElementById('tab-hunt');
            if (huntTab && huntTab.classList.contains('active')) {
                updateHuntTab();
            }
        }, 2000);
        
        renderHarvestingEntries();
        updateBurstDisplayUI();
        updateHuntTab();
        updateWagerDisplayUI();
        
        setTimeout(function() {
            showSplash();
        }, 500);
    }
    
    var checkChart = setInterval(function() {
        if (typeof Chart !== 'undefined') {
            clearInterval(checkChart);
            createUI();
            clearInterval(checkChart);
        }
    }, 500);
    
    setTimeout(function() {
        if (typeof Chart === 'undefined') {
            clearInterval(checkChart);
            createUI();
        }
    }, 5000);
})();
