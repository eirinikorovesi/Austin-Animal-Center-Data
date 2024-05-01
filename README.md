#Austin Animal Center Data
It is with no doubt that Austin is 
Database Design

Table: animal_intakes
The animal_intakes table is designed to store detailed records of all animals brought into the shelter. Below is an outline of the table structure and the rationale behind the design choices.

### Table: `animal_intakes`

The animal_intakes table is designed to store detailed records of all animals brought into the shelter. Below is an outline of the table structure and the rationale behind the design choices

| Column Name      | Data Type                   | Description                                                      |
|------------------|-----------------------------|------------------------------------------------------------------|
| `intake_id`      | SERIAL (Primary Key)        | Auto-incrementing identifier for each record.                    |
| `animal_id`      | TEXT                        | Non-unique identifier assigned to each animal.                   |
| `name`           | TEXT                        | Name of the animal, if provided.                                 |
| `datetime`       | TIMESTAMP WITHOUT TIME ZONE | Date and time when the animal was brought into the shelter.      |
| `intake_type`    | TEXT                        | Intake category (e.g., stray, surrendered).                   |
| `intake_condition` | TEXT                      | Condition of the animal upon arrival (e.g., healthy, injured).   |
| `animal_type`    | TEXT                        | Type of animal (e.g., dog, cat).                                 |
| `sex_upon_intake` | TEXT                       | Sex and reproductive status of the animal at intake.             |
| `age_value`      | SMALLINT                    | Numeric value representing the animal's age.                |
| `age_unit`       | VARCHAR(10)                 | Unit of measurement for the age (e.g., years, months, weeks).    |
| `breed`          | TEXT                        | Breed of the animal, if known.                                   |
| `color`          | TEXT                        | Color of the animal's fur, feathers, etc.                        |


**Design Rationale**

- intake_id: This is a unique identifier for each intake event, ensuring that each record is distinctly accessible and manageable.
- animal_id: While this ID is unique to each animal, it is not unique to each intake event, as animals may be brought to the shelter more than once.
age_value and age_unit:
- Splitting the Age upon Intake into numeric (age_value) and textual (age_unit) components allow for more straightforward and efficient numerical analyses and queries.
    - age_value uses SMALLINT to efficiently store the age numerically while ensuring the data fits within the typical range of animal ages.
    - age_unit is defined as VARCHAR(10) to optimize storage and enforce consistency in the units of measurement.
- I remove Monthyear column because I can extract it from datetime column. I also removed found_location because all data it's in Austin city and I don't need the exact location for any analysis.
**Data Normalization**

Redundant fields like MonthYear have been removed to normalize the database and reduce redundancy, as this information can be derived from the datetime column.


### Copy the csv into the table
```sql
COPY animal_intakes (
	Animal_id, 
	Name, 
	DateTime,
	Intake_Type,
	Intake_Condition,
	Animal_Type,
	Sex_upon_Intake,
	Age_upon_Intake,
	Breed,
	Color)
FROM 'filepath'
WITH (FORMAT CSV,HEADER);
```
**Populate age_value and age_unit**

To populate the age_value and age_unit we need to split the age_upon_intake. Since age_upon_intake is a text and age_value is stored as smallint, I need to cast the split_part as an integer.

```sql
UPDATE animal_intakes
SET age_value = CAST(SPLIT_PART(age_upon_intake, ' ', 1) AS INTEGER),
    age_unit = SPLIT_PART(age_upon_intake, ' ', 2);
```

**Standardizing the age_unit**
Since I want to check the average and median age of animals in the shelter, I need to standardize the age in one unit. Looking for the average and median now, as they are, will give me the numbers 3.37 and 2, respectively, but I don't know if it is in days, weeks, months, or years. Running a SELECT DISTINCT(age_unit) query, I saw that the age units are: day, days, month, months, week, weeks, year, years, and null (only 1)

```sql
UPDATE animal_intakes
SET age_unit = CASE
    WHEN age_unit IN ('day', 'days') THEN 'days'
    WHEN age_unit IN ('week', 'weeks') THEN 'weeks'
    WHEN age_unit IN ('month', 'months') THEN 'months'
    WHEN age_unit IN ('year', 'years') THEN 'years'
    ELSE age_unit
END;
```


#### Handling Data Anomalies: Negative Age Values

After populating the age_value and age_uni, I selected (with "distinct") the columns and identified records in the `animal_intakes` table with negative age values. Given that negative ages are not feasible and because there are only 13 rows, I assumed that the real age is the number and I decided to keep the rows but update the table with their absolute values.


  ```sql
  UPDATE animal_intakes
  SET age_value = ABS(age_value) -- abs = absolute value
  WHERE age_value < 0;
```

#### Checking the null %
I used the formula below to calculate the % of nulls for each column. 
```sql
select 
(100.0 - count(column)*100.0/count(*)) as null_percentage
from animal_intakes
;
```


#### % of animal ids that appear more than once in the dataset
To find this percentage I need:
 
- the total number of rows with animal ids that appear more than once, and
- the total number of animal id (duplicates or not)
divide them * 100.

(total number of rows with animal ids that appear more than once / total number of animal id (duplicates or not) ) * 100

**the total number of rows with animal ids that appear more than once is:**

```sql
select sum(count)as total_duplicates
from 
(select
animal_id, count(*) as count
from animal_intakes
group by animal_id
having count(*) > 1
) as id_duplicates
;
```
**the total number of animal ids is:**

```sql
select count(animal_id)
from animal_intakes
;
```

So:
```sql
select (
select sum(count)as total_duplicates
from 
(select
animal_id, count(*) as count
from animal_intakes
group by animal_id
having count(*) > 1
) as id_duplicates)
/
(select count(animal_id)
from animal_intakes
	) *100 as duplicate_percentage
;
```
<<percentage photo>>

The percentage of 18.09% you calculated indicates that 18.09% of the total entries (or records) in the animal_intakes database represent instances where the animal_id appears more than once. This suggests that about 18% of the entries are for animals that have been admitted to the shelter more than once.

Having that in mind, I wanted to check the whole table to see if I can see any pattern or help me draw more questions so I run this query:
```sql
select a.*
from animal_intakes a
inner join(
select animal_id
from animal_intakes
group by animal_id
having count(*) >1
) as duplicates on a.animal_id = duplicates.animal_id
order by animal_id,datetime;
```

Here we can see that 29261 animals out of the 161712 (18.09%) return back to the shelter, some of them more than twice as well. 
Based on this, I want to investigate the following:
1. from those animals who come to the shelter what is the time in between they come back, so basically, what is the average and median time until the animal gets back in the shelter?
2. more dogs cats or what kind of animal returns most to the shelter?
3. what is the reason of return?

1. Average and Median Time Between Entries for Returning Animals
```sql
WITH TimeDifferences AS (
    SELECT 
        animal_id, 
        datetime,
        LAG(datetime) OVER (PARTITION BY animal_id ORDER BY datetime) AS previous_datetime,
        EXTRACT(DAY FROM (datetime - LAG(datetime) OVER (PARTITION BY animal_id ORDER BY datetime))) AS days_between_visits
    FROM 
        animal_intakes
)
SELECT 
    AVG(days_between_visits) AS average_days_between_visits,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY days_between_visits) AS median_days_between_visits
FROM 
    TimeDifferences
WHERE 
    days_between_visits IS NOT NULL;
```

**Insights from the Data**
- The large difference between the average and median suggests that the data is right-skewed, meaning there are a few animals with very long intervals between shelter visits that are increasing the average. These could be outliers or specific cases where animals were lost for a long time or had long periods of successful adoption before returning.
- The median being significantly lower than the average suggests that more than half of the animals return within about 97 days. 

**No of each animal type**

To count the actual number of dogs,cats etc in the shelter, considering the fact that there 29261 that has return to the shelter, we need to count the ones with distinct animal id to make sure we don't count twice the same animal. This method will give us a clearer picture of how many different animals we have dealt with.

```sql
select animal_type, count(distinct(animal_id)) as unique_animal_count
from animal_intakes
group by animal_type
order by unique_animal_count desc
;
```
Results:

|Animal Type | Unique_Animal_count|
-------------|--------------------|
|    Dog     |	      75664       |
|    Cat     |	      60226       |
|    Other   |	       8419       |
|    Bird    |	        812       |
|  Livestock |	         27       |

Sum = 145.148

2. Most Common Animal Type Among Returnees 

```sql
WITH Returnees AS (
    SELECT animal_id
    FROM animal_intakes
    GROUP BY animal_id
    HAVING COUNT(*) > 1
)
SELECT animal_type, COUNT(DISTINCT animal_id) AS returnee_count
FROM animal_intakes
WHERE animal_id IN (SELECT animal_id FROM Returnees)
GROUP BY animal_type
ORDER BY returnee_count DESC;	
```	
Results:

|Animal Type | Unique_Animal_count|
-------------|--------------------|
|    Dog     |	      10107       |
|    Cat     |	       2553       |
|    Other   |	         37       |	

Sum = 12.697 

That means that 8% (12697/145148 * 100) of the of the total animals entering the shelter (with unique ids) are returning.
Now we have 2 different percentages. 
- 8% focuses on the breadth of the issue across the animal population.
- 18% emphasizes the depth of the issue in terms of resource use and operational challenges


3. Frequency of different intake types among animals that have been admitted more than once:

```sql
SELECT intake_type, COUNT(*) AS count
FROM animal_intakes
WHERE animal_id IN (
	SELECT animal_id 
	FROM animal_intakes 
	GROUP BY animal_id 
	HAVING COUNT(*) > 1
)
GROUP BY intake_type
ORDER BY count DESC;
```
Results:

|Intake Type | Count              |
-------------|--------------------|
|    Stray   |	      15486       |
|  Owner Surrender   |	    10902 |
|  Public Assist   |	 2675     |
|    Abandoned   |  167           |
|  Euthanasia Request  |21       |
|  Wildlife   |	       10       |



