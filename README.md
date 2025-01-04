# Change Data Capture ( cdc)
hello world Example that uses LOB BASE CDC for on premise SQL Server  
to complete this article you have to be admin on the DB and not using SQL Express Edition
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




