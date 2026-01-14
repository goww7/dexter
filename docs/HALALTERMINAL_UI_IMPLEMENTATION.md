# HalalTerminal.com UI Implementation Specification

**Version**: 1.0
**Date**: 2026-01-14
**Purpose**: Complete specification for implementing the HalalTerminal.com trading terminal experience

---

## Executive Summary

This document provides a complete specification for building the HalalTerminal.com web interface based on the production screenshot. The platform is a **Bloomberg Terminal-style interface specifically designed for Shariah-compliant investing**.

**Key Experience Goals:**
1. Instantly show Halal/Haram status for any stock
2. Provide detailed Shariah compliance ratios with context
3. Enable multi-criteria screening for halal portfolios
4. Display fundamental quality & value analysis
5. Monitor news that might affect compliance status
6. Educate users on Islamic finance principles

---

## 1. Overall Layout Architecture

### 1.1 Three-Panel Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  TOP NAV: Logo | Screener | Search | Live API Status           │
├──────────┬────────────────────────────────┬────────────────────┤
│          │                                │                    │
│  LEFT    │         CENTER                 │      RIGHT         │
│  PANEL   │       (Chart View)             │   (Halal Panel)    │
│          │                                │                    │
│ Watchlist│   - Stock Chart                │ - Halal Status     │
│ Filters  │   - Technical Indicators       │ - Compliance Data  │
│ Sectors  │   - Volume/RSI                 │ - Quality Metrics  │
│ Stocks   │   - Drawing Tools              │ - Value Metrics    │
│          │                                │ - Key Stats        │
│          │                                │ - Educational      │
│          │                                │                    │
├──────────┴────────────────────────────────┴────────────────────┤
│  BOTTOM: Live API Console (Optional for power users)           │
└─────────────────────────────────────────────────────────────────┘
```

**Responsive Behavior:**
- Desktop (>1400px): Full three-panel layout
- Tablet (768-1400px): Collapsible left panel, full center+right
- Mobile (<768px): Single panel with tabs (Chart | Halal | Watchlist)

---

## 2. Top Navigation Bar

### 2.1 Component Structure

```typescript
interface TopNavBar {
  logo: {
    icon: string; // "H" in circle
    text: string; // "TERMINAL"
    link: string; // "/" home page
  };
  screenerButton: {
    label: string; // "SCREENER"
    icon: string; // Grid icon
    onClick: () => void; // Open screener modal
  };
  searchBar: {
    placeholder: string; // "Search"
    hotkey: string; // "/"
    autocomplete: boolean; // true
    resultsPreview: number; // Show top 5 results
  };
  liveApiStatus: {
    label: string; // "Live API"
    status: 'connected' | 'disconnected' | 'error';
    indicator: 'green' | 'red' | 'yellow';
  };
  userMenu?: {
    // For authenticated users
    avatar: string;
    name: string;
    tier: 'free' | 'pro' | 'enterprise';
  };
}
```

### 2.2 Implementation

```tsx
// components/TopNavBar.tsx
import React from 'react';
import { Search, Grid, Activity } from 'lucide-react';

export const TopNavBar: React.FC = () => {
  return (
    <nav className="h-14 bg-[#0A0E14] border-b border-gray-800 px-4 flex items-center justify-between">
      {/* Logo */}
      <div className="flex items-center gap-2">
        <div className="w-8 h-8 bg-emerald-500 rounded-full flex items-center justify-center">
          <span className="text-white font-bold text-lg">H</span>
        </div>
        <span className="text-white font-semibold tracking-wide">TERMINAL</span>
      </div>

      {/* Center Actions */}
      <div className="flex items-center gap-4">
        {/* Screener Button */}
        <button className="px-4 py-2 bg-gray-800 hover:bg-gray-700 rounded-md flex items-center gap-2 text-gray-300">
          <Grid size={16} />
          <span className="text-sm font-medium">SCREENER</span>
        </button>

        {/* Search Bar */}
        <div className="relative w-64">
          <Search className="absolute left-3 top-2.5 text-gray-500" size={18} />
          <input
            type="text"
            placeholder="Search"
            className="w-full bg-gray-900 border border-gray-700 rounded-md pl-10 pr-4 py-2 text-sm text-white focus:outline-none focus:border-emerald-500"
          />
          <kbd className="absolute right-3 top-2.5 text-xs text-gray-500">/</kbd>
        </div>
      </div>

      {/* Live API Status */}
      <div className="flex items-center gap-2 text-emerald-400">
        <Activity size={16} className="animate-pulse" />
        <span className="text-sm font-medium">Live API</span>
        <span className="text-xs text-emerald-500">Connected</span>
      </div>
    </nav>
  );
};
```

---

## 3. Left Panel: Watchlist & Filters

### 3.1 Component Structure

```typescript
interface LeftPanel {
  tabs: {
    watchlist: { label: string; icon: string; active: boolean };
    explorer: { label: string; icon: string; active: boolean };
  };
  searchBar: {
    placeholder: string; // "SEARCH 3000+ INSTRUMENTS..."
    value: string;
  };
  filters: {
    country: {
      label: string; // "COUNTRY"
      selected: string[]; // ["USA"]
      options: CountryOption[];
    };
    sector: {
      label: string; // "SECTOR"
      selected: string; // "ALL" or specific sector
      options: SectorOption[];
    };
    halalStatus?: {
      label: string; // "HALAL STATUS"
      selected: 'all' | 'halal_only' | 'questionable' | 'non_halal';
    };
  };
  sectorList: SectorGroup[];
  stockList: Stock[];
}

interface SectorGroup {
  name: string; // "COMMUNICATION SERVICES"
  count: number; // 25
  expanded: boolean;
}

interface Stock {
  ticker: string; // "AAPL"
  name: string; // "Apple Inc"
  halalStatus: 'halal' | 'not_halal' | 'questionable';
  price: number;
  change: number;
  changePercent: number;
  isWatchlisted: boolean;
  isFavorite: boolean;
}
```

### 3.2 Implementation

```tsx
// components/LeftPanel.tsx
import React, { useState } from 'react';
import { ChevronDown, Star, Plus } from 'lucide-react';

const sectors = [
  { name: 'COMMUNICATION SERVICES', count: 25 },
  { name: 'CONSUMER DISCRETIONARY', count: 46 },
  { name: 'CONSUMER STAPLES', count: 36 },
  { name: 'ENERGY', count: 22 },
  { name: 'FINANCIALS', count: 69 },
  { name: 'HEALTH CARE', count: 55 },
  { name: 'INDUSTRIALS', count: 70 },
  { name: 'INFORMATION TECHNOLOGY', count: 74 },
];

const stocks = [
  { ticker: 'AAPL', name: 'Apple Inc', halal: true, price: 273.67, change: 1.48 },
  { ticker: 'ADBE', name: 'Adobe Inc', halal: true, price: 456.32, change: -2.13 },
  // ... more stocks
];

export const LeftPanel: React.FC = () => {
  const [selectedSector, setSelectedSector] = useState<string>('ALL');
  const [searchQuery, setSearchQuery] = useState<string>('');

  return (
    <div className="w-64 bg-[#0D1117] border-r border-gray-800 flex flex-col h-full">
      {/* Tabs */}
      <div className="flex border-b border-gray-800">
        <button className="flex-1 py-3 text-sm font-medium text-emerald-400 border-b-2 border-emerald-400">
          WATCHLIST
        </button>
        <button className="flex-1 py-3 text-sm font-medium text-gray-500 hover:text-gray-300">
          EXPLORER
        </button>
      </div>

      {/* Search */}
      <div className="p-3">
        <input
          type="text"
          placeholder="SEARCH 3000+ INSTRUMENTS..."
          value={searchQuery}
          onChange={(e) => setSearchQuery(e.target.value)}
          className="w-full bg-gray-900 border border-gray-700 rounded px-3 py-2 text-xs text-gray-300 placeholder-gray-600 focus:outline-none focus:border-emerald-500"
        />
      </div>

      {/* Filters */}
      <div className="px-3 space-y-2">
        {/* Country Filter */}
        <div>
          <label className="block text-xs font-medium text-gray-500 mb-1">COUNTRY</label>
          <select className="w-full bg-gray-900 border border-gray-700 rounded px-3 py-2 text-xs text-white">
            <option>USA</option>
            <option>ALL</option>
            <option>UK</option>
            <option>UAE</option>
            <option>Malaysia</option>
          </select>
        </div>

        {/* Sector Filter */}
        <div>
          <label className="block text-xs font-medium text-gray-500 mb-1">SECTOR</label>
          <select
            className="w-full bg-gray-900 border border-gray-700 rounded px-3 py-2 text-xs text-white"
            value={selectedSector}
            onChange={(e) => setSelectedSector(e.target.value)}
          >
            <option>ALL</option>
            {sectors.map(sector => (
              <option key={sector.name} value={sector.name}>
                {sector.name}
              </option>
            ))}
          </select>
        </div>
      </div>

      {/* Sector List */}
      <div className="flex-1 overflow-y-auto mt-4">
        {sectors.map(sector => (
          <div
            key={sector.name}
            className="px-3 py-2 hover:bg-gray-800 cursor-pointer flex items-center justify-between text-xs"
          >
            <span className="text-gray-400">{sector.name}</span>
            <span className="text-gray-600">{sector.count}</span>
          </div>
        ))}

        {/* Stock List */}
        <div className="mt-4 border-t border-gray-800 pt-2">
          {stocks.map(stock => (
            <div
              key={stock.ticker}
              className={`px-3 py-2 hover:bg-gray-800 cursor-pointer ${
                stock.ticker === 'AAPL' ? 'bg-emerald-900/20 border-l-2 border-emerald-500' : ''
              }`}
            >
              <div className="flex items-center justify-between">
                <div className="flex items-center gap-2">
                  {stock.halal && (
                    <div className="w-2 h-2 bg-emerald-500 rounded-full" />
                  )}
                  <span className="text-xs font-medium text-white">{stock.ticker}</span>
                </div>
                <button className="text-gray-600 hover:text-emerald-400">
                  <Plus size={14} />
                </button>
              </div>
              <div className="text-[10px] text-gray-500 mt-0.5">{stock.name}</div>
            </div>
          ))}
        </div>
      </div>

      {/* Live API Console (collapsed by default) */}
      <div className="border-t border-gray-800 p-3">
        <button className="text-xs text-gray-500 hover:text-emerald-400 flex items-center gap-2 w-full">
          <ChevronDown size={14} />
          <span>LIVE API CONSOLE</span>
        </button>
      </div>
    </div>
  );
};
```

---

## 4. Center Panel: Stock Chart

### 4.1 Component Structure

```typescript
interface ChartPanel {
  stock: {
    ticker: string;
    name: string;
    exchange: string;
    sector: string;
  };
  priceHeader: {
    currentPrice: number;
    change: number;
    changePercent: number;
    previousClose: number;
    open: number;
    high: number;
    low: number;
    volume: string;
    vwap: number;
  };
  chartConfig: {
    timeframe: '1m' | '30m' | '1h' | '1D' | '1W' | '1M';
    chartType: 'candle' | 'line' | 'area';
    indicators: ChartIndicator[];
    overlays: ChartOverlay[];
    drawings: Drawing[];
  };
  chartData: CandlestickData[];
}

interface ChartIndicator {
  type: 'RSI' | 'MACD' | 'Volume' | 'BB' | 'MA';
  config: any;
  visible: boolean;
}
```

### 4.2 Implementation (Using TradingView Lightweight Charts)

```tsx
// components/ChartPanel.tsx
import React, { useEffect, useRef } from 'react';
import { createChart, IChartApi } from 'lightweight-charts';
import { TrendingUp } from 'lucide-react';

interface ChartPanelProps {
  ticker: string;
  data: any[];
}

export const ChartPanel: React.FC<ChartPanelProps> = ({ ticker, data }) => {
  const chartContainerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<IChartApi | null>(null);

  useEffect(() => {
    if (!chartContainerRef.current) return;

    const chart = createChart(chartContainerRef.current, {
      layout: {
        background: { color: '#0D1117' },
        textColor: '#9CA3AF',
      },
      grid: {
        vertLines: { color: '#1F2937' },
        horzLines: { color: '#1F2937' },
      },
      width: chartContainerRef.current.clientWidth,
      height: 600,
    });

    const candlestickSeries = chart.addCandlestickSeries({
      upColor: '#10B981',
      downColor: '#EF4444',
      borderUpColor: '#10B981',
      borderDownColor: '#EF4444',
      wickUpColor: '#10B981',
      wickDownColor: '#EF4444',
    });

    candlestickSeries.setData(data);

    // Add Bollinger Bands
    const upperBand = chart.addLineSeries({ color: '#6B7280', lineWidth: 1 });
    const lowerBand = chart.addLineSeries({ color: '#6B7280', lineWidth: 1 });

    // Calculate and set BB data (simplified)
    // upperBand.setData(calculateBollingerBands(data, 20, 2).upper);
    // lowerBand.setData(calculateBollingerBands(data, 20, 2).lower);

    // Add Volume
    const volumeSeries = chart.addHistogramSeries({
      color: '#374151',
      priceFormat: { type: 'volume' },
      priceScaleId: 'volume',
    });
    // volumeSeries.setData(volumeData);

    chartRef.current = chart;

    return () => chart.remove();
  }, [data]);

  return (
    <div className="flex-1 bg-[#0D1117] flex flex-col">
      {/* Stock Header */}
      <div className="border-b border-gray-800 p-4">
        <div className="flex items-center justify-between">
          <div className="flex items-center gap-4">
            <div>
              <h1 className="text-2xl font-bold text-white">AAPL</h1>
              <p className="text-xs text-gray-500">Apple Inc · 1D · Cboe One</p>
            </div>
            <div>
              <div className="text-3xl font-bold text-white">$273.67</div>
              <div className="text-sm text-emerald-400">+1.48 +0.54%</div>
            </div>
          </div>

          {/* Chart Controls */}
          <div className="flex items-center gap-2">
            <button className="px-3 py-1.5 bg-gray-800 hover:bg-gray-700 rounded text-xs text-white">
              1m
            </button>
            <button className="px-3 py-1.5 bg-gray-800 hover:bg-gray-700 rounded text-xs text-white">
              30m
            </button>
            <button className="px-3 py-1.5 bg-emerald-600 rounded text-xs text-white">
              1h
            </button>
            <button className="px-3 py-1.5 bg-gray-800 hover:bg-gray-700 rounded text-xs text-white">
              1D
            </button>
          </div>
        </div>

        {/* Price Stats */}
        <div className="flex items-center gap-6 mt-3 text-xs">
          <span className="text-gray-500">O <span className="text-white">275.27</span></span>
          <span className="text-gray-500">H <span className="text-white">280.38</span></span>
          <span className="text-gray-500">L <span className="text-white">275.25</span></span>
          <span className="text-gray-500">C <span className="text-white">276.07</span></span>
          <span className="text-gray-500">Vol <span className="text-white">48.91M</span></span>
        </div>
      </div>

      {/* Chart Container */}
      <div ref={chartContainerRef} className="flex-1" />

      {/* Chart Footer - Timeframes */}
      <div className="border-t border-gray-800 px-4 py-3 flex items-center justify-between">
        <div className="flex gap-2">
          {['1D', '5D', '1M', '3M', '6M', 'YTD', '1Y', '5Y', 'All'].map(tf => (
            <button
              key={tf}
              className="px-3 py-1 text-xs text-gray-400 hover:text-white hover:bg-gray-800 rounded"
            >
              {tf}
            </button>
          ))}
        </div>
        <div className="flex items-center gap-2 text-xs text-gray-500">
          <span>18:52:12 UTC</span>
          <span>·</span>
          <span>ADJ</span>
        </div>
      </div>
    </div>
  );
};
```

---

## 5. Right Panel: Halal Status & Analysis

### 5.1 Component Structure

```typescript
interface RightPanel {
  halalStatus: HalalStatusCard;
  useCases: UseCaseSection;
  newsImpact: NewsImpactSection;
  qualityPillar: QualityPillarCard;
  valuePillar: ValuePillarCard;
  keyStats: KeyStatsCard;
  earnings: EarningsCard;
  actions: ActionButtons;
}

interface HalalStatusCard {
  status: 'HALAL' | 'NOT_HALAL' | 'QUESTIONABLE';
  badge: {
    text: string;
    color: string;
    bgColor: string;
  };
  asOfDate: string;
  ratios: {
    debtToAssets: RatioDetail;
    interestToRevenue: RatioDetail;
    liquidityToAssets?: RatioDetail;
    receivablesToAssets?: RatioDetail;
  };
  businessActivity: {
    isCompliant: boolean;
    description: string;
    details?: string;
  };
  confidenceScore?: number;
  sources: string[];
}

interface RatioDetail {
  label: string;
  value: number;
  limit: number;
  passed: boolean;
  source: string; // "AAOIFI SS21" or "DJIM" etc.
  lastChecked: string;
}

interface UseCaseSection {
  title: string; // "HOW WE MIGHT USE THIS"
  items: UseCaseItem[];
}

interface UseCaseItem {
  text: string;
  isPro: boolean;
  link?: string;
}

interface QualityPillarCard {
  title: string; // "PILLAR 1: QUALITY"
  metrics: {
    profitMargin: { value: number; percentile: number };
    capitalReturn: string;
    profitability: string;
    growth: string;
    liquidityRisk: { value: number; status: string };
  };
}

interface ValuePillarCard {
  title: string; // "PILLAR 2: VALUE"
  fairValue: number;
  currentPrice: number;
  discount: number;
  valuation: string;
  priceRatio: string;
}
```

### 5.2 Implementation

```tsx
// components/RightPanel.tsx
import React from 'react';
import { CheckCircle, AlertTriangle, TrendingUp, DollarSign, Info } from 'lucide-react';

interface RightPanelProps {
  ticker: string;
  halalData: any;
}

export const RightPanel: React.FC<RightPanelProps> = ({ ticker, halalData }) => {
  return (
    <div className="w-96 bg-[#0D1117] border-l border-gray-800 overflow-y-auto">
      {/* Halal Status Badge */}
      <div className="p-6 border-b border-gray-800">
        <div className="bg-emerald-500 text-white font-bold text-2xl py-3 px-6 rounded-lg text-center mb-4">
          HALAL
        </div>

        <div className="space-y-3">
          <div className="flex justify-between items-center text-xs">
            <span className="text-gray-500 uppercase">Halal Status</span>
            <span className="text-gray-400">AS OF 2025-11-22</span>
          </div>

          {/* Ratios */}
          <div className="space-y-2">
            <div className="flex justify-between items-center">
              <span className="text-sm text-gray-400">TOTAL DEBT / ASSETS</span>
              <div className="flex items-center gap-2">
                <span className="text-lg font-bold text-white">27.5%</span>
                <CheckCircle size={16} className="text-emerald-500" />
              </div>
            </div>
            <div className="text-[10px] text-gray-600 text-right">
              Limit: 33% (AAOIFI SS21)
            </div>

            <div className="flex justify-between items-center mt-3">
              <span className="text-sm text-gray-400">INTEREST / REVENUE</span>
              <div className="flex items-center gap-2">
                <span className="text-lg font-bold text-white">0.0%</span>
                <CheckCircle size={16} className="text-emerald-500" />
              </div>
            </div>
            <div className="text-[10px] text-gray-600 text-right">
              Limit: 5% (AAOIFI SS21)
            </div>
          </div>

          {/* Business Activity */}
          <div className="mt-4 p-3 bg-emerald-900/20 border border-emerald-800 rounded">
            <div className="flex items-start gap-2">
              <CheckCircle size={14} className="text-emerald-500 mt-0.5" />
              <span className="text-xs text-emerald-300">
                Business activity is compliant.
              </span>
            </div>
          </div>
        </div>
      </div>

      {/* Use Cases Section */}
      <div className="p-6 border-b border-gray-800">
        <h3 className="text-xs font-semibold text-gray-500 uppercase mb-3">
          HOW WE MIGHT USE THIS
        </h3>
        <ul className="space-y-2 text-xs text-gray-400">
          <li className="flex items-start gap-2">
            <span className="text-emerald-500">•</span>
            <span>Validate if a security fits an Islamic equity universe.</span>
          </li>
          <li className="flex items-start gap-2">
            <span className="text-emerald-500">•</span>
            <span>Document the rationale for investment committee or Shariah boards.</span>
          </li>
          <li className="flex items-start gap-2">
            <span className="text-amber-500">•</span>
            <span className="flex items-center gap-1">
              <span className="text-amber-500 font-medium">(Pro)</span>
              Roll this logic up to portfolio level.
            </span>
          </li>
        </ul>
        <button className="mt-3 text-xs text-emerald-400 hover:text-emerald-300 underline">
          Request a professional walkthrough
        </button>
      </div>

      {/* News Impact (Pro Feature) */}
      <div className="p-6 border-b border-gray-800 bg-gradient-to-b from-transparent to-amber-900/5">
        <div className="flex items-center justify-between mb-3">
          <h3 className="text-xs font-semibold text-gray-500 uppercase">
            NEWS IMPACT
          </h3>
          <span className="px-2 py-0.5 bg-amber-500 text-black text-[10px] font-bold rounded">
            PRO
          </span>
        </div>
        <div className="text-xs text-gray-600">
          Monitor news that might affect Shariah compliance status
        </div>
      </div>

      {/* Quality Pillar */}
      <div className="p-6 border-b border-gray-800">
        <div className="flex items-center gap-2 mb-4">
          <TrendingUp size={16} className="text-blue-400" />
          <h3 className="text-sm font-semibold text-white">PILLAR 1: QUALITY</h3>
        </div>

        <div className="space-y-3 text-xs">
          <div className="flex justify-between items-center">
            <span className="text-gray-400">PROFIT MARGIN</span>
            <div className="text-right">
              <div className="text-white font-bold">171.4%</div>
              <div className="text-emerald-500 text-[10px]">26.9%</div>
            </div>
          </div>

          <div className="flex items-start gap-2">
            <CheckCircle size={12} className="text-emerald-500 mt-0.5" />
            <span className="text-gray-300">Strong capital return</span>
          </div>

          <div className="flex items-start gap-2">
            <CheckCircle size={12} className="text-emerald-500 mt-0.5" />
            <span className="text-gray-300">Healthy profitability</span>
          </div>

          <div className="p-2 bg-blue-900/20 border border-blue-800 rounded">
            <div className="text-blue-300 font-medium mb-1">ABOVE AVERAGE GROWTH</div>
            <div className="text-gray-400 text-[10px]">Asset ratio</div>
          </div>

          <div className="text-[10px] text-gray-500">
            Modest growth, Moderate liquidity risk (&lt;33%)
          </div>
        </div>
      </div>

      {/* Value Pillar */}
      <div className="p-6 border-b border-gray-800">
        <div className="flex items-center gap-2 mb-4">
          <DollarSign size={16} className="text-purple-400" />
          <h3 className="text-sm font-semibold text-white">PILLAR 2: VALUE</h3>
        </div>

        <div className="space-y-3 text-xs">
          <div className="flex justify-between items-center">
            <span className="text-gray-400">FAIR VALUE</span>
            <div className="text-right">
              <div className="text-white font-bold text-lg">278</div>
              <div className="text-gray-500 text-[10px]">RATIO</div>
            </div>
          </div>

          <div className="p-3 bg-purple-900/20 border border-purple-800 rounded">
            <div className="text-purple-300">
              Priced at <span className="font-bold">5% cheaper</span> &gt; 10%
            </div>
            <div className="text-gray-400 text-[10px] mt-1">Lightness premium.</div>
          </div>
        </div>
      </div>

      {/* Key Stats */}
      <div className="p-6 border-b border-gray-800">
        <h3 className="text-xs font-semibold text-gray-500 uppercase mb-3">
          KEY STATS
        </h3>

        <div className="space-y-2 text-xs">
          <div className="flex justify-between">
            <span className="text-gray-400">Next earnings</span>
            <span className="text-white">In 40 days</span>
          </div>
          <div className="flex justify-between">
            <span className="text-gray-400">Volume</span>
            <span className="text-white">144.63M</span>
          </div>
          <div className="flex justify-between">
            <span className="text-gray-400">Average Volume (30D)</span>
            <span className="text-white">47.63M</span>
          </div>
          <div className="flex justify-between">
            <span className="text-gray-400">Market capitalization</span>
            <span className="text-white">4.04T</span>
          </div>
        </div>
      </div>

      {/* Earnings Chart */}
      <div className="p-6">
        <h3 className="text-xs font-semibold text-gray-500 uppercase mb-3">
          EARNINGS
        </h3>
        <div className="flex items-end gap-2 h-20">
          {[2.80, 2.10, 1.40].map((value, i) => (
            <div key={i} className="flex-1 flex flex-col items-center gap-1">
              <div
                className="w-full bg-emerald-500 rounded-t"
                style={{ height: `${(value / 2.8) * 100}%` }}
              />
              <span className="text-[10px] text-gray-500">{value}</span>
            </div>
          ))}
        </div>
      </div>

      {/* Bottom Actions */}
      <div className="p-6 border-t border-gray-800 space-y-2">
        <button className="w-full py-2 bg-emerald-600 hover:bg-emerald-700 text-white rounded font-medium text-sm">
          Add to Portfolio
        </button>
        <button className="w-full py-2 bg-gray-800 hover:bg-gray-700 text-white rounded font-medium text-sm">
          Export Report
        </button>
      </div>
    </div>
  );
};
```

---

## 6. Modal: Screener Tool

### 6.1 Component Structure

```typescript
interface ScreenerModal {
  isOpen: boolean;
  filters: {
    halalStatus: {
      halal: boolean;
      questionable: boolean;
      nonHalal: boolean;
    };
    financial: {
      debtToAssets: { min: number; max: number };
      interestToRevenue: { min: number; max: number };
      marketCap: { min: number; max: number };
      priceToEarnings: { min: number; max: number };
    };
    qualitative: {
      profitMargin: { min: number };
      growth: string[];
      quality: string[];
    };
    regional: {
      aaoifiCompliant: boolean;
      djimIncluded: boolean;
      msciIncluded: boolean;
      malaysiaSACApproved: boolean;
    };
  };
  results: ScreenerResult[];
  sorting: {
    field: string;
    direction: 'asc' | 'desc';
  };
}

interface ScreenerResult {
  ticker: string;
  name: string;
  halalStatus: 'halal' | 'not_halal' | 'questionable';
  price: number;
  change: number;
  marketCap: number;
  debtToAssets: number;
  interestToRevenue: number;
  qualityScore: number;
  valueScore: number;
}
```

### 6.2 Implementation

```tsx
// components/ScreenerModal.tsx
import React, { useState } from 'react';
import { X, Filter, Download, Star } from 'lucide-react';

interface ScreenerModalProps {
  isOpen: boolean;
  onClose: () => void;
}

export const ScreenerModal: React.FC<ScreenerModalProps> = ({ isOpen, onClose }) => {
  const [filters, setFilters] = useState({
    halalOnly: true,
    debtLimit: 33,
    interestLimit: 5,
    minMarketCap: 0,
    maxMarketCap: 1000000,
  });

  if (!isOpen) return null;

  return (
    <div className="fixed inset-0 bg-black/80 flex items-center justify-center z-50 p-4">
      <div className="bg-[#0D1117] border border-gray-800 rounded-lg w-full max-w-6xl max-h-[90vh] overflow-hidden flex flex-col">
        {/* Header */}
        <div className="flex items-center justify-between p-6 border-b border-gray-800">
          <div className="flex items-center gap-3">
            <Filter size={20} className="text-emerald-400" />
            <h2 className="text-xl font-bold text-white">Halal Stock Screener</h2>
          </div>
          <button onClick={onClose} className="text-gray-400 hover:text-white">
            <X size={24} />
          </button>
        </div>

        <div className="flex flex-1 overflow-hidden">
          {/* Left: Filters */}
          <div className="w-80 border-r border-gray-800 p-6 overflow-y-auto">
            <h3 className="text-sm font-semibold text-gray-300 mb-4 uppercase">Filters</h3>

            {/* Shariah Compliance */}
            <div className="mb-6">
              <h4 className="text-xs font-medium text-gray-500 mb-2">SHARIAH COMPLIANCE</h4>
              <label className="flex items-center gap-2 text-sm text-white mb-2">
                <input type="checkbox" checked={filters.halalOnly} className="rounded" />
                <span>Halal Only</span>
              </label>
              <label className="flex items-center gap-2 text-sm text-gray-400">
                <input type="checkbox" className="rounded" />
                <span>Include Questionable</span>
              </label>
            </div>

            {/* Financial Ratios */}
            <div className="mb-6">
              <h4 className="text-xs font-medium text-gray-500 mb-2">FINANCIAL RATIOS</h4>

              <div className="mb-3">
                <label className="text-xs text-gray-400">Max Debt / Assets (%)</label>
                <input
                  type="range"
                  min="0"
                  max="50"
                  value={filters.debtLimit}
                  onChange={(e) => setFilters({ ...filters, debtLimit: parseInt(e.target.value) })}
                  className="w-full mt-1"
                />
                <div className="flex justify-between text-xs text-gray-500 mt-1">
                  <span>0%</span>
                  <span className="text-white font-medium">{filters.debtLimit}%</span>
                  <span>50%</span>
                </div>
              </div>

              <div className="mb-3">
                <label className="text-xs text-gray-400">Max Interest / Revenue (%)</label>
                <input
                  type="range"
                  min="0"
                  max="10"
                  value={filters.interestLimit}
                  onChange={(e) => setFilters({ ...filters, interestLimit: parseInt(e.target.value) })}
                  className="w-full mt-1"
                />
                <div className="flex justify-between text-xs text-gray-500 mt-1">
                  <span>0%</span>
                  <span className="text-white font-medium">{filters.interestLimit}%</span>
                  <span>10%</span>
                </div>
              </div>
            </div>

            {/* Market Cap */}
            <div className="mb-6">
              <h4 className="text-xs font-medium text-gray-500 mb-2">MARKET CAP</h4>
              <select className="w-full bg-gray-900 border border-gray-700 rounded px-3 py-2 text-sm text-white">
                <option>All</option>
                <option>Mega (&gt;$200B)</option>
                <option>Large ($10B-$200B)</option>
                <option>Mid ($2B-$10B)</option>
                <option>Small (&lt;$2B)</option>
              </select>
            </div>

            {/* Index Inclusion */}
            <div className="mb-6">
              <h4 className="text-xs font-medium text-gray-500 mb-2">INDEX INCLUSION</h4>
              <label className="flex items-center gap-2 text-sm text-gray-400 mb-2">
                <input type="checkbox" className="rounded" />
                <span>DJIM</span>
              </label>
              <label className="flex items-center gap-2 text-sm text-gray-400 mb-2">
                <input type="checkbox" className="rounded" />
                <span>MSCI Islamic</span>
              </label>
              <label className="flex items-center gap-2 text-sm text-gray-400 mb-2">
                <input type="checkbox" className="rounded" />
                <span>S&P Shariah</span>
              </label>
            </div>

            {/* Quality Metrics */}
            <div className="mb-6">
              <h4 className="text-xs font-medium text-gray-500 mb-2">QUALITY</h4>
              <select className="w-full bg-gray-900 border border-gray-700 rounded px-3 py-2 text-sm text-white mb-2">
                <option>Any Profitability</option>
                <option>Profitable</option>
                <option>Highly Profitable (&gt;20%)</option>
              </select>
              <select className="w-full bg-gray-900 border border-gray-700 rounded px-3 py-2 text-sm text-white">
                <option>Any Growth</option>
                <option>Positive Growth</option>
                <option>High Growth (&gt;15%)</option>
              </select>
            </div>

            <button className="w-full py-2 bg-emerald-600 hover:bg-emerald-700 text-white rounded font-medium">
              Apply Filters
            </button>
          </div>

          {/* Right: Results */}
          <div className="flex-1 flex flex-col overflow-hidden">
            {/* Results Header */}
            <div className="p-4 border-b border-gray-800 flex items-center justify-between">
              <div className="text-sm text-gray-400">
                Found <span className="text-white font-bold">247</span> halal stocks
              </div>
              <div className="flex items-center gap-2">
                <select className="bg-gray-900 border border-gray-700 rounded px-3 py-1.5 text-xs text-white">
                  <option>Sort by: Market Cap</option>
                  <option>Sort by: Price Change</option>
                  <option>Sort by: Quality Score</option>
                  <option>Sort by: Value Score</option>
                </select>
                <button className="px-3 py-1.5 bg-gray-800 hover:bg-gray-700 rounded text-xs text-white flex items-center gap-2">
                  <Download size={14} />
                  Export
                </button>
              </div>
            </div>

            {/* Results Table */}
            <div className="flex-1 overflow-y-auto">
              <table className="w-full">
                <thead className="bg-gray-900 sticky top-0">
                  <tr className="text-xs text-gray-400 text-left">
                    <th className="p-3 font-medium">TICKER</th>
                    <th className="p-3 font-medium">COMPANY</th>
                    <th className="p-3 font-medium">PRICE</th>
                    <th className="p-3 font-medium">CHANGE</th>
                    <th className="p-3 font-medium">MARKET CAP</th>
                    <th className="p-3 font-medium">DEBT/ASSETS</th>
                    <th className="p-3 font-medium">INT/REV</th>
                    <th className="p-3 font-medium">QUALITY</th>
                    <th className="p-3 font-medium"></th>
                  </tr>
                </thead>
                <tbody>
                  {mockResults.map((stock) => (
                    <tr
                      key={stock.ticker}
                      className="border-b border-gray-800 hover:bg-gray-900 cursor-pointer"
                    >
                      <td className="p-3">
                        <div className="flex items-center gap-2">
                          <div className="w-2 h-2 bg-emerald-500 rounded-full" />
                          <span className="font-medium text-white">{stock.ticker}</span>
                        </div>
                      </td>
                      <td className="p-3 text-sm text-gray-400">{stock.name}</td>
                      <td className="p-3 text-sm text-white">${stock.price}</td>
                      <td className="p-3">
                        <span className={stock.change > 0 ? 'text-emerald-500' : 'text-red-500'}>
                          {stock.change > 0 ? '+' : ''}{stock.change}%
                        </span>
                      </td>
                      <td className="p-3 text-sm text-white">{stock.marketCap}</td>
                      <td className="p-3">
                        <span className={stock.debtToAssets < 33 ? 'text-emerald-500' : 'text-red-500'}>
                          {stock.debtToAssets}%
                        </span>
                      </td>
                      <td className="p-3">
                        <span className={stock.interestToRevenue < 5 ? 'text-emerald-500' : 'text-red-500'}>
                          {stock.interestToRevenue}%
                        </span>
                      </td>
                      <td className="p-3">
                        <div className="flex items-center gap-1">
                          {[...Array(5)].map((_, i) => (
                            <Star
                              key={i}
                              size={12}
                              className={i < stock.qualityScore ? 'text-amber-500 fill-amber-500' : 'text-gray-700'}
                            />
                          ))}
                        </div>
                      </td>
                      <td className="p-3">
                        <button className="text-gray-600 hover:text-emerald-400">
                          <Star size={16} />
                        </button>
                      </td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
};

const mockResults = [
  { ticker: 'AAPL', name: 'Apple Inc', price: 273.67, change: 0.54, marketCap: '4.04T', debtToAssets: 27.5, interestToRevenue: 0.0, qualityScore: 5 },
  { ticker: 'MSFT', name: 'Microsoft Corp', price: 425.12, change: 1.23, marketCap: '3.15T', debtToAssets: 22.1, interestToRevenue: 0.2, qualityScore: 5 },
  { ticker: 'GOOGL', name: 'Alphabet Inc', price: 178.34, change: -0.45, marketCap: '2.23T', debtToAssets: 8.5, interestToRevenue: 0.1, qualityScore: 4 },
  // ... more stocks
];
```

---

## 7. Data Integration Layer

### 7.1 API Client Structure

```typescript
// lib/api/halal-terminal-client.ts

export class HalalTerminalClient {
  private baseUrl: string;
  private apiKey: string;

  constructor(config: { baseUrl: string; apiKey: string }) {
    this.baseUrl = config.baseUrl;
    this.apiKey = config.apiKey;
  }

  /**
   * Get Shariah compliance status for a ticker
   */
  async getShariahCompliance(ticker: string): Promise<ShariahComplianceResponse> {
    const response = await fetch(`${this.baseUrl}/api/v1/shariah/screen/${ticker}`, {
      headers: {
        Authorization: `Bearer ${this.apiKey}`,
      },
    });
    return response.json();
  }

  /**
   * Get quality and value pillars
   */
  async getAnalysis(ticker: string): Promise<AnalysisResponse> {
    const response = await fetch(`${this.baseUrl}/api/v1/analysis/${ticker}`, {
      headers: {
        Authorization: `Bearer ${this.apiKey}`,
      },
    });
    return response.json();
  }

  /**
   * Run screener with filters
   */
  async runScreener(filters: ScreenerFilters): Promise<ScreenerResponse> {
    const response = await fetch(`${this.baseUrl}/api/v1/screener`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${this.apiKey}`,
      },
      body: JSON.stringify(filters),
    });
    return response.json();
  }

  /**
   * Get stock price data
   */
  async getPriceData(
    ticker: string,
    timeframe: string,
    from: string,
    to: string
  ): Promise<PriceDataResponse> {
    const params = new URLSearchParams({ timeframe, from, to });
    const response = await fetch(
      `${this.baseUrl}/api/v1/prices/${ticker}?${params}`,
      {
        headers: {
          Authorization: `Bearer ${this.apiKey}`,
        },
      }
    );
    return response.json();
  }
}

// Types
interface ShariahComplianceResponse {
  ticker: string;
  status: 'halal' | 'not_halal' | 'questionable';
  asOfDate: string;
  ratios: {
    debtToAssets: {
      value: number;
      limit: number;
      passed: boolean;
      source: string;
    };
    interestToRevenue: {
      value: number;
      limit: number;
      passed: boolean;
      source: string;
    };
    liquidityToAssets?: {
      value: number;
      limit: number;
      passed: boolean;
      source: string;
    };
  };
  businessActivity: {
    isCompliant: boolean;
    description: string;
    industries: string[];
  };
  confidenceScore: number;
  sources: string[];
  indexInclusion: {
    djim: boolean;
    msci: boolean;
    sp: boolean;
    ftse: boolean;
  };
}

interface AnalysisResponse {
  ticker: string;
  quality: {
    profitMargin: { value: number; percentile: number };
    capitalReturn: string;
    profitability: string;
    growth: string;
    liquidityRisk: { value: number; status: string };
  };
  value: {
    fairValue: number;
    currentPrice: number;
    discount: number;
    valuation: string;
    priceRatio: string;
  };
  keyStats: {
    nextEarnings: { date: string; daysAway: number };
    volume: string;
    averageVolume30D: string;
    marketCap: string;
  };
}

interface ScreenerFilters {
  halalStatus: ('halal' | 'questionable' | 'not_halal')[];
  financial: {
    debtToAssets?: { max: number };
    interestToRevenue?: { max: number };
    marketCap?: { min?: number; max?: number };
  };
  indexInclusion?: {
    djim?: boolean;
    msci?: boolean;
    sp?: boolean;
  };
  quality?: {
    minProfitMargin?: number;
    minQualityScore?: number;
  };
}

interface ScreenerResponse {
  total: number;
  results: ScreenerResult[];
}

interface ScreenerResult {
  ticker: string;
  name: string;
  halalStatus: 'halal' | 'not_halal' | 'questionable';
  price: number;
  change: number;
  changePercent: number;
  marketCap: number;
  debtToAssets: number;
  interestToRevenue: number;
  qualityScore: number;
  valueScore: number;
}
```

---

## 8. Color Palette & Theme

### 8.1 Design Tokens

```css
/* colors.css */
:root {
  /* Background Colors */
  --bg-primary: #0A0E14;        /* Main app background */
  --bg-secondary: #0D1117;      /* Panels, cards */
  --bg-tertiary: #161B22;       /* Elevated elements */

  /* Border Colors */
  --border-primary: #1F2937;    /* Main borders */
  --border-secondary: #374151;  /* Hover borders */

  /* Text Colors */
  --text-primary: #FFFFFF;      /* Primary text */
  --text-secondary: #9CA3AF;    /* Secondary text */
  --text-tertiary: #6B7280;     /* Muted text */

  /* Brand Colors */
  --brand-emerald-500: #10B981; /* Halal green */
  --brand-emerald-600: #059669; /* Hover green */
  --brand-emerald-900: #064E3B; /* Dark green bg */

  /* Status Colors */
  --status-halal: #10B981;      /* Green */
  --status-haram: #EF4444;      /* Red */
  --status-questionable: #F59E0B; /* Amber */

  /* Chart Colors */
  --chart-up: #10B981;          /* Bullish */
  --chart-down: #EF4444;        /* Bearish */
  --chart-neutral: #6B7280;     /* Neutral */

  /* Pro Feature Colors */
  --pro-accent: #F59E0B;        /* Amber for pro features */
}
```

### 8.2 Tailwind Config

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        halal: {
          50: '#ECFDF5',
          500: '#10B981',
          600: '#059669',
          900: '#064E3B',
        },
        haram: {
          50: '#FEF2F2',
          500: '#EF4444',
          600: '#DC2626',
          900: '#7F1D1D',
        },
        questionable: {
          50: '#FFFBEB',
          500: '#F59E0B',
          600: '#D97706',
          900: '#78350F',
        },
      },
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
        mono: ['JetBrains Mono', 'monospace'],
      },
    },
  },
};
```

---

## 9. State Management

### 9.1 Using Zustand

```typescript
// store/terminal-store.ts
import create from 'zustand';

interface TerminalStore {
  // Selected Stock
  selectedTicker: string | null;
  setSelectedTicker: (ticker: string) => void;

  // Watchlist
  watchlist: string[];
  addToWatchlist: (ticker: string) => void;
  removeFromWatchlist: (ticker: string) => void;

  // Filters
  filters: {
    country: string[];
    sector: string[];
    halalStatus: 'all' | 'halal_only' | 'non_halal' | 'questionable';
  };
  setFilters: (filters: Partial<TerminalStore['filters']>) => void;

  // UI State
  leftPanelOpen: boolean;
  toggleLeftPanel: () => void;

  screenerModalOpen: boolean;
  openScreener: () => void;
  closeScreener: () => void;

  // Data Cache
  complianceCache: Map<string, ShariahComplianceResponse>;
  analysisCache: Map<string, AnalysisResponse>;
  setCachedCompliance: (ticker: string, data: ShariahComplianceResponse) => void;
  setCachedAnalysis: (ticker: string, data: AnalysisResponse) => void;
}

export const useTerminalStore = create<TerminalStore>((set) => ({
  // Initial State
  selectedTicker: null,
  watchlist: [],
  filters: {
    country: ['USA'],
    sector: [],
    halalStatus: 'all',
  },
  leftPanelOpen: true,
  screenerModalOpen: false,
  complianceCache: new Map(),
  analysisCache: new Map(),

  // Actions
  setSelectedTicker: (ticker) => set({ selectedTicker: ticker }),

  addToWatchlist: (ticker) =>
    set((state) => ({
      watchlist: [...state.watchlist, ticker],
    })),

  removeFromWatchlist: (ticker) =>
    set((state) => ({
      watchlist: state.watchlist.filter((t) => t !== ticker),
    })),

  setFilters: (filters) =>
    set((state) => ({
      filters: { ...state.filters, ...filters },
    })),

  toggleLeftPanel: () =>
    set((state) => ({
      leftPanelOpen: !state.leftPanelOpen,
    })),

  openScreener: () => set({ screenerModalOpen: true }),
  closeScreener: () => set({ screenerModalOpen: false }),

  setCachedCompliance: (ticker, data) =>
    set((state) => {
      const newCache = new Map(state.complianceCache);
      newCache.set(ticker, data);
      return { complianceCache: newCache };
    }),

  setCachedAnalysis: (ticker, data) =>
    set((state) => {
      const newCache = new Map(state.analysisCache);
      newCache.set(ticker, data);
      return { analysisCache: newCache };
    }),
}));
```

---

## 10. Accessibility (WCAG 2.1 AA)

### 10.1 Requirements

1. **Keyboard Navigation**
   - All interactive elements must be keyboard accessible
   - Clear focus indicators (emerald ring)
   - Skip navigation links
   - Arrow key navigation in lists

2. **Screen Reader Support**
   - ARIA labels for all icons
   - ARIA live regions for dynamic content
   - Proper heading hierarchy (h1 → h2 → h3)
   - Alt text for all images/charts

3. **Color Contrast**
   - Text on background: minimum 4.5:1 ratio
   - Large text: minimum 3:1 ratio
   - Status indicators: not relying on color alone (use icons)

4. **Responsive Text**
   - Support text zoom up to 200%
   - No horizontal scrolling at 200% zoom
   - Minimum font size: 14px body text

### 10.2 Implementation Example

```tsx
// Accessible Halal Badge
<div
  role="status"
  aria-label={`Shariah compliance status: ${status}`}
  className="bg-emerald-500 text-white font-bold text-2xl py-3 px-6 rounded-lg text-center"
>
  <span className="sr-only">
    This stock is Shariah compliant based on AAOIFI standards
  </span>
  HALAL
</div>

// Accessible Stock List
<ul role="list" aria-label="Watchlist stocks">
  {stocks.map((stock, index) => (
    <li key={stock.ticker}>
      <button
        onClick={() => selectStock(stock.ticker)}
        onKeyDown={(e) => handleKeyNav(e, index)}
        aria-label={`${stock.ticker} ${stock.name} - ${stock.halalStatus ? 'Halal' : 'Not Halal'}`}
        className="focus:ring-2 focus:ring-emerald-500"
      >
        {/* Stock content */}
      </button>
    </li>
  ))}
</ul>
```

---

## 11. Performance Optimization

### 11.1 Optimization Strategies

1. **Code Splitting**
```typescript
// Lazy load heavy components
const ChartPanel = lazy(() => import('./components/ChartPanel'));
const ScreenerModal = lazy(() => import('./components/ScreenerModal'));
```

2. **Data Caching**
```typescript
// Cache compliance data for 24 hours
const CACHE_TTL = 24 * 60 * 60 * 1000; // 24 hours

function getCachedCompliance(ticker: string) {
  const cached = localStorage.getItem(`compliance_${ticker}`);
  if (!cached) return null;

  const { data, timestamp } = JSON.parse(cached);
  if (Date.now() - timestamp > CACHE_TTL) {
    localStorage.removeItem(`compliance_${ticker}`);
    return null;
  }

  return data;
}
```

3. **Virtual Scrolling**
```tsx
// For long stock lists, use react-window
import { FixedSizeList } from 'react-window';

<FixedSizeList
  height={600}
  itemCount={stocks.length}
  itemSize={50}
  width="100%"
>
  {({ index, style }) => (
    <div style={style}>
      <StockItem stock={stocks[index]} />
    </div>
  )}
</FixedSizeList>
```

4. **WebSocket for Live Data**
```typescript
// Real-time price updates
const ws = new WebSocket('wss://api.halalterminal.com/ws');

ws.onmessage = (event) => {
  const { ticker, price, change } = JSON.parse(event.data);
  updatePrice(ticker, price, change);
};
```

---

## 12. Testing Strategy

### 12.1 Unit Tests

```typescript
// __tests__/utils/shariah-compliance.test.ts
import { checkShariahCompliance } from '@/utils/shariah-compliance';

describe('Shariah Compliance Checker', () => {
  it('should mark stock as halal when all ratios pass', () => {
    const result = checkShariahCompliance({
      debtToAssets: 25.0,
      interestToRevenue: 0.5,
      businessActivity: 'technology',
    });

    expect(result.status).toBe('halal');
    expect(result.ratios.debtToAssets.passed).toBe(true);
    expect(result.ratios.interestToRevenue.passed).toBe(true);
  });

  it('should mark stock as not halal when debt exceeds limit', () => {
    const result = checkShariahCompliance({
      debtToAssets: 40.0, // Above 33% limit
      interestToRevenue: 0.5,
      businessActivity: 'technology',
    });

    expect(result.status).toBe('not_halal');
    expect(result.ratios.debtToAssets.passed).toBe(false);
  });

  it('should mark stock as not halal when business is haram', () => {
    const result = checkShariahCompliance({
      debtToAssets: 20.0,
      interestToRevenue: 0.5,
      businessActivity: 'conventional_banking', // Haram
    });

    expect(result.status).toBe('not_halal');
    expect(result.businessActivity.isCompliant).toBe(false);
  });
});
```

### 12.2 Integration Tests

```typescript
// __tests__/integration/halal-terminal-flow.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { HalalTerminal } from '@/pages/index';

describe('HalalTerminal User Flow', () => {
  it('should display halal status when user selects a stock', async () => {
    render(<HalalTerminal />);

    // Select AAPL from watchlist
    const appleStock = screen.getByText('AAPL');
    fireEvent.click(appleStock);

    // Wait for halal status to load
    await waitFor(() => {
      expect(screen.getByText('HALAL')).toBeInTheDocument();
    });

    // Check ratios are displayed
    expect(screen.getByText(/TOTAL DEBT \/ ASSETS/i)).toBeInTheDocument();
    expect(screen.getByText(/27.5%/i)).toBeInTheDocument();
  });

  it('should filter stocks by halal status in screener', async () => {
    render(<HalalTerminal />);

    // Open screener
    const screenerButton = screen.getByText('SCREENER');
    fireEvent.click(screenerButton);

    // Enable "Halal Only" filter
    const halalOnlyCheckbox = screen.getByLabelText('Halal Only');
    fireEvent.click(halalOnlyCheckbox);

    // Apply filters
    const applyButton = screen.getByText('Apply Filters');
    fireEvent.click(applyButton);

    // Verify only halal stocks are shown
    await waitFor(() => {
      const stocks = screen.getAllByRole('row');
      stocks.forEach(stock => {
        expect(stock).toContainElement(screen.getByTitle('Halal'));
      });
    });
  });
});
```

### 12.3 E2E Tests (Playwright)

```typescript
// e2e/halal-terminal.spec.ts
import { test, expect } from '@playwright/test';

test('complete user journey: search → view → analyze', async ({ page }) => {
  // Navigate to terminal
  await page.goto('https://halalterminal.com');

  // Search for a stock
  await page.fill('input[placeholder="Search"]', 'AAPL');
  await page.keyboard.press('Enter');

  // Wait for stock to load
  await expect(page.locator('text=Apple Inc')).toBeVisible();

  // Check halal status appears
  await expect(page.locator('text=HALAL').first()).toBeVisible();

  // Verify ratios are displayed
  await expect(page.locator('text=TOTAL DEBT / ASSETS')).toBeVisible();
  await expect(page.locator('text=27.5%')).toBeVisible();

  // Open screener
  await page.click('button:has-text("SCREENER")');
  await expect(page.locator('text=Halal Stock Screener')).toBeVisible();

  // Apply filter
  await page.check('label:has-text("Halal Only")');
  await page.click('button:has-text("Apply Filters")');

  // Verify results
  await expect(page.locator('text=Found')).toContainText('247 halal stocks');
});
```

---

## 13. Deployment Architecture

### 13.1 Infrastructure

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=https://api.halalterminal.com
      - NEXT_PUBLIC_WS_URL=wss://api.halalterminal.com/ws
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/halalterminal
      - REDIS_URL=redis://redis:6379
      - FINANCIAL_DATASETS_API_KEY=${FINANCIAL_DATASETS_API_KEY}
    depends_on:
      - db
      - redis

  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=halalterminal
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass

  redis:
    image: redis:7
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### 13.2 Next.js Deployment (Vercel)

```javascript
// next.config.js
module.exports = {
  reactStrictMode: true,
  swcMinify: true,
  images: {
    domains: ['api.halalterminal.com'],
  },
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'https://api.halalterminal.com/:path*',
      },
    ];
  },
};
```

---

## 14. Implementation Roadmap

### Week 1-2: Core Layout & Navigation
- [ ] Set up Next.js project with TypeScript
- [ ] Implement three-panel layout
- [ ] Build top navigation bar
- [ ] Create left panel (watchlist + filters)
- [ ] Design token setup + Tailwind config

### Week 3-4: Halal Status Panel
- [ ] Build halal status badge component
- [ ] Implement ratio display cards
- [ ] Add "How We Might Use This" section
- [ ] Create key stats component
- [ ] Add earnings chart

### Week 5-6: Chart Integration
- [ ] Integrate TradingView Lightweight Charts
- [ ] Add technical indicators (BB, RSI, MA)
- [ ] Implement chart controls (timeframe, zoom)
- [ ] Add volume histogram
- [ ] Drawing tools setup

### Week 7-8: Screener Tool
- [ ] Build screener modal
- [ ] Implement filter system
- [ ] Create results table with sorting
- [ ] Add export functionality
- [ ] Index inclusion filters

### Week 9-10: Quality & Value Analysis
- [ ] Build Quality Pillar component
- [ ] Build Value Pillar component
- [ ] Calculate quality metrics (profitability, growth)
- [ ] Calculate value metrics (fair value, discount)
- [ ] Integrate with backend APIs

### Week 11-12: Data Integration & API
- [ ] Build API client
- [ ] Implement data caching layer
- [ ] WebSocket for live prices
- [ ] Connect to Islamic finance tools from Dexter
- [ ] Error handling + retry logic

### Week 13-14: Testing & Polish
- [ ] Write unit tests (80% coverage)
- [ ] Write integration tests
- [ ] E2E tests with Playwright
- [ ] Accessibility audit (WCAG 2.1 AA)
- [ ] Performance optimization

### Week 15-16: Deployment & Launch
- [ ] Set up production infrastructure
- [ ] Deploy to Vercel + AWS
- [ ] Configure CDN (Cloudflare)
- [ ] Set up monitoring (Sentry, LogRocket)
- [ ] Launch beta

---

## 15. Success Metrics

### 15.1 User Engagement
- **Daily Active Users (DAU)**: Target 1,000 in month 1
- **Session Duration**: Target 10+ minutes average
- **Screener Usage**: 40% of users run screener weekly
- **Watchlist Size**: Average 10-15 stocks per user

### 15.2 Technical Performance
- **Page Load Time**: < 2 seconds (p95)
- **API Response Time**: < 500ms (p95)
- **Chart Render Time**: < 1 second
- **Uptime**: 99.9%

### 15.3 Business Metrics
- **Free → Pro Conversion**: Target 5%
- **Monthly Recurring Revenue (MRR)**: Target $50K in month 6
- **Customer Acquisition Cost (CAC)**: < $100
- **Lifetime Value (LTV)**: > $600

---

## Conclusion

This specification provides a complete blueprint for implementing the HalalTerminal.com interface shown in the screenshot. The design balances:

1. **Bloomberg-style professional UI** for traders and advisors
2. **Shariah compliance transparency** with detailed ratios and sources
3. **Educational context** to help users understand Islamic finance
4. **Multi-source validation** for authoritative compliance verdicts
5. **Performance optimization** for real-time data and charts

**Next Steps:**
1. Review this specification with stakeholders
2. Set up development environment
3. Begin Week 1-2 implementation (Core Layout)
4. Schedule weekly demos to gather feedback

The complete codebase structure, components, APIs, and deployment strategy are now fully documented and ready for implementation.

---

**Document Version**: 1.0
**Last Updated**: 2026-01-14
**Total Pages**: 75+ sections covering all implementation details
