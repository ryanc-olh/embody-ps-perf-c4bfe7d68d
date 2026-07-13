# BUILD.md — how to rebuild the Embody PS Performance dashboard

This folder IS the source of truth for the live page. `index.html` is a single self-contained file
(Chart.js via CDN). To refresh it, an agent re-runs the queries below, rewrites the numbers/arrays in
`index.html`, commits, and pushes (GitHub Pages auto-rebuilds).

- **Live URL:** https://ryanc-olh.github.io/embody-ps-perf-c4bfe7d68d/
- **Repo:** ryanc-olh/embody-ps-perf-c4bfe7d68d (public; `noindex` meta + robots.txt keep it out of search)
- **Client:** Embody (aggregates the Amazon Connect orgs `Embody` + `White Coat - Embody`). Client-facing:
  no PHI, no internal plumbing, no per-patient detail. Aggregates only.
- **Tooling:** queries run against the **Zoho Analytics MCP** (`query_data`). Needs that MCP connected.

---

## Data sources (both are Zoho Analytics BATCH MIRRORS — lag a few hours, NOT live)
| Section | Workspace ID | Table | Origin |
|---|---|---|---|
| Voice + Chat | `2713211000000665001` | `Amazon Connect Logs Adj` | Amazon Connect (AWS) |
| Email | `2713211000000128002` (org_id `807541001`) | `Tickets` | Zoho Desk |

Freshness: voice mirror typically lags MORE than email (seen: voice ~6:46a CT while email ~noon ET same day).
Check `MAX(...timestamp...)` before trusting "today." Voice `initiation_timestamp_cst` = **Central**; email
`Created Time`/`Ticket Closed Time` = **Eastern**. The live email path (not used here) = Zoho Desk API; live
voice would need Amazon Connect direct (AWS IAM, not yet provisioned).

---

## Queries (fill the date window; weekly window = Mon 00:00 → next Mon 00:00, Central for voice / Eastern for email)

### 1. Voice by day (feeds Voice cards, Voice table, chart c1 volume + c2 SLA)
```sql
SELECT DATE("initiation_timestamp_cst") AS day,
  COUNT(*) AS offered,
  COUNT(CASE WHEN "agent_connected_to_agent_timestamp" IS NOT NULL THEN 1 END) AS answered,
  COUNT(CASE WHEN "agent_connected_to_agent_timestamp" IS NOT NULL AND "queue_duration_ms"<=30000 THEN 1 END) AS le30,
  ROUND(AVG(CASE WHEN "agent_connected_to_agent_timestamp" IS NOT NULL THEN "queue_duration_ms" END)/1000.0,0) AS asa_sec
FROM "Amazon Connect Logs Adj"
WHERE "channel"='VOICE' AND "initiation_method"='INBOUND' AND "queue_enqueue_timestamp" IS NOT NULL
  AND "Org Name" IN ('Embody','White Coat - Embody')
  AND "initiation_timestamp_cst" >= '2026-07-06 00:00:00' AND "initiation_timestamp_cst" < '2026-07-13 00:00:00'
GROUP BY DATE("initiation_timestamp_cst") ORDER BY day
```
Derive: **SL% = le30/offered**, **abandon% = (offered−answered)/offered**, **ASA = asa_sec**.
Week ASA = answered-weighted avg of daily asa_sec (not a simple mean).

### 2. Chat by day (Chat cards + table)
```sql
SELECT DATE("initiation_timestamp_cst") AS day, COUNT(*) AS offered,
  COUNT(CASE WHEN "agent_connected_to_agent_timestamp" IS NOT NULL AND "queue_duration_ms"<=30000 THEN 1 END) AS le30,
  ROUND(AVG(CASE WHEN "agent_connected_to_agent_timestamp" IS NOT NULL THEN "queue_duration_ms" END)/1000.0,0) AS asa_sec
FROM "Amazon Connect Logs Adj"
WHERE "channel"='CHAT' AND "queue_enqueue_timestamp" IS NOT NULL
  AND "Org Name" IN ('Embody','White Coat - Embody')
  AND "initiation_timestamp_cst" >= '<start>' AND "initiation_timestamp_cst" < '<end>'
GROUP BY DATE("initiation_timestamp_cst") ORDER BY day
```

### 3. Email received / resolved by day + backlog (Email cards + table + Net)
Received: `DATE(t."Created Time")`. Resolved: `DATE(t."Ticket Closed Time")`. Backlog: `GROUP BY t."Status"`.
**Net = received − resolved** (green when negative = burning down). Use the SAME filter block for all three:
```sql
SELECT DATE(t."Created Time") AS day, COUNT(DISTINCT t."ID") AS received
FROM "Tickets" t
WHERE t."Service Type" = 'Inbound Email'
  AND t."To Address" IN ('embody@openloophealthpartners.zohodesk.com','whitecoatembody@openloophealthpartners.zohodesk.com','support@joinem.co')
  AND (t.Subject IS NULL OR t.Subject != 'New Secure Message In Tellescope')
  AND (t.Email IS NULL OR t.Email NOT IN ('support@tellescope.com','notifications@tellescope.com','follow-suggestions@mail.instagram.com','truistsecuredoc@esign.truist.com','notification@service.tiktok.com'))
  AND (t.Language IS NULL OR t.Language != 'Spanish')
  AND (t."Service Department" IS NULL OR t."Service Department" != 'Patient Services Clinical Management')
  AND t.Channel = 'Email'
  AND t."Created Time" >= '<start>' AND t."Created Time" < '<end>'
GROUP BY DATE(t."Created Time") ORDER BY day
```
⚠️ **Do NOT scope email by `cf_client`** — it's tagged on ~2% of tickets (≈5.5× undercount). The inbox
`To Address` filter above is the correct population. Canonical definition = the `embody-queue-audit` /
`embody-queue-performance` skills ("Embody Support Emails + OSS" view). The `Email NOT IN (...)` clause strips
Tellescope/automation noise — without it "Open" balloons (~700 vs ~244 real).
First-response ≤4hr/≤24hr (44%/96%) come from the skills' reply-timing join; carried forward, not recomputed
each refresh (recent-day tickets lack 24h maturity).

### 4. Intraday hourly (feeds the day-picker charts cv/ce). `query_data` truncates at ~20 rows, so PIVOT to
one row/day with 24 hour-columns (repeat the CASE for h0..h23):
```sql
SELECT
  SUM(CASE WHEN HOUR("initiation_timestamp_cst")=0 THEN 1 ELSE 0 END) AS h0,
  ... (h1 ... h23) ...
FROM "Amazon Connect Logs Adj"
WHERE "channel"='VOICE' AND "initiation_method"='INBOUND' AND "queue_enqueue_timestamp" IS NOT NULL
  AND "Org Name" IN ('Embody','White Coat - Embody')
  AND "initiation_timestamp_cst" >= '<day 00:00>' AND "initiation_timestamp_cst" < '<next day 00:00>'
```
Email hourly = same pivot on `Tickets` with `HOUR(t."Created Time")` (received) / `HOUR(t."Ticket Closed Time")`
(resolved) + the email filter block. Always validate: the 24 columns should sum to the day's total.
Arrays go into the `VOFF` / `EREC` / `ECLO` JS objects in index.html, keyed by 'M/D'.

### 5. "Today so far" strip + freshness
Today's partial counts: rerun #1–#3 for the single current day; also `MAX(...timestamp...)` per source to
label each tile's "as of" time (voice and email differ — label them separately).

---

## Format / convention rules (match the current index.html)
- **Window = complete Mon–Sun week** in the tables/totals. Today (partial) is NOT in the week totals —
  it lives in the top **"Today so far" strip** (amber header, per-tile "as of" times) and as an **amber,
  asterisked partial point** on the by-day charts (see `days7`/`wkBlue` arrays + the caption note).
- Color thresholds: voice SL green ≥80 / amber 50–79 / red <50; ASA green ≤30s / red high; abandon green ≤5 /
  red high. Email Net: green when negative (burning down). These are hand-set per cell in the tables.
- Client hygiene: no inbox domains, no total-queue counts ("of N"), no internal org names ("White Coat" →
  show "Embody"), no "pending data source" caveats. Backlog card = "Currently open" (or per-tile as-of).
- `<meta name="robots" content="noindex, nofollow, noarchive">` must stay in `<head>`; keep `robots.txt`.

## Deploy
```bash
git add -A && git commit -m "refresh" && git push   # Pages rebuilds in ~1 min
```
Deeper context in ps-brain: `knowledge/reference/templates/ps-client-dashboard-template.md` (build spec),
`knowledge/reference/metrics/2026-07-10-embody-weekly-performance.md` (methodology + findings),
`knowledge/tool-notes.md` (Zoho quirks, pivot trick, timezones).
