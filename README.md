# Change Data Capture ( cdc)
hello world Example that uses LOB BASE CDC for on premise SQL Server  
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
- create the following tables
```sql
  
CREATE TABLE [dbo].CDC_Control (
	[ID] [int] NOT NULL primary key (id),
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
- to start the loading of data , we have to start with the first **__$start_lsn**
![image](https://github.com/user-attachments/assets/54138cb7-5627-407b-b738-4f66d644d74d)


### **get Insert /update / delete**
```sql

--Author :- issam qasas
--date written 5-jan-2025

alter procedure cdc
as 
DECLARE @last_lsn BINARY(10);
	

-- Retrieve the last processed LSN from the control table
SELECT @last_lsn = LastLSN
FROM CDC_Control
WHERE TableName = 'users';
--print @last_lsn

if @last_lsn is null
begin
	begin tran
	
	select @last_lsn = min(__$start_lsn)	
	FROM cdc.dbo_Users_CT

--	if @last_lsn is not null
--	begin	
--		insert into dbo.cdc_Control(id,lastLSN,tablename)values(1,@last_lsn,'Users')
--	end
--print @last_lsn
	
------------- Insert Operation --------------------
	;WITH LatestInserts AS (
		SELECT
		   id, FirstName, LastName, Email, ROW_NUMBER() OVER (PARTITION BY Id ORDER BY __$seqval desc ) AS seq_rank
		FROM cdc.dbo_users_ct
		WHERE __$operation = 2  --insert operation
		AND __$start_lsn >= @last_lsn  -- first time >= , then > only
	) 
	INSERT INTO dbo.TargetUsers (id, FirstName, LastName, Email)
	select id, FirstName, LastName, Email from LatestInserts where seq_rank=1
	---------------------------------------------------
	-----------update operation -----------------------


	;WITH LatestUpdates AS (
		SELECT
	*,        ROW_NUMBER() OVER (PARTITION BY ID ORDER BY __$seqval DESC) AS seq_rank
		FROM cdc.dbo_users_ct
		WHERE __$operation = 4  -- Update operation
		AND __$start_lsn >= @Last_LSN  --in the begining , then >
	)  

	UPDATE tu
	SET 
		tu.FirstName = lu.FirstName,
		tu.LastName = lu.LastName,
		tu.Email = lu.Email
    
	FROM dbo.TargetUsers tu
	JOIN LatestUpdates lu
		ON tu.ID = lu.Id
	WHERE lu.seq_rank = 1;  -- Only the latest version

	-------------------DElete operation-------------------------------
	;WITH LatestDelete AS (
		SELECT *,ROW_NUMBER() OVER (PARTITION BY Id ORDER BY __$seqval desc ) AS seq_rank
       
		 FROM cdc.dbo_users_ct
		WHERE __$operation in (1,2)  --Delete , insert  operation
		AND __$start_lsn >= @last_lsn  -- first time >= , then > only
	) 

	delete tu
	from dbo.targetusers tu inner join LatestDelete Ld on Ld.id=tu.id  where ld.seq_rank=1 and ld.__$operation=1

	----------------------------------
	
		SELECT @last_lsn=  MAX(__$start_lsn) FROM cdc.dbo_users_ct
	if @last_lsn is not null
	begin
		insert into dbo.cdc_Control(id,lastLSN,tablename)values(1,@last_lsn,'Users')
	end
	COMMIT TRANSACTION;
end
else
begin
	begin tran
	print '> then' 
	print @last_lsn
------------- Insert Operation --------------------
	;WITH LatestInserts AS (
		SELECT
		   id, FirstName, LastName, Email, ROW_NUMBER() OVER (PARTITION BY Id ORDER BY __$seqval desc ) AS seq_rank
		FROM cdc.dbo_users_ct
		WHERE __$operation = 2  --insert operation
		AND __$start_lsn > @last_lsn  -- first time >= , then > only
	) 
	INSERT INTO dbo.TargetUsers (id, FirstName, LastName, Email)
	select id, FirstName, LastName, Email from LatestInserts where seq_rank=1
	---------------------------------------------------
	-----------update operation -----------------------


	;WITH LatestUpdates AS (
		SELECT
	*,        ROW_NUMBER() OVER (PARTITION BY ID ORDER BY __$seqval DESC) AS seq_rank
		FROM cdc.dbo_users_ct
		WHERE __$operation = 4  -- Update operation
		AND __$start_lsn > @Last_LSN  --in the begining , then >
	)  

	UPDATE tu
	SET 
		tu.FirstName = lu.FirstName,
		tu.LastName = lu.LastName,
		tu.Email = lu.Email
    
	FROM dbo.TargetUsers tu
	JOIN LatestUpdates lu
		ON tu.ID = lu.Id
	WHERE lu.seq_rank = 1;  -- Only the latest version

	-------------------DElete operation-------------------------------
	;WITH LatestDelete AS (
		SELECT *,ROW_NUMBER() OVER (PARTITION BY Id ORDER BY __$seqval desc ) AS seq_rank
       
		 FROM cdc.dbo_users_ct
		WHERE __$operation in (1,2)  --Delete , insert  operation
		AND __$start_lsn > @last_lsn  -- first time >= , then > only
	) 

	delete tu
	from dbo.targetusers tu inner join LatestDelete Ld on Ld.id=tu.id  where ld.seq_rank=1 and ld.__$operation=1

	----------------------------------
	update dbo.cdc_Control set lastLSN=@last_lsn where tablename='Users'

	COMMIT TRANSACTION;

end

```






