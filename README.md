# 📊 Feature Performance Dashboard — Excel Power Pivot Project

An end-to-end Excel analytics project using **Power Query**, **Power Pivot (DAX)**, and **PivotCharts** to analyze feature usage, user behavior, feedback sentiment, and data quality across a product platform.

---

## 📁 Data Model

### Dimension Tables
| Table | Fields |
|-------|--------|
| `teams` | Team, Department, TeamLead |
| `users` | UserID, Age, Gender, EmploymentStatus, Location |
| `tags` | Tag, Description |
| `dates` | Date, Year, Month, MonthName, MonthYear, Quarter, Weekday, Day |
| `features` | FeatureID, FeatureName, Team, RolloutMonth, ProductOwner |

### Fact Tables
| Table | Fields |
|-------|--------|
| `scroll_depth` | ScrollID, UserID, FeatureID, ScrollPercent, SessionDate |
| `click_logs` | ClickID, UserID, FeatureID, ClickTimestamp, TimeSpentSeconds |
| `feedback_log` | FeedbackID, UserID, FeatureID, Category, SentimentScore, Comment, Timestamp |

### Bridge Table
| Table | Fields |
|-------|--------|
| `fecomponent_tags` | FeatureID, Tag |

> **Note:** M:M relationships cannot exist in Power Pivot. The `fecomponent_tags` bridge table resolves the M:M relationship between `features` and `tags` into two 1:M relationships.

---

## 🔧 Data Preparation (Power Query)

### Data Quality Audit
For each fact table (`click_logs`, `scroll_depth`, `feedback_log`):

1. **Create a copy** of the raw data before cleaning.
2. **Add `MissingFlag`** — custom column:
   ```
   = if [UserID] = null or [FeatureID] = null then "Missing" else "Ok"
   ```
3. **Group by** `UserID`, `FeatureID`, `ClickTimestamp` → operation: *All Rows* → column named `GroupedRows`.
4. **Add `DuplicateFlag`** — custom column:
   ```
   = if Table.RowCount([GroupedRows]) > 1 then "Duplicate" else "Ok"
   ```
5. **Expand** `GroupedRows`, then delete duplicate columns (`GroupedRows.UserID`, `GroupedRows.FeatureID`, `GroupedRows.ClickTimestamp`).
6. **Add `SourceTable`** — custom column: `= "table_name"`.
7. **Unpivot** `MissingFlag` and `DuplicateFlag` → rename `Attribute` → `IssueType`, `Value` → `IssueStatus`.
8. **Filter** `IssueStatus` — uncheck "Ok", keep only "Duplicate" and "Missing".
9. **Group by** `SourceTable`, `IssueType`, `IssueStatus` → operation: *Count Rows* → column named `IssueCount`.
10. Repeat for each table, renaming each result `tablename_issues_summary`.
11. **Append** all `_summary` tables into one query → name it `data_quality_summary`.
12. **Close & Load** → connection only (add to data model).

### Data Cleaning
- Remove duplicates and missing values from fact tables based on the audit.
- For **dimension tables**, remove duplicates/missing values from the ID field (required for 1:M relationships in Power Pivot).

---

## 🔗 Table Relationships (Power Pivot — Diagram View)

All relationships are **1:M** (one-to-many):

```
users       ──UserID──>    click_logs
users       ──UserID──>    scroll_depth
users       ──UserID──>    feedback_log

features    ──FeatureID──> click_logs
features    ──FeatureID──> scroll_depth
features    ──FeatureID──> feedback_log
features    ──FeatureID──> fecomponent_tags  (bridge)

tags        ──Tag──>       fecomponent_tags  (bridge)

dates       ──Date──>      feedback_log      (Timestamp field, converted to Date type in Power Query)
```

> Layout convention: Dimension tables on the **left**, bridge table in the **middle**, fact tables on the **right**.

---

## 📐 DAX Measures & Calculated Columns

### `click_logs` Table — Measures
```dax
ClickCount := COUNTROWS(click_logs)

AvgTime := AVERAGE(click_logs[TimeSpentSeconds])

AvgScrollByTag :=
    CALCULATE(
        AVERAGE(scroll_depth[ScrollPercent]),
        TREATAS(VALUES(fecomponent_tags[FeatureID]), scroll_depth[FeatureID])
    )
```

### `click_logs` Table — Calculated Columns
```dax
FeatureClickCount =
    CALCULATE(
        COUNTROWS(click_logs),
        ALLEXCEPT(click_logs, click_logs[FeatureID])
    )

LowUsageFlag = IF(click_logs[FeatureClickCount] < 150, "Low", "Ok")
```

### `feedback_log` Table — Measures
```dax
AvgSentimentByTag :=
    CALCULATE(
        AVERAGE(feedback_log[SentimentScore]),
        TREATAS(VALUES(fecomponent_tags[FeatureID]), feedback_log[FeatureID])
    )

TotalFeedback := COUNTA(feedback_log[FeedbackID])

BadFeedbackCount :=
    CALCULATE(
        COUNTROWS(feedback_log),
        feedback_log[BadFeedbackFlag] = "Bad"
    )

AvgSentiment := AVERAGE(feedback_log[SentimentScore])
```

### `feedback_log` Table — Power Query Column
```
BadFeedbackFlag = if [SentimentScore] <= 2 then "Bad" else "Good"
```

### `users` Table — Calculated Column
```dax
AgeGroup =
    SWITCH(TRUE(),
        [Age] < 18,  "Under 18",
        [Age] <= 24, "18-24",
        [Age] <= 34, "25-34",
        [Age] <= 44, "35-44",
        [Age] <= 54, "45-54",
        [Age] <= 64, "55-64",
        "65+"
    )
```

---

## ❓ Business Questions Answered (PivotTables Sheet)

| # | Question | Pivot Setup | Filter |
|---|----------|-------------|--------|
| 1 | Which features are most used? | `FeatureName` (Rows) × `ClickCount` (Values) | Between 170–200 |
| 2 | Which design tags perform best? | `Tag` (Rows) × `AvgScrollByTag` (Values) | Top 15 |
| 3 | Do different types of people behave differently? | `AgeGroup` (Rows) × `AvgTime` (Values) | — |
| 4 | What kind of feedback are we getting? | `Category` (Rows) × `Count of FeedbackID` (Values) | — |
| 5 | Do users like our features more over time? | `MonthName` (Rows) × `AvgSentiment` (Values) | Sort A–Z |
| 6 | Are any features underperforming? | `FeatureName` (Rows) × `ClickCount` (Values) + `LowUsageFlag` filter (Low only) | Between 140–150 |
| 7 | Is our data clean and reliable? | `IssueStatus` (Rows) × `IssueCount` (Values) | — |

---

## 🖥️ Dashboard Layout (Features Performance Dashboard Sheet)

**Canvas:** 12-column width × 40-row height cells | Outline border | Title in `B1:Q1`

| Component | Cell Height | Font | Size |
|-----------|------------|------|------|
| Title (`B1:Q1`) | — | Segoe UI | 26 |
| Scorecard title | 3.2 cells | Segoe UI | 16 |
| Scorecard value | 3.3 cells | Segoe UI | 48 |
| Insight card title | 5.2 cells | Segoe UI | 26 |
| Insight card values | 5.3 cells | Segoe UI | 14 |
| Top chart | 12.6 cells | — | — |
| Middle chart | 6.5 cells | — | — |
| Heatmap | 16.3 cells | — | — |
| Doughnut chart | 4.6 cells | — | — |
| Bottom chart | 11.6 cells | — | — |

### Dashboard Components

| Visual | Source Question | Type |
|--------|----------------|------|
| Scorecard: Features with Low Usage | Q6 | `COUNTA` formula referencing pivot |
| Scorecard: Features with Bad Feedback | Q8 | `GETPIVOTDATA` formula |
| Line Chart: Avg Sentiment Over Time | Q5 | 2D Line |
| Insight Card: Top feature, duplicate count, missing count | Q1, Q7 | Formula + `GETPIVOTDATA` |
| Bar Chart: Avg Time Spent by Age Group | Q3 | 2D Clustered Bar (sorted smallest to largest) |
| Doughnut Chart: Feedback by Category | Q4 | Doughnut |
| Bar Chart: Top 5 Most Used Features | Q1 | 2D Clustered Bar (top 5, sorted 179–200) |
| Heatmap: Avg Scroll % by Tag | Q2 | Transposed values + conditional formatting (green–yellow–red) |

### Key Formulas Used
```excel
-- Features with Low Usage (scorecard)
=COUNTA(PivotTables!A78:A80)

-- Bad Feedback Count (scorecard)
=GETPIVOTDATA("[Measures].[Count of FeedbackID]",PivotTables!$A$93,
  "[feedback_log].[BadFeedbackFlag]","[feedback_log].[BadFeedbackFlag].&[Bad]")

-- Insight Card
="-"&PivotTables!A6&" was used the most"&CHAR(10)
="-"&GETPIVOTDATA("[Measures].[Sum of IssuesCount]",PivotTables!$A$87,
  "[data_quality_summary].[IssueStatus]","[data_quality_summary].[IssueStatus].&[Duplicate]")
  &" duplicate values found in the raw data"&CHAR(10)
="-"&GETPIVOTDATA("[Measures].[Sum of IssuesCount]",PivotTables!$A$87,
  "[data_quality_summary].[IssueStatus]","[data_quality_summary].[IssueStatus].&[Missing]")
  &" missing values found in the raw datasets"
```

---

## 🛠️ Tools & Features Used

- **Power Query** — data cleaning, transformation, issue flagging, appending tables
- **Power Pivot** — data model, DAX measures, calculated columns, relationships
- **PivotTables & PivotCharts** — analysis and visualization
- **Conditional Formatting** — color scale heatmap (green → yellow → red)
- **View Settings** — Gridlines and Formula Bar hidden for clean dashboard presentation

---

## 📌 Notes

- Dimension table ID columns must be **unique and non-null** before creating relationships in Power Pivot.
- M:M relationships are resolved via the `fecomponent_tags` **bridge table**, creating two 1:M links.
- The `feedback_log` Timestamp field is converted to **Date type** in Power Query to enable the date dimension relationship.
- All charts and scorecards are pasted to the dashboard sheet and styled with outside borders.
