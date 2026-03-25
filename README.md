# Bofor Sales Kanban

A single-file HTML sales pipeline board built as a **Dynamics 365 web resource**. It reads live opportunity data from the D365 Web API and renders it as a Kanban board with owner filters, smart filters, card sorting, and a Won column — all without any backend, build step, or external dependency.

---

## What it does

- Renders open opportunities in three pipeline columns: **Develop → Propose → Close**
- Shows **Won** opportunities in a fourth column (current calendar year)
- Filters cards by **owner** — defaults to the logged-in user on load
- Four **smart filter** pills that can be combined (AND logic):
  - 🔥 **Hot** — probability > 60 %
  - 💰 **Large** — estimated revenue > 100 000 €
  - ⏰ **Closing soon** — estimated close date within 30 days
  - 🧹 **Needs cleanup** — close date in the past, or `schedulefollowup_qualify` in the past
- **Sort** cards within each column by revenue (default), estimated close date, or probability
- Column headers show a count of cards hidden by the active filter
- **Search** across opportunity topic and customer name in real time
- Cards link directly to the opportunity record in D365
- Refresh button re-fetches all data without a page reload

---

## How it works

### Data source

Everything comes from the **D365 OData Web API v9.2** via two fetch calls made on load:

| Query | Purpose |
|---|---|
| `GET /api/data/v9.2/opportunities?$filter=statecode eq 0 ...` | Open pipeline opportunities |
| `GET /api/data/v9.2/opportunities?$filter=statecode eq 1 ...` | Won opportunities (current year) |

Pagination is handled automatically — the helper `fetchAllPages()` follows `@odata.nextLink` until all records are fetched.

### Fields used

| Field | Used for |
|---|---|
| `topic` | Card title |
| `_customerid_value`, `customeridname` | Customer name on card |
| `estimatedvalue` | Revenue display and sort |
| `closeprobability` | Probability badge and sort |
| `estimatedclosedate` | Close date badge, Closing soon filter |
| `stepname` | Column assignment (see below) |
| `_ownerid_value`, `owneridname` | Owner filter buttons |
| `actualclosedate`, `actualvalue` | Won column |
| `schedulefollowup_qualify` | Needs cleanup filter |
| `opportunityid` | Card link to record |

### Column assignment

The `stepname` field is mapped to columns with a simple regex:

```
/develop/i  → Develop column
/propose/i  → Propose column
/close/i    → Close column
```

This handles prefixed stage names like `2-Develop`, `3-Propose`, `4-Close` that are common in D365 sales process configurations. Opportunities whose `stepname` does not match any pattern are silently ignored.

### Current user detection

On load, the board resolves the current user via `Xrm.Utility.getGlobalContext().getUserName()` (available inside the D365 iframe context). The owner filter defaults to that user automatically. If `Xrm` is not available (e.g. local testing), it falls back to showing all opportunities.

### No dependencies

- No jQuery, no React, no bundler
- No external CSS frameworks
- Google Fonts (`Segoe UI`) used only where available — falls back to system sans-serif
- One `<img>` tag pointing to the Bofor logo on bofor.com (can be replaced with a base64 inline image if needed)

---

## Deploying to any D365 Sales instance

### Requirements

- D365 Sales (any version supporting Web API v9.2 — this covers all versions from 2016 Update 1 onward)
- A user account with the **System Customizer** or **System Administrator** security role to create web resources
- Opportunities must use a sales process with stage names containing `Develop`, `Propose`, and/or `Close` (case-insensitive). If your stage names differ, edit the regex map at the top of the script.

### Step-by-step deployment

1. **Download** `bofor_kanban.html` from this repository.

2. In D365, go to **Settings → Customizations → Customize the System** (or open any Solution you want to deploy into).

3. In the solution, select **Web Resources → New**.

4. Fill in:
   - **Name**: e.g. `bofor_saleskanban` (no spaces, no special chars — this becomes the URL path)
   - **Display name**: Sales Kanban
   - **Type**: Web Page (HTML)

5. Click **Choose File** and upload `bofor_kanban.html`.

6. **Save and Publish**.

7. The board is now accessible at:
   ```
   https://[your-org].crm.dynamics.com/WebResources/bofor_saleskanban
   ```

8. Optionally, add it as a **Site Map** entry (App → Edit → add a Sub Area pointing to the web resource URL) so users can navigate to it from the Sales Hub sidebar.

### Adapting stage names

Open the HTML file and find the `STAGE_MAP` section near the top of the `<script>` block:

```js
const STAGE_PATTERNS = [
  { key: 'develop', regex: /develop/i },
  { key: 'propose', regex: /propose/i },
  { key: 'close',   regex: /close/i   },
];
```

Change the regex patterns to match whatever `stepname` values your D365 sales process uses.

### Adapting the currency / revenue threshold

The **Large** smart filter threshold (100 000 €) and currency symbol are defined as constants at the top of the script:

```js
const LARGE_THRESHOLD = 100000;
const CURRENCY_SYMBOL = '€';
```

---

## File structure

```
bofor_kanban.html        ← the entire app, self-contained
README.md
```

There is intentionally only one file. D365 web resources are deployed as individual files; keeping everything in one HTML file means deployment is a single upload with no build step.

---

## Development / local testing

Because the board calls `/api/data/v9.2/...`, which only resolves inside a D365 environment, you cannot run it fully against a live CRM from `file://`. Options:

- **Upload and test in D365** — simplest. Edit the file, re-upload, publish, hard-refresh.
- **Use a reverse proxy** — proxy `localhost/api/data/v9.2` to your real CRM org URL and open the file via a local HTTP server.
- **Mock data** — the script contains a `DEMO_MODE` flag at the top. Set it to `true` to render with hardcoded sample opportunities instead of fetching from the API.

---

## Known limitations

- The board is read-only — cards cannot be dragged between columns. D365 enforces stage transitions through the business process flow engine; arbitrary drag-and-drop would bypass validation rules.
- Won column shows only the current calendar year.
- The board respects D365 security roles at the API level — users only see records they have access to — but the owner filter UI shows all owners that appear in the fetched result set.

---

## License

THIS IS A WORKING SAMPLE FOR ANYONE TO USE, SHARE OR MODIFY
