# Data Quality Audit – Error Summary Function

## Context

As part of an external data quality audit conducted by a Big 4 consulting firm, I designed a single-cell Excel formula to automatically classify data errors across **526 real estate records**.

The goal was to replace manual, row-by-row inspection with a systematic, auditable check — producing a human-readable error summary per site that could be reviewed by Finance and Real Estate leadership.

## What It Does

The formula checks each row for **10 distinct data quality rules** and concatenates all detected errors into a single cell, separated by ` | `. If no errors are found, it returns `"OK"`.

**Example outputs:**

| Result | Meaning |
|---|---|
| `OK` | No anomalies detected |
| `Active site with 0 surface` | Site marked Active but Total Developed Surface = 0 |
| `JV=No but 100%` | Joint Venture flag inconsistency |
| `Check Owned/Rented relationship` | ODS + RDS does not add up to TDS |
| `Active site with 0 surface \| JV=No but 100%` | Multiple errors on the same row |

## The Formula

```excel
=LET(
  SAct,  [@[Start of activity]],
  EAct,  [@[End of activity]],
  SProd, [@[Start Of Production]],
  EProd, [@[End Of Production]],
  Status,[@Status],
  JV,    [@[JV fully consolidated]],
  Pct,   [@[% Control]],
  TDS,   [@[Total Developed Surface]],
  ODS,   [@[Owned Developed Surface]],
  RDS,   [@[Rented Developed Surface]],
  Ctry,  [@Country],
  SID,   [@[Site ID]],
  tol,   0.5,

  msg, TEXTJOIN(" | ", TRUE,
    IF(AND(Status="Closed", SAct="",  EAct=""),
       "Activity dates missing for closed site",
       IF(AND(Status="Closed", OR(SAct="", EAct<>"")),
          "Activity dates incomplete for closed site", "")),

    IF(AND(SAct<>"", EAct<>"", EAct<SAct),
       "End<Start", ""),

    IF(AND(Status="Closed", SProd="", EProd=""),
       "Production dates missing for closed site",
       IF(AND(Status="Closed", OR(SProd="", EProd<>"")),
          "Production dates incomplete for closed site", "")),

    IF(AND(Status="Closed", EAct="", EProd=""),
       "Dates missing in a closed site", ""),

    IF(AND(Pct<>"", OR(Pct<0, Pct>100)),
       "% out of range", ""),

    IF(AND(JV="No", Pct=100),
       "JV=No but 100%", ""),

    IF(AND(TDS<>"", ODS<>"", RDS<>"", ODS+RDS < TDS - tol),
       "Check Owned/Rented relationship", ""),

    IF(AND(Status="Active", TDS<>"", TDS=0),
       "Active site with 0 surface", ""),

    IF(COUNTIFS([Country], Ctry, [Site ID], SID) > 1,
       "Dup SiteID within Country", "")
  ),

  IF(OR(msg="", msg=" "), "OK", msg)
)
```

## Rules Explained

| # | Rule | Logic |
|---|---|---|
| 1 | Activity dates missing for closed site | Status = Closed AND both Start and End of activity are blank |
| 2 | Activity dates incomplete for closed site | Status = Closed AND only one of Start/End is filled |
| 3 | End < Start | End of activity is earlier than Start of activity |
| 4 | Production dates missing for closed site | Status = Closed AND both production dates are blank |
| 5 | Production dates incomplete for closed site | Status = Closed AND only one production date is filled |
| 6 | Dates missing in a closed site | Status = Closed AND both End of activity AND End of production are blank |
| 7 | % out of range | % Control is filled but outside 0–100 range |
| 8 | JV=No but 100% | JV fully consolidated = No but % Control = 100 (contradiction) |
| 9 | Check Owned/Rented relationship | ODS + RDS < TDS − 0.5 m² tolerance (surface areas do not add up) |
| 10 | Active site with 0 surface | Status = Active AND Total Developed Surface = 0 |
| 11 | Dup SiteID within Country | Same Site ID appears more than once within the same Country |

## Design Decisions

**Why `LET()`?**
Named variables make each rule self-documenting and eliminate repeated column references. Without `LET()`, the formula would be illegible and difficult to audit or maintain.

**Why `TEXTJOIN(" | ", TRUE, ...)`?**
`TRUE` as the second argument ignores empty strings, so only triggered rules appear in the output. Multiple errors on the same row are concatenated cleanly into one readable cell.

**Why a 0.5 m² tolerance in rule 9?**
Floating-point rounding in surface area calculations can produce sub-1 m² differences that are not real data errors. The tolerance prevents false positives on valid records.

**Why `COUNTIFS()` for duplicate detection (rule 11)?**
Site IDs may legitimately repeat across countries (a standard internal code used in multiple regions). Scoping the check to Country + Site ID avoids false positives.

## Results

Across 526 records, the formula identified **6 anomalies**:

| Error | Count |
|---|---|
| Active site with 0 surface area | 4 |
| Check Owned/Rented relationship | 1 |
| JV=No but 100% | 1 |

Each anomaly was documented and escalated to the responsible team with a corrective action summary for the audit delivery.

![Data quality output](../docs/data_quality.png)
