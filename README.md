# Conversion Retraction for Disqualified Leads (Google Ads)

Build an automated pipeline that retracts poor-quality conversions from Google Ads using offline conversion uploads — so the algorithm stops optimizing toward spam, auto-DQs, and ICP mismatches.

## What This Skill Does

When a lead converts through Google Ads (fills out a form, books a call, etc.), that conversion gets fed back to Google's algorithm as a positive signal. Google then looks for more people like that person.

The problem: not all conversions are good conversions. If a spam bot fills out your form, or someone outside your ICP books a call and gets disqualified — Google still treats those as wins. Over time, your campaigns attract more of the wrong people because the algorithm is being trained on dirty data.

**This skill walks you through building an automated retraction pipeline that:**

1. Identifies which disqualified leads should be retracted (and which should not)
2. Pulls the right data from your CRM via a data warehouse
3. Formats it into Google Ads' offline conversion upload format
4. Schedules daily automatic uploads that tell Google: "ignore these conversions"

The result: Google's algorithm gradually stops serving ads to profiles that match your junk leads, and starts focusing spend on people who actually look like your real customers.

## Who This Is For

- Performance marketers running lead gen campaigns on Google Ads
- Agency teams managing conversion tracking for clients
- Marketing ops teams responsible for CRM-to-ad-platform data flows

## Prerequisites

Before starting, you need:

- **Google Ads account** with offline conversion tracking enabled
- **CRM** (e.g., HubSpot, Salesforce) with closed-loss reason tracking on deals or contacts
- **GCLID capture** — your forms or CRM must be storing the Google Click ID from each ad click
- **Data warehouse or ETL tool** (e.g., Funnel.io, Supermetrics, BigQuery) that can query CRM data and output to Google Sheets
- **Google Sheets** access for the retraction upload files

## How to Use

### As a Claude Code Skill

Drop the `SKILL.md` file into your `.claude/commands/` directory:

```bash
cp SKILL.md .claude/commands/conversion-retraction.md
```

Then invoke it in Claude Code:

```
/conversion-retraction
```

Claude will walk you through each step interactively — identifying which leads to retract, building the SQL query, setting up the Sheets, and configuring the upload schedule in Google Ads.

### As a Manual SOP

Read through `SKILL.md` from top to bottom. Each step includes:

- What to do and why
- Exact field names, formats, and configurations
- Common mistakes and how to avoid them
- Edge cases and decision frameworks

## File Structure

```
├── README.md       ← You are here. Overview and setup instructions.
├── SKILL.md        ← The full skill. Detailed step-by-step procedure with
│                     decision logic, SQL templates, and troubleshooting.
└── sources.md      ← Google's official documentation and reference links.
```

## Key Decisions This Skill Helps You Make

| Decision | Framework |
|----------|-----------|
| Which closed-loss reasons to retract? | Retract **quality** problems (spam, ICP mismatch, auto-DQ). Keep **intent** problems (competitor, timing, budget). |
| Which conversion actions to retract? | Retract MQL and SQL. Generally do not retract Closed-Won/Revenue. |
| How to handle "Competitor" losses? | Do **not** retract. These were real, qualified prospects — useful signal for the algorithm. |
| How to handle edge cases (no-shows, wrong service)? | Depends on pattern. Discuss with client. See Step 1 in SKILL.md. |
| One sheet or multiple? | One sheet per conversion action. Never combine MQL and SQL retractions. |

## Quick Reference

- **Google Ads retraction window:** 60 days (use 55-day filter for safety buffer)
- **Upload format:** Google Click ID, Conversion Name, Conversion Time, Adjustment Type
- **Adjustment type for retraction:** `RETRACT`
- **Refresh cadence:** Sheet refreshes daily → Google Ads pulls daily
- **Conversion name matching:** Case-sensitive, whitespace-sensitive, must be exact

## Important Notes

- Always confirm the retraction list and conversion actions with the client before the schedule goes live
- Retractions take 24-48 hours to reflect in Google Ads reporting
- You cannot retract conversions older than 60 days — this is a Google limitation
- Each conversion action needs its own Sheet and upload schedule
- If GCLID capture is not set up in the CRM, that must be fixed first — it's a prerequisite, not something this skill can work around
