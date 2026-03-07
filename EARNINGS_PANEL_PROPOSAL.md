# Proposal: Earnings Calendar Panel for Finance Variant

**Issue:** #1010  
**Author:** nKOxxx  
**Date:** March 7, 2026  
**Status:** Ready for Implementation

---

## Summary

Add a dedicated "Earnings Calendar" panel to the Finance variant showing upcoming and recent earnings reports for major companies. Addresses request from #1010.

---

## Problem

The current Finance variant lacks visibility into earnings reports - a critical data point for traders and investors. Users must leave World Monitor to check earnings calendars on Yahoo Finance, Bloomberg, or company IR pages.

**User Quote (#1010):** *"Those 2 panels are quite important for financiers"*

---

## Proposed Solution

### Two-Panel Approach

**1. Upcoming Earnings Panel**
- List companies reporting in next 7/30 days
- Show: Company name, ticker, report date, expected EPS
- Filter by: Market cap, sector, importance

**2. Recent Earnings Panel**  
- Companies that reported in last 7 days
- Show: Company name, ticker, actual EPS vs expected, surprise %
- AI summary of key takeaways (optional)

---

## Data Sources

**Primary:** Yahoo Finance API (free tier available)
- `https://query1.finance.yahoo.com/v1/finance/calendar/earnings`
- No API key required
- Returns: symbol, name, report date, EPS estimate/actual

**Backup:** Alpha Vantage (free tier: 5 calls/min, 500/day)
- `https://www.alphavantage.co/query?function=EARNINGS_CALENDAR`
- Requires API key
- More detailed data

**Company List:** S&P 500 + Nasdaq 100 + Regional majors (for Gulf variant: ADNOC, Emirates NBD, etc.)

---

## Technical Implementation

### 1. Proto Definition

**File:** `proto/worldmonitor/market/v1/earnings.proto`

```protobuf
syntax = "proto3";

package worldmonitor.market.v1;

import "buf/validate/validate.proto";
import "sebuf/http/annotations.proto";

service EarningsService {
  rpc GetEarningsCalendar(GetEarningsCalendarRequest)
    returns (GetEarningsCalendarResponse) {
    option (sebuf.http.config) = {
      path: "/api/market/v1/earnings-calendar"
      method: POST
    };
  }
}

message GetEarningsCalendarRequest {
  // "upcoming" or "recent"
  string view = 1;
  
  // Days ahead/behind (default: 7)
  int32 days = 2;
  
  // Filter by market cap: "mega", "large", "all"
  string market_cap_filter = 3;
}

message GetEarningsCalendarResponse {
  repeated EarningsReport reports = 1;
}

message EarningsReport {
  string symbol = 1;
  string company_name = 2;
  
  // Report date as Unix epoch milliseconds
  int64 report_date = 3 [
    (sebuf.http.int64_encoding) = INT64_ENCODING_NUMBER
  ];
  
  // Time of day: "bmo" (before market open), "amc" (after market close)
  string report_time = 4;
  
  // EPS data (null for upcoming)
  double eps_estimate = 5;
  double eps_actual = 6;
  
  // Surprise % ((actual - estimate) / estimate * 100)
  double surprise_percent = 7;
  
  // Revenue data (if available)
  double revenue_estimate = 8;
  double revenue_actual = 9;
  
  // Classification
  string market_cap_tier = 10;  // "mega", "large", "mid"
  string sector = 11;
}
```

### 2. Server Implementation

**File:** `server/worldmonitor/market/v1/earnings.ts`

```typescript
import type { ServerContext, GetEarningsCalendarRequest, GetEarningsCalendarResponse } from './service_server';

const YAHOO_EARNINGS_URL = 'https://query1.finance.yahoo.com/v1/finance/calendar/earnings';

export async function getEarningsCalendar(
  req: GetEarningsCalendarRequest,
  _ctx: ServerContext
): Promise<GetEarningsCalendarResponse> {
  const symbols = getMajorSymbols(req.market_cap_filter);
  const reports = await fetchYahooEarnings(symbols, req.view, req.days);
  
  return { reports };
}

async function fetchYahooEarnings(
  symbols: string[], 
  view: string, 
  days: number
): Promise<EarningsReport[]> {
  // Fetch from Yahoo Finance
  // Cache in Redis (TTL: 1 hour for upcoming, 6 hours for recent)
  // Parse and return
}

function getMajorSymbols(tier: string): string[] {
  if (tier === 'mega') return MEGA_CAP_SYMBOLS;  // Top 50
  if (tier === 'large') return LARGE_CAP_SYMBOLS; // Top 500
  return ALL_SYMBOLS;
}
```

### 3. Panel Component

**File:** `src/components/EarningsPanel.ts`

```typescript
import { Panel } from './Panel';
import { t } from '@/services/i18n';
import { MarketServiceClient } from '@/generated/client/worldmonitor/market/v1/service_client';

type ViewMode = 'upcoming' | 'recent';

export class EarningsPanel extends Panel {
  private viewMode: ViewMode = 'upcoming';
  private client = new MarketServiceClient('', { fetch: (...args) => globalThis.fetch(...args) });

  constructor(id: string) {
    super({ id, title: t('panels.earnings'), showCount: true });
    this.fetchEarnings();
  }

  private async fetchEarnings(): Promise<void> {
    const data = await this.client.getEarningsCalendar({
      view: this.viewMode,
      days: this.viewMode === 'upcoming' ? 7 : 7,
      market_cap_filter: 'mega'
    });
    
    this.setCount(data.reports.length);
    this.render(data.reports);
  }

  private render(reports: EarningsReport[]): void {
    const html = reports.map(r => `
      <div class="earnings-item">
        <div class="earnings-header">
          <span class="symbol">${escapeHtml(r.symbol)}</span>
          <span class="company">${escapeHtml(r.company_name)}</span>
          <span class="date">${formatDate(r.report_date)}</span>
        </div>
        ${r.eps_estimate ? `
          <div class="earnings-estimate">
            Est: $${r.eps_estimate.toFixed(2)}
          </div>
        ` : ''}
        ${r.eps_actual ? `
          <div class="earnings-actual ${r.surprise_percent >= 0 ? 'positive' : 'negative'}">
            Actual: $${r.eps_actual.toFixed(2)} 
            (${r.surprise_percent >= 0 ? '+' : ''}${r.surprise_percent.toFixed(1)}%)
          </div>
        ` : ''}
      </div>
    `).join('');

    this.setContent(html);
  }
}
```

### 4. Add to Finance Variant

**File:** `src/config/variants/finance.ts`

Add to panels config:
```typescript
earnings: { name: 'Earnings Calendar', enabled: true, priority: 2 },
```

---

## UI Mockup

```
┌─────────────────────────────────────┐
│ EARNINGS CALENDAR              [15] │
├─────────────────────────────────────┤
│ [Upcoming] [Recent]                 │
├─────────────────────────────────────┤
│ AAPL  Apple Inc.          Mar 12    │
│       Est: $1.50                    │
│                                     │
│ MSFT  Microsoft           Mar 13    │
│       Est: $2.89                    │
│                                     │
│ NVDA  NVIDIA Corp         Mar 14    │
│       Est: $4.92                    │
├─────────────────────────────────────┤
│ Mega Cap • 7 Days • Filter ▼       │
└─────────────────────────────────────┘
```

**Recent View:**
```
│ TSLA  Tesla Inc.          Mar 7     │
│       $0.73 vs $0.60 est  (+21.7%)  │
│       [AI Summary: Beat on...]      │
```

---

## Implementation Approach

**Phase 1: Core**
- Add proto definitions
- Implement Yahoo Finance fetcher
- Basic panel component

**Phase 2: Polish**
- Add to Finance variant config
- Styling matching WM design
- Error handling & loading states

**Phase 3: Enhancements (Optional)**
- AI summary integration
- Historical chart
- Sector filtering

---

## Success Criteria

- [ ] Panel shows upcoming earnings for S&P 50
- [ ] Panel shows recent earnings with surprise %
- [ ] Data updates hourly (cached)
- [ ] Works in Finance variant
- [ ] Mobile responsive

---

## Open Questions

1. **Data scope:** S&P 500 only, or include international (FTSE, Nikkei, GCC)?
2. **AI summaries:** Use existing WM AI service, or skip for MVP?
3. **Real-time:** Show intraday updates, or daily batch sufficient?

---

## Files to Modify

```
proto/worldmonitor/market/v1/
├── earnings.proto          (NEW)
└── service.proto           (add rpc)

server/worldmonitor/market/v1/
├── earnings.ts             (NEW - handler)
└── handler.ts              (register)

src/components/
└── EarningsPanel.ts        (NEW)

src/config/variants/
└── finance.ts              (add panel)

api/market/v1/
└── [rpc].ts                (gateway)
```

---

**Ready to implement. Seeking maintainer approval on approach.**
