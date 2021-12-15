## Steps to import DIRECT1 data to the STOP_MAILING_LIST table

#### 1. Check if there are records present in the load table to begin with


``` sql
select * from non_deliverable_address;

select * from non_deliverable_acs_address;

```
 ##### If records are present there, delete them

``` sql
delete from non_deliverable_address;

delete from non_deliverable_acs_address;
```
#### 2.  Load the file acs_undeliverables... into non_deliverable_acs_address 

#### 3. Insert the contents of this table into non_deliverable_address

``` sql
INSERT INTO non_deliverable_address
            (mailingaddress,
             mailingcity,
             mailingstate,
             mailingzip)
SELECT address1,
       LEFT(citystate, Length(citystate) - 8) city,
       LEFT(RIGHT(citystate, Strpos(Reverse(citystate), ' ' ) + 2), 2) state,
       RIGHT(citystate, 5) zip
FROM   non_deliverable_acs_address ;
```
#### 4. Update mailing address to match with address table data

``` sql
UPDATE non_deliverable_address
SET    mailingaddress = Upper(mailingaddress),
       mailingcity = Upper(mailingcity);
```

#### 5. Get address_id based on mailing address match
``` sql
update non_deliverable_address
set (county, tax_id, address_id, motivation, extra1, extra2, extra3)
  = (a.county, a.tax_id, a.id,'VM', null, null, null)    
  from address a
   WHERE a.mailing_zip =  mailingzip
 --    and a.mailing_street_name = mailingaddress
     and REPLACE(REPLACE(REPLACE(a.mailing_street_name, ' ', ''), ',', ''), 'POBOX', '') || COALESCE (a.mailing_unit_type, '') || COALESCE (a.mailing_unit_number, '')  
	     = REPLACE(REPLACE(REPLACE(mailingaddress, ' ', ''), ',', ''), 'POBOX', '')  
	 and address_id is null;

  ```
 #### 6. Run the script below  ** IMPORTANT Only run the script for undeliverable addresses, NOT vacant property **
 
``` sql
do $$
declare d int;
declare i int;
declare ttl_recs int;
declare dt varchar(12);
declare s_message varchar(255); 
begin
-- set heading
dt = current_date;
raise info ' *** Loading undeliverable address records from YLHQ for all counties on %. *** ',  dt;
--
ttl_recs = (select count(*) from non_deliverable_address );
raise info 'There are: % records in non_deliverable_address ', ttl_recs;
--
-- insure we have no records with empty address_id 
delete from  non_deliverable_address
  where address_id is  null;
get diagnostics d = row_count;
raise info 'deleted: % records from non_deliverable_address because there were empty address_ids ', d;
--
-- Insert the records into stop mailing list indicating undeliverable address
INSERT INTO stop_mailing_list 
        (id, address_id, created_date, customer_id, flag)
SELECT nextval('stop_list_seq'), address_id, current_date, 0, 'R'  -- "0" means all clients, not just one.
 FROM   non_deliverable_address t
 WHERE  t.address_id is not null
   and NOT EXISTS (SELECT 1 FROM stop_mailing_list s 
			WHERE s.address_id = t.address_id AND s.customer_id = 0);
get diagnostics i = row_count;
raise info 'inserted: % records into stop_mailing_list', i;
--
--
s_message = '';
/*
--  For undeliverable - Load the motivation points in for these addresses  (absentee)
update  motivation_points
  set absentee = 2,  -- We give 2 instead of 1 because they came from the mailing house
  absentee_expiry = (select current_date + interval '1 month' * absentee_expiry  
                        from motivation_points_param ),
  modified_date = current_date					
  where id in (select  motivation_points_id
from non_deliverable_address s, address a, motivation_points m
where a.id = s.address_id
  and a.motivation_points_id = m.id);
get diagnostics d =  row_count;  -- Grab
raise info 'updated: % motivation point records with absentee', d;
s_message =   (select 'Mot:' || d || ' abs. ');
--
--  Now insert the remaining into motivation_points_tmp and use APP to pull into motivation_points due to the foreign key issue
-- 
insert into motivation_points_tmp  (county, tax_id, absentee)
  select a.county, a.tax_id, 2  -- We give 2 instead of 1 because they came from the mailing house
  from non_deliverable_address s, address a
   where s.address_id is not null
     and a.id = s.address_id
     and a.motivation_points_id is null;  -- Only insert if no motivation points assigned to address yet
get diagnostics d =  row_count;  -- Grab
raise info 'inserted: % absentee records into motivation_point_tmp ', d;  	  
s_message =   (select s_message || ' Ins: '|| d || ' abs. ');
--
*/
-- report results
insert into processing_hist (parameter_1, parameter_2, id, task_name, processed_records, total_records, 
							 task_status, status_message, task_finish_date)
  select 'All',   'All' ,
  nextval('processing_ht_seq'), 'Load Undeliverable Records', i,   ttl_recs, 
  'Completed', 'Completed Successfully. ' || s_message, now();
--
get diagnostics d = row_count;
raise info ' *** Successfully loaded undeliverable address records and motivation points % *** ', d;
raise info ' *** If any records were inserted into motivation_point_tmp, be sure to load those records into MOTIVATION_POINTS table using the app: [ Load Motivation Points Data ] *** ';
--
end;
$$;

```
 #### 7. If any records were inserted into motivation_point_tmp, be sure to load records into MOTIVATION_POINTS table using the app:( https://investor-machine-front.herokuapp.com )and "Load Motivation Points Data"

 #### 8. Once everything is done above, delete the old rows from our temp tables
``` sql
delete from non_deliverable_address;

```

