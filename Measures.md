# 📐 DAX Measures — Carelytics Healthcare Analytics

> **Project:** Carelytics – Power BI Healthcare Analytics Dashboard  
> **Author:** Kanika Dogra  
> **Tool:** Power BI Desktop  
> **DAX Version:** Compatible with Power BI Desktop (March 2024+)  

All measures live in a dedicated **`_Measures`** table (hidden, display-folder organised).  
Reference tables: `Patients`, `Staff`, `Beds`, `Departments`, `Date`.

---

## 📁 Measure Table Structure

```
_Measures/
├── 📂 Patient Volume
│   ├── Total Patients
│   ├── Total Inpatients
│   ├── Total Outpatients
│   └── Total Deaths
├── 📂 Length of Stay
│   ├── Avg LOS (Days)
│   └── % Patients LOS > 10 Days
├── 📂 Financial
│   ├── Total Revenue
│   └── Avg Treatment Cost per Patient
├── 📂 Operational
│   ├── Bed Occupancy Rate (%)
│   ├── Avg ER Waiting Time (mins)
│   └── Patient-to-Staff Ratio
├── 📂 Satisfaction
│   └── Avg Patient Feedback Score
└── 📂 Time Intelligence
    ├── Total Patients MoM %
    └── Revenue YTD
```

---

## 👥 Patient Volume

### Total Patients

Counts all patient admission records (both Inpatient and Outpatient).

```dax
Total Patients = 
COUNTROWS( Patients )
```

**Usage:** KPI card on Hospital Summary page.  
**Notes:** Apply page-level filters (Department, Month) to slice automatically via relationships.

---

### Total Inpatients

```dax
Total Inpatients = 
CALCULATE(
    COUNTROWS( Patients ),
    Patients[PatientType] = "Inpatient"
)
```

**Usage:** KPI card; used as denominator in bed occupancy and LOS calculations.

---

### Total Outpatients

```dax
Total Outpatients = 
CALCULATE(
    COUNTROWS( Patients ),
    Patients[PatientType] = "Outpatient"
)
```

---

### Total Deaths

```dax
Total Deaths = 
CALCULATE(
    COUNTROWS( Patients ),
    Patients[WardStatus] = "Deceased"
)
```

**Usage:** KPI card on Patients Summary page. Format as integer with red conditional formatting when above threshold.

---

### ICU Occupancy Count

```dax
ICU Occupancy Count = 
CALCULATE(
    COUNTROWS( Patients ),
    Patients[WardStatus] = "ICU"
)
```

---

## ⏳ Length of Stay

### Avg LOS (Days)

Average length of stay across discharged inpatients. Excludes outpatients and currently active (undischarged) patients.

```dax
Avg LOS (Days) = 
CALCULATE(
    AVERAGEX(
        Patients,
        Patients[LengthOfStay_days]
    ),
    Patients[PatientType] = "Inpatient",
    NOT ISBLANK( Patients[LengthOfStay_days] )
)
```

**Usage:** KPI card on Patients Summary. Slice by Department or AgeGroup to compare ICU vs General Ward, or Children vs Elderly.  
**Format:** Decimal, 1 decimal place, suffix " days".

---

### % Patients with LOS > 10 Days

Highlights the proportion of long-stay cases — a key operational risk indicator.

```dax
% Patients LOS > 10 Days = 
VAR LongStay =
    CALCULATE(
        COUNTROWS( Patients ),
        Patients[LengthOfStay_days] > 10,
        NOT ISBLANK( Patients[LengthOfStay_days] )
    )
VAR TotalDischarged =
    CALCULATE(
        COUNTROWS( Patients ),
        NOT ISBLANK( Patients[LengthOfStay_days] )
    )
RETURN
    DIVIDE( LongStay, TotalDischarged, 0 )
```

**Usage:** KPI card or gauge visual. Format as percentage. Apply alert rule: red if > 20%.

---

## 💰 Financial

### Total Revenue

```dax
Total Revenue = 
SUM( Patients[TreatmentCost] )
```

**Usage:** Hospital Summary KPI card and bar chart segmented by AgeGroup / Department.  
**Format:** ₹ #,##0 (INR, no decimals for large amounts).

---

### Avg Treatment Cost per Patient

```dax
Avg Treatment Cost per Patient = 
DIVIDE(
    [Total Revenue],
    [Total Patients],
    0
)
```

**Usage:** KPI card. Compare by Department to identify highest-cost care areas.  
**Format:** ₹ #,##0.00

---

### Revenue by Age Group

Used as a quick inline measure for the stacked bar chart on Hospital Summary.

```dax
Revenue by Age Group = 
CALCULATE(
    [Total Revenue],
    ALLEXCEPT( Patients, Patients[AgeGroup] )
)
```

> **Note:** Use `Patients[AgeGroup]` on the axis of the bar chart and `[Total Revenue]` as the value — this standalone measure is typically not needed unless you're building a tooltip table.

---

## 🛏️ Operational

### Bed Occupancy Rate (%)

```dax
Bed Occupancy Rate (%) = 
DIVIDE(
    SUM( Beds[OccupiedBeds] ),
    SUM( Beds[TotalBeds] ),
    0
)
```

**Usage:** Trend line on Hospital Summary (axis = Date[WeekNumber] or Date[MonthName]).  
**Format:** Percentage, 1 decimal place.  
**Alert:** Apply conditional background — amber if > 85%, red if > 95%.

---

### Avg ER Waiting Time (mins)

Calculated only for Emergency department patients where waiting time is captured.

```dax
Avg ER Waiting Time (mins) = 
CALCULATE(
    AVERAGE( Patients[ERWaitingTime_mins] ),
    NOT ISBLANK( Patients[ERWaitingTime_mins] )
)
```

**Usage:** KPI card filtered to Emergency Department. Compare across departments to spot bottlenecks.  
**Format:** Integer, suffix " mins".

---

### Patient-to-Staff Ratio

Number of (inpatient) patients per active staff member. Lower is better.

```dax
Patient-to-Staff Ratio = 
VAR ActiveStaff =
    CALCULATE(
        COUNTROWS( Staff ),
        Staff[IsActive] = TRUE()
    )
RETURN
    DIVIDE( [Total Inpatients], ActiveStaff, 0 )
```

**Usage:** KPI card on Hospital Summary. Slice by Department to identify under-staffed wards.  
**Format:** Decimal, 2 decimal places (e.g., 4.25 patients per staff).

---

## ⭐ Satisfaction

### Avg Patient Feedback Score

```dax
Avg Patient Feedback Score = 
CALCULATE(
    AVERAGE( Patients[FeedbackScore] ),
    NOT ISBLANK( Patients[FeedbackScore] )
)
```

**Usage:** KPI card and bar chart by Department. Scale is 1–5.  
**Format:** Decimal, 2 decimal places. Use a star-rating tooltip visual.  
**Conditional Format:** Red < 3.0, Amber 3.0–3.9, Green ≥ 4.0.

---

## 📅 Time Intelligence

> **Pre-requisite:** `Date` table must be marked as a **Date Table** in Power BI (Table tools → Mark as date table → `Date[Date]`).

---

### Total Patients — Month-over-Month % Change

```dax
Total Patients MoM % = 
VAR CurrentMonth = [Total Patients]
VAR PreviousMonth =
    CALCULATE(
        [Total Patients],
        DATEADD( Date[Date], -1, MONTH )
    )
RETURN
    DIVIDE(
        CurrentMonth - PreviousMonth,
        PreviousMonth,
        BLANK()
    )
```

**Usage:** KPI card with trend arrow. Positive % = more admissions vs last month.  
**Format:** Percentage, 1 decimal place. Show ▲ / ▼ icon via conditional formatting.

---

### Revenue — Year-to-Date (YTD)

```dax
Revenue YTD = 
CALCULATE(
    [Total Revenue],
    DATESYTD( Date[Date] )
)
```

**Usage:** YTD revenue card on Hospital Summary. Compare to prior year with a secondary measure if multi-year data is available.  
**Format:** ₹ #,##0

---

### Revenue — Prior Year YTD (for comparison)

```dax
Revenue Prior Year YTD = 
CALCULATE(
    [Total Revenue],
    DATESYTD(
        SAMEPERIODLASTYEAR( Date[Date] )
    )
)
```

---

### Revenue YTD vs Prior Year %

```dax
Revenue YTD vs PY % = 
DIVIDE(
    [Revenue YTD] - [Revenue Prior Year YTD],
    [Revenue Prior Year YTD],
    BLANK()
)
```

**Usage:** Delta KPI card — shows year-on-year revenue growth.

---

## 🔖 Formatting Conventions

| Measure Category | Format String | Example |
|---|---|---|
| Counts | `0` | `1,284` |
| Currency (INR) | `"₹ " & FORMAT([Measure], "#,##0")` | `₹ 45,20,000` |
| Percentages | `0.0%` | `87.3%` |
| Ratios | `0.00` | `4.25` |
| Time (minutes) | `0" mins"` | `38 mins` |
| Scores | `0.00` | `4.10` |

---

## 🛡️ Measure Authoring Best Practices

1. **Use `VAR` / `RETURN`** — improves readability and avoids repeated expression evaluation.
2. **Always use `DIVIDE(numerator, denominator, alternateResult)`** — never raw `/` division to avoid division-by-zero errors.
3. **Filter with `CALCULATE` + explicit filter arguments** — avoid mixing `FILTER(ALL(...))` with simple predicates unnecessarily.
4. **Avoid `COUNTBLANK`** — use `NOT ISBLANK(...)` inside `CALCULATE` for cleaner semantics.
5. **Prefix measure names** by folder implicitly — e.g., don't abbreviate to `Avg LOS`; write `Avg LOS (Days)` so units are self-documenting.
6. **Mark the Date table** — all `DATESYTD`, `DATEADD`, `SAMEPERIODLASTYEAR` functions require a properly marked calendar table to work correctly.
