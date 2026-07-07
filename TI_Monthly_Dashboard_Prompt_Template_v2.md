# Monthly Lead Gen Dashboard — Reusable Prompt Template v2 (TI)

Copy everything below into a new chat each month, replacing `[MONTH]` / `[YEAR]` and attaching that month's **YTD (year-to-date cumulative) exports**. This regenerates the TI Lead Generation dashboard — including MoM comparisons and the Deeper Insights sections — scoped to whichever month you name.

**What changed from v1:** all data files are now YTD-cumulative (one export covering Jan 1 → end of target month), which enables (a) automatic MoM comparisons in every section, (b) correct "new vs returning" lead detection, (c) full-history SQL scoring, and (d) the four Deeper Insights sections. You no longer provide a prior-month anything — the prior month is derived from the same YTD files.

---

## PROMPT TO PASTE

I need a monthly lead generation dashboard for **[MONTH] [YEAR]**, following the exact structure, methodology, and design described below. Build it as a single self-contained HTML file with Chart.js. All attached data files are **YTD-cumulative (Jan 1 [YEAR] → end of [MONTH])**; scope the dashboard to [MONTH] and derive the prior month from the same files for MoM.

### Files I'm attaching (all YTD-cumulative)

1. **Three form-fill exports (csv)** — exhaustive website form-fill exports, one file each. Actual column names in these files (they differ from the Pardot files below):
   - email → `email`
   - created date → `created` — **this is a Unix timestamp (seconds), not a date string.** Convert it (epoch → datetime) before month-scoping.
   - company → `account_name` (there is **no** `Company` column in the form files)
   - first / last name → `first_name` / `last_name`
   - UTM/landing fields → `first_touch_url`, `ga_landing_c`
   - also present: `ad_web`, `lead_source`

   The files:
   - `generic_request_demo.csv` — **contact form** (demo request)
   - `dynamic_request_demo_form.csv` — **contact form** (demo request)
   - `general_leadership_webform.csv` — **gated resource download** (leadership webform)

   Notes: the three files don't have identical column sets (the leadership form carries extras like `download_type`, `pardot_asset_link`) but share the core columns. Filenames may carry download suffixes like `__2_` — ignore those. Some Pardot files contain embedded newlines inside quoted fields, so always use a proper CSV parser (never count rows with `wc -l`).

   Role reminder: the two demo-request forms are the "contact forms" (the SQL auto-qualify trigger); the leadership webform is the gated resource.

2. **`EN_TI_Paid_Social_Leads.csv`** — Pardot export of native LinkedIn Lead Gen Form leads. Pardot column names: `Email`, `Created Date` (normal datetime string, not epoch), `Company`, `First Name`, `Last Name`, `Score`, `Grade`, `Lead Status`, `Source`, plus `utm_source`/`utm_medium` and metadata fields. These are NOT in the form files — separate source. Count as leads, always classified **LinkedIn (Paid Social)**.

3. **`EN_TI_All_Contact_Form_Leads.csv`** and **`EN_TI_All_Leadership_Form_Leads.csv`** — Pardot exports (same column style) of the same demo/leadership form leads already in the form files. Use ONLY to look up each email's current Pardot `Score`. Do NOT count these as additional leads.

   **Max-score lookup:** take the max Pardot Score across all three Pardot files if an email appears more than once. These two files just need to *cover every email* appearing in the form files — their own date range doesn't matter.

4. **Confirmed Google Ads leads (YTD)** — leads manually confirmed as coming from Google Ads. May arrive as a full form-fill export (with `Work Email` and `Company` columns) — extract the emails. Used for the Google Ads classification override (see Step 2, including the conflicting-UTM refinement).

5. **Service line breakdown (YTD screenshot/table)** — pasted table + screenshot of the service-line tracker showing **all months Jan → [MONTH]** as columns. Two breakdowns: a Sales side (the two demo-request forms) and a Resource side (leadership webform). Transcribe the **target month AND the prior month** columns (prior month is needed for the MoM chips). Always echo the transcribed values back to me for confirmation — screenshot reads can err.

**Google Ads confirmed leads (YTD):** `[PASTE EMAIL LIST OR ATTACH EXPORT]`

**Service line breakdown (YTD):** `[PASTE TABLE + ATTACH SCREENSHOT(S)]`

---

### Step 0 — Recon before processing

- Parse every file with a real CSV reader and report row counts and month-by-month date coverage per file. If the YTD files look truncated (e.g., a month with implausibly few rows), flag it before proceeding.
- Confirm the target month exists in the data and state the detected year.

### Step 1 — Exclude spam/test/internal entries

Before any counting, drop rows matching:

- **Company** contains (case-insensitive). Match against `account_name` in the form files and `Company` in the Pardot/Google Ads files:
  - `Trial Interactive`, `TransPerfect` — substring match
  - `moving company` — substring match
  - `TI` and `TP` — **word-boundary match only** (standalone words; don't match "Continental"/"Optum").
- **Work email** on the `@transperfect` or `@tp` domain (prefix match on `@tp`).
- **The word `test`** (case-insensitive, word-boundary) in **name, email, or company field only** — NOT a whole-row scan (Pardot metadata like `Lists` contains "…TEST HK" strings that false-positive).
- **First + Last name combo** (case-insensitive substring): Johann George, Diana Andrea, Emily Hunsbusher, Jake Angevine, Isaiah Howard, Duran Goodyear, Rob Brodie, Ryan Draving. Match against name fields AND the email local-part.

Apply to all four lead sources before anything else. Report kept/dropped per file with reason counts. (For reference: the leadership form typically has a very high exclusion rate — internal staff and "Test1992" rows — that's expected, not an error.)

### Step 2 — Classify each lead's channel

Priority order:

1. Email in the confirmed Google Ads list → **Google Ads (Paid Search)** — **UNLESS** the lead's own first-touch UTM explicitly resolves to a different named channel (e.g. `pardot/email`), in which case that named channel wins. A lead with *no* UTM still gets the Google Ads override. (This "conflicting-UTM refinement" prevents a stale Google Ads list entry from overriding fresh contradictory UTM evidence.)
2. Otherwise parse utm_source/utm_medium from `first_touch_url`, falling back to `ga_landing_c`:
   - google + cpc → **Google Ads (Paid Search)**
   - google + display → **Google Display (Paid)**
   - linkedin + organic-social → **LinkedIn (Organic Social)**
   - pardot or act-on + email → **Email Nurture (Pardot/Act-On)**
   - chatgpt.com in URL → **AI Referral (ChatGPT)**
3. gclid present with no UTM → **Google Ads (Paid Search)**
4. No UTM at all → **Direct / Organic (No UTM)**
5. Every row from the Paid Social CSV → **LinkedIn (Paid Social)** — always its own channel.

All channel classification and trend charts are computed from the raw lead data. There is no separate channel spreadsheet.

### Step 3 — Funnel / SQL scoring

- Merge each email's max Pardot Score across the three Pardot files.
- **SQL** if EITHER: ever filled a demo-request form (auto 100 pts — "SQL – Contact Form Fill"), OR max score ≥ 100 without ever filling one ("SQL – Score Qualified"). Contact-form fill takes priority if both.
- Otherwise **MQL**.

### Step 4 — Scope to the target month AND derive the prior month

- **Unique new leads** = unique people whose first-ever touch (earliest Created date across all sources) falls in [MONTH]. Primary cohort for funnel/channel sections.
- **Total submissions** = all rows dated in [MONTH] (includes repeat visitors; report separately).
- **Prior-month equivalents:** compute the same stats (new leads, submissions, MQL/SQL split, per-form uniques, channel distribution, non-demo-origin counts, share of leads scoring ≥100) for the prior month from the same YTD data. These power the MoM chips.
- **Trailing 6-month view** for trend charts (or as much history as the YTD window allows — e.g., a March dashboard has only a 3-month trend; label accordingly).

### Step 5 — Service line breakdown

- Transcribe the target month and prior month columns from the pasted YTD table + screenshot, both Sales and Resource sides. Echo values back for confirmation and flag them as screenshot-transcribed in the dashboard's MoM note.
- Service lines: TI General, eClinical / multiple products, TMF, eTMF, CTMS, SSU, eISF, eArchive, LMS, AI / ML, Summits, Other.
- **Summits caveat:** separate event/registration line outside the demo-form funnel — flag, don't blend. Sanity check: Sales side excl. Summits vs. the month's combined demo-contact uniques; a small gap (repeat/overlap submitters) is fine, a large one isn't.

### Step 6 — "Which engagement types lead to a contact form"

For [MONTH]-cohort leads whose first touch was a gated resource or LinkedIn (Paid Social) ad lead: how many, at ANY point in the dataset, later filled a demo-request form? Report "X of Y" per origin. Also check nurture-touched leads. Include the prior-month origin counts for the MoM note.

### Steps 7–10 — Deeper Insights (computed from the full YTD data)

- **Lead Quality Trend:** per-month cohorts (new leads, SQL count, SQL rate %) across the YTD window — combo chart, bars + rate line.
- **Channel Quality Scorecard:** all YTD leads by first-touch channel — leads, SQL, SQL rate, avg max score. Table + callout comparing paid channels' SQL rates (historically Google Ads converts far better than LinkedIn Paid Social — call out the current gap and what it implies for budget).
- **Lifetime engagement→demo conversion:** across ALL YTD cohorts, how many resource-origin and LinkedIn-ad-origin leads ever filled a demo form, with the converts listed (email, first-touch date, demo date, lag days).
- **Warm Pipeline:** MQLs scoring 50–99 with no form fill, total count + top 10 by score (email, score, first-touch date, channel). Note: contains lead emails — flag if the dashboard may be shared beyond marketing/sales.

Skip analyses with no signal rather than padding (e.g., account clustering with <5 multi-lead companies, or free-email analysis when nearly all leads are corporate).

---

### Dashboard sections, in this exact order

1. **Header** — gradient dark-blue banner. Title "TI Lead Generation Dashboard — [Month] [Year]", pill badges: new unique leads + total submissions, and a MoM badge (% change in total submissions vs. prior month, pink "down" style when negative; "n/a" only if no prior-month data exists).

2. **KPI row (5 cards, colored left edges):** New Unique Leads (+ total submissions sub-label), MQL, SQL Total, SQL via Contact Form, SQL via Engagement Score — **each with a MoM delta chip** (▲/▼ n vs [prior month]; green = good direction, pink = bad; down is good for nothing on this row).

3. **[Month] Form Fill Breakdown** — table with columns: Form Type | [Prior Month] | [Month] | MoM chip. Combine the two demo forms into one "Contact Form" row (check cross-form overlap first; note it). Leadership webform and LinkedIn Ad Lead Gen Form as own rows. TOTAL row. Note below: generic/dynamic split, repeat submitters by email, overlap, and any [Month] submitters who first touched in a prior month.

4. **[Month] Form Fills by Service Line** — two horizontal bar charts (Sales teal, Resource magenta), only non-zero lines, each card's desc carrying the month total + MoM chip vs the prior month's transcribed total. Amber caveat box: Summits caveat + sanity check + a note that prior-month values are screenshot-transcribed, listing them for verification.

5. **[Month] Funnel: New Leads → MQL → SQL** — hand-built proportional bars (never a chart library) + side panel with the two SQL paths and a "vs [prior month] cohort" MoM line (New/MQL/SQL chips).

6. **Which Engagement Types Lead to a Contact Form? ([Month] cohort)** — big stat (X of Y) + three origin rows + MoM sentence in the subtitle (this month's non-demo-origin count vs prior month's).

7. **Channel Performance — [Month]** — 2×2 grid: donut (62% cutout) of new-lead share by channel; Total / Google Ads / LinkedIn (Paid Social) trend bars (prior months gray, current month magenta). Under the section subtitle, a row of per-channel MoM chips (channel: n ▲/▼ Δ (prior p)).

8. **Which Channels Drove [Month]'s SQLs?** — horizontal stacked bar (MQL / SQL–Contact Form / SQL–Score Qualified) by first-touch channel, largest total at top. Subtitle carries SQL MoM chips. Callout: flag any channel converting to SQL almost entirely via score with no form fills (possible manual score adjustment).

9. **Which Channels Drove [Month]'s MQLs?** — single-series horizontal bar, colors matching the donut. Subtitle carries the MQL MoM chip.

10. **Engagement Score Distribution — [Month]** — column chart, bins: No Score, 25-49, 50-74, 75-99, 100-149, 150-199, 200-299, 300-499, 500+. Blue below 100, magenta at/above, gray for No Score. Subtitle: share of cohort ≥100 this month vs prior month.

11–14. **Deeper Insights — [YTD window]** banner, then: Lead Quality Trend combo chart; Channel Quality Scorecard table + budget callout; Lifetime engagement→demo conversion (progress bars + converts list + lag commentary); Warm Pipeline table (no bold styling on its last row — it's not a totals row).

---

### Design requirements

- Single HTML file, inline CSS and JS, no build step.
- Chart.js from `https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js` — exact version string (verify the version exists — e.g. via the npm registry — before finalizing; nonexistent CDN paths fail silently and blank every chart).
- Visual language: page background `#f4f6fa`, white cards with `#e2e8f0` borders and 12–14px radii, gradient header (`#003b71` → `#0a5a9c`), KPI cards with 4px colored left edges, rounded chart bars, tabular-numeric tables with hover states, pill badges, small MoM delta chips (green `#e9f7ef`/`#1e7a45` good, pink `#fdeef4`/`#b02a63` bad, gray flat).
- Brand colors: Dark Blue `#003b71`, Light Blue `#139dd8`, Light Blue 2 `#4199d8`, Teal `#3bbfb5`, Lime `#99cc33`, Magenta `#ec388a`, Cream `#e7e3da`, light blue-gray `#e0e8f5`. Arial throughout.
- Channel color map (donut + Section 9 bars must match): Direct/Organic dark blue, Google Ads light blue, Google Display `#5a7fa8`, LinkedIn Paid Social magenta, LinkedIn Organic light blue 2, Email Nurture teal, AI Referral lime.
- No data labels baked onto bars/slices (no chartjs-plugin-datalabels) — hover tooltips only.
- KPI cards and funnel bars are hand-built divs so exact numbers are always visible.
- Embed all data as one JSON object (`const D = {...}`) — no hardcoded numbers in JS logic, and no hardcoded narrative numbers in HTML that can go stale (MoM badge and notes must be computed/derived, not typed).

### Validation before delivering (all required)

1. Embedded JSON parses (extract `const D = {...}` and `json.loads` it).
2. Extracted `<script>` block passes `node --check`.
3. Cross-check every rendered number (KPI cards, header badges, engagement headline, MoM chips) against the computed data — no stale text.
4. Internal consistency: MQL + SQL = cohort size; form-table Contact = generic + dynamic − overlap; score bins sum to cohort; engagement Y = non-demo-origin count.
5. Confirm the only external reference is the Chart.js CDN.
6. When editing generated code with find-and-replace, verify the anchor string is unique — a too-short anchor can splice into the wrong chart block and truncate it (this has happened; node --check catches it, so never skip it).

### Delivery

Present the final HTML file, then summarize: headline numbers with MoM, what changed vs prior month, any data-quality flags (UTM-poor classification, truncated history, screenshot-transcription items awaiting confirmation), and the single most actionable insight of the month.
