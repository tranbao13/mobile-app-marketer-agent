---
name: mmp-reporting-assistant
display_name: MMP Reporting Assistant — Adjust Account Health & App Snapshots
description: >
  Teaches Claude to act as a senior mobile app marketing analyst using Adjust MMP data.
  Provides two core workflows: (1) full account health check across all apps,
  (2) per-app snapshot dashboard with install trends, revenue breakdown, and DAU charts.
  Flags revenue anomalies based on company monetization model (IAA, IAP, subscription).
version: 1.0.0
author: tranbao13
license: MIT
tags:
  - marketing
  - mobile advertisement
  - mmp
  - adjust
  - analytics
  - ua
  - user-acquisition
  - advertising revenue
requires:
  mcp:
    - name: adjust
      description: Adjust MCP reporting server (adjust-copilot or @adjustcom/mcp-server)
platforms:
  - claude.ai
  - claude-desktop
  - claude-code
---

# ─────────────────────────────────────────────────────────────
# Skill: MMP Reporting
# Version: 1.0.0
# Requires: Adjust MCP server connected
# ─────────────────────────────────────────────────────────────

## Overview

This skill handles two core MMP reporting workflows:

  Trigger 1 — Initial account health check
  Trigger 2 — App snapshot report (full dashboard)

Both workflows call the Adjust MCP reporting tool using natural language
questions and process the returned CSV to generate marketer-ready outputs.


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## TRIGGER 1 — Initial account health check
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

### Activation

Triggered by any of the following prompts (or close paraphrases):
  - "Do an initial check"
  - "Check my MMP account status"
  - "Summarize my MMP account"
  - "What apps do I have in my account"
  - "Give me an account overview"

---

### Step 1 — Pull account-wide data

Call the Adjust reporting tool with:

  "Show all apps with their installs, sessions, daus, revenue, and
  ad_revenue for the last 30 days, broken down by app"

Parse the returned CSV. Each row represents one app for the period.

---

### Step 2 — Compute per-app metrics

For each app row, compute:

  true_total_revenue = revenue + ad_revenue
    (revenue field = IAP only; ad_revenue is separate — always sum both)

  is_active = (installs > 0) OR (sessions > 0) OR (daus > 0)

  has_revenue = true_total_revenue > 0

---

### Step 3 — Apply company-level context from profile

The profile defines company-wide defaults — not per-app rules.
Use profile.business.monetization as the baseline expectation for
ALL active apps in the account.

Use profile.business.notes to catch known exceptions before flagging.
Example: if notes mention "one banking app — zero revenue expected",
do not flag that app even if it shows zero revenue.

You do NOT need to match apps individually against the profile.
App discovery is fully handled by the API response.

---

### Step 4 — Classify each app into a bucket

  BUCKET A — Active
    Condition: is_active = true
    Meaning: app has production data in the past 30 days

  BUCKET B — Inactive
    Condition: is_active = false (all metrics = 0)
    Meaning: app may be paused, in test/sandbox mode, or sunset
    Do NOT assume it is broken — state possibilities

  BUCKET C — Revenue anomaly (needs confirmation)
    Condition: is_active = true
      AND profile monetization includes any of: IAA, IAP, subscription
      AND has_revenue = false
      AND app is NOT covered by an exception in profile.business.notes

    Do NOT immediately flag as broken. Instead, ask the marketer:

    "I noticed [App Name] has [X] installs and [Y] sessions in the past
    30 days but zero revenue recorded. Based on your profile, your apps
    are expected to generate [monetization model] revenue.

    Is this app:
    a) An IAA/IAP app where revenue tracking may be broken?
    b) A non-monetized app (e.g. finance, brand, utility)?
    c) A new app still in soft launch with monetization not yet active?

    Your answer helps me flag this correctly."

    After the marketer responds, classify accordingly and note the
    confirmed model in your reply so they can optionally add it to
    their profile notes.

---

### Step 5 — Format and output

Produce the following markdown report:

---
## MMP Account Health Check
*[Account Name] · Adjust · Last 30 days · [start_date] to [end_date]*

**Summary**
- Total apps in account: X
- Active (production data present): Y
- Inactive (no data in past 30 days): Z
- Revenue anomalies detected: N

---

### ✅ Active apps (Y)

| App | Platform | Installs | Sessions | Avg DAU | Revenue |
|-----|----------|----------|----------|---------|---------|
| [name] | iOS | 1,820 | 33,598 | 621 | $43.8M |

---

### ⚠️ Inactive apps (Z)

These apps recorded zero installs and sessions in the past 30 days.
This may be expected (paused campaigns) or may need investigation.

| App | Platform | Possible reason |
|-----|----------|-----------------|
| [name] | Android | Paused, sandbox mode, or sunset |

---

### 🚨 Revenue anomalies (N)

These apps are active but showing zero revenue despite being configured
as [monetization model] apps. This requires investigation.

| App | Platform | Monetization | Installs | Sessions | Revenue |
|-----|----------|--------------|----------|----------|---------|
| [name] | iOS | IAA | 5,200 | 41,000 | $0 |

**Why this matters:** A [monetization] app with sessions but zero revenue
typically means the ad SDK is not firing, mediation is misconfigured,
or revenue postbacks to Adjust are broken. Left undetected, this causes
underreported ROAS and incorrect campaign decisions.

---

*To investigate any app in detail:*
*"Show me a snapshot report of [App Name] in the last 3 months"*


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## TRIGGER 2 — App snapshot report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

### Activation

Triggered by any of the following (or close paraphrases):
  - "Show me a snapshot report of [App]"
  - "Give me a dashboard for [App]"
  - "Quick report on [App] in [period]"
  - "How is [App] performing"
  - "Show me [App]'s performance in [period]"

---

### Step 1 — Identify app and date range

App identification:
  - Match [App] against profile app names (case-insensitive)
  - If iOS and Android versions both exist and prompt is ambiguous:
    ask "Which platform — iOS, Android, or both?"
  - If no match found: ask the marketer to clarify

Date range:
  - Extract from prompt if specified ("last 3 months", "Q1", "last week")
  - Default if not specified: last 90 days
  - Compute start_date (YYYY-MM-DD) and end_date (YYYY-MM-DD)

---

### Step 2 — Pull three data sets from Adjust

Make these three tool calls in sequence:

  QUERY A — Daily installs by network:
  "Show daily installs broken down by network for [App Name]
  from [start_date] to [end_date]"

  QUERY B — Daily revenue:
  "Show daily revenue and ad_revenue for [App Name]
  from [start_date] to [end_date]"

  QUERY C — Daily engagement:
  "Show daily sessions and DAU for [App Name]
  from [start_date] to [end_date]"

If a query returns no data, note it in the dashboard header and
skip the corresponding chart. Do not fabricate placeholder data.

---

### Step 3 — Process and aggregate the data

From QUERY A (installs by network):
  organic_installs   = sum of rows where network = "Organic"
                       + sum of rows where network contains "Organic Search"
  untrusted_installs = sum of rows where network = "Untrusted Devices"
  paid_installs      = sum of all remaining network rows
  total_installs     = organic + untrusted + paid
  organic_pct        = organic_installs / total_installs × 100

From QUERY B (revenue):
  iap_revenue_total  = sum of all `revenue` or `all_revenue` column values
  ad_revenue_total   = sum of all `ad_revenue` column values
  true_total_revenue = iap_revenue_total + ad_revenue_total
  iap_pct            = iap_revenue_total / true_total_revenue × 100
  ad_pct             = ad_revenue_total / true_total_revenue × 100

From QUERY C (engagement):
  avg_dau            = mean of dau column (ignore null/zero days)
  peak_dau           = max of dau column
  total_sessions     = sum of sessions column
  dau_first_week_avg = mean of first 7 non-null dau values
  dau_last_week_avg  = mean of last 7 non-null dau values
  dau_trend_pct      = (dau_last_week_avg - dau_first_week_avg)
                       / dau_first_week_avg × 100

Weekly revenue aggregation for Chart 2:
  Group daily revenue rows into 7-day buckets (week starting each Monday
  or simply every 7 days from start_date). Sum iap and ad within each week.

7-day moving average for DAU Chart 3:
  For each day i (where i ≥ 6), compute mean of days [i-6 .. i].
  Leave null for first 6 days.

---

### Step 4 — Determine health badge

  🟢 Healthy:
    - Revenue > 0 if monetization is not [none]
    - dau_trend_pct > -10% (DAU stable or growing)
    - No paid channels missing if paid UA is expected

  🟡 Watch:
    - dau_trend_pct between -10% and -30%
    - Revenue present but declining
    - Untrusted installs > 5% of total

  🔴 Alert:
    - dau_trend_pct < -30%
    - Revenue = 0 for monetized app
    - Install cliff: any 7-day period shows >40% drop vs prior period

---

### Step 5 — Generate insight bullets

Compute and include 3–5 of the following (only those relevant to the data):

  DAU trend:
    "DAU declined [X]% over the period — from [first_week_avg] in [start_month]
    to [last_week_avg] in [end_month]. No recovery signal visible."
    OR
    "DAU grew [X]% over the period — from [first_week_avg] to [last_week_avg].
    Positive engagement trend."

  Paid UA:
    "Zero paid UA channels detected — 100% of installs are organic.
    Growth is entirely dependent on store visibility and word of mouth."
    OR
    "[N] paid channels detected ([channel names]). Paid installs represent
    [paid_pct]% of total."

  Revenue concentration:
    If max single-day revenue > 5× daily average:
    "Revenue is highly concentrated — peak single-day revenue ([date]: [amount])
    is [X]× the daily average. High whale dependency; monitor cohort LTV."

  Untrusted devices:
    If untrusted_pct > 5%:
    "Untrusted device installs represent [X]% of total. Adjust is flagging
    these as failing fraud checks. They inflate raw install counts and may
    skew attribution data."

  Ad revenue health (IAA apps only):
    If ad_pct < 1% and profile monetization includes IAA:
    "Ad revenue is [X]% of total revenue — unusually low for an IAA app.
    Check ad SDK initialization and mediation waterfall configuration."

  Positive signals:
    If revenue > 0 and dau_trend_pct > -5%:
    "Revenue pipeline healthy — [monetization type] revenue recording
    consistently with no gaps in the reporting period."

---

### Step 6 — Output HTML dashboard artifact

Generate a self-contained HTML artifact using the template below.
All data must be embedded inline as JavaScript objects.
Use Chart.js (CDN: https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js).

---

## HTML DASHBOARD TEMPLATE

Use this exact structure. Replace [PLACEHOLDERS] with computed values.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
.dash{padding:1.25rem 0}
.hdr{display:flex;justify-content:space-between;align-items:flex-start;margin-bottom:1.25rem;gap:1rem;flex-wrap:wrap}
.app-name{font-size:20px;font-weight:500;color:var(--color-text-primary);margin-bottom:4px}
.app-meta{font-size:12px;color:var(--color-text-secondary);line-height:1.7}
.badge{padding:5px 13px;border-radius:20px;font-size:12px;font-weight:500;white-space:nowrap}
.b-green{background:#EAF3DE;color:#3B6D11}
.b-watch{background:#FAEEDA;color:#854F0B}
.b-alert{background:#FCEBEB;color:#A32D2D}
.kpi-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:10px;margin-bottom:12px}
.kpi{background:var(--color-background-secondary);border:0.5px solid var(--color-border-tertiary);border-radius:var(--border-radius-lg);padding:12px 14px}
.kpi-label{font-size:10px;color:var(--color-text-tertiary);text-transform:uppercase;letter-spacing:.5px;margin-bottom:5px}
.kpi-val{font-size:21px;font-weight:500;color:var(--color-text-primary);line-height:1.1}
.kpi-sub{font-size:10px;color:var(--color-text-tertiary);margin-top:3px}
.chart-card{background:var(--color-background-secondary);border:0.5px solid var(--color-border-tertiary);border-radius:var(--border-radius-lg);padding:14px;margin-bottom:10px}
.ct{font-size:13px;font-weight:500;color:var(--color-text-primary);margin-bottom:2px}
.cs{font-size:11px;color:var(--color-text-tertiary);margin-bottom:12px}
.cw{position:relative;height:170px}
.ins{background:var(--color-background-secondary);border:0.5px solid var(--color-border-tertiary);border-radius:var(--border-radius-lg);padding:14px}
.ins-title{font-size:13px;font-weight:500;color:var(--color-text-primary);margin-bottom:10px}
.ii-row{display:flex;gap:9px;margin-bottom:9px;font-size:12px;color:var(--color-text-secondary);line-height:1.5}
.ii-row:last-child{margin-bottom:0}
.ii{flex-shrink:0;width:17px;height:17px;border-radius:50%;display:flex;align-items:center;justify-content:center;font-size:9px;font-weight:700;margin-top:1px}
.ii-w{background:#FAEEDA;color:#854F0B}
.ii-ok{background:#EAF3DE;color:#3B6D11}
.ii-i{background:#E6F1FB;color:#185FA5}
.ii-a{background:#FCEBEB;color:#A32D2D}
.em{font-weight:500;color:var(--color-text-primary)}
</style>

<div class="dash">
  <div class="hdr">
    <div>
      <div class="app-name">[APP_NAME]</div>
      <div class="app-meta">[PLATFORM] · [VERTICAL] · [MONETIZATION] · [DATE_RANGE]</div>
    </div>
    <div class="badge [BADGE_CLASS]">[BADGE_EMOJI] [BADGE_LABEL]</div>
  </div>

  <div class="kpi-grid">
    <div class="kpi">
      <div class="kpi-label">Total installs</div>
      <div class="kpi-val" id="v-inst">—</div>
      <div class="kpi-sub" id="s-org">—</div>
    </div>
    <div class="kpi">
      <div class="kpi-label">Untrusted devices</div>
      <div class="kpi-val" id="v-unt">—</div>
      <div class="kpi-sub" id="s-unt">of total installs</div>
    </div>
    <div class="kpi">
      <div class="kpi-label">Avg DAU</div>
      <div class="kpi-val" id="v-dau">—</div>
      <div class="kpi-sub" id="s-dau">—</div>
    </div>
    <div class="kpi">
      <div class="kpi-label">Total revenue</div>
      <div class="kpi-val" id="v-rev">—</div>
      <div class="kpi-sub">IAP + Ad revenue</div>
    </div>
    <div class="kpi">
      <div class="kpi-label">IAP revenue</div>
      <div class="kpi-val" id="v-iap">—</div>
      <div class="kpi-sub" id="s-iap">of total</div>
    </div>
    <div class="kpi">
      <div class="kpi-label">Ad revenue</div>
      <div class="kpi-val" id="v-adr">—</div>
      <div class="kpi-sub" id="s-adr">of total</div>
    </div>
  </div>

  <div class="chart-card">
    <div class="ct">Daily installs — organic vs untrusted devices</div>
    <div class="cs" id="inst-sub">—</div>
    <div class="cw"><canvas id="ch-inst"></canvas></div>
  </div>

  <div class="chart-card">
    <div class="ct">Weekly revenue — IAP vs ad revenue</div>
    <div class="cs">Aggregated by week · daily figures can be spiky</div>
    <div class="cw"><canvas id="ch-rev"></canvas></div>
  </div>

  <div class="chart-card">
    <div class="ct">Daily active users</div>
    <div class="cs">With 7-day moving average</div>
    <div class="cw"><canvas id="ch-dau"></canvas></div>
  </div>

  <div class="ins">
    <div class="ins-title">Quick evaluation</div>
    [INSERT_INSIGHT_ITEMS_HERE]
    <!-- Each insight follows this pattern:
    <div class="ii-row">
      <div class="ii ii-w">!</div>
      <div><span class="em">Insight headline</span> — supporting detail with specific numbers.</div>
    </div>
    Icon classes: ii-w (warning), ii-ok (healthy), ii-i (info), ii-a (alert)
    -->
  </div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js"></script>
<script>
// ── Embed all data here ──────────────────────────────────────
// Format: { "YYYY-MM-DD": { o: organic, u: untrusted, p: paid } }
const instData = [PASTE_INSTALL_DATA_OBJECT];

// Format: { "YYYY-MM-DD": { r: iap_revenue, a: ad_revenue } }
const revData = [PASTE_REVENUE_DATA_OBJECT];

// Format: { "YYYY-MM-DD": { s: sessions, d: dau } }
const engData = [PASTE_ENGAGEMENT_DATA_OBJECT];

// ── Date range ───────────────────────────────────────────────
function dateRange(s, e) {
  const out = [], c = new Date(s);
  while (c <= new Date(e)) { out.push(c.toISOString().slice(0,10)); c.setDate(c.getDate()+1); }
  return out;
}
const dates = dateRange("[START_DATE]", "[END_DATE]");

// ── Formatters ───────────────────────────────────────────────
const fmtR = v => v >= 1e6 ? '$'+(v/1e6).toFixed(1)+'M' : v >= 1e3 ? '$'+(v/1e3).toFixed(0)+'K' : '$'+v.toFixed(0);
const fmtN = v => v >= 1e3 ? (v/1e3).toFixed(1)+'K' : String(v);
const pct  = (a, b) => b > 0 ? (a/b*100).toFixed(1)+'%' : '—';

// ── Compute KPIs ─────────────────────────────────────────────
let tO=0, tU=0, tP=0, tR=0, tA=0, dauSum=0, dauCnt=0;
dates.forEach(d => {
  if (instData[d]) { tO += instData[d].o||0; tU += instData[d].u||0; tP += instData[d].p||0; }
  if (revData[d])  { tR += revData[d].r||0; tA += revData[d].a||0; }
  if (engData[d] && engData[d].d) { dauSum += engData[d].d; dauCnt++; }
});
const tI = tO + tU + tP;
const tRev = tR + tA;

document.getElementById('v-inst').textContent = fmtN(tI);
document.getElementById('s-org').textContent  = pct(tO, tI) + ' organic';
document.getElementById('v-unt').textContent  = fmtN(tU);
document.getElementById('s-unt').textContent  = pct(tU, tI) + ' of installs';
document.getElementById('v-dau').textContent  = dauCnt ? Math.round(dauSum/dauCnt) : '—';
document.getElementById('s-dau').textContent  = 'Peak: ' + Math.max(...dates.map(d => engData[d]?.d||0));
document.getElementById('v-rev').textContent  = fmtR(tRev);
document.getElementById('v-iap').textContent  = fmtR(tR);
document.getElementById('s-iap').textContent  = pct(tR, tRev) + ' of total';
document.getElementById('v-adr').textContent  = fmtR(tA);
document.getElementById('s-adr').textContent  = pct(tA, tRev) + ' of total';

// Installs chart subtitle
const hasPaid = tP > 0;
document.getElementById('inst-sub').textContent = hasPaid
  ? 'Paid UA detected · ' + pct(tP, tI) + ' paid installs'
  : 'No paid UA channels detected · 100% organic acquisition';

// ── Chart config defaults ────────────────────────────────────
const dark = matchMedia('(prefers-color-scheme: dark)').matches;
const gc   = dark ? 'rgba(255,255,255,0.07)' : 'rgba(0,0,0,0.06)';
const tc   = dark ? '#888780' : '#888780';
const base = {
  responsive: true, maintainAspectRatio: false,
  plugins: {
    legend: { labels: { color: tc, font: { size: 11 }, boxWidth: 10, padding: 8 } },
    tooltip: { mode: 'index', intersect: false }
  },
  scales: {
    x: { ticks: { color: tc, font: { size: 10 }, maxTicksLimit: 9, maxRotation: 0 }, grid: { color: gc } },
    y: { ticks: { color: tc, font: { size: 10 } }, grid: { color: gc } }
  }
};

// ── Chart 1: Daily installs ──────────────────────────────────
new Chart(document.getElementById('ch-inst'), {
  type: 'bar',
  data: {
    labels: dates.map(d => d.slice(5)),
    datasets: [
      { label: 'Organic',           data: dates.map(d => instData[d]?.o||0), backgroundColor: 'rgba(93,184,122,0.8)',  borderRadius: 1, borderSkipped: false },
      { label: 'Untrusted devices', data: dates.map(d => instData[d]?.u||0), backgroundColor: 'rgba(232,89,60,0.75)', borderRadius: 1, borderSkipped: false },
      { label: 'Paid',              data: dates.map(d => instData[d]?.p||0), backgroundColor: 'rgba(74,144,217,0.8)',  borderRadius: 1, borderSkipped: false }
    ]
  },
  options: { ...base }
});

// ── Chart 2: Weekly revenue ──────────────────────────────────
const wk = [], wIAP = [], wAd = [];
for (let i = 0; i < dates.length; i += 7) {
  const sl = dates.slice(i, i+7);
  wk.push(sl[0].slice(5));
  let ip = 0, ad = 0;
  sl.forEach(d => { if (revData[d]) { ip += revData[d].r||0; ad += revData[d].a||0; } });
  wIAP.push(Math.round(ip));
  wAd.push(Math.round(ad));
}
new Chart(document.getElementById('ch-rev'), {
  type: 'bar',
  data: {
    labels: wk,
    datasets: [
      { label: 'IAP revenue', data: wIAP, backgroundColor: 'rgba(155,89,182,0.8)',  borderRadius: 3, borderSkipped: false },
      { label: 'Ad revenue',  data: wAd,  backgroundColor: 'rgba(245,166,35,0.85)', borderRadius: 3, borderSkipped: false }
    ]
  },
  options: {
    ...base,
    scales: { ...base.scales, y: { ...base.scales.y, ticks: { color: tc, font: { size: 10 }, callback: v => fmtR(v) } } },
    plugins: { ...base.plugins, tooltip: { mode: 'index', intersect: false, callbacks: { label: ctx => ctx.dataset.label + ': ' + fmtR(ctx.raw) } } }
  }
});

// ── Chart 3: DAU + 7-day MA ──────────────────────────────────
const dauD = dates.map(d => engData[d]?.d || null);
const ma7  = dauD.map((_, i) => {
  if (i < 6) return null;
  const sl = dauD.slice(i-6, i+1).filter(v => v !== null);
  return sl.length ? Math.round(sl.reduce((a,b) => a+b, 0) / sl.length) : null;
});
new Chart(document.getElementById('ch-dau'), {
  type: 'line',
  data: {
    labels: dates.map(d => d.slice(5)),
    datasets: [
      { label: 'DAU',       data: dauD, borderColor: 'rgba(74,144,217,0.5)',  backgroundColor: 'rgba(74,144,217,0.08)', borderWidth: 1.5, pointRadius: 0, fill: true,  tension: 0.3 },
      { label: '7-day avg', data: ma7,  borderColor: '#E8593C',               backgroundColor: 'transparent',           borderWidth: 2,   pointRadius: 0, fill: false, tension: 0.4 }
    ]
  },
  options: { ...base }
});
</script>
```

---

## Notes for skill contributors

- Adjust MCP tool returns CSV — always parse headers from the first row
- Revenue field naming may vary (`revenue` vs `all_revenue`) — handle both
- Some days may be missing from the API response — treat missing days as 0
- The `Untrusted Devices` network is Adjust's internal fraud flag — always
  separate it from organic and paid in any network breakdown
- If the account has multiple apps, Query A (installs by network) will return
  rows for the queried app only if the app name is specified correctly
- Test all queries manually before locking in trigger phrases
