# 🏥 Meridian Care Network: Diabetic 30-Day Readmission Risk Stratification & Capacity Analysis

**Analyst:** Daniella Egbuniwe  
**Domain:** Healthcare Analytics & Population Health  
**Tools Used:** Power BI Desktop, DAX, Power Query (M Language), Microsoft Excel, Microsoft Word  
**Dataset:** UCI Diabetes 130-US Hospitals Dataset (1999–2008)  

---

## 📌 Project Overview & Problem Statement

At **Meridian Care Network**, 30-day diabetic readmission rates have been rising. Historically, discharge planning staff treated all diabetic inpatients uniformly because no rapid, data-driven method existed to identify higher-risk cohorts at discharge.

The objective of this project is to build an operational, data-driven **Risk Stratification Dashboard** that enables discharge planners and the VP of Population Health to target limited care-coordination staff and budgets toward the highest-risk diabetic discharges.

---

## 🛠️ The 5 Ws Strategic Framework

Before querying or analyzing data, the project scope was aligned with stakeholders using the **5 Ws Framework**:
* **What:** Reduce avoidable 30-day diabetic readmissions by establishing a risk stratification model and determining required care-coordination staffing.
* **Who:** VP of Population Health (Executive Stakeholder) and Hospital Discharge Planners (End Users).
* **Where:** Meridian Care Network inpatient hospital settings.
* **When:** At the point of patient discharge, focusing on 30-day post-discharge outcomes.
* **Why:** Uniform care coordination is operationally infeasible and financially wasteful. Targeted intervention maximizes clinical impact and avoids value-based care financial penalties.

---

## 📊 Dataset Understanding & Granularity Audit

* **Source Data:** UCI Machine Learning Repository — Diabetes 130-US Hospitals (101,766 raw inpatient encounters).
* **Granularity Unit:** Each row represents a **single hospital encounter/stay**, not a unique patient.
* **Unique Patient Audit:** Utilizing DAX (`DISTINCTCOUNT(patient_nbr)`), the dataset was audited to reveal **~69,960 unique patients** across 99,343 cleaned encounters (averaging **1.42 visits per patient**).
* **Granularity Rationale:** Stay-level granularity was strictly preserved. Deduplicating patient IDs would erase historical readmissions and distort true 30-day rate calculations.

---

## 🧹 Data Cleaning, Transformations & Reconciliation

Data cleaning and ETL logic were executed in **Power Query**, with full auditing tracked in `Data_Cleaning_Log_Daniella_Egbuniwe.xlsx`.

### Key Cleaning Steps:
1. **Ineligible Discharge Exclusion:** Filtered out deceased and hospice discharges (`discharge_disposition_id` in **11, 13, 14, 19, 20**). Patients who are deceased or in hospice care cannot medically be readmitted; leaving them in artificially deflates the readmission rate denominator.
2. **Target Variable Creation:** Engineered binary `30day_readmission` flag (`1` if `readmitted` = `<30`, else `0`).
3. **Lookup Table Standardization & Joins:** Converted ID columns to Whole Number format across files. Split `IDS_mapping.csv` into 3 reference queries (`dim_admission_type`, `dim_discharge_disposition`, `dim_admission_source`), applied Left Outer Merges, and disabled "Enable Load" on reference tables to streamline the data model.
4. **Missing Value Standardization:** Standardized missing text placeholders (`?`) to `"Unknown"` across all categorical fields. Retained explicit clinical `"None"` values in laboratory outcome fields (`max_glu_serum`, `A1Cresult`).
5. **Text Formatting & Whitespace Audit:** Applied `Text.Trim` and `Text.Clean` to dimension lookup descriptions, preventing offset alignment and duplicate matrix categories.

### Row Count Reconciliation Table:

| Pipeline Step | Starting Row Count | Rows Changed / Removed | Ending Row Count | Technical & Business Justification |
| :--- | :---: | :---: | :---: | :--- |
| **Raw Import** | 101,766 | 0 | 101,766 | Initial dataset load from `diabetic_encounter_raw.csv`. |
| **Filter Ineligible Discharges** | 101,766 | **-2,423** | **99,343** | Excluded Expired/Hospice codes (11, 13, 14, 19, 20). |
| **Relational Merges & Cleaning** | 99,343 | 0 | 99,343 | Applied Left Outer Joins and standardized `?` placeholders. |
| **Final Analysis Dataset** | **101,766** | **-2,423** | **99,343** | **100% verified encounter dataset ready for DAX & modeling.** |

---

## 📈 Feature Engineering & Risk Scoring Model

To operationalize risk at discharge, four custom columns were constructed:

1. **`total_prior_visits`** = `[number_outpatient] + [number_emergency] + [number_inpatient]`
2. **`Risk Score`** = Additive DAX rating (0 to 4):
   ```dax
   Risk Score = 
     IF(diabetic_encounter_raw[number_inpatient] >= 1, 1, 0) +
     IF(diabetic_encounter_raw[time_in_hospital] >= 5, 1, 0) +
     IF(diabetic_encounter_raw[num_medications] >= 15, 1, 0) +
     IF(diabetic_encounter_raw[admission_type_id] = 1, 1, 0)
   ```
3. **`Risk Tier`** = `IF([Risk Score] >= 2, "High Risk Tier", "Standard Tier")`
4. **`Diagnosis Band`** = Grouped diagnostic complexity (`1-3 (Low)`, `4-6 (Moderate)`, `7-8 (High)`, `9+ (Very High)`).
5. **`LOS Tier`** = Hospital length of stay grouping (`1-2 Days`, `3-6 Days`, `7+ Days`).

---

## 📐 Core DAX Measures Glossary

All DAX measures are centralized inside a dedicated `_Measures` table:

```dax
// 1. Core KPIs
Total Encounters = COUNTROWS(diabetic_encounter_raw)

30Day Readmissions = SUM(diabetic_encounter_raw[30day_readmission])

30Day Readmission Rate = DIVIDE([30Day Readmissions], [Total Encounters], 0)

// 2. High-Risk Tier Metrics
High Risk Encounters = CALCULATE([Total Encounters], diabetic_encounter_raw[Risk Tier] = "High Risk Tier")

High Risk Share % = DIVIDE([High Risk Encounters], [Total Encounters], 0)

High Risk Readmission Rate = CALCULATE([30Day Readmission Rate], diabetic_encounter_raw[Risk Tier] = "High Risk Tier")

% of Readmissions Captured = 
DIVIDE(
    CALCULATE([30Day Readmissions], diabetic_encounter_raw[Risk Tier] = "High Risk Tier"),
    [30Day Readmissions],
    0
)

// 3. Operational Staffing & Workload What-If Modeling
Weekly High Risk Encounters = [High Risk Encounters] / 52

Weekly Coordination Hours = ([Weekly High Risk Encounters] * [Minutes Per Patient Value]) / 60

Required Care Coordinators (FTE) = DIVIDE([Weekly Coordination Hours], 35, 0)
```

---

## 🎯 Analytical Findings & Stakeholder Answers

### Core Project Brief Questions (Section 7 Audit)
1. **Overall Baseline Readmission Rate:** **11.39%** (11,314 readmissions across 99,343 encounters).
2. **Subgroup Variation:**
   * **Age:** Readmission rates increase progressively with age, peaking in the `70-80` and `80-90` age brackets.
   * **Length of Stay:** Stays ≥ 5 days exhibit a readmission rate of **13.2%**, compared to **9.8%** for shorter stays.
   * **Admission Type:** Emergency admissions account for the highest total volume of readmissions.
3. **Primary Risk Markers:** Prior inpatient visits (`number_inpatient` ≥ 1) showed the strongest individual association with 30-day readmissions (>20% readmission rate).
4. **Risk Flag & Tier Allocation:** The High-Risk Tier (≥ 2) captures **56.20% of encounters** and **68.35% of all 30-day readmissions**, with an elevated readmission rate of **13.85%**.
5. **Operational Staffing Impact:** At a standard allocation of **45 minutes per patient**, coordinating the High-Risk Tier requires **23.0 FTE Care Coordinators** (compared to 40.9 FTEs for 100% un-stratified coverage).

### Bonus In-Depth Questions
* **Bonus Q1 — Diagnosis Depth:** Readmission rates scale directly with the number of diagnoses, rising from **7.4%** for 1–3 diagnoses up to **12.6%** for encounters with **9+ diagnoses**.
* **Bonus Q2 — Disposition Dynamics:** After controlling for length of stay (LOS Tiers), patients discharged to **Skilled Nursing Facilities (SNFs)** and **Home Health Care** consistently demonstrate elevated readmission rates across short, medium, and long stays.

---

## 📑 Data Dictionary Appendix Summary

The report model optimizes schema performance by maintaining **26 active report fields** and hiding **28 granular/unused fields** (such as individual drug dosage tracking columns, `weight`, and `payer_code`). Full descriptions and allowed values are documented in the project repository's `Data_Cleaning_Log_Daniella_Egbuniwe.xlsx` file.

---

## 💡 Strategic Recommendations

1. **EHR Discharge Integration:** Deploy the 4-factor Risk Score directly into the hospital EHR system to flag high-risk patients automatically at discharge.
2. **Resource Allocation:** Deploy **23 FTE Care Coordinators** dedicated exclusively to High-Risk Tier discharges.
3. **Specialized Post-Acute Handoffs:** Establish targeted care-transition protocols for discharges to Skilled Nursing Facilities and Home Health agencies.
