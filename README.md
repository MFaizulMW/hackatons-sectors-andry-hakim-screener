# Andry Hakim Transformation Screener

An autonomous n8n workflow that scans the Indonesia Stock Exchange (IDX) every weekday morning for **"transformation" stocks** in the style of Indonesian investor Andry Hakim: small companies going through a new controlling owner, an early rights issue (1st or 2nd), and an owner who does not dump shares.

**Track:** Automation & Workflows · **Data:** [Sectors API](https://sectors.app) · **Built with:** n8n, Claude (Haiku + Sonnet), Google Sheets

> Inspired by Andry Hakim's publicly shared framework. Not his recommendation, not investment advice, and no automated trading.

---

## Problem

Retail investors who follow this framework have to check 900+ IDX tickers by hand every day: rights issue calendars, ownership filings, price, fundamentals, and news. That is slow, and doing it by feel makes it biased. This workflow turns the framework into **fixed, objective rules** that run on their own and explain every decision.

## What's in this repo

| File | What it is |
|---|---|
| `wf1_v4_andry_hakim.json` | **WF1, the screener** (34 nodes). Runs at 07:30 WIB Mon–Fri (live), or when called by WF2 (backtest). |
| `wf2_backtest_runner.json` | **WF2, the backtest runner.** On a schedule, it calls WF1 with a historical `as_of_date`, stepping 60 days from 2022-03-31 to 2026-09-30. |
| `README.md` | This file. |

All logic lives inside the n8n Code nodes (JavaScript) in the JSON files.

## How it works

```
Trigger (07:30 WIB live  |  WF2 backtest)
  └─ Konfigurasi: all rules + point-in-time date windows
① RADAR            Rights issue calendar · Owner whitelist screener · Market-wide ownership filings (change of control)
② MARKET GATE      90-day price: market cap ≤ Rp10T, median daily value ≥ Rp50M, dilution, rights price vs market
③ CHECKS           Revenue growth / profit turnaround / negative equity · Rights issue # (1st/2nd) · 12-month owner selling
④ AI NARRATIVE     Claude Haiku reads 120 days of news → real transformation vs internal restructuring / bailout
⑤ SCORE (0–100)    Hard gates + 6 pillars, pass ≥ 60
   └─ (backtest) performance from signal to today
   └─ Claude Sonnet writes a short thesis
⑥ REPORT           Every run is logged to Google Sheets (`run_log`), including rejected tickers and why
```

Every stage has a fall-through to the report, so a run with 0 picks is still logged with reasons.

### Scoring (100 points)

| Pillar | Max | Rule |
|---|---|---|
| Catalyst | 30 | New controller 30 · 1st rights issue 30 · 2nd rights issue 20 · 3rd+ 0 |
| Owner commitment | 25 | Rights price close to market + no selling by major holders (+5 if they added) |
| Owner quality | 15 | Whitelisted owner or Sectors conglomerate tag 15 · strong controller per news (AI) 8 |
| Room to grow | 15 | Pre-event market cap ≤ Rp0.5T 15 · ≤ Rp2T 10 · ≤ Rp5T 5 |
| Fundamentals | 10 | Revenue growth ≥ 20% OR profit turnaround |
| Timing | 5 | Price ≤ 2× its 90-day low |

**Hard gates (instant reject):**
- Owner sold ≥ 1% in 12 months.
- Negative equity without a change of control.
- AI classifies the event as internal restructuring, tender offer, or a financial rescue.
- Dilution ≥ 10× without a strong owner.

All parameters are **frozen** in version `v4-beku-2026-10-08`. No ticker is hardcoded. The only manual list is a small owner whitelist (`Prajogo Pangestu`, `Barito Pacific`, `Multi Artha Pratama`). Most owner quality comes from Sectors' own conglomerate tags.

## Sectors API endpoints used

| Endpoint | Used for |
|---|---|
| `/v2/corporate-actions/?type=right_issue` | Rights issue calendar (radar) |
| `/v2/filings/` | Market-wide change-of-control detection, and 12-month owner behaviour per candidate (incl. `idx_conglomerates_group_slug`) |
| `/v2/companies/` (`where`) | Owner whitelist, revenue growth, profit turnaround, negative equity screens |
| `/v2/company/corporate-actions/{symbol}/` | Full rights issue history → which number this rights issue is |
| `/v2/daily/{symbol}/` | Market cap, liquidity, rights price vs market, timing, backtest performance |
| `/v2/news/` | News for the AI narrative check |

## Backtest without look-ahead

WF2 replays history: for each `as_of_date`, WF1 only queries data **up to that date** (calendar, filings, price, news, and the last annual report already published). Performance is measured only **after** the decision, from the signal price to the latest close, and is never used in scoring.

Example: PANI's 1st rights issue (Aug 2022) passes with score 86. Its 2nd rights issue is rejected because the company had become too large (~Rp63T).

**Honest note:** PANI passes partly because its controller is in the owner whitelist, which was chosen with hindsight. PANI is an illustration that the logic works, not proof of predictive power.

## Setup

1. Import `wf1_v4_andry_hakim.json` and `wf2_backtest_runner.json` into n8n.
2. Create credentials (no keys are stored in this repo):
   - **Header Auth** for Sectors: name `Authorization`, value = your Sectors API key.
   - **Anthropic** API key (Claude Haiku 4.5 + Claude Sonnet).
   - **Google Sheets** OAuth, plus a sheet with a `run_log` tab. Columns: `timestamp, mode, as_of_date, ri_dalam_jendela, akuisisi_terdeteksi, kandidat, lolos, simbol_lolos, pesan`.
3. Publish WF1. In WF2, set **Jalankan WF1** to WF1's ID, then publish WF2 to start the backtest. Unpublish WF2 when the last date is done.
4. (Optional) Enable the Telegram node for live alerts.

## Limitations

- Sectors ownership filings are sparse before 2022 (before 2024 for some tickers), so older takeovers may be missed.
- The rights issue calendar does not say who exercised. Owner commitment is inferred from the rights price and filings.
- Thresholds (20% growth, Rp10T cap, etc.) are judgment-based round numbers, frozen to avoid data snooping.
- Price data may not be adjusted for rights issues or splits, so performance figures should be sanity-checked.
