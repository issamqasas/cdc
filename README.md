# Change Data Capture ( cdc)
FULL Example for loading data from source to destination  using LOB BASE CDC for on premise SQL Server  
to complete this article you have to be admin on the DB and not using SQL Express Edition and you should have SQL Server Agent Enabled.

## Definition of change Data Capture
Change Data Capture (CDC) is a process that allows detecting, recording, and tracking changes made to data in a Database.  
Transactional databases indeed store all changes in a transaction log. Change Data Capture (CDC) methods like Log-Based CDC utilize these transaction logs to capture insert, update, and delete operations as they occur.  
![image](https://github.com/user-attachments/assets/753324f8-64a9-48fd-81d0-fb97b6df9e7c)
## steps to acheive CDC
we will asume you have a testDB , and we will create a **user** table as source, and the destination will be a **tagetuser**
## **step 1: Check for CDC** 
to check if CDC is enabled on your DB , write the following sql statment
```sql
name,is_cdc_enabled from sys.databases
```
![image](https://github.com/user-attachments/assets/dc8acc35-41ae-4cbd-a366-b1ef3ff6fcbf)

## **step 2: Creating Table** 
```sql
-- Create a Users table 
CREATE TABLE Users 
(    
   ID int NOT NULL PRIMARY KEY,    
   FirstName varchar(30),    
   LastName varchar(30),    
   Email varchar(50) 
)
```
## **step 3: Enabling CDC** 
- First we need to Enable CDC on DB level
```sql
use TestDb
go
EXEC sys.sp_cdc_enable_db  
GO
```
![image](https://github.com/user-attachments/assets/c3912b66-844a-459c-b9ad-babda8c9e2c8)


- then We have to enable the CDC on the source table, which in our case **users**
```sql


-- Enable CDC for the table
EXEC sys.sp_cdc_enable_table
    @source_schema = N'dbo',
    @source_name = N'Users',
    @role_name = NULL;
GO
```

![image](https://github.com/user-attachments/assets/fc052167-81bf-4bf7-800f-ff1635078c6d)
you can see the new system tables ,
![image](https://github.com/user-attachments/assets/671a47c9-01af-448b-b9f8-7fc6b1e801db)

- To disable the table 
```sql
EXEC sys.sp_cdc_disable_table
    @source_schema = N'dbo',
    @source_name = N'Users',
	  @capture_instance = N'dbo_Users'
  
GO
```

- to check for all tables that have cdc enabled
```sql
-- Verify if the table is enabled for CDC
SELECT * FROM cdc.change_tables;
GO
```
![image](https://github.com/user-attachments/assets/8d48ec74-a2df-4120-8e9c-3ed14ee4ab73)

## **Step 4: Insert values Within The Table(s)**
```sql
INSERT INTO Users Values (7, 'ali', 'isa', 'isa@yahoo.com')
INSERT INTO Users Values (5, 'omar', 'othman', 'omar@yahoo.com')
INSERT INTO USERS Values (6, 'mohammad', 'sayed', 'mohammad@gmail.com')
```
- to cheeck if the insert data are captured
```sql
  select * from cdc.dbo_USERS_CT
```
![image](https://github.com/user-attachments/assets/8f385641-6235-45f5-8d08-58714bfe0c23)
- update and delete records
```sql
DELETE FROM Users WHERE ID = 1
UPDATE Users SET LastName = 'Snow' WHERE ID = 2
DELETE FROM Users WHERE ID = 3
```
and the Result is shown below  

![image](https://github.com/user-attachments/assets/65fbe9df-954f-483d-a732-c633d92da921)

- when inspecting the cdc table for the source user table for the last 4 records.
```sql
select * from cdc.dbo_USERS_CT 

```
![image](https://github.com/user-attachments/assets/84275ea1-fda3-417e-9b35-5afa75dbb601)  

The additional columns include

**__$start_lsn and __$end_lsn** that show the commit log sequence number (LSN) assigned by the SQL Server Engine to the recorded change
**__$seqval** that shows the order of that change related to other changes in the same transaction,  
**__$operation** that shows the operation type of the change, where 1 = delete, 2 = insert, 3 = update (before change), and 4 = update (after change)  
**__$update_mask** that is a bit mask defined for each captured column, identifying the updating columns  


## Notes:-
- if you issue an update statment , there will be two records captured with the same **__$start_lsn**
- if you inlcude Begin Tran and Commit , then all the records in CDC table will have the same **__$start_lsn**
```sql
begin tran
	DELETE FROM Users WHERE ID = 2
	INSERT INTO Users Values (2, 'aaaa', 'isa', 'isa@yahoo.com')
	UPDATE Users SET LastName = 'qasas' WHERE ID = 2

commit
```
![image](https://github.com/user-attachments/assets/de9b09be-a81e-40db-97a2-3cf8279fc00b)

- you can use the function **sys.fn_cdc_map_lsn_to_time** to convert the lsn to Dattime
```sql
SELECT
CONVERT(dateTime, sys.fn_cdc_map_lsn_to_time(__$start_lsn)) AS TableTimeStamp, *
FROM
    cdc.dbo_Users_CT 
```
![image](https://github.com/user-attachments/assets/20de07fb-77af-46e1-94b0-4731e62f00ea)

## **step 5: Loading the Data from Source table to Destination table**
i will devide this operation into two parts (FUll Load , incremental load )
the steps as follows
1. enable the CDC
2. run the Full_load script
3. run the incremental script
    
- create the following tables
  before inserting new data on users table , you have to make a full load for source users to targetusers , then enable the cdc ,
  we will assume no one is currently working on users table after the full load and before the enabling of CDC .
  
```sql
  
CREATE TABLE [dbo].CDC_Control (
	[ID] [bigint] IDENTITY(1,1) NOT NULL primary key (id),
	LastLSN  binary(10) NULL,
	[TableName] [varchar](100) NULL
	)
	go
	
CREATE TABLE [dbo].[TargetUsers](
	[ID] [int] NOT NULL primary key (id),
	[FirstName] [varchar](30) NULL,
	[LastName] [varchar](30) NULL,
	[Email] [varchar](50) NULL,
)
GO
```

### **Full Load**
```sql
USE [TestDB]
GO
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER procedure [dbo].[cdc_v3_Full_load] 
as
begin try

begin tran

declare @lsn binary(10) =
	(select max(LastLSN) from dbo.cdc_Control
	where tablename = 'users');
with incr_Users as
(
	select
		row_number() over
			(
				partition by ID
				order by
					__$start_lsn desc,
					__$seqval desc
			) as __$rn,
		*
	from cdc.dbo_users_CT
	where
		__$operation <> 3 and
		__$start_lsn > @lsn
)

--select * from incr_Users


, full_Users as
(
	select *
	from dbo.users
	--where id =-1
)
, cte_Users as
(
select
	ID, firstname,lastname,Email, __$operation
from incr_Users
where __$rn = 1
union all
select
	ID, FirstName,lastname,Email, 2 as __$operation
from full_Users
)
--select * from cte_Users


merge dbo.targetusers as trg using cte_Users as src on trg.ID=src.ID
when matched and __$operation = 1 then delete
when matched and __$operation <> 1 then update set trg.FirstName = src.firstname,trg.LAstName=src.LastName,trg.email=src.email
when not matched by target and __$operation <> 1 then insert (id,firstname, lastname,email) values (src.ID, src.firstname,src.lastname,src.email);

insert into dbo.cdc_control (LASTLSN,Tablename)
values(isnull((select max(__$start_lsn) from cdc.dbo_users_CT),0),'Users')

commit
end try
begin catch

Rollback

select ERROR_MESSAGE()

end catch
```
### **incremental Load**
```sql
USE [TestDB]
GO
/****** Object:  StoredProcedure [dbo].[cdc_v3_Incremental]    Script Date: 1/6/2025 3:48:38 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER procedure [dbo].[cdc_v3_Incremental] 
as
begin try

begin tran

declare @lsn binary(10) =
	(select max(LastLSN) from dbo.cdc_Control
	where tablename = 'users');
with incr_Users as
(
	select
		row_number() over
			(
				partition by ID
				order by
					__$start_lsn desc,
					__$seqval desc
			) as __$rn,
		*
	from cdc.dbo_users_CT
	where
		__$operation <> 3 and
		__$start_lsn > @lsn
)

--select * from incr_Users


, full_Users as
(
	select *
	from dbo.users
	where id =-1
)
, cte_Users as
(
select
	ID, firstname,lastname,Email, __$operation
from incr_Users
where __$rn = 1
union all
select
	ID, FirstName,lastname,Email, 2 as __$operation
from full_Users
)
--select * from cte_Users


merge dbo.targetusers as trg using cte_Users as src on trg.ID=src.ID
when matched and __$operation = 1 then delete
when matched and __$operation <> 1 then update set trg.FirstName = src.firstname,trg.LAstName=src.LastName,trg.email=src.email
when not matched by target and __$operation <> 1 then insert (id,firstname, lastname,email) values (src.ID, src.firstname,src.lastname,src.email);

insert into dbo.cdc_control (LASTLSN,Tablename)
values(isnull((select max(__$start_lsn) from cdc.dbo_users_CT),0),'Users')

commit
end try
begin catch

Rollback

select ERROR_MESSAGE()

end catch
```
### References :-
- https://codingsight.com/implementing-incremental-load-using-change-data-capture-sql-server/  
- https://estuary.dev/enable-sql-server-change-data-capture/
- https://towardsdev.com/change-data-capture-example-using-microsoft-sql-server-log-based-cdc-vs-trigger-based-cdc-a848b649e29a
- https://www.sqlshack.com/change-data-capture-for-auditing-sql-server/








