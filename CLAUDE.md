# BookKeepingWidgets Project Instructions

## Repository Info
- **GitHub**: https://github.com/missioncritical842/BookKeepingWidgets
- **GitHub Pages**: https://missioncritical842.github.io/BookKeepingWidgets/
- **Current Version**: v1.18

## Grist Documents Using These Widgets

### Headwaters Book Keeping
- **Doc ID**: `hcmQR7zX3i5HqNwzCbi1jn`

### Christ the Anchor Book Keeping
- **Doc ID**: `kEfMiMXXMwg5YGYfhrAnvo`

## Widgets
| Widget | GitHub Pages URL |
|--------|------------------|
| Form1099NEC.html | `https://missioncritical842.github.io/BookKeepingWidgets/Form1099NEC.html?v1.18` |
| PayrollFilingsHelper.html | `https://missioncritical842.github.io/BookKeepingWidgets/PayrollFilingsHelper.html?v1.18` |
| donationenvelopes.html | `https://missioncritical842.github.io/BookKeepingWidgets/donationenvelopes.html?v1.18` |
| donationreceipts.html | `https://missioncritical842.github.io/BookKeepingWidgets/donationreceipts.html?v1.18` |
| envelopes.html | `https://missioncritical842.github.io/BookKeepingWidgets/envelopes.html?v1.18` |
| lastmonthpandl.html | `https://missioncritical842.github.io/BookKeepingWidgets/lastmonthpandl.html?v1.18` |
| lastyearcashflow.html | `https://missioncritical842.github.io/BookKeepingWidgets/lastyearcashflow.html?v1.18` |
| lastyearpandl.html | `https://missioncritical842.github.io/BookKeepingWidgets/lastyearpandl.html?v1.18` |
| pandlalltime.html | `https://missioncritical842.github.io/BookKeepingWidgets/pandlalltime.html?v1.18` |
| quarterlypandl.html | `https://missioncritical842.github.io/BookKeepingWidgets/quarterlypandl.html?v1.18` |

## Commit Requirements
- **Always increment version number** in commit message (e.g., "Description of changes (v1.17)")
- After committing, provide updated GitHub Pages URLs with new version as cache-busting parameter: `?v1.X`
- Push to origin after commit

## Grist MCP Integration - CRITICAL
When working with Grist data:
1. **Always use Grist MCP tools** (`list_tables`, `list_columns`, `list_records`) to verify actual table and column names
2. **Never create fallback or wildcard lookups** - reference exact column/table names from the schema
3. **Never guess column names** - query the schema first using `mcp__grist__list_columns`
4. Both Grist docs have similar schemas but may have slight differences (e.g., AccountsBackup only exists in Christ the Anchor)

### Common Column Lookup
Before referencing any column in widget code, run:
```
mcp__grist__list_columns(doc_id="<doc_id>", table_id="<table_name>")
```
