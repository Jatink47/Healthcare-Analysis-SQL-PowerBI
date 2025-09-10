# Healthcare-Analysis-SQL-PowerBI

# SQL Queries

## 1. Encounter Types and Costs
```sql
SELECT 
    encounterclass,
    COUNT(*) AS encounter_count,
    ROUND(AVG(base_encounter_cost),2) AS avg_base_cost,
    ROUND(AVG(total_claim_cost),2) AS avg_total_cost,
    ROUND(SUM(payer_coverage),2) AS total_coverage
FROM encounters
GROUP BY encounterclass
ORDER BY encounter_count DESC;
```

## 2. yearwise  Encounter Trends
```sql
SELECT 
    DATE_FORMAT(start, '%Y') as year,
    COUNT(*) as encounter_count
FROM encounters
GROUP BY DATE_FORMAT(start, '%Y')
ORDER BY year;
```

## 3. Percentage of encounterclass with repsective to year

```sql
select  date_format(start,"%Y") as year,
		ROUND(sum(case when encounterclass = "ambulatory" then 1 else 0 end)/count(*)*100,2) as ambulatory,
        ROUND(sum(case when encounterclass = "wellness" then 1 else 0 end)/count(*)*100,2) as wellness,
        ROUND(sum(case when encounterclass = "outpatient" then 1 else 0 end)/count(*)*100,2) as outpatient,
        ROUND(sum(case when encounterclass = "urgentcare" then 1 else 0 end)/count(*)*100,2) as urgentcare,
        ROUND(sum(case when encounterclass = "emergency" then 1 else 0 end)/count(*)*100,2) as emergency,
        ROUND(sum(case when encounterclass = "inpatient" then 1 else 0 end)/count(*)*100,2) as inpatient
        from encounters
group by year        
```

## 4. Encounters solved under 24 hour vs over 24 overs 
```sql 
SELECT 
    ROUND(SUM(CASE WHEN TIMESTAMPDIFF(HOUR, start, stop) < 24 THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS under_24_hours,
    ROUND(SUM(CASE WHEN TIMESTAMPDIFF(HOUR, start, stop) >= 24 THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS over_24_hours
FROM encounters;

```

## Patients

## 1. Number of unique patient admitted  each qtr 
```sql 
select 
CONCAT(CAST(YEAR(start) as char), "-", CAST(QUARTER(start) as char)) as year_qtr,
COUNT(DISTINCT patient) as unique_patients
from encounters 
group by year_qtr
order by year_qtr
```

## 2. Number of patient who readmitted within 30 days 
```sql 
WITH CTE AS (
select  patient, start, stop,
 lead(start) over(partition by patient order by start asc) as next_admission
 from encounters 
)

select count(distinct patient) as readmissions
from cte 
where datediff(next_admission,stop) <30 
```

## 3. Which patient had most readmission 
```sql 
WITH CTE AS (
select  patient, start, stop,
 lead(start) over(partition by patient order by start asc) as next_admission
 from encounters 
)
select patient, count(*) as readmissions
from cte 
where datediff(next_admission,stop) < 30 
group by patient
order by readmissions desc
```
## 4. Patient Demographics Distribution
```sql
SELECT 
    gender,
    race,
    ethnicity,
    COUNT(*) as patient_count,
    ROUND(AVG(healthcare_expenses), 2) as avg_healthcare_expenses
FROM patients 
GROUP BY gender, race, ethnicity
ORDER BY patient_count DESC;
```

## 5. Geographic Healthcare Costs location of paitnets count> 100 
```sql
SELECT p.county as country,
    p.state,
    p.city,
    COUNT(DISTINCT p.id) as patient_count,
    round(AVG(p.healthcare_expenses),2) as avg_patient_expenses,
    round(AVG(e.total_claim_cost),2) as avg_encounter_cost,
    COUNT(DISTINCT o.id) as healthcare_facilities
FROM patients p
JOIN encounters e ON p.id = e.patient
JOIN organizations o ON e.organization = o.id
GROUP BY p.county,p.state, p.city
HAVING patient_count > 100
ORDER BY avg_encounter_cost DESC;
```

## Procedures & Conditions

## 1. Top 10 most frequently done procedures 
```sql
select DESCRIPTION,
     count(*)  as procedure_count,
     round(avg(base_cost),2) as avg_base_cost
from procedures
group by description
order by procedure_count desc
limit 10 
```
## 2. Top 10 procedures with highest base cost 
```sql
select DESCRIPTION,
     round(avg(base_cost),2) as avg_base_cost,
     count(*)  as procedure_count
from procedures
group by description
order by avg_base_cost desc
limit 10 
```
## 3. Top Conditions by Age Group
```sql
SELECT 
    c.description,
    CASE 
        WHEN TIMESTAMPDIFF(YEAR, p.birthdate, c.start) < 18 THEN 'Pediatric'
        WHEN TIMESTAMPDIFF(YEAR, p.birthdate, c.start) BETWEEN 18 AND 65 THEN 'Adult'
        ELSE 'Elderly'
    END as age_group,
    COUNT(*) as condition_count
FROM conditions c
JOIN patients p ON c.patient = p.id
GROUP BY c.description, age_group
ORDER BY condition_count DESC;
```
## 4. Chronic vs Acute Conditions
```sql
SELECT 
    IF(stop IS NULL, 'Chronic', 'Acute') AS condition_type,
    COUNT(*) AS condition_count,
		Round(AVG(DATEDIFF(CASE WHEN stop IS NULL THEN CURDATE() ELSE stop END,start))/365,2) AS avg_duration_years
FROM conditions
GROUP BY condition_type;
```
## 5. Most Common Conditions
```sql
SELECT 
    description,
    COUNT(*) as condition_count,
    COUNT(DISTINCT patient) as unique_patients
FROM conditions
GROUP BY description
ORDER BY condition_count DESC
LIMIT 20;
```
## Healthcare Utilization Analysis

## 1.Payer Coverage Effectiveness  for last one year in dataset 
 ```sql
 SELECT 
    py.name as payer_name,
    COUNT(DISTINCT e.patient) as covered_patients,
    AVG(e.payer_coverage/e.total_claim_cost * 100) as avg_coverage_percentage,
    SUM(e.total_claim_cost) as total_claims,
    SUM(e.payer_coverage) as total_coverage
FROM payers py
JOIN encounters e ON py.id = e.payer
WHERE e.total_claim_cost > 0
GROUP BY py.id, py.name 
ORDER BY avg_coverage_percentage DESC;
```

## 2.Provider Performance Analysis
```sql
SELECT 
    pv.name as provider_name,
    pv.speciality,
    o.name as organization,
    COUNT(e.id) as total_encounters,
    round(AVG(e.total_claim_cost),2) as avg_encounter_cost,
    COUNT(DISTINCT e.patient) as unique_patients,
    round(AVG(DATEDIFF(e.stop, e.start))) as avg_days_of_stay
FROM providers pv
JOIN organizations o ON pv.organization = o.id
JOIN encounters e ON pv.id = e.provider
WHERE e.start BETWEEN '2019-01-01' AND '2020-12-31'
GROUP BY pv.id, pv.name, pv.speciality, o.name
ORDER BY total_encounters DESC;
```
## 3.Patient Journey - Conditions to Treatments  highlighting complex cases (procdure > 2) 
```sql
SELECT 
    p.gender, p.race,
    c.description as conditions,
    COUNT(DISTINCT pr.code) as unique_procedures,
    round(AVG(pr.base_cost),2) as avg_procedure_cost,
    COUNT(DISTINCT m.code) as unique_medications
FROM patients p
JOIN conditions c ON p.id = c.patient
LEFT JOIN procedures pr ON c.encounter = pr.encounter
LEFT JOIN medications m ON c.encounter = m.encounter
GROUP BY p.gender, p.race, c.description
HAVING COUNT(DISTINCT pr.code) > 2
ORDER BY avg_procedure_cost DESC;
```
## 4. Patient Observation
```sql
SELECT 
    o.PATIENT, o.DATE, o.DESCRIPTION, o.VALUE, o.UNITS,
    p.FIRST, p.LAST, p.GENDER
FROM observations o
JOIN patients p ON o.PATIENT = p.Id;
```
## Disease Management Analysis

## 1.Allergy Impact on Treatment
```sql
SELECT 
    a.description as allergy,
    c.description as conditions,
    COUNT(DISTINCT m.code) as alternative_medications,
    ROUND(AVG(m.totalcost),2) as avg_medication_cost
FROM allergies a
JOIN patients p ON a.patient = p.Id
JOIN conditions c ON p.Id = c.patient
JOIN medications m ON c.encounter = m.encounter
WHERE a.stop IS NULL
GROUP BY a.description, c.description
ORDER BY avg_medication_cost DESC ,alternative_medications DESC;
```


## 2.Medication Effectiveness by Condition
```sql
SELECT  c.description as conditions, m.description as medication,
    COUNT(*) as prescription_count,
    ROUND(AVG(DATEDIFF(IFNULL(c.stop, CURDATE()), c.start)),2) as avg_condition_duration,
    ROUND(AVG(m.totalcost),2) as avg_medication_cost
FROM conditions c
JOIN medications m ON c.encounter = m.encounter 
    AND m.reasondescription = c.description
GROUP BY c.description, m.description
HAVING prescription_count > 5
ORDER BY avg_condition_duration ASC, prescription_count DESC;
```

## 3.Comorbidity Analysis
```sql
SELECT 
    c1.description as primary_condition,
    c2.description as secondary_condition,
    COUNT(*) as co_occurrence_count,
    round(AVG(e.total_claim_cost),2) as avg_total_cost
FROM conditions c1
JOIN conditions c2 ON c1.patient = c2.patient AND c1.code != c2.code
JOIN encounters e ON c1.encounter = e.id
WHERE c1.start <= c2.start
GROUP BY c1.description, c2.description
HAVING co_occurrence_count > 10
ORDER BY co_occurrence_count DESC;
```sql
