# Embody PS Performance dashboard — deploy notes

**Live URL:** https://ryanc-olh.github.io/embody-ps-perf-c4bfe7d68d/
**Source of truth:** this repo. `index.html` is the whole dashboard (self-contained, Chart.js via CDN).
`noindex` meta + `robots.txt` keep it out of search results (link-shareable, no login).

## Updating (when Ryan says "update Embody")
1. Claude re-pulls the data from Zoho and rebuilds `index.html` here, then commits locally.
2. Ryan runs the push (the only step Claude can't do — safety block on pushing client data to public GitHub):
   ```bash
   cd ~/embody-ps-dash && git add -A && git commit -m "refresh" && git push
   ```
   GitHub Pages rebuilds in ~1 min. Same URL, fresh data.

## Data sources (all via Zoho Analytics, batch mirrors, lag a few hours)
- Voice + Chat → "Amazon Connect Logs Adj" (ws 2713211000000665001), org filter `Org Name IN ('Embody','White Coat - Embody')`, `channel`, `initiation_method='INBOUND'`.
- Email → Zoho Desk "Tickets" (ws 2713211000000128002), inbox `To Address` filter, minus Tellescope/Spanish/Clinical noise.
