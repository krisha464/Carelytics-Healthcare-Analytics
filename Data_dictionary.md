# 📖 Data Dictionary — Carelytics Healthcare Analytics

> **Project:** Carelytics – Power BI Healthcare Analytics Dashboard  
> **Author:** Kanika Dogra  
> **Tool:** Power BI Desktop (Power Query + DAX)  
---

## 🗂️ Table Overview

| Table Name | Description | Row Grain |
|---|---|---|
| `Patients` | Core fact table — one row per patient admission | One row per admission |
| `Staff` | Staff headcount by department and role | One row per staff member |
| `Beds` | Daily bed occupancy snapshot per department | One row per department per date |
| `Departments` | Department reference / lookup | One row per department |
| `Date` | Standard calendar dimension | One row per calendar day |

---

## 🏥 Table: `Patients`

The primary fact table. Each row represents a unique patient admission episode.

| Column Name | Data Type | Example Values | Description |
|---|---|---|---|
| `PatientID` | Text | `P-00142`, `P-00891` | Unique identifier for each patient admission. Primary key. |
| `PatientName` | Text | `Arjun Sharma`, `Priya Mehta` | Full name of the patient. PII — mask in production reports. |
| `Gender` | Text | `Male`, `Female`, `Other` | Patient's self-reported gender. Used for demographic slicing. |
| `Age` | Whole Number | `34`, `72`, `8` | Patient's age at time of admission (years). |
| `AgeGroup` | Text | `Child (0–17)`, `Adult (18–64)`, `Elderly (65+)` | Derived age bucket used for LOS and revenue breakdowns. Calculated in Power Query. |
| `City` | Text | `Ludhiana`, `Delhi`, `Chandigarh` | Patient's city of residence. Used for geographic distribution visuals. |
| `AdmissionDate` | Date | `2024-01-15` | Date the patient was admitted to the hospital. |
| `DischargeDate` | Date | `2024-01-22`, *(blank if still admitted)* | Date the patient was discharged. Blank for active admissions. |
| `PatientType` | Text | `Inpatient`, `Outpatient` | Indicates whether the patient was admitted overnight (Inpatient) or visited for same-day care (Outpatient). |
| `Department` | Text | `Cardiology`, `Orthopedics`, `Pediatrics`, `Emergency`, `ICU` | The department responsible for the patient's primary care. Foreign key to `Departments[Department]`. |
| `WardStatus` | Text | `Normal Ward`, `ICU`, `Discharged`, `Deceased` | Patient's current or final ward status. Used in Patient Status breakdowns. |
| `TreatmentCost` | Decimal Number | `45200.50`, `12800.00` | Total billed treatment cost for the episode (INR). |
| `ERWaitingTime_mins` | Whole Number | `18`, `45`, `120` | Time in minutes from ER arrival to first clinical contact. Relevant only for Emergency department. Null for non-ER patients. |
| `FeedbackScore` | Decimal Number | `4.2`, `3.8`, `5.0` | Patient satisfaction score collected at discharge. Scale: 1 (Very Poor) to 5 (Excellent). Null if patient deceased or not yet discharged. |
| `LengthOfStay_days` | Whole Number | `3`, `7`, `14` | Computed column: `DischargeDate − AdmissionDate` in days. Null for active admissions. See Power Query note below. |
| `IsDeceased` | True/False | `TRUE`, `FALSE` | Boolean flag derived from `WardStatus = "Deceased"`. Used in mortality KPI calculations. |

> **Power Query Note — `LengthOfStay_days`:**  
> Calculated as a custom column:  
> `= Duration.Days([DischargeDate] - [AdmissionDate])`  
> Rows where `DischargeDate` is null are left as null (active patients).

---

## 👩‍⚕️ Table: `Staff`

One row per staff member. Used to calculate patient-to-staff ratios.

| Column Name | Data Type | Example Values | Description |
|---|---|---|---|
| `StaffID` | Text | `S-001`, `S-098` | Unique identifier for each staff member. |
| `StaffName` | Text | `Dr. Ravi Kumar` | Full name of the staff member. |
| `Department` | Text | `Cardiology`, `ICU`, `Emergency` | Department the staff member is assigned to. Foreign key to `Departments[Department]`. |
| `Role` | Text | `Doctor`, `Nurse`, `Technician`, `Admin` | Job function of the staff member. |
| `IsActive` | True/False | `TRUE`, `FALSE` | Whether the staff member is currently active. Filter to `TRUE` for operational ratios. |

---

## 🛏️ Table: `Beds`

Daily snapshot of bed availability and occupancy per department. Useful for trend charts.

| Column Name | Data Type | Example Values | Description |
|---|---|---|---|
| `BedID` | Text | `B-ICU-01`, `B-CARD-14` | Unique identifier for each physical bed. |
| `Department` | Text | `ICU`, `Cardiology`, `General Ward` | Department the bed belongs to. |
| `Date` | Date | `2024-03-01` | The snapshot date. Foreign key to `Date[Date]`. |
| `IsOccupied` | True/False | `TRUE`, `FALSE` | Whether the bed was occupied on this date. |
| `TotalBeds` | Whole Number | `20`, `50` | Total beds available in the department on this date. |
| `OccupiedBeds` | Whole Number | `17`, `42` | Number of beds occupied in the department on this date. |

---

## 🏢 Table: `Departments`

Lookup/reference table for department metadata.

| Column Name | Data Type | Example Values | Description |
|---|---|---|---|
| `Department` | Text | `Cardiology`, `Orthopedics`, `ICU` | Department name. Primary key — joins to `Patients`, `Staff`, `Beds`. |
| `DepartmentType` | Text | `Clinical`, `Surgical`, `Critical Care`, `Emergency` | Broad category grouping for filtering. |
| `HeadOfDepartment` | Text | `Dr. Anita Singh` | Current department head. Used in tooltip or drill-through context. |
| `FloorNumber` | Whole Number | `1`, `3`, `5` | Floor location within the hospital building. |

---

## 📅 Table: `Date`

Standard Power BI calendar dimension, marked as a Date Table.

| Column Name | Data Type | Example Values | Description |
|---|---|---|---|
| `Date` | Date | `2024-01-01` | Calendar date. Primary key. Mark this table as a Date Table in Power BI. |
| `Year` | Whole Number | `2024`, `2025` | Calendar year. |
| `Quarter` | Text | `Q1`, `Q2`, `Q3`, `Q4` | Quarter label. |
| `Month` | Whole Number | `1` – `12` | Month number. Used for sorting. |
| `MonthName` | Text | `January`, `February` | Full month name. |
| `MonthShort` | Text | `Jan`, `Feb` | Abbreviated month name. |
| `WeekNumber` | Whole Number | `1` – `53` | ISO week number. |
| `DayOfWeek` | Text | `Monday`, `Tuesday` | Day name. |
| `IsWeekend` | True/False | `TRUE`, `FALSE` | True if Saturday or Sunday. |
| `IsCurrentMonth` | True/False | `TRUE`, `FALSE` | Dynamic flag: True if the date falls in the current calendar month. |

---

## 🔗 Relationships

```
Patients[Department]    →  Departments[Department]   (Many-to-One)
Staff[Department]       →  Departments[Department]   (Many-to-One)
Beds[Department]        →  Departments[Department]   (Many-to-One)
Patients[AdmissionDate] →  Date[Date]                (Many-to-One, active)
Beds[Date]              →  Date[Date]                (Many-to-One)
```

> All relationships use a **single-direction filter** from the dimension side to the fact side unless explicitly noted. Cross-filter direction is single unless a specific measure requires bidirectional filtering.

---

## 🧠 Calculated Columns (Power Query)

| Table | Column Name | Logic |
|---|---|---|
| `Patients` | `AgeGroup` | `if [Age] < 18 then "Child (0–17)" else if [Age] < 65 then "Adult (18–64)" else "Elderly (65+)"` |
| `Patients` | `LengthOfStay_days` | `Duration.Days([DischargeDate] - [AdmissionDate])` |
| `Patients` | `IsDeceased` | `[WardStatus] = "Deceased"` |
| `Patients` | `LOS_Bucket` | `if [LengthOfStay_days] < 5 then "<5 Days" else if [LengthOfStay_days] <= 10 then "5–10 Days" else ">10 Days"` |

---

