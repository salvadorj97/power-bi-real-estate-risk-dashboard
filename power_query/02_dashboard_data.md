# 02 – Dashboard Data: Multi-Table Joins and Transformations

## Data Model Overview

The dashboard consolidates four source tables into a single analytical layer (`Dashboard Data`), with one row per project and 17 gate-level status columns.

```
Download ──────────────────────────────────────────────────────┐
    │                                                           │
    ├── LEFT JOIN ── CARGO                                      │
    │   (on Reference = Project reference)                      │
    │                                                           │
    ├── LEFT JOIN ── RealEstateProjectWP_Transactional          ▼
    │   (on Reference = Real Estate Project)           Dashboard Data
    │                                                  (1 row per project)
    └── LEFT JOIN ── WorkPackages_Referential
        (on WP.Work package = Reference)
```

## Key Transformations

### 1. Numeric Column Cleaning

Several columns arrive as text or mixed types and require cleaning before aggregation.

```powerquery
// Replace nulls with 0 before type conversion (avoids errors in aggregations)
#"Replaced Financial Amount Blanks" = Table.ReplaceValue(
    #"Expanded CARGO Details", null, 0, Replacer.ReplaceValue, {"Financial Amount"}
),

// Validate "Year of impact" — accept only plausible calendar years (1900–2100)
#"Cleaned and Validated Year of impact" = Table.TransformColumns(
    #"Set Financial Amount Type",
    {{"Year of impact", each
        let
            converted    = try Number.FromText(Text.From(_)) otherwise null,
            isPlausible  = (converted <> null) and (converted >= 1900) and (converted <= 2100)
        in
            if isPlausible then converted else null,
    type number}}
),

// Clean "Total Developed Surface" — remove space thousand-separators before conversion
#"Cleaned Total Developed Surface" = Table.TransformColumns(
    #"Cleaned and Validated Year of impact",
    {{"Total Developed Surface", each
        let
            noSpaces  = Text.Replace(Text.From(_), " ", ""),
            asNumber  = try Number.FromText(noSpaces) otherwise null
        in
            asNumber,
    type number}}
)
```

### 2. Grouping and Aggregation

After joining all four tables (one row per work package per project), rows are grouped back to one row per project.

Aggregations applied:

| Output Column | Aggregation |
|---|---|
| `Total Developed Surface Max` | Max across work packages |
| `CAPEX Max` | Max across work packages |
| `Financial Amount Sum` | Sum across work packages |
| `Overall Project Status` | Max status rank across all work packages |
| `Is Any Task Late` | Logical OR (Max of boolean) |
| `A11_ProjectStatus` … `E04.05_ProjectStatus` | Max status rank for that specific gate |

### 3. Gate-Level Status Aggregation

Each of the 17 gate columns is computed by filtering to work packages matching that gate ID and taking the worst (highest rank) status.

```powerquery
GetGateStatus = (gateName as text, groupData as table) as text =>
    let
        // Filter to work packages whose ID exactly matches this gate
        filteredRows   = Table.SelectRows(
            groupData,
            each Text.Clean(Text.Trim(Text.From([Ref.Work Package ID])))
               = Text.Clean(Text.Trim(gateName))
        ),
        statuses       = filteredRows[WP.CalculatedStatus],
        ranks          = List.Transform(statuses, each GetStatusRank(_)),
        maxRank        = if List.IsEmpty(ranks) then -100 else List.Max(ranks)
    in
        GetStatusFromRank(maxRank)
```

Note: exact string matching (`=`) is used instead of `Text.Contains` to avoid cross-contamination between gate IDs that share prefixes (e.g. `E04.00` vs `E04.04`).

## Notes on Anonimisation

- SharePoint URLs have been replaced with `https://your-tenant.sharepoint.com/sites/your-site/`
- Column names referencing internal systems or business units have been generalised
- No project reference data, financial amounts, or country-level records are included in this repository
