# Austin Animal Center Data

## Database Design

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

To populate the age_value and age_unit we need to split the age_upon_intake. Since age_upon_intake is a text and age_value is stored as smallint, I need to cast the split_part as integer.

```sql
UPDATE animal_intakes
SET age_value = CAST(SPLIT_PART(age_upon_intake, ' ', 1) AS INTEGER),
    age_unit = SPLIT_PART(age_upon_intake, ' ', 2);
```
#### Handling Data Anomalies: Negative Age Values

After populating the age_value and age_uni, I selected with distinct the columns and I identified records in the `animal_intakes` table with negative age values. Given that negative ages are not feasible and because there are only 13 rows, I assumed that the real age is the number and I decided to keep the rows but update the table with their absolute values.


  ```sql
  UPDATE animal_intakes
  SET age_value = ABS(age_value) -- abs = absolute value
  WHERE age_value < 0;
```

#### Checking the null %
I used the below formula to calculate the % of nulls for each column. 
```sql
select 
(100.0 - count(column)*100.0/count(*)) as null_percentage
from animal_intakes
;
```
