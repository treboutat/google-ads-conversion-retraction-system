# Conversion Retraction Setup

Walk through setting up offline conversion retraction for disqualified leads in Google Ads. This prevents bad-quality conversions (spam, auto-DQs, ICP mismatches) from being fed back to the ad platform as positive signals — which would otherwise train the algorithm to find more of the wrong people.

## Arguments

$ARGUMENTS - Optional: client name, step to resume from

Examples:
- `/conversion-retraction` - starts from Step 1
- `/conversion-retraction adam` - starts setup for Adam
- `/conversion-retraction adam step-4` - resumes at SQL query step

## Why This Matters

Google Ads optimizes toward whatever you tell it is a conversion. If a spam lead submits a form and that form submission is tracked as an MQL, Google now thinks "this is the kind of person I should find more of." Retraction tells Google: "that conversion I reported? Ignore it. That person was junk." Over time, this materially improves lead quality because the algorithm stops optimizing toward disqualified profiles.

Without retraction, you're essentially paying Google to find you more spam.

---

## Workflow

### Step 1: Identify Closed-Loss Reasons to Retract

Open the client's CRM (typically HubSpot) and review the closed-loss reason field. This field may live on **deals** or **contacts** depending on how the client has configured their reporting pipeline — confirm which object they use before pulling data.

**Pull the full list of closed-loss reason values.** Then classify each one:

**Retract these (genuinely bad conversions):**
- `AutoDQ` — automatically disqualified by lead scoring or routing rules
- `Not in private practice` — or whatever the client's primary ICP disqualifier is (e.g., "Not a decision-maker", "Wrong company size", "Wrong industry")
- `No longer in private practice` — was once ICP-qualified but no longer is
- `DIY DQ` / `Self-service DQ` — lead self-identified as not a fit
- `Spam` — bot submissions, fake info, test entries
- `Unsubscribed` — opted out before any meaningful engagement
- `Duplicate` — same person submitted multiple times
- `Invalid contact info` — fake email, disconnected phone, unreachable

**Do NOT retract these:**
- `Competitor` — this person was a real, qualified prospect who chose a competitor. That's a viable SQL. The platform should absolutely keep finding people like this.
- `Lost - Timing` / `Not ready` — qualified but wrong timing. Still a good lead profile.
- `Lost - Price` / `Budget` — qualified but couldn't afford it. Still directionally useful signal.
- `Nurture` / `Long-term pipeline` — not closed-won yet, but not a bad lead.

**The distinction is intent vs. quality.** Retract leads that were never real prospects (quality problem). Keep leads that were real prospects but didn't convert (intent/timing problem). Google can't control whether someone buys — but it can control whether the person who clicks is actually in your ICP.

**Edge cases to discuss with the client:**
- `No-show` — depends. If the lead booked a call and ghosted, they were at least somewhat qualified. If it's a pattern of bot bookings, retract.
- `Wrong service` — depends on how different the services are. If someone wanted Service A but you only offer Service B, and the audiences are materially different, retract.
- `Unresponsive` — similar to no-show. If they filled out a form with real info but never replied, they were probably real. Generally do NOT retract unless the client sees a pattern of fake submissions.

**Action:** Confirm the final retraction list with the client or account lead before proceeding. Document the agreed-upon list somewhere accessible (Slack thread, shared doc, or in the client's config). This list will be referenced when building the SQL query.

---

### Step 2: Determine Which Conversion Actions to Retract

Not every conversion action should be retracted. Review the client's Google Ads conversion actions and classify each:

**Typically retract:**
- **MQL (Marketing Qualified Lead)** — this is the most common retraction target. If someone became an MQL but was later disqualified, retract it.
- **SQL (Sales Qualified Lead)** — if the client tracks SQL as a separate conversion action and the lead was DQ'd after qualifying, retract it.
- **Booked Call / Demo Request** — if tracked as a conversion, retract for DQ'd leads.

**Typically do NOT retract:**
- **New Customer / Closed-Won / Revenue** — if someone actually became a customer and later churned or was problematic, that's a different issue. The original conversion was real. Do not retract genuine customers.
- **Micro-conversions** (page views, video plays, content downloads) — these aren't tied to lead quality in the same way. Retracting a PDF download because the person later turned out to be spam adds noise, not signal.

**Important nuance:** Each conversion action you want to retract requires its own separate retraction pipeline (its own query, its own Sheet, its own upload schedule). So be deliberate — don't retract 5 different conversion actions if only MQL and SQL matter. More pipelines = more maintenance.

**Action:** Confirm with the client which conversion actions to retract. Write down the exact conversion action names as they appear in Google Ads — spelling and capitalization must match exactly, or the upload will silently fail.

To find the exact names:
1. Google Ads → Goals → Conversions → Summary
2. Note the exact "Conversion action" name for each one you're retracting
3. Copy-paste — don't retype

---

### Step 3: Set Up the Data Source in the Data Warehouse

Go to your data warehouse (e.g., Funnel.io, Supermetrics, or a custom ETL pipeline) and locate or create the **Offline Conversion Import** data source that connects your CRM to Google Sheets.

**Required fields in the data source:**

| Field | Description | Notes |
|-------|-------------|-------|
| **Google Click ID (GCLID)** | The click identifier from the original ad click | Must be present on the CRM record. If the client isn't capturing GCLIDs, stop here — you need to fix tracking first. |
| **Conversion Name** | The exact name of the conversion action in Google Ads | If available as a dynamic field, map it. If not, you'll set it as a static value per query (see Step 4). |
| **Conversion Time** | When the original conversion happened | Format: `yyyy-MM-dd HH:mm:ss` or ISO 8601. Must be the time of the *original* conversion, not the retraction time. |
| **Adjustment Type** | The type of adjustment | Will always be `RETRACT` for this use case. Can also be `RESTATE` if adjusting value rather than removing. |

**If GCLID is missing from CRM records:**
This is a blocker. The retraction file requires GCLIDs to match conversions back to specific ad clicks. If the client isn't capturing GCLIDs:
1. Check if they have a hidden field on their forms capturing `gclid` from the URL parameter
2. Check if their CRM has a "Google Click ID" or "GCLID" property (HubSpot has this natively if Google Ads integration is connected)
3. If neither exists, you need to set up GCLID capture before you can do retraction. This is a separate workstream.

**If Conversion Name is not available as a dynamic field:**
This is common. Most data warehouses don't have "Conversion Name" as a mappable CRM field because it's a Google Ads concept, not a CRM concept. In this case, you'll hardcode it as a static value in the query (Step 4). This means you need one query per conversion action.

---

### Step 4: Write the SQL Query

Build a query that pulls the records to retract. The query runs against your data warehouse's offline conversion import table.

**Query structure:**

```sql
SELECT
    gclid                           AS "Google Click ID",
    '<CONVERSION_NAME>'             AS "Conversion Name",      -- Static value, e.g., 'MQL'
    conversion_time                 AS "Conversion Time",       -- Original conversion timestamp
    'RETRACT'                       AS "Adjustment Type"
FROM
    offline_conversion_import       -- Or whatever your table is named
WHERE
    conversion_mql > 0              -- Only rows where this conversion fired
    AND closed_loss_reason IN (
        'AutoDQ',
        'Not in private practice',
        'No longer in private practice',
        'DIY DQ',
        'Spam',
        'Unsubscribed'
        -- Use the exact list from Step 1
    )
```

**Critical details:**

1. **`<CONVERSION_NAME>` must exactly match Google Ads.** If your Google Ads conversion is called "MQL - Form Submit", the static value must be `'MQL - Form Submit'` — not `'MQL'`, not `'mql - form submit'`. Case-sensitive. Whitespace-sensitive.

2. **The `WHERE` clause on conversion count** (e.g., `conversion_mql > 0`) ensures you only retract records where that specific conversion actually fired. Without this filter, you'd try to retract conversions that never existed, which Google will reject.

3. **The closed-loss reasons must use the exact CRM values.** If HubSpot stores it as `"Auto DQ"` (with a space) and your query says `'AutoDQ'` (no space), you'll get zero results. Pull the actual values from the CRM first.

4. **Date filtering is optional but recommended.** Google Ads can only process retractions within the last 55 days (the documented limit is 60, but build in a buffer). Adding `AND conversion_time >= DATEADD(day, -55, GETDATE())` reduces the payload and avoids upload errors on old records.

**Validation before proceeding:**
- Run the query and check the row count. Sanity-check it:
  - If you get 0 rows: something is wrong with your filters. Check field names and values.
  - If you get thousands of rows on a small account: your filters might be too broad. Spot-check a few records.
  - Compare the count to total leads in that period. If you're retracting more than ~40-50% of all conversions, double-check with the client that this is expected.

---

### Step 5: Create the Google Sheet

**Do not start from scratch.** Copy an existing retraction sheet from another account to use as a template. This ensures the column headers and formatting are exactly what Google Ads expects.

1. Find an existing retraction sheet from another client's offline uploads folder
2. Make a copy: `File → Make a copy`
3. Save the copy in the client's offline uploads folder (Google Drive), named clearly: `[Client Name] - MQL Retraction Upload`
4. Update the sheet's data connection to point to the new SQL query from Step 4:
   - Open the sheet's data connector (usually under `Data → Data connectors` or similar, depending on your warehouse integration)
   - Replace the old query with your new query
   - Test the connection — click "Refresh" or "Preview" and confirm data appears
5. Verify the columns match what Google Ads expects:
   - Column A: `Google Click ID`
   - Column B: `Conversion Name`
   - Column C: `Conversion Time`
   - Column D: `Adjustment Type`
   - **No extra columns.** No header formatting. No merged cells. Google Ads parses this file rigidly.

**Common mistakes:**
- Extra whitespace in column headers (e.g., `"Google Click ID "` with a trailing space) — will cause upload failure
- Date format mismatch — Google wants `yyyy-MM-dd HH:mm:ss` or similar. If your warehouse outputs `MM/dd/yyyy`, you'll need to reformat in the Sheet.
- Sheet has multiple tabs — Google Ads reads the first tab only. Make sure data is on Sheet1.

---

### Step 6: Schedule the Sheet Refresh

The Google Sheet needs to pull fresh data from your warehouse automatically so new DQ'd leads are included without manual intervention.

1. In the Google Sheet, go to the data connector settings
2. Set **automatic refresh** to every **24 hours**
3. Choose a refresh time that's at least 2-3 hours *before* the Google Ads upload schedule (Step 7). This ensures the sheet has fresh data when Google pulls it.
   - Example: If the Google Ads upload runs at 8 AM, schedule the sheet refresh for 5 AM.
4. Verify the first automatic refresh runs successfully — check the sheet the next day.

**If your data warehouse doesn't support scheduled refresh natively:**
- Some connectors require a paid tier for scheduled refresh (e.g., Supermetrics)
- Alternatively, set up a Google Apps Script trigger that calls `SpreadsheetApp.flush()` or your connector's refresh API on a timer
- As a last resort, set a calendar reminder to manually refresh weekly (not ideal, but better than never)

---

### Step 7: Set Up the Upload Schedule in Google Ads

1. In Google Ads, navigate to: **Tools & Settings → Measurement → Conversions**
2. Click **Uploads** in the left sidebar
3. Click **Schedules** tab
4. Click the **+** button to create a new schedule
5. Select **Google Sheets** as the source
6. Authenticate and select the Google Sheet you created in Step 5
7. Set frequency to **Daily**
8. Set the time to run *after* the Sheet refresh completes (see Step 6 timing note)
9. Save the schedule

**After saving, manually trigger the first upload** to verify everything works:
- Go to Uploads → click "Upload" → select the same Sheet → Upload
- Check the results:
  - **"Successfully processed"** = working correctly
  - **"Partially processed"** = some rows failed. Download the error report and check why. Common issues:
    - GCLID not found (click happened too long ago, or GCLID is from a different Google Ads account)
    - Conversion name doesn't match any conversion action
    - Conversion time is outside the 60-day window
    - Duplicate retraction (already retracted this conversion)
  - **"Failed"** = structural issue with the file (wrong columns, wrong format, etc.)

**The 60-day window:**
Google Ads can only process retractions for conversions that happened within the last 60 days. Older records will be rejected. This is expected and not an error — it just means you can't retroactively clean up historical data beyond ~2 months. The ongoing daily schedule handles future DQ'd leads automatically.

---

### Step 8: Repeat for Each Conversion Action

Each conversion action requires its own complete pipeline:

| Conversion Action | Query | Sheet | Upload Schedule |
|-------------------|-------|-------|-----------------|
| MQL | Filters for `conversion_mql > 0` | `[Client] - MQL Retraction` | Daily, 8 AM |
| SQL | Filters for `conversion_sql > 0` | `[Client] - SQL Retraction` | Daily, 8 AM |

For each additional conversion action:
1. Duplicate the Google Sheet from Step 5 (don't create from scratch)
2. Update the `<CONVERSION_NAME>` static value in the query to match the new conversion action
3. Update the `WHERE` clause to filter for that specific conversion column (e.g., `conversion_sql > 0` instead of `conversion_mql > 0`)
4. Create a new upload schedule in Google Ads pointing to the new sheet
5. Test with a manual upload

**Naming convention for sheets:** `[Client Name] - [Conversion Action] Retraction Upload`
- e.g., `Acme Corp - MQL Retraction Upload`
- e.g., `Acme Corp - SQL Retraction Upload`

---

## Post-Setup Verification Checklist

After everything is live, verify the system end-to-end:

- [ ] **Sheet refresh works** — Check the Sheet 24 hours later. Row count should update if new DQ'd leads exist.
- [ ] **Upload runs on schedule** — Check Google Ads → Uploads → History. You should see a daily entry.
- [ ] **Retractions are processing** — In the upload history, click into a run and confirm rows are being processed (not all failing).
- [ ] **Conversion counts are adjusting** — In Google Ads reporting, retracted conversions will reduce the reported conversion count for the day they originally occurred. This may take 24-48 hours to reflect.
- [ ] **No false retractions** — Spot-check 5-10 retracted records in the CRM. Confirm they are genuinely DQ'd leads that should not count as conversions.

---

## Ongoing Maintenance

**Monthly check-in:**
- Review upload history for persistent errors
- Check if new closed-loss reasons have been added to the CRM that should be included in the retraction list
- Verify the data source connection hasn't expired (some warehouse connectors require periodic re-auth)

**When adding a new conversion action later:**
- Copy an existing retraction sheet
- Update the conversion name and query filter
- Create a new upload schedule
- Do NOT modify the existing sheets — each conversion action stays isolated

**When the client adds new DQ reasons:**
- Update the `IN (...)` list in the SQL query for every retraction sheet
- Document the change and the date
- The next sheet refresh will automatically include the new reason

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Upload shows 0 rows processed | Sheet is empty or has wrong headers | Check Sheet data and column names |
| "Conversion action not found" errors | Conversion name in Sheet doesn't match Google Ads exactly | Copy-paste the name from Google Ads |
| "GCLID not recognized" errors | Click is from a different account or older than 60 days | Filter query to last 55 days; verify account ID |
| Upload succeeds but conversion counts unchanged | Retractions take 24-48hrs to reflect in reporting | Wait and re-check |
| Sheet stops refreshing | Data warehouse auth expired | Re-authenticate the connector |
| Row count keeps growing indefinitely | Query isn't date-filtered, accumulating all historical DQs | Add date filter to limit to last 55 days |

---

## Key Principles

1. **Retract quality, not intent.** Bad leads = retract. Good leads who didn't buy = keep.
2. **Exact naming matters.** Conversion names, column headers, and CRM field values must match precisely.
3. **One sheet per conversion action.** Don't combine MQL and SQL retractions in one file.
4. **The 60-day limit is real.** Don't stress about historical data. Focus on catching DQ'd leads going forward.
5. **Test before scheduling.** Always run a manual upload first to catch formatting issues before the daily schedule goes live.
