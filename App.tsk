import React, { useState, useEffect, useMemo, useRef } from 'react';
import { 
  LineChart, Line, XAxis, YAxis, Tooltip, ResponsiveContainer, ReferenceLine 
} from 'recharts';
import { 
  TrendingUp, ShieldAlert, Activity, DollarSign, Award, Clock, 
  AlertTriangle, RefreshCw, Zap, Play, CheckCircle2, XCircle
} from 'lucide-react';

// ==========================================
// 1. QUANT & BLACK-SCHOLES MATHEMATICAL ENGINE
// ==========================================

function cdf(x: number): number {
  const a1 = 0.254829592, a2 = -0.284496736, a3 = 1.421413741;
  const a4 = -1.453152027, a5 = 1.061405429, p = 0.3275911;
  const sign = x < 0 ? -1 : 1;
  const absX = Math.abs(x) / Math.sqrt(2.0);
  const t = 1.0 / (1.0 + p * absX);
  const y = 1.0 - (((((a5 * t + a4) * t) + a3) * t + a2) * t + a1) * t * Math.exp(-absX * absX);
  return 0.5 * (1.0 + sign * y);
}

function pdf(x: number): number {
  return Math.exp(-0.5 * x * x) / Math.sqrt(2 * Math.PI);
}

interface OptionGreeks {
  price: number;
  delta: number;
  gamma: number;
  theta: number;
  vega: number;
}

function calculateBlackScholes(
  S: number,      // Underlying Spot Price
  K: number,      // Strike Price
  T: number,      // Time to Expiration in years
  r: number,      // Risk-free interest rate
  sigma: number,  // Volatility
  type: 'CALL' | 'PUT'
): OptionGreeks {
  if (T <= 0) {
    const intrinsic = type === 'CALL' ? Math.max(0, S - K) : Math.max(0, K - S);
    return { price: intrinsic, delta: type === 'CALL' ? (S > K ? 1 : 0) : (S < K ? -1 : 0), gamma: 0, theta: 0, vega: 0 };
  }

  const d1 = (Math.log(S / K) + (r + 0.5 * sigma * sigma) * T) / (sigma * Math.sqrt(T));
  const d2 = d1 - sigma * Math.sqrt(T);

  let price = 0;
  let delta = 0;

  if (type === 'CALL') {
    price = S * cdf(d1) - K * Math.exp(-r * T) * cdf(d2);
    delta = cdf(d1);
  } else {
    price = K * Math.exp(-r * T) * cdf(-d2) - S * cdf(-d1);
    delta = cdf(d1) - 1;
  }

  const gamma = pdf(d1) / (S * sigma * Math.sqrt(T));
  const vega = (S * pdf(d1) * Math.sqrt(T)) / 100; // per 1% change
  const thetaCall = (-(S * pdf(d1) * sigma) / (2 * Math.sqrt(T)) - r * K * Math.exp(-r * T) * cdf(d2)) / 365;
  const thetaPut = (-(S * pdf(d1) * sigma) / (2 * Math.sqrt(T)) + r * K * Math.exp(-r * T) * cdf(-d2)) / 365;
  const theta = type === 'CALL' ? thetaCall : thetaPut;

  return {
    price: Math.max(0.01, price),
    delta,
    gamma,
    theta,
    vega
  };
}

// ==========================================
// 2. TYPES AND INTERFACES
// ==========================================

export interface OptionContract {
  id: string;
  ticker: string;
  strike: number;
  type: 'CALL' | 'PUT';
  dte: number;
  iv: number;
  bid: number;
  ask: number;
  greeks: OptionGreeks;
  volume: number;
  openInterest: number;
}

export interface Position {
  id: string;
  type: 'STOCK' | 'OPTION';
  symbol: string;
  contract?: OptionContract;
  qty: number; // positive = long, negative = short
  entryPrice: number;
  currentPrice: number;
  marginReq: number;
}

export interface OrderBookLevel {
  price: number;
  size: number;
  total: number;
}

export interface ExecutionLog {
  id: string;
  time: string;
  msg: string;
  type: 'EXEC' | 'WARN' | 'EVENT' | 'GAME';
}

export type GameMode = 'FREE_PLAY' | 'IV_CRUSH' | 'GAMMA_SCALP' | 'MARGIN_RESCUE';

// ==========================================
// 3. MAIN TERMINAL APPLICATION
// ==========================================

export default function QuantTerminal() {
  // --- Market State ---
  const [spotPrice, setSpotPrice] = useState<number>(185.00);
  const [baseIV, setBaseIV] = useState<number>(0.35); // 35%
  const [highVolMode, setHighVolMode] = useState<boolean>(false);
  const [riskFreeRate] = useState<number>(0.045); // 4.5%
  
  // --- Portfolio State ---
  const [cashBalance, setCashBalance] = useState<number>(100000);
  const [positions, setPositions] = useState<Position[]>([]);
  const [logs, setLogs] = useState<ExecutionLog[]>([]);

  // --- Selected Trading Target ---
  const [selectedStrike, setSelectedStrike] = useState<number>(185);
  const [selectedType, setSelectedType] = useState<'CALL' | 'PUT'>('CALL');
  const [tradeQty, setTradeQty] = useState<number>(1);

  // --- Mini Game Engine State ---
  const [activeMode, setActiveMode] = useState<GameMode>('FREE_PLAY');
  const [gameTimer, setGameTimer] = useState<number>(0);
  const [gameScore, setGameScore] = useState<number | null>(null);
  const [gameStatus, setGameStatus] = useState<'IDLE' | 'RUNNING' | 'WON' | 'LOST'>('IDLE');
  const [gameTargetScore, setGameTargetScore] = useState<string>('');

  const addLog = (msg: string, type: 'EXEC' | 'WARN' | 'EVENT' | 'GAME' = 'EXEC') => {
    const time = new Date().toLocaleTimeString();
    setLogs(prev => [{ id: Math.random().toString(), time, msg, type }, ...prev.slice(0, 49)]);
  };

  // --- Generator: Option Chain Data ---
  const optionChain = useMemo(() => {
    const strikes = [165, 170, 175, 180, 185, 190, 195, 200, 205];
    const T = 30 / 365; // 30 DTE
    const spreadMultiplier = highVolMode ? 2.5 : 1.0;

    const chain: { strike: number; call: OptionContract; put: OptionContract }[] = [];

    strikes.forEach(K => {
      const callGreeks = calculateBlackScholes(spotPrice, K, T, riskFreeRate, baseIV, 'CALL');
      const putGreeks = calculateBlackScholes(spotPrice, K, T, riskFreeRate, baseIV, 'PUT');

      const callSpread = Math.max(0.05, callGreeks.price * 0.02 * spreadMultiplier);
      const putSpread = Math.max(0.05, putGreeks.price * 0.02 * spreadMultiplier);

      const call: OptionContract = {
        id: `NVDA-30D-${K}-C`,
        ticker: 'NVDA',
        strike: K,
        type: 'CALL',
        dte: 30,
        iv: baseIV,
        bid: parseFloat((callGreeks.price - callSpread / 2).toFixed(2)),
        ask: parseFloat((callGreeks.price + callSpread / 2).toFixed(2)),
        greeks: callGreeks,
        volume: Math.floor(Math.abs(Math.sin(K)) * 1400) + 120,
        openInterest: Math.floor(Math.abs(Math.cos(K)) * 8000) + 500,
      };

      const put: OptionContract = {
        id: `NVDA-30D-${K}-P`,
        ticker: 'NVDA',
        strike: K,
        type: 'PUT',
        dte: 30,
        iv: baseIV,
        bid: parseFloat((putGreeks.price - putSpread / 2).toFixed(2)),
        ask: parseFloat((putGreeks.price + putSpread / 2).toFixed(2)),
        greeks: putGreeks,
        volume: Math.floor(Math.abs(Math.cos(K)) * 1200) + 100,
        openInterest: Math.floor(Math.abs(Math.sin(K)) * 7500) + 400,
      };

      chain.push({ strike: K, call, put });
    });

    return chain;
  }, [spotPrice, baseIV, highVolMode, riskFreeRate]);

  // --- Portfolio & Greek Calculations ---
  const portfolioMetrics = useMemo(() => {
    let marketValue = 0;
    let totalMarginReq = 0;
    let netDelta = 0;
    let netGamma = 0;
    let netTheta = 0;
    let netVega = 0;
    let unrealizedPnL = 0;

    positions.forEach(pos => {
      let currentVal = 0;
      let unitPrice = 0;

      if (pos.type === 'STOCK') {
        unitPrice = spotPrice;
        currentVal = pos.qty * unitPrice;
        netDelta += pos.qty;
      } else if (pos.contract) {
        const T = pos.contract.dte / 365;
        const liveGreeks = calculateBlackScholes(
          spotPrice, 
          pos.contract.strike, 
          T, 
          riskFreeRate, 
          baseIV, 
          pos.contract.type
        );
        unitPrice = liveGreeks.price;
        currentVal = pos.qty * unitPrice * 100;

        netDelta += liveGreeks.delta * pos.qty * 100;
        netGamma += liveGreeks.gamma * pos.qty * 100;
        netTheta += liveGreeks.theta * pos.qty * 100;
        netVega += liveGreeks.vega * pos.qty * 100;
      }

      const positionPnL = (unitPrice - pos.entryPrice) * pos.qty * (pos.type === 'OPTION' ? 100 : 1);
      unrealizedPnL += positionPnL;
      marketValue += currentVal;
      totalMarginReq += pos.marginReq;
    });

    const netLiquidationValue = cashBalance + marketValue;
    const maintenanceMargin = totalMarginReq;
    const availableBuyingPower = Math.max(0, netLiquidationValue - maintenanceMargin);

    return {
      netLiquidationValue,
      maintenanceMargin,
      availableBuyingPower,
      unrealizedPnL,
      netDelta,
      netGamma,
      netTheta,
      netVega
    };
  }, [positions, cashBalance, spotPrice, baseIV, riskFreeRate]);

  // ==========================================
  // 4. REAL-TIME TICKER & RANDOM EVENT SIMULATOR
  // ==========================================

  useEffect(() => {
    const interval = setInterval(() => {
      // Small stock price random walk
      const delta = (Math.random() - 0.49) * (highVolMode ? 1.2 : 0.35);
      setSpotPrice(prev => parseFloat(Math.max(10, prev + delta).toFixed(2)));

      // Random Market Shock Engine (1% chance per tick in Free Play)
      if (activeMode === 'FREE_PLAY' && Math.random() < 0.02) {
        const eventType = Math.floor(Math.random() * 3);
        if (eventType === 0) {
          addLog("EVENT: Volatility Crush! Industry IV dropped by 20%", "EVENT");
          setBaseIV(prev => Math.max(0.15, prev * 0.8));
        } else if (eventType === 1) {
          addLog("EVENT: Federal Reserve Signal! Interest rate shift triggered volatility spike.", "EVENT");
          setBaseIV(prev => prev * 1.2);
        } else {
          const shift = (Math.random() > 0.5 ? 1 : -1) * (spotPrice * 0.04);
          addLog(`EVENT: Major Block Trade! Ticker moved by $${shift.toFixed(2)}`, "EVENT");
          setSpotPrice(prev => parseFloat((prev + shift).toFixed(2)));
        }
      }
    }, 1000);

    return () => clearInterval(interval);
  }, [highVolMode, activeMode, spotPrice]);

  // ==========================================
  // 5. MINI-GAME SCENARIO ENGINES
  // ==========================================

  const startMiniGame = (mode: GameMode) => {
    setActiveMode(mode);
    setGameStatus('RUNNING');
    setGameScore(null);

    if (mode === 'IV_CRUSH') {
      setSpotPrice(200.00);
      setBaseIV(1.20); // 120% IV
      setCashBalance(50000);
      setGameTimer(30); // 30 seconds to trade
      setPositions([]);
      setGameTargetScore("Construct a Delta-Neutral trade (e.g. Short Iron Condor/Straddle) to stay green post-earnings.");
      addLog("GAME STARTED: Earnings in 30 seconds. IV is 120%. Build a delta-neutral structure!", "GAME");
    } else if (mode === 'GAMMA_SCALP') {
      setSpotPrice(100.00);
      setBaseIV(0.50);
      setCashBalance(25000);
      setGameTimer(45);
      
      // Give initial Long Straddle
      const T = 30 / 365;
      const call = calculateBlackScholes(100, 100, T, riskFreeRate, 0.50, 'CALL');
      const put = calculateBlackScholes(100, 100, T, riskFreeRate, 0.50, 'PUT');

      const initialPositions: Position[] = [
        {
          id: 'GAME-STRADDLE-CALL',
          type: 'OPTION',
          symbol: 'NVDA $100 CALL',
          contract: { id: 'C100', ticker: 'NVDA', strike: 100, type: 'CALL', dte: 30, iv: 0.5, bid: call.price, ask: call.price, greeks: call, volume: 100, openInterest: 100 },
          qty: 10,
          entryPrice: call.price,
          currentPrice: call.price,
          marginReq: call.price * 10 * 100
        },
        {
          id: 'GAME-STRADDLE-PUT',
          type: 'OPTION',
          symbol: 'NVDA $100 PUT',
          contract: { id: 'P100', ticker: 'NVDA', strike: 100, type: 'PUT', dte: 30, iv: 0.5, bid: put.price, ask: put.price, greeks: put, volume: 100, openInterest: 100 },
          qty: 10,
          entryPrice: put.price,
          currentPrice: put.price,
          marginReq: put.price * 10 * 100
        }
      ];

      setPositions(initialPositions);
      setGameTargetScore("Keep Net Delta within [-50, +50] by scalping stock as price swings violently!");
      addLog("GAME STARTED: Long Straddle active. Trade underlying shares to keep Net Delta near 0!", "GAME");
    } else if (mode === 'MARGIN_RESCUE') {
      setSpotPrice(150.00);
      setBaseIV(0.40);
      setCashBalance(1000);
      setGameTimer(30);

      // Over-leveraged toxic positions
      const toxicCall = calculateBlackScholes(150, 140, 30/365, riskFreeRate, 0.40, 'CALL');
      const toxicPos: Position[] = [
        {
          id: 'TOXIC-SHORT-CALL',
          type: 'OPTION',
          symbol: 'NVDA $140 CALL (SHORT)',
          contract: { id: 'C140', ticker: 'NVDA', strike: 140, type: 'CALL', dte: 30, iv: 0.4, bid: toxicCall.price, ask: toxicCall.price, greeks: toxicCall, volume: 100, openInterest: 100 },
          qty: -15,
          entryPrice: 2.00,
          currentPrice: toxicCall.price,
          marginReq: 45000 // Huge margin penalty
        }
      ];

      setPositions(toxicPos);
      setGameTargetScore("Close/Hedge toxic short legs to get Available Buying Power > $0 before timer expires!");
      addLog("GAME STARTED: MARGIN CALL WARNING! Buying power negative. Defuse toxic short positions!", "GAME");
    }
  };

  // Game Loop Timer logic
  useEffect(() => {
    if (gameStatus !== 'RUNNING' || activeMode === 'FREE_PLAY') return;

    const timer = setInterval(() => {
      setGameTimer(prev => {
        if (prev <= 1) {
          clearInterval(timer);
          evaluateGameEnd();
          return 0;
        }

        // Intra-game event triggers
        if (activeMode === 'GAMMA_SCALP') {
          // Push wild swings
          const swing = (Math.random() - 0.5) * 4.5;
          setSpotPrice(sp => parseFloat((sp + swing).toFixed(2)));
        }

        return prev - 1;
      });
    }, 1000);

    return () => clearInterval(timer);
  }, [gameStatus, activeMode, portfolioMetrics]);

  const evaluateGameEnd = () => {
    if (activeMode === 'IV_CRUSH') {
      // Trigger instant IV crush down to 30%
      setBaseIV(0.30);
      const finalPnL = portfolioMetrics.unrealizedPnL;
      if (finalPnL > 500 && Math.abs(portfolioMetrics.netDelta) < 150) {
        setGameStatus('WON');
        setGameScore(finalPnL);
        addLog(`CHALLENGE PASSED! You crushed volatility and made +$${finalPnL.toFixed(2)}`, "GAME");
      } else {
        setGameStatus('LOST');
        setGameScore(finalPnL);
        addLog(`CHALLENGE FAILED! Net PnL: $${finalPnL.toFixed(2)}. Unhedged Delta or bad structure.`, "WARN");
      }
    } else if (activeMode === 'GAMMA_SCALP') {
      const deltaDev = Math.abs(portfolioMetrics.netDelta);
      if (deltaDev <= 60 && portfolioMetrics.unrealizedPnL > -1000) {
        setGameStatus('WON');
        setGameScore(portfolioMetrics.unrealizedPnL);
        addLog(`CHALLENGE PASSED! Excellent delta discipline. Final Delta Dev: ${deltaDev.toFixed(1)}`, "GAME");
      } else {
        setGameStatus('LOST');
        setGameScore(portfolioMetrics.unrealizedPnL);
        addLog(`CHALLENGE FAILED! Delta drifted too far (${deltaDev.toFixed(1)}). You blew up from gamma exposure.`, "WARN");
      }
    } else if (activeMode === 'MARGIN_RESCUE') {
      if (portfolioMetrics.availableBuyingPower > 0) {
        setGameStatus('WON');
        setGameScore(portfolioMetrics.availableBuyingPower);
        addLog(`RESCUE SUCCESSFUL! Account saved from liquidation. Excess Margin: $${portfolioMetrics.availableBuyingPower.toFixed(2)}`, "GAME");
      } else {
        setGameStatus('LOST');
        setGameScore(portfolioMetrics.availableBuyingPower);
        addLog("RESCUE FAILED! Risk desk force-liquidated your remaining positions.", "WARN");
      }
    }
  };

  // ==========================================
  // 6. ORDER EXECUTION SYSTEM
  // ==========================================

  const executeTrade = (
    type: 'STOCK' | 'OPTION',
    side: 'BUY' | 'SELL',
    qty: number,
    contract?: OptionContract
  ) => {
    const actualQty = side === 'BUY' ? qty : -qty;

    if (type === 'STOCK') {
      const executionPrice = side === 'BUY' ? spotPrice + 0.05 : spotPrice - 0.05; // Slippage
      const cost = executionPrice * qty;

      if (side === 'BUY' && portfolioMetrics.availableBuyingPower < cost) {
        addLog("REJECTED: Insufficient Buying Power for Stock Purchase", "WARN");
        return;
      }

      setCashBalance(prev => prev - (executionPrice * actualQty));
      
      setPositions(prev => {
        const existing = prev.find(p => p.type === 'STOCK');
        if (existing) {
          const newQty = existing.qty + actualQty;
          if (newQty === 0) return prev.filter(p => p.id !== existing.id);
          return prev.map(p => p.id === existing.id ? { ...p, qty: newQty } : p);
        }
        return [...prev, {
          id: `STOCK-NVDA-${Date.now()}`,
          type: 'STOCK',
          symbol: 'NVDA Shares',
          qty: actualQty,
          entryPrice: executionPrice,
          currentPrice: executionPrice,
          marginReq: executionPrice * Math.abs(actualQty) * 0.5 // 50% margin
        }];
      });

      addLog(`EXECUTED: ${side} ${qty} NVDA Shares @ $${executionPrice.toFixed(2)}`);
    } 
    else if (type === 'OPTION' && contract) {
      const fillPrice = side === 'BUY' ? contract.ask : contract.bid;
      const totalPremium = fillPrice * qty * 100;

      if (side === 'BUY' && portfolioMetrics.availableBuyingPower < totalPremium) {
        addLog("REJECTED: Insufficient Buying Power for Option Premium", "WARN");
        return;
      }

      const marginPerContract = side === 'SELL' ? (contract.strike * 100 * 0.20) : 0; // Short option margin requirement

      setCashBalance(prev => prev - (fillPrice * actualQty * 100));

      setPositions(prev => {
        const existing = prev.find(p => p.contract?.id === contract.id);
        if (existing) {
          const newQty = existing.qty + actualQty;
          if (newQty === 0) return prev.filter(p => p.id !== existing.id);
          return prev.map(p => p.id === existing.id ? { ...p, qty: newQty } : p);
        }
        return [...prev, {
          id: `OPT-${contract.id}-${Date.now()}`,
          type: 'OPTION',
          symbol: `${contract.ticker} $${contract.strike} ${contract.type}`,
          contract,
          qty: actualQty,
          entryPrice: fillPrice,
          currentPrice: fillPrice,
          marginReq: marginPerContract * Math.abs(actualQty)
        }];
      });

      addLog(`EXECUTED: ${side} ${qty}x ${contract.ticker} $${contract.strike} ${contract.type} @ $${fillPrice.toFixed(2)}`);
    }
  };

  const closePosition = (id: string) => {
    const pos = positions.find(p => p.id === id);
    if (!pos) return;

    if (pos.type === 'STOCK') {
      executeTrade('STOCK', pos.qty > 0 ? 'SELL' : 'BUY', Math.abs(pos.qty));
    } else if (pos.contract) {
      executeTrade('OPTION', pos.qty > 0 ? 'SELL' : 'BUY', Math.abs(pos.qty), pos.contract);
    }
  };

  // ==========================================
  // 7. PAYOFF GRAPH CURVE GENERATOR
  // ==========================================

  const payoffData = useMemo(() => {
    const range = 0.25; // +/- 25% price spectrum
    const minPrice = Math.floor(spotPrice * (1 - range));
    const maxPrice = Math.ceil(spotPrice * (1 + range));
    const step = Math.max(1, Math.floor((maxPrice - minPrice) / 30));

    const points = [];

    for (let p = minPrice; p <= maxPrice; p += step) {
      let expPnL = 0;
      let currPnL = 0;

      positions.forEach(pos => {
        if (pos.type === 'STOCK') {
          expPnL += (p - pos.entryPrice) * pos.qty;
          currPnL += (p - pos.entryPrice) * pos.qty;
        } else if (pos.contract) {
          // Expiration PnL
          const intrinsicAtExpiry = pos.contract.type === 'CALL' 
            ? Math.max(0, p - pos.contract.strike) 
            : Math.max(0, pos.contract.strike - p);
          
          expPnL += (intrinsicAtExpiry - pos.entryPrice) * pos.qty * 100;

          // Current Model Price at dynamic spot 'p'
          const T = pos.contract.dte / 365;
          const simGreeks = calculateBlackScholes(p, pos.contract.strike, T, riskFreeRate, baseIV, pos.contract.type);
          currPnL += (simGreeks.price - pos.entryPrice) * pos.qty * 100;
        }
      });

      points.push({
        price: p,
        ExpirationPnL: parseFloat(expPnL.toFixed(2)),
        CurrentPnL: parseFloat(currPnL.toFixed(2))
      });
    }

    return points;
  }, [positions, spotPrice, baseIV, riskFreeRate]);

  // Selected Option for Order Ticket
  const currentSelectedContract = useMemo(() => {
    const row = optionChain.find(r => r.strike === selectedStrike);
    return row ? (selectedType === 'CALL' ? row.call : row.put) : null;
  }, [optionChain, selectedStrike, selectedType]);

  // ==========================================
  // 8. RENDER BLOOMBERG-STYLE TERMINAL UI
  // ==========================================

  return (
    <div className="min-h-screen bg-slate-950 text-slate-100 font-mono text-xs flex flex-col select-none border-t-2 border-cyan-500">
      
      {/* HEADER / TOP TICKER BAR */}
      <header className="bg-slate-900 border-b border-slate-800 p-2 flex flex-wrap items-center justify-between gap-4">
        <div className="flex items-center gap-3">
          <div className="bg-cyan-500/10 text-cyan-400 p-1.5 rounded border border-cyan-500/30 font-bold flex items-center gap-2">
            <Activity className="w-4 h-4 text-cyan-400" /> QUANT-DESK v4.2
          </div>
          <div className="flex items-center gap-2 bg-slate-950 px-2.5 py-1 rounded border border-slate-800">
            <span className="text-slate-400">TICKER:</span>
            <span className="font-bold text-yellow-400">NVDA</span>
            <span className="text-slate-500">|</span>
            <span className="text-slate-400">SPOT:</span>
            <span className="font-bold text-emerald-400">${spotPrice.toFixed(2)}</span>
            <span className="text-slate-500">|</span>
            <span className="text-slate-400">IV:</span>
            <span className="font-bold text-cyan-300">{(baseIV * 100).toFixed(1)}%</span>
          </div>
        </div>

        {/* Volatility Control Toggle */}
        <div className="flex items-center gap-4">
          <label className="flex items-center gap-2 cursor-pointer bg-slate-950 px-2 py-1 rounded border border-slate-800 hover:border-slate-700">
            <input 
              type="checkbox" 
              checked={highVolMode} 
              onChange={e => setHighVolMode(e.target.checked)} 
              className="accent-cyan-500"
            />
            <span className={highVolMode ? "text-rose-400 font-bold" : "text-slate-400"}>
              {highVolMode ? "HIGH VOL SPREADS ACTIVE" : "NORMAL SPREADS"}
            </span>
          </label>

          <button 
            onClick={() => { setCashBalance(100000); setPositions([]); setActiveMode('FREE_PLAY'); addLog("Terminal state fully reset."); }}
            className="flex items-center gap-1 bg-slate-800 hover:bg-slate-700 text-slate-300 px-2.5 py-1 rounded transition"
          >
            <RefreshCw className="w-3 h-3" /> RESET
          </button>
        </div>
      </header>

      {/* MINI-GAME MODES BANNER */}
      <section className="bg-slate-900/60 border-b border-slate-800 p-2 flex flex-wrap items-center justify-between gap-2">
        <div className="flex items-center gap-2">
          <Award className="w-4 h-4 text-amber-400" />
          <span className="font-bold text-slate-300 uppercase tracking-wider">Quant Challenges:</span>
        </div>

        <div className="flex items-center gap-2">
          <button 
            onClick={() => startMiniGame('IV_CRUSH')}
            className={`px-2.5 py-1 rounded border font-semibold flex items-center gap-1 transition ${activeMode === 'IV_CRUSH' ? 'bg-amber-500/20 border-amber-500 text-amber-300' : 'bg-slate-950 border-slate-800 text-slate-400 hover:border-amber-500/50'}`}
          >
            <Zap className="w-3 h-3 text-amber-400" /> IV Crush
          </button>
          <button 
            onClick={() => startMiniGame('GAMMA_SCALP')}
            className={`px-2.5 py-1 rounded border font-semibold flex items-center gap-1 transition ${activeMode === 'GAMMA_SCALP' ? 'bg-cyan-500/20 border-cyan-500 text-cyan-300' : 'bg-slate-950 border-slate-800 text-slate-400 hover:border-cyan-500/50'}`}
          >
            <TrendingUp className="w-3 h-3 text-cyan-400" /> Gamma Scalper
          </button>
          <button 
            onClick={() => startMiniGame('MARGIN_RESCUE')}
            className={`px-2.5 py-1 rounded border font-semibold flex items-center gap-1 transition ${activeMode === 'MARGIN_RESCUE' ? 'bg-rose-500/20 border-rose-500 text-rose-300' : 'bg-slate-950 border-slate-800 text-slate-400 hover:border-rose-500/50'}`}
          >
            <ShieldAlert className="w-3 h-3 text-rose-400" /> Margin Rescue
          </button>
        </div>

        {/* Active Game Stats Display */}
        {activeMode !== 'FREE_PLAY' && (
          <div className="flex items-center gap-4 bg-slate-950 px-3 py-1 rounded border border-amber-500/40">
            <div className="flex items-center gap-1 text-amber-400">
              <Clock className="w-3.5 h-3.5" />
              <span className="font-bold text-sm">{gameTimer}s</span>
            </div>
            <div className="text-slate-300 max-w-md truncate">{gameTargetScore}</div>
            {gameStatus === 'WON' && <span className="text-emerald-400 font-bold flex items-center gap-1"><CheckCircle2 className="w-4 h-4"/> PASSED</span>}
            {gameStatus === 'LOST' && <span className="text-rose-400 font-bold flex items-center gap-1"><XCircle className="w-4 h-4"/> FAILED</span>}
          </div>
        )}
      </section>

      {/* MAIN DASHBOARD GRID */}
      <div className="flex-1 grid grid-cols-12 gap-2 p-2 overflow-hidden">
        
        {/* LEFT COLUMN: OPTION CHAIN MATRIX (6 COLS) */}
        <div className="col-span-12 lg:col-span-6 bg-slate-900 rounded border border-slate-800 flex flex-col overflow-hidden">
          <div className="bg-slate-950 p-2 border-b border-slate-800 font-bold text-slate-400 flex justify-between items-center">
            <span>OPTION CHAIN MATRIX (30 DTE)</span>
            <span className="text-slate-600 text-3xs">CLICK ROW TO LOAD ORDER TICKET</span>
          </div>

          <div className="flex-1 overflow-auto">
            <table className="w-full text-center border-collapse">
              <thead className="bg-slate-950 text-slate-500 sticky top-0 border-b border-slate-800">
                <tr>
                  <th colSpan={4} className="py-1 border-r border-slate-800 text-emerald-400 bg-emerald-950/20">CALLS</th>
                  <th className="py-1 bg-slate-900 text-yellow-400">STRIKE</th>
                  <th colSpan={4} className="py-1 border-l border-slate-800 text-rose-400 bg-rose-950/20">PUTS</th>
                </tr>
                <tr className="border-b border-slate-800 text-2xs">
                  <th className="py-1">BID</th>
                  <th>ASK</th>
                  <th>DELTA</th>
                  <th className="border-r border-slate-800">IV</th>
                  <th className="bg-slate-900"></th>
                  <th className="border-l border-slate-800">BID</th>
                  <th>ASK</th>
                  <th>DELTA</th>
                  <th>IV</th>
                </tr>
              </thead>
              <tbody className="divide-y divide-slate-800/50">
                {optionChain.map((row) => {
                  const isAtm = Math.abs(row.strike - spotPrice) < 3;
                  const isSelectedCall = selectedStrike === row.strike && selectedType === 'CALL';
                  const isSelectedPut = selectedStrike === row.strike && selectedType === 'PUT';

                  return (
                    <tr key={row.strike} className={`hover:bg-slate-800/40 transition ${isAtm ? 'bg-yellow-500/5' : ''}`}>
                      {/* CALL BID/ASK */}
                      <td 
                        onClick={() => { setSelectedStrike(row.strike); setSelectedType('CALL'); }} 
                        className={`py-1.5 cursor-pointer hover:bg-emerald-500/20 text-emerald-400 ${isSelectedCall ? 'bg-emerald-500/30 font-bold' : ''}`}
                      >
                        {row.call.bid.toFixed(2)}
                      </td>
                      <td 
                        onClick={() => { setSelectedStrike(row.strike); setSelectedType('CALL'); }}
                        className={`py-1.5 cursor-pointer hover:bg-emerald-500/20 text-emerald-300 ${isSelectedCall ? 'bg-emerald-500/30 font-bold' : ''}`}
                      >
                        {row.call.ask.toFixed(2)}
                      </td>
                      <td className="text-slate-400">{row.call.greeks.delta.toFixed(2)}</td>
                      <td className="text-slate-500 border-r border-slate-800">{(row.call.iv * 100).toFixed(0)}%</td>

                      {/* STRIKE */}
                      <td className={`font-bold bg-slate-950 ${isAtm ? 'text-yellow-400 underline' : 'text-slate-200'}`}>
                        {row.strike}
                      </td>

                      {/* PUT BID/ASK */}
                      <td 
                        onClick={() => { setSelectedStrike(row.strike); setSelectedType('PUT'); }}
                        className={`py-1.5 border-l border-slate-800 cursor-pointer hover:bg-rose-500/20 text-rose-400 ${isSelectedPut ? 'bg-rose-500/30 font-bold' : ''}`}
                      >
                        {row.put.bid.toFixed(2)}
                      </td>
                      <td 
                        onClick={() => { setSelectedStrike(row.strike); setSelectedType('PUT'); }}
                        className={`py-1.5 cursor-pointer hover:bg-rose-500/20 text-rose-300 ${isSelectedPut ? 'bg-rose-500/30 font-bold' : ''}`}
                      >
                        {row.put.ask.toFixed(2)}
                      </td>
                      <td className="text-slate-400">{row.put.greeks.delta.toFixed(2)}</td>
                      <td className="text-slate-500">{(row.put.iv * 100).toFixed(0)}%</td>
                    </tr>
                  );
                })}
              </tbody>
            </table>
          </div>

          {/* QUICK UNDERLYING SHARES SCALPER */}
          <div className="p-2 border-t border-slate-800 bg-slate-950 flex items-center justify-between">
            <span className="text-slate-400">UNDERLYING SHARES TRADER:</span>
            <div className="flex gap-2">
              <button 
                onClick={() => executeTrade('STOCK', 'BUY', 100)}
                className="bg-emerald-600 hover:bg-emerald-500 text-white px-3 py-1 rounded font-bold transition"
              >
                BUY 100 NVDA @ ${spotPrice.toFixed(2)}
              </button>
              <button 
                onClick={() => executeTrade('STOCK', 'SELL', 100)}
                className="bg-rose-600 hover:bg-rose-500 text-white px-3 py-1 rounded font-bold transition"
              >
                SHORT 100 NVDA @ ${spotPrice.toFixed(2)}
              </button>
            </div>
          </div>
        </div>

        {/* MIDDLE COLUMN: ORDER TICKET & LIVE ORDER BOOK (3 COLS) */}
        <div className="col-span-12 md:col-span-6 lg:col-span-3 flex flex-col gap-2">
          
          {/* INSTITUTIONAL ORDER TICKET */}
          <div className="bg-slate-900 rounded border border-slate-800 p-2.5 flex flex-col gap-2">
            <div className="border-b border-slate-800 pb-1.5 font-bold text-slate-300 flex justify-between">
              <span>ORDER TICKET</span>
              <span className="text-cyan-400">{currentSelectedContract?.id}</span>
            </div>

            {currentSelectedContract && (
              <div className="space-y-2">
                <div className="grid grid-cols-2 gap-1 text-2xs bg-slate-950 p-1.5 rounded border border-slate-800">
                  <div><span className="text-slate-500">Theoretical:</span> ${currentSelectedContract.greeks.price.toFixed(2)}</div>
                  <div><span className="text-slate-500">Delta:</span> {currentSelectedContract.greeks.delta.toFixed(3)}</div>
                  <div><span className="text-slate-500">Gamma:</span> {currentSelectedContract.greeks.gamma.toFixed(3)}</div>
                  <div><span className="text-slate-500">Theta:</span> {currentSelectedContract.greeks.theta.toFixed(3)}</div>
                </div>

                <div className="flex items-center justify-between">
                  <span className="text-slate-400">QUANTITY (CONTRACTS):</span>
                  <input 
                    type="number" 
                    min="1" 
                    max="100" 
                    value={tradeQty} 
                    onChange={e => setTradeQty(Math.max(1, parseInt(e.target.value) || 1))}
                    className="bg-slate-950 border border-slate-700 rounded w-16 px-2 py-0.5 text-right font-bold text-yellow-400"
                  />
                </div>

                <div className="grid grid-cols-2 gap-2 pt-1">
                  <button 
                    onClick={() => executeTrade('OPTION', 'BUY', tradeQty, currentSelectedContract)}
                    className="bg-emerald-600 hover:bg-emerald-500 text-white font-bold py-1.5 rounded transition flex flex-col items-center"
                  >
                    <span>BUY / LONG</span>
                    <span className="text-3xs opacity-80">ASK: ${currentSelectedContract.ask.toFixed(2)}</span>
                  </button>
                  <button 
                    onClick={() => executeTrade('OPTION', 'SELL', tradeQty, currentSelectedContract)}
                    className="bg-rose-600 hover:bg-rose-500 text-white font-bold py-1.5 rounded transition flex flex-col items-center"
                  >
                    <span>SELL / SHORT</span>
                    <span className="text-3xs opacity-80">BID: ${currentSelectedContract.bid.toFixed(2)}</span>
                  </button>
                </div>
              </div>
            )}
          </div>

          {/* DYNAMIC REAL-TIME ORDER BOOK LADDER */}
          <div className="flex-1 bg-slate-900 rounded border border-slate-800 p-2 flex flex-col overflow-hidden">
            <div className="border-b border-slate-800 pb-1 font-bold text-slate-400 flex justify-between">
              <span>MARKET DEPTH</span>
              <span className="text-slate-500">LEVEL II LADDER</span>
            </div>

            <div className="flex-1 overflow-auto mt-2 space-y-1">
              {/* ASKS (RED) */}
              <div className="space-y-0.5">
                {[0.03, 0.02, 0.01].map((offset, i) => {
                  const askP = spotPrice + offset;
                  const size = Math.floor(Math.sin(spotPrice + i) * 300) + 100;
                  return (
                    <div key={i} className="relative flex justify-between px-1.5 py-0.5 text-rose-400 bg-rose-950/20 rounded">
                      <div className="absolute left-0 top-0 bottom-0 bg-rose-500/10 rounded" style={{ width: `${Math.min(100, size / 5)}%` }} />
                      <span className="z-10">${askP.toFixed(2)}</span>
                      <span className="z-10 font-bold">{size}</span>
                    </div>
                  );
                })}
              </div>

              {/* SPREAD INDICATOR */}
              <div className="text-center py-1 bg-slate-950 border-y border-slate-800 text-yellow-400 font-bold">
                --- SPOT: ${spotPrice.toFixed(2)} ---
              </div>

              {/* BIDS (GREEN) */}
              <div className="space-y-0.5">
                {[0.01, 0.02, 0.03].map((offset, i) => {
                  const bidP = spotPrice - offset;
                  const size = Math.floor(Math.cos(spotPrice + i) * 300) + 100;
                  return (
                    <div key={i} className="relative flex justify-between px-1.5 py-0.5 text-emerald-400 bg-emerald-950/20 rounded">
                      <div className="absolute left-0 top-0 bottom-0 bg-emerald-500/10 rounded" style={{ width: `${Math.min(100, size / 5)}%` }} />
                      <span className="z-10">${bidP.toFixed(2)}</span>
                      <span className="z-10 font-bold">{size}</span>
                    </div>
                  );
                })}
              </div>
            </div>
          </div>
        </div>

        {/* RIGHT COLUMN: PORTFOLIO RISK, PAYOFF GRAPH & LOGS (3 COLS) */}
        <div className="col-span-12 md:col-span-6 lg:col-span-3 flex flex-col gap-2 overflow-hidden">
          
          {/* ACCOUNT METRICS & AGGREGATED GREEKS */}
          <div className="bg-slate-900 rounded border border-slate-800 p-2.5 space-y-2">
            <div className="font-bold text-slate-300 border-b border-slate-800 pb-1 flex justify-between items-center">
              <span>PORTFOLIO RISK & NAV</span>
              <DollarSign className="w-4 h-4 text-emerald-400" />
            </div>

            <div className="grid grid-cols-2 gap-2">
              <div className="bg-slate-950 p-1.5 rounded border border-slate-800">
                <div className="text-slate-500 text-3xs">NET LIQ VALUE</div>
                <div className="text-sm font-bold text-emerald-400">${portfolioMetrics.netLiquidationValue.toFixed(2)}</div>
              </div>
              <div className="bg-slate-950 p-1.5 rounded border border-slate-800">
                <div className="text-slate-500 text-3xs">AVAIL BUYING PWR</div>
                <div className={`text-sm font-bold ${portfolioMetrics.availableBuyingPower < 0 ? 'text-rose-400 animate-pulse' : 'text-slate-200'}`}>
                  ${portfolioMetrics.availableBuyingPower.toFixed(2)}
                </div>
              </div>
            </div>

            {/* Portfolio Greek Summary Bar */}
            <div className="grid grid-cols-4 gap-1 text-center bg-slate-950 p-1.5 rounded border border-slate-800">
              <div>
                <div className="text-slate-500 text-3xs">Δ DELTA</div>
                <div className={`font-bold ${portfolioMetrics.netDelta >= 0 ? 'text-emerald-400' : 'text-rose-400'}`}>
                  {portfolioMetrics.netDelta.toFixed(1)}
                </div>
              </div>
              <div>
                <div className="text-slate-500 text-3xs">Γ GAMMA</div>
                <div className="font-bold text-cyan-400">{portfolioMetrics.netGamma.toFixed(2)}</div>
              </div>
              <div>
                <div className="text-slate-500 text-3xs">Θ THETA</div>
                <div className="font-bold text-amber-400">{portfolioMetrics.netTheta.toFixed(1)}</div>
              </div>
              <div>
                <div className="text-slate-500 text-3xs">ν VEGA</div>
                <div className="font-bold text-purple-400">{portfolioMetrics.netVega.toFixed(1)}</div>
              </div>
            </div>
          </div>

          {/* PAYOFF CURVE GRAPH */}
          <div className="bg-slate-900 rounded border border-slate-800 p-2 flex flex-col h-44">
            <div className="font-bold text-slate-400 text-3xs mb-1 flex justify-between">
              <span>EXPIRATION vs CURRENT P&L CURVE</span>
              <span className="text-cyan-400">BLUE: Current | PURPLE: Expiry</span>
            </div>
            <div className="flex-1 w-full">
              <ResponsiveContainer width="100%" height="100%">
                <LineChart data={payoffData}>
                  <XAxis dataKey="price" stroke="#475569" fontSize={9} tickLine={false} />
                  <YAxis stroke="#475569" fontSize={9} tickLine={false} />
                  <Tooltip 
                    contentStyle={{ backgroundColor: '#020617', borderColor: '#334155', fontSize: '10px' }} 
                  />
                  <ReferenceLine y={0} stroke="#64748b" strokeDasharray="3 3" />
                  <ReferenceLine x={spotPrice} stroke="#eab308" strokeDasharray="3 3" label={{ value: 'SPOT', fill: '#eab308', fontSize: 8 }} />
                  <Line type="monotone" dataKey="CurrentPnL" stroke="#38bdf8" strokeWidth={1.5} dot={false} />
                  <Line type="monotone" dataKey="ExpirationPnL" stroke="#a855f7" strokeWidth={1.5} dot={false} />
                </LineChart>
              </ResponsiveContainer>
            </div>
          </div>

          {/* ACTIVE POSITIONS LIST */}
          <div className="flex-1 bg-slate-900 rounded border border-slate-800 p-2 flex flex-col overflow-hidden">
            <div className="font-bold text-slate-400 border-b border-slate-800 pb-1 mb-1">
              POSITIONS ({positions.length})
            </div>

            <div className="flex-1 overflow-auto space-y-1">
              {positions.length === 0 ? (
                <div className="text-slate-600 text-center py-4">NO OPEN POSITIONS</div>
              ) : (
                positions.map(pos => {
                  const pnl = pos.type === 'STOCK' 
                    ? (spotPrice - pos.entryPrice) * pos.qty
                    : (pos.contract ? (calculateBlackScholes(spotPrice, pos.contract.strike, pos.contract.dte/365, riskFreeRate, baseIV, pos.contract.type).price - pos.entryPrice) * pos.qty * 100 : 0);

                  return (
                    <div key={pos.id} className="bg-slate-950 p-1.5 rounded border border-slate-800 flex justify-between items-center text-2xs">
                      <div>
                        <div className="font-bold text-slate-200">{pos.symbol}</div>
                        <div className="text-slate-500">QTY: {pos.qty} @ ${pos.entryPrice.toFixed(2)}</div>
                      </div>
                      <div className="text-right">
                        <div className={`font-bold ${pnl >= 0 ? 'text-emerald-400' : 'text-rose-400'}`}>
                          ${pnl.toFixed(2)}
                        </div>
                        <button 
                          onClick={() => closePosition(pos.id)}
                          className="text-3xs text-rose-400 hover:underline uppercase"
                        >
                          CLOSE
                        </button>
                      </div>
                    </div>
                  );
                })
              )}
            </div>
          </div>

        </div>

      </div>

      {/* FOOTER: REAL-TIME EXECUTION LOG TERMINAL */}
      <footer className="h-24 bg-slate-950 border-t border-slate-800 p-2 overflow-y-auto font-mono text-3xs">
        <div className="text-slate-500 font-bold mb-1 border-b border-slate-900 pb-0.5">EXECUTION LOG & AUDIT TRAIL</div>
        <div className="space-y-0.5">
          {logs.map(log => (
            <div key={log.id} className="flex gap-2">
              <span className="text-slate-600">[{log.time}]</span>
              <span className={
                log.type === 'WARN' ? 'text-rose-400 font-bold' :
                log.type === 'EVENT' ? 'text-yellow-400 font-bold' :
                log.type === 'GAME' ? 'text-cyan-400 font-bold' : 'text-emerald-400'
              }>
                {log.msg}
              </span>
            </div>
          ))}
        </div>
      </footer>

    </div>
  );
}
