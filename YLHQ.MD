## Steps to import YLHQ data to the STOP_MAILING_LIST table

#### 1. Check if there is records in the table ylhq_import

``` sql
select * from analysis.ylhq_import;
```
 ##### If records are present there, delete them

``` sql
delete from analysis.ylhq_import;
```
#### 2. Import records from the file to the table ylhq_import

Note that we only need to import address_id and customer_id from the file to the table. 
This can be performed using DBeaver as it provides a feature to import data from .csv file to the target table.

#### 3. Check and validate the number of rows imported from the file to the table using the following query.
``` sql
select count(*) from analysis.ylhq_import;
```
#### 4. Check if there are any records in the ylhq_import with null value in the address_id field.
``` sql
select * from analysis.ylhq_import where address_id is null;
```
##### If such records are present, delete them from the ylhq_import table
``` sql
delete from analysis.ylhq_import where address_id is null;
```
#### 5. Add these records from ylhq_import table to the stop_mailing_list  table in the public schema using following query
``` sql
INSERT INTO public.stop_mailing_list (id, address_id, customer_id, created_date, modified_date, flag)
  (SELECT 	nextval('stop_list_seq'),
        	address_id         AS address_id,
        	customer_id        AS customer_id,
        	Now() :: timestamp AS created_date,
        	Now() :: timestamp AS updated_date,
        	'R'                AS flag
  FROM   analysis.ylhq_import); 
  ```
 #### 6. Note down the total rows inserted and validate it.
