# Oracle Important Info

## Anonymous Block - Random Data Generator # 01

```
BEGIN
	FOR a IN 900000001..900009999 LOOP            
        INSERT INTO balance (account_id, bank_balance, created_timestamp, updated_timestamp) 
        VALUES (a, ROUND(dbms_random.value(101, 999),0), trunc(systimestamp), trunc(systimestamp));
        
 	END loop;
	COMMIT; 
END;
/
```

## Anonymous Block - Random Data Generator # 02

```
DECLARE  
	CURSOR c1 IS SELECT * FROM balance;
BEGIN  
	FOR row IN c1  LOOP    
		FOR a IN 10..120000 LOOP            
			INSERT INTO balance (account_id, bank_balance, created_timestamp, updated_timestamp)
			VALUES ((row.ban + a), ROUND(dbms_random.value(11, 99),0),trunc(systimestamp),trunc(systimestamp));     
		END loop;   
	END LOOP;  
	COMMIT; 
END;
```

## Anonymous Block - Random Data Generator # 03

```
DECLARE
  -- non member
  v_non_mem_01_min number := 1000001;
  v_non_mem_01_max number := 1999999;
  v_non_mem_01_past_payments number := 3;

  
	-- member
  v_mem_01_min number := 2000001;
  v_mem_01_max number := 2999999;
  v_mem_01_past_payments number:= 7;
  
	
	-- 
  v_func_return boolean := false;

  --  data type
  v_only_non_member_data number := 1;
  v_only_member_data number := 2;
  v_non_member_plus_member_data number := 3;
  v_add_data_ind number := v_non_member_plus_member_data;
	
	--
  v_sql VARCHAR2(4000);

  ------------------------------------------------------------
  --- non member function
  ------------------------------------------------------------
  function non_mem_data_func
  return boolean is
  begin
    --dbms_output.put_line('non_mem_data_func() => start');
    -- 01: add data non member
    Insert /*+ append parallel(6) */
    into non_member (account_id, non_member_id, non_member_name, service_start_date, CREATED_TIMESTAMP, CREATED_BY, UPDATED_TIMESTAMP, UPDATED_BY)
    select  account_id, non_member_id, non_member_name, service_start_date, CREATED_TIMESTAMP, CREATED_BY, UPDATED_TIMESTAMP, UPDATED_BY
    from (
    				select 	account_id, 
										round(dbms_random.value(100000001, 999999999),0) as non_member_id,
    								('NON-MEM-NM' || '-' || upper(dbms_random.string('l', 35))) as non_member_name,
										SERVICE_START_DATE,
    								systimestamp as CREATED_TIMESTAMP,
    								'auto' as CREATED_BY,
    								systimestamp as UPDATED_TIMESTAMP,
    								'auto' as UPDATED_BY
    				from tmp_perf_data
    				where non_member_id between v_non_mem_01_min and v_non_mem_01_max
    				and is_member = 'no'    				
    			);
    commit;
    --dbms_output.put_line('non_mem_data_func() => end');
    return true;
  end non_mem_data_func;

  ------------------------------------------------------------
  --- member function
  ------------------------------------------------------------
  function mem_data_func
  return boolean is
  begin
    --dbms_output.put_line('mem_data_func() => start');
    -- 01 - add data in member table
    Insert /*+ append parallel(6) */
    into non_member (account_id, non_member_id, non_member_name, service_start_date, CREATED_TIMESTAMP, CREATED_BY, UPDATED_TIMESTAMP, UPDATED_BY)
    select  account_id, non_member_id, non_member_name, service_start_date, CREATED_TIMESTAMP, CREATED_BY, UPDATED_TIMESTAMP, UPDATED_BY
    from (
    				select 	account_id, 
										round(dbms_random.value(1000000001, 9999999999),0) as member_id,
    								('MEM-NM' || '-' || upper(dbms_random.string('l', 35))) as member_name,
										SERVICE_START_DATE,
    								systimestamp as CREATED_TIMESTAMP,
    								'auto' as CREATED_BY,
    								systimestamp as UPDATED_TIMESTAMP,
    								'auto' as UPDATED_BY
    				from tmp_perf_data
    				where member_id between v_mem_01_min and v_mem_01_max
    				and is_member = 'yes'
    		);
    commit;
    --dbms_output.put_line('mem_data_func() => end');
    return true;
  end mem_data_func;
  

BEGIN
  -- 01 : truncate tables
  EXECUTE IMMEDIATE 'truncate table member';
  EXECUTE IMMEDIATE 'truncate table non_member';
  
  -- 04 : add data in temp table
  Insert /*+ append parallel(6) */
  into   tmp_perf_data (account_id, SERVICE_START_DATE, is_member)
  select a.account_id, a.SERVICE_START_DATE,  a.is_member
  from (
          -- non member
          SELECT  level, (101000000+ level) as account_id,
                  TO_TIMESTAMP(ADD_MONTHS(sysdate,-2),'yyyy-mm-dd hh24:mi:ss') as SERVICE_START_DATE,
                  'no' as is_member
          from dual connect by Level <= 999999
          union all
          -- member
          SELECT  level, (201000000+ level) as account_id,
                  TO_TIMESTAMP(ADD_MONTHS(sysdate,-4),'yyyy-mm-dd hh24:mi:ss') as SERVICE_START_DATE,
                  'yes' as is_member
          from dual connect by Level <= 999999
          
       ) a;
  commit;

  -- 05 - add member data
  --dbms_output.put_line('v_add_data_ind = '|| v_add_data_ind);
  IF ( v_add_data_ind = 1 ) THEN
    v_func_return := non_mem_data_func();
  ELSIF ( v_add_data_ind = 2 ) THEN
    v_func_return := mem_data_func();
  ELSIF ( v_add_data_ind = 3 ) THEN
    v_func_return := mem_data_func();
    v_func_return := non_mem_data_func();
  END IF;
	
	Insert /*+ append parallel(6) */
	into past_payments (account_id, payment_date)
	select  *
	from    (
						SELECT a.account_id, b.payment_date
						FROM tmp_account_ids a
						CROSS JOIN (
													select ADD_MONTHS(trunc(sysdate),-4) as payment_date from dual
													union all
													select ADD_MONTHS(trunc(sysdate),-3) as payment_date from dual
													union all
													select ADD_MONTHS(trunc(sysdate),-2) as payment_date from dual
													union all
													select ADD_MONTHS(trunc(sysdate),-1) as payment_date from dual
													union all
													select trunc(sysdate) as payment_date from dual
											 ) b
						where a.account_id between v_non_mem_01_min and v_non_mem_01_max
						union all
						SELECT a.account_id, b.payment_date
						FROM tmp_account_ids a
						CROSS JOIN (
													select ADD_MONTHS(trunc(sysdate),-6) as payment_date from dual
													union all
													select ADD_MONTHS(trunc(sysdate),-5) as payment_date from dual
													union all
													select ADD_MONTHS(trunc(sysdate),-4) as payment_date from dual
													union all
													select ADD_MONTHS(trunc(sysdate),-3) as payment_date from dual
													union all
													select ADD_MONTHS(trunc(sysdate),-2) as payment_date from dual
													union all
													select ADD_MONTHS(trunc(sysdate),-1) as payment_date from dual
													union all
													select trunc(sysdate) as payment_date from dual
											 ) b
						where a.account_id between v_mem_01_min and v_mem_01_min
				)
	order by account_id, payment_date;
	commit;
  
END;
/

```

## Previous Record data in current row

```
select  account_id, payment_month, balance, 
        LAG(tier_end) OVER (PARTITION BY account_id ORDER BY payment_month) prev_month_balance,
        row_number() over(partition by account_id order by payment_month) as row_nbr
from payment
```

## Split table into chunks

```
select a.account_id as account_id, ntile(5) over (order by account_id) as bucket_nbr
from (select * from balance where rownum <= 150000 order by account_id) a;
```

## Bulk Random Data Generator

- Generates '999999' rows of data

```
SELECT  level, (101+ level) as account_id,
      TO_TIMESTAMP(ADD_MONTHS(sysdate,-2),'yyyy-mm-dd hh24:mi:ss') as START_DATE,
      TO_TIMESTAMP(ADD_MONTHS(sysdate,12),'yyyy-mm-dd hh24:mi:ss') as END_DATE
from dual 
connect by Level <= 999999
```

## Single Field Random Data Generator

```
select  translate(dbms_random.string('a', 20), 'abcXYZ', '158249') as name_01,
        ('A1' || '-' || upper(translate(dbms_random.string('l', 16), 'abcdefghijkln', '0123456789opqrstuvwxyz01234567899')))  as name_02,
        REGEXP_REPLACE(SYS_GUID(), '(.{8})(.{4})(.{4})(.{4})(.{12})', '\1-\2-\3-\4-\5') GUID,
        round(dbms_random.value(11, 30),0) as AMOUNT
from dual;
```

## Bulk Insert using Random Data Generator

- Inserts '999999' rows of data

```
Insert /*+ append parallel(8) */
into balance (account_id, bank_balance, created_timestamp, updated_timestamp)
select account_id, bank_balance, created_timestamp, updated_timestamp
from   (
					SELECT  level, (101+ level) as account_id,
									ROUND(dbms_random.value(11, 99),0) as bank_balance,
									trunc(systimestamp) as created_timestamp,
									trunc(systimestamp) as updated_timestamp
					from dual 
					connect by Level <= 999999
					union
					SELECT  level, (201+ level) as account_id,
									ROUND(dbms_random.value(11, 99),0) as bank_balance,
									trunc(systimestamp) as created_timestamp,
									trunc(systimestamp) as updated_timestamp
					from dual 
					connect by Level <= 999999
			 );
```

## Convert date to hh:mm:ss

```
with q1 as (
             -- seconds
             select round(24 * 60 * 60* (END_TIME - start_time)) as total_time
             from execution_log
             where process_name = 'build_car'
           )
SELECT total_time, 
    (TO_CHAR(TRUNC(total_time/3600),'FM9900') || ':' ||
    TO_CHAR(TRUNC(MOD(total_time,3600)/60),'FM00') || ':' ||
    TO_CHAR(MOD(total_time,60),'FM00')) as total_time
FROM q1;

```

## Convert date to seconds

```
select round(24 * 60 * 60* (END_TIME - start_time)) as total_time
from execution_log
where process_name = 'build_car' 
and row_count > 0;

```

## Convert List 

```
select column_value as account_nbr, 0 as amount 
from (
		select *  
		from table(sys.odcinumberlist(12001, 12002, 12003, 	12004, 12005))
 )
```
## Timestamp 2 Milliseconds

```
SELECT
     EXTRACT(DAY FROM(sys_extract_utc(systimestamp) - to_timestamp('1970-01-01', 'YYYY-MM-DD'))) * 86400000
    + to_number(TO_CHAR(sys_extract_utc(systimestamp), 'SSSSSFF3')) as ms
  FROM dual;
```

## Timestamps

```
-- substract 1 year
SELECT systimestamp - NUMTOYMINTERVAL(1, 'year') FROM dual;

-- substract 5 days
SELECT (systimestamp - interval '5' day)
FROM dual;

-- 
select membership_start_date, service_start_date,
       case
            when membership_start_date <= service_start_date then 'true'
            else 'false'
       end as flag
from  (
select add_months(sysdate, -1) as membership_start_date, trunc(sysdate) as service_start_date 
from dual
union 
select add_months(sysdate, 1) as membership_start_date, trunc(sysdate) as service_start_date
from dual);
```

## Lock Table Verification

```
- Verify locked session
select 	c.owner, c.object_name, c.object_type,
				b.sid, b.serial#, b.status, b.osuser,
				b.machine
fromv $locked_object a, v$session b, dba_objects c
whereb.sid = a.session_id
anda.object_id = c.object_id;

-- kill session
alter system kill session '1526, 45866';
```

## Merge two tables without common columns

```
SELECT q1.account_id, q2.log_nbr
FROM   (SELECT   account_id, ROW_NUMBER() OVER (ORDER BY account_id) AS rn
        FROM     balance) q1
JOIN   (SELECT   log_nbr, ROW_NUMBER() OVER (ORDER BY log_nbr) AS rn
        FROM     execution_log) q2 
ON q1.rn = q2.rn
```

## Delete from Select

```
DELETE FROM execution_log a WHERE EXISTS (SELECT 0 FROM execution_log_tmp b WHERE a.exec_id = b.exec_id);        
```


