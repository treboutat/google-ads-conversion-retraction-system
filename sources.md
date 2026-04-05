# Sources and References

Official documentation and reference material for conversion retraction setup.

## Google Ads Documentation

### Offline Conversion Imports
- **Import offline conversions** — Google's primary guide for uploading offline conversion data, including retractions and restatements.
  - Covers: file format, required columns, upload methods (manual, scheduled, API)
  - Key detail: Adjustment type must be `RETRACT` or `RESTATE`

### Conversion Adjustments
- **About conversion adjustments** — Explains the difference between retracting (removing a conversion entirely) and restating (changing the conversion value).
  - Key detail: Adjustments can only be applied to conversions within the last 60 days
  - Key detail: Adjustments are processed asynchronously — changes appear in reporting within 24-48 hours

### Upload Scheduling
- **Schedule conversion uploads** — How to set up recurring automatic uploads from Google Sheets.
  - Covers: Connecting a Google Sheet, setting frequency, troubleshooting failed uploads
  - Key detail: Google Ads reads from the first tab of the connected Sheet only

### Conversion Tracking Setup
- **Set up offline conversion tracking** — Prerequisites for offline conversion imports, including how to configure conversion actions and enable the Google Click ID (GCLID) parameter.
  - Key detail: Auto-tagging must be enabled in the Google Ads account for GCLIDs to be appended to URLs

## File Format Reference

### Required Columns for Retraction Upload

| Column | Header (exact) | Format | Example |
|--------|----------------|--------|---------|
| A | `Google Click ID` | String | `EAIaIQobChMI...` |
| B | `Conversion Name` | String (exact match to Google Ads) | `MQL - Form Submit` |
| C | `Conversion Time` | `yyyy-MM-dd HH:mm:ss` | `2026-03-15 14:30:00` |
| D | `Adjustment Type` | `RETRACT` or `RESTATE` | `RETRACT` |

### Column Header Rules
- Headers must be in Row 1
- Headers are case-sensitive and whitespace-sensitive
- No extra columns — Google Ads ignores columns it doesn't recognize, but unexpected columns can cause parsing issues
- No merged cells, no formatting, no formulas in headers

### Conversion Name Matching
- The value in the `Conversion Name` column must **exactly** match the name of the conversion action in Google Ads
- Case-sensitive: `MQL` is not the same as `mql`
- Whitespace-sensitive: `MQL ` (trailing space) will not match `MQL`
- To find exact names: Google Ads → Goals → Conversions → Summary → copy the name directly

### Conversion Time Rules
- Must be the timestamp of the **original conversion**, not the retraction time
- Accepted formats: `yyyy-MM-dd HH:mm:ss`, `yyyy-MM-ddTHH:mm:ss`, `yyyy-MM-dd HH:mm:ss+00:00`
- Timezone: Uses the Google Ads account timezone by default. Include timezone offset if your data is in a different timezone.

### GCLID Rules
- Must be from the same Google Ads account you're uploading to
- GCLIDs expire — clicks older than 60 days cannot have their conversions adjusted
- Each GCLID can have multiple conversions; the retraction targets a specific conversion action + time combination
- Invalid or unrecognized GCLIDs are silently skipped (check upload results for "rows with errors")

## Upload Error Reference

| Error | Meaning | Fix |
|-------|---------|-----|
| `ConversionActionNotFound` | Conversion name doesn't match any action in the account | Copy-paste the exact name from Google Ads |
| `GclidNotFound` | GCLID not recognized or from a different account | Verify GCLID is from the correct account; may be expired |
| `ConversionAlreadyRetracted` | This specific conversion was already retracted | Safe to ignore — duplicate retraction attempts are harmless |
| `TooOldToAdjust` | Original conversion is older than 60 days | Expected for historical records; filter query to 60 days |
| `InvalidConversionTime` | Timestamp format not recognized | Use `yyyy-MM-dd HH:mm:ss` format |
| `MissingRequiredField` | A required column is empty for this row | Check for null GCLIDs or conversion times in your data source |

## CRM Reference (HubSpot)

### GCLID Capture
- HubSpot natively captures GCLID when the Google Ads integration is connected
- Property name: `hs_google_click_id` (on contacts)
- If the integration isn't connected, GCLIDs must be captured via hidden form fields from the `gclid` URL parameter

### Closed-Loss Reasons
- Property name: `closed_lost_reason` (on deals) or custom property on contacts
- Values are client-configured — always pull the actual values from the CRM before building your SQL query
- Common values: `AutoDQ`, `Spam`, `Competitor`, `Not in ICP`, `No budget`, `Bad timing`

### Deal vs. Contact Reporting
- Some clients track conversion quality on **deals** (closed-loss reason on the deal object)
- Others track it on **contacts** (lifecycle stage or custom status property)
- Confirm which object the client uses before building your query — the join path to GCLID differs

## Data Warehouse Reference (Funnel.io)

### Offline Conversion Import Data Source
- Data source type: Offline Conversion Import (or CRM-specific connector)
- Required fields: GCLID, conversion timestamp, conversion counts per action
- Static fields: Useful for hardcoding `Conversion Name` and `Adjustment Type` when they're not available as dynamic CRM fields

### Google Sheets Output
- Funnel.io can output query results directly to a Google Sheet
- Scheduled refresh: Available on paid plans; set to daily
- Sheet refresh timing: Schedule 2-3 hours before the Google Ads upload to ensure fresh data

## Related Concepts

### Enhanced Conversions
- Enhanced conversions use first-party data (email, phone) instead of GCLIDs for conversion matching
- Retraction via enhanced conversions follows a similar process but uses different identifiers
- If the client uses enhanced conversions instead of GCLID-based tracking, the upload format differs — consult Google's enhanced conversion adjustment docs

### Conversion Value Rules
- An alternative to retraction for some use cases
- Conversion value rules adjust the *value* of conversions based on audience signals, rather than removing them entirely
- Useful for: assigning different values to different lead sources, adjusting for geographic quality differences
- Not a replacement for retraction — value rules don't remove bad conversions, they re-weight them
