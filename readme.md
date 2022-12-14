# To deidentify an OMOP dataset you need a few items to take place. 

1.	Any source field needs to be zeroed out
2.	All dates associated with a person must be skewed
3.	Any zip code needs to be mapped to the 3 digit equivalent
4.	If your primary key's are a direct copy from your EHR you need to create a crosswalk and overwrite each


## 1. Nulling Fields, SQL Statement, See Excel sheet for columns to null for the other tables I am providing examples for the first three in the sheet.

```bash
UPDATE person
SET person_source_value=NULL,
gender_source_value=NULL,
gender_source_concept_id=NULL,
race_source_value=NULL,
race_source_concept_id=NULL,
ethnicity_source_value=NULL,
ethnicity_source_concept_id=NULL
WHERE person_id is not null;

UPDATE visit_occurrence
SET visit_source_value=NULL,
visit_source_concept_id=NULL,
admitted_from_source_value=NULL,
discharged_to_source_value=NULL
WHERE visit_occurrence_id is not null;

UPDATE visit_detail
visit_detail_source_value=NULL,
visit_detail_source_concept_id=NULL,
admitted_from_source_value=NULL,
discharged_to_source_value=NULL,
WHERE visit_detail_id is not null;
```

## 2. Skewing Dates:

Step 1 Run a SQL query (program) creating a crosswalk table of Person ID and Random Integer, example below, I’d prefer to use a more robust random number generator than this but if I’m limiting myself to SQL, it seems the best option I found to generate a random number 60 and lower. This represents 60 days, we are going to subtract this integer from all dates to generate our new dates. Note You need to store these offsets to be referenced, I’m putting them into crosswalk_date_skews.

```bash
SELECT person_id, 1.0 + floor(60 * RAND(convert(varbinary, newid()))) as random_date_offset
INTO OMOP.Crosswalk_date_skews
FROM OMOP.person
```

### Step 2

Run the following SQL statement for each date column

With regular fields you just need to run the following

```bash
UPDATE person
SET birth_datetime = (
SELECT DATEADD(day, (sf.random_date_offset*-1), pr.birth_datetime) AS DateAdd;
FROM person as pr
LEFT JOIN Crosswalk_date_skews on pr.person_id= Crosswalk_date_skews.person_id as sf
WHERE pr.person_id = person.id
);
```

The person table is the worst as we split apart the year, month, day of birth. Step one you have to concatenate them all together, step two you subtract the skewed date, then you update individual components. 

```bash
UPDATE person
SET year_of_birth = (
SELECT other_col
FROM person
WHERE person. person _id = person.id
);
```

Another example table where you update a skew column

```bash
UPDATE visit_occurrence
SET visit_start_date = (
SELECT DATEADD(day, (sf.random_date_offset*-1), pr. visit_start_date) AS DateAdd;
FROM visit_occurrence as pr
LEFT JOIN Crosswalk_date_skews on pr.person_id= Crosswalk_date_skews.person_id as sf
WHERE pr.person_id = person.id
);

UPDATE visit_occurrence
SET visit_start_datetime = (
SELECT DATEADD(day, (sf.random_date_offset*-1), pr. visit_start_datetime) AS DateAdd;
FROM visit_occurrence as pr
LEFT JOIN Crosswalk_date_skews on pr.person_id= Crosswalk_date_skews.person_id as sf
WHERE pr.person_id = person.id
);
```



## 3. Deidentifying location ID

### Step 1. Locations come with 3, 5 or 9 digit zip codes in OMOP. 
Each one gets assigned a location_id. You cannot just concatenate the location ID to the first three for 5 and 9 because you can still successfully guess based on population size the original zip codes. Instead we do something similar to the skewed dates. Step 1 we build a table to contain a crosswalk. The original ID’s and the location_id for the three digit zip

```bash
SELECT location_id as source_location_id, location_id as target_location_id
INTO OMOP.LOCATION_Crosswalk
FROM OMOP.LOCATION as fo
LEFT JOIN LOCATION as so on SUBSTRING(fo.location_id, 1, 3)=so.zip
```


### Step 2 update the original location_id’s with the crosswalk where it maps to the three digit zip

```bash
UPDATE person
SET location_id = (
SELECT lc.target_location_id
FROM PERSON as pr
LEFT JOIN LOCATION_Crosswalk as lc on pr.location_id=lc.source_location_id
WHERE pr.location_id = source_location_id
);


UPDATE person
SET location_id = (
SELECT lc.target_location_id
FROM CARE_SITE as pr
LEFT JOIN LOCATION_Crosswalk as lc on pr.location_id=lc.source_location_id
WHERE pr.location_id = source_location_id
);
```

## 4.  If your primary key's are a direct copy from your EHR you need to create a crosswalk and overwrite each


I highly recommend finding a smarter way to compute random numbers. Use referential integrity on both columns to ensure you don't get a duplicate ID number

```bash
SELECT person_id, 1.0 + floor(9223372036854775807 * RAND(convert(varbinary, newid()))) as random_generated_person_id
INTO OMOP.Crosswalk_person_id
FROM OMOP.person

SELECT visit_occurrence_id, 1.0 + floor(9223372036854775807 * RAND(convert(varbinary, newid()))) as random_generated_visit_id
INTO OMOP.Crosswalk_visit_occurrence
FROM OMOP.visit_occurrence
```

# OMOP has this to say about considerations on deidentifying the dataset https://ohdsi.github.io/CommonDataModel/cdmPrivacy.html
