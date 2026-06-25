# 03 – Status Logic: Priority Rules and Rank Functions

## The Problem with Simple Status Lookups

A real estate project has many work packages, each with its own status. To aggregate these into a single "project risk level," you need a consistent rule for which status "wins" when multiple statuses exist.

A simple lookup table is not enough: the priority depends on the **type** of status, not just its value.

## Priority Hierarchy

```
1. Severity label     (High risk > Medium risk > Low risk)
2. Task time status   (Late)
3. OP Status          (any non-empty value)
4. Due Diligence flag (DD Yes → "No Status (DD Yes)")
5. Fallback           ("No Status (DD No)")
```

## The Function

```powerquery
DefineWorkPackageStatus = (CurrentRow as record) as text =>
    let
        // Safely extract each input, defaulting to empty string if null
        severity   = if CurrentRow[WP.Severity]         is null then "" else CurrentRow[WP.Severity],
        taskStatus = if CurrentRow[WP.Task time status] is null then "" else CurrentRow[WP.Task time status],
        opStatus   = if CurrentRow[WP.OP Status]        is null then "" else CurrentRow[WP.OP Status],
        ddFlag     = if CurrentRow[Ref.Due Diligence]   is null then "" else CurrentRow[Ref.Due Diligence],

        // Inner function: convert severity label to a numeric priority
        GetSeverityPriority = (s as text) as number =>
            if      s = "High risk"   then 3
            else if s = "Medium risk" then 2
            else if s = "Low risk"    then 1
            else 0,

        // Normalise task status for robust string comparison
        cleanedTask = Text.Lower(Text.Trim(Text.Clean(taskStatus)))
    in
        // Apply priority rules in order
        if   GetSeverityPriority(severity) > 0 then severity          // Rule 1: Severity
        else if cleanedTask = "late"            then "Late"            // Rule 2: Late task
        else if Text.Trim(opStatus) <> ""       then opStatus          // Rule 3: OP Status
        else if ddFlag = "Yes"                  then "No Status (DD Yes)"  // Rule 4: DD flag
        else                                         "No Status (DD No)"   // Rule 5: Fallback
```

## Status Rank System

To compare and aggregate statuses numerically, each status maps to a rank:

```powerquery
GetStatusRank = (statusText as text) as number =>
    let lowerStatus = Text.Lower(statusText) in
        if      lowerStatus = "late"              then  6
        else if lowerStatus = "high risk"         then  5
        else if lowerStatus = "medium risk"       then  4
        else if lowerStatus = "low risk"          then  3
        else if lowerStatus = "open"              then  2
        else if lowerStatus = "upcoming"          then  1.5
        else if lowerStatus = "pending approval"  then  1
        else if lowerStatus = "pending"           then  0.5
        else if lowerStatus = "approved"          then  0
        else if lowerStatus = "not applicable"    then -0.01
        else if lowerStatus = "recycled"          then -0.05
        else if lowerStatus = "no status (dd yes)" then -0.1
        else if lowerStatus = "no status (dd no)"  then -0.1
        else if lowerStatus = "n/a for project"   then -100
        else -1  // Unknown
```

The reverse function converts a rank back to a label for display:

```powerquery
GetStatusFromRank = (rank as number) as text =>
    if      rank =  6    then "Late"
    else if rank =  5    then "High risk"
    else if rank =  4    then "Medium risk"
    else if rank =  3    then "Low risk"
    else if rank =  2    then "Open"
    else if rank =  1.5  then "Upcoming"
    else if rank =  1    then "Pending Approval"
    else if rank =  0.5  then "Pending"
    else if rank =  0    then "Approved"
    else if rank = -0.01 then "Not applicable"
    else if rank = -0.05 then "Recycled"
    else if rank = -0.1  then "No Status (DD)"
    else if rank = -100  then "N/A for Project"
    else "Unknown"
```

## Why Ranks Instead of Direct String Comparison?

Using numeric ranks allows:

- `List.Max(ranks)` to find the worst status across any list of work packages
- Consistent aggregation at project level and gate level with the same function
- Easy extension: adding a new status only requires inserting one line in each function

## Usage Example

```powerquery
// Add calculated status column to each work package row
#"Added WorkPackageCalculatedStatus" = Table.AddColumn(
    #"Set WP.Is Task Late Type",
    "WP.CalculatedStatus",
    each DefineWorkPackageStatus(_),
    type text
),

// Aggregate to project level: take the worst status across all work packages
"Overall Project Status" = each GetStatusFromRank(
    List.Max(List.Transform([WP.CalculatedStatus], each GetStatusRank(_)))
)
```
