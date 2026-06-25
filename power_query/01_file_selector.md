# 01 – SharePoint File Selector

## Problem

When using `SharePoint.Files()` in Power Query and sorting by `Date modified` to pick the latest file, there is a subtle bug:

> **SharePoint updates the `Date modified` timestamp on a folder whenever anyone opens or browses it** — even without editing any file.

This means Power BI can silently load stale data. No error is raised.

## Solution

Name source files with a `YYYYMMDD` prefix (e.g. `20251104_RMT_Download.xlsx`).
Extract the date from the filename itself and sort by that — ignoring SharePoint metadata.

## Annotated M Code

```powerquery
let
    // 1. Connect to SharePoint file library
    Source = SharePoint.Files(
        "https://your-tenant.sharepoint.com/sites/your-site/",
        [ApiVersion = 15]
    ),

    // 2. Filter to the target folder and file pattern
    Keep = Table.SelectRows(
        Source,
        each Text.Contains(Text.Lower([Name]), "_rmt_download")
            and [Extension] = ".xlsx"
            and Text.Contains(
                Text.Lower([Folder Path]),
                "/shared documents/data storage/01_risk management table/"
            )
    ),

    // 3. Extract date from filename prefix (YYYYMMDD)
    //    This is the KEY FIX: we ignore [Date modified] from SharePoint metadata
    //    and instead parse the date from the filename itself.
    WithDates = Table.AddColumn(
        Keep,
        "FileDate",
        each
            let
                nm = [Name],
                // Extract year, month, day from positions 0-7 of filename
                y  = try Number.FromText(Text.Start(nm, 4))     otherwise null,
                m  = try Number.FromText(Text.Middle(nm, 4, 2)) otherwise null,
                d  = try Number.FromText(Text.Middle(nm, 6, 2)) otherwise null
            in
                if y <> null and m <> null and d <> null
                then #date(y, m, d)
                else null,  // Non-conforming filenames return null and sink to the bottom
        type date
    ),

    // 4. Sort: newest filename date first; use [Date modified] as tiebreaker only
    Sorted = Table.Sort(
        WithDates,
        {{"FileDate", Order.Descending}, {"Date modified", Order.Descending}}
    ),

    // 5. Pick the first row (= most recent valid file) or raise a clear error
    PickFile =
        if Table.RowCount(Sorted) > 0
        then Sorted{0}
        else error "No *_RMT_Download.xlsx file found in SharePoint folder.",

    // 6. Load the binary content and open as Excel workbook
    Content = PickFile[Content],
    WB      = Excel.Workbook(Content, null, true),

    // 7. Try to load a named table called "Download", fall back to sheet, then first table
    TryTable = try WB{[Item = "Download", Kind = "Table"]}[Data] otherwise null,
    TrySheet =
        if TryTable = null
        then try WB{[Item = "Download", Kind = "Sheet"]}[Data] otherwise null
        else null,
    FromNamed =
        if TryTable <> null then TryTable
        else if TrySheet <> null then Table.PromoteHeaders(TrySheet, [PromoteAllScalars = true])
        else error "No tables or sheets found in workbook.",

    // 8. Add a short "Reference" key (first 5 chars of the "Ref" column) for joins
    WithRef =
        if List.Contains(Table.ColumnNames(FromNamed), "Ref")
        then Table.AddColumn(FromNamed, "Reference", each Text.Start(Text.From([Ref]), 5), type text)
        else FromNamed
in
    WithRef
```

## Why This Works

| Approach | Risk |
|---|---|
| Sort by `[Date modified]` (default) | Folder browsing silently resets the timestamp → wrong file loaded |
| Sort by extracted filename date | Deterministic, reproducible, independent of SharePoint metadata |

## Naming Convention Required

Source files must follow: `YYYYMMDD_SomeName.xlsx`

Example: `20251104_RMT_Download.xlsx`

Files that do not match this pattern receive `null` as their `FileDate` and are always ranked below conforming files.
