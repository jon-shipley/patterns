# MSSQL Performance & Diagnostics

## Table of contents
1. [Indexes](#Indexes)
2. [Deadlocks](#Deadlocks)

## Indexes

### Identify unused indexes

[Source](https://blog.sqlauthority.com/2011/01/04/sql-server-2008-unused-index-script-download/)

```tsql
-- Unused Index Script
-- Original Author: Pinal Dave 
SELECT TOP 25
o.name AS ObjectName
, i.name AS IndexName
, i.index_id AS IndexID
, dm_ius.user_seeks AS UserSeek
, dm_ius.user_scans AS UserScans
, dm_ius.user_lookups AS UserLookups
, dm_ius.user_updates AS UserUpdates
, p.TableRows
, 'DROP INDEX ' + QUOTENAME(i.name)
+ ' ON ' + QUOTENAME(s.name) + '.'
+ QUOTENAME(OBJECT_NAME(dm_ius.OBJECT_ID)) AS 'drop statement'
FROM sys.dm_db_index_usage_stats dm_ius
INNER JOIN sys.indexes i ON i.index_id = dm_ius.index_id 
AND dm_ius.OBJECT_ID = i.OBJECT_ID
INNER JOIN sys.objects o ON dm_ius.OBJECT_ID = o.OBJECT_ID
INNER JOIN sys.schemas s ON o.schema_id = s.schema_id
INNER JOIN (SELECT SUM(p.rows) TableRows, p.index_id, p.OBJECT_ID
FROM sys.partitions p GROUP BY p.index_id, p.OBJECT_ID) p
ON p.index_id = dm_ius.index_id AND dm_ius.OBJECT_ID = p.OBJECT_ID
WHERE OBJECTPROPERTY(dm_ius.OBJECT_ID,'IsUserTable') = 1
AND dm_ius.database_id = DB_ID()
AND i.type_desc = 'nonclustered'
AND i.is_primary_key = 0
AND i.is_unique_constraint = 0
ORDER BY (dm_ius.user_seeks + dm_ius.user_scans + dm_ius.user_lookups) ASC
GO
```

### Identify potential Missing Indexes

[Source](https://blog.sqlauthority.com/2011/01/03/sql-server-2008-missing-index-script-download/)

```tsql
-- Missing Index Script
-- Original Author: Pinal Dave 
SELECT TOP 25
dm_mid.database_id AS DatabaseID,
dm_migs.avg_user_impact*(dm_migs.user_seeks+dm_migs.user_scans) Avg_Estimated_Impact,
dm_migs.last_user_seek AS Last_User_Seek,
OBJECT_NAME(dm_mid.OBJECT_ID,dm_mid.database_id) AS [TableName],
'CREATE INDEX [IX_' + OBJECT_NAME(dm_mid.OBJECT_ID,dm_mid.database_id) + '_'
+ REPLACE(REPLACE(REPLACE(ISNULL(dm_mid.equality_columns,''),', ','_'),'[',''),']','') 
+ CASE
WHEN dm_mid.equality_columns IS NOT NULL
AND dm_mid.inequality_columns IS NOT NULL THEN '_'
ELSE ''
END
+ REPLACE(REPLACE(REPLACE(ISNULL(dm_mid.inequality_columns,''),', ','_'),'[',''),']','')
+ ']'
+ ' ON ' + dm_mid.statement
+ ' (' + ISNULL (dm_mid.equality_columns,'')
+ CASE WHEN dm_mid.equality_columns IS NOT NULL AND dm_mid.inequality_columns 
IS NOT NULL THEN ',' ELSE
'' END
+ ISNULL (dm_mid.inequality_columns, '')
+ ')'
+ ISNULL (' INCLUDE (' + dm_mid.included_columns + ')', '') AS Create_Statement
FROM sys.dm_db_missing_index_groups dm_mig
INNER JOIN sys.dm_db_missing_index_group_stats dm_migs
ON dm_migs.group_handle = dm_mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details dm_mid
ON dm_mig.index_handle = dm_mid.index_handle
WHERE dm_mid.database_ID = DB_ID()
ORDER BY Avg_Estimated_Impact DESC
GO
```

### Index Count Report for all tables

[Source](https://blog.sqlauthority.com/2012/10/09/sql-server-identify-numbers-of-non-clustered-index-on-tables-for-entire-database/)

```tsql
SELECT COUNT(i.TYPE) NoOfIndex,
[schema_name] = s.name, table_name = o.name
FROM sys.indexes i
INNER JOIN sys.objects o ON i.[object_id] = o.[object_id] INNER JOIN sys.schemas s ON o.[schema_id] = s.[schema_id] WHERE o.TYPE IN ('U')
AND i.TYPE = 2
GROUP BY s.name, o.name
ORDER BY schema_name, table_name
```

### List unindexed Foreign Keys

[Source](https://www.sqlshack.com/index-foreign-key-columns-sql-server/)

```tsql
CREATE TABLE #TempForeignKeys (TableName varchar(100), ForeignKeyName varchar(100) , ObjectID int)
INSERT INTO #TempForeignKeys 
SELECT OBJ.NAME, ForKey.NAME, ForKey .[object_id] 
FROM sys.foreign_keys ForKey
INNER JOIN sys.objects OBJ
ON OBJ.[object_id] = ForKey.[parent_object_id]
WHERE OBJ.is_ms_shipped = 0
 
CREATE TABLE #TempIndexedFK (ObjectID int)
INSERT INTO #TempIndexedFK  
SELECT ObjectID      
FROM sys.foreign_key_columns ForKeyCol
JOIN sys.index_columns IDXCol
ON ForKeyCol.parent_object_id = IDXCol.[object_id]
JOIN #TempForeignKeys FK
ON  ForKeyCol.constraint_object_id = FK.ObjectID
WHERE ForKeyCol.parent_column_id = IDXCol.column_id 
 
SELECT * FROM #TempForeignKeys WHERE ObjectID NOT IN (SELECT ObjectID FROM #TempIndexedFK)
 
DROP TABLE #TempForeignKeys
DROP TABLE #TempIndexedFK
```

## Deadlocks

### Identify deadlock events

```tsql
SELECT * FROM sys.event_log
WHERE event_type = 'deadlock';
WITH CTE AS (
SELECT CAST(event_data AS XML)  AS [target_data_XML]
FROM sys.fn_xe_telemetry_blob_target_read_file('dl',
null, null, null)
)
SELECT target_data_XML.value('(/event/@timestamp)[1]',
'DateTime2') AS Timestamp,
target_data_XML.query('/event/data[@name=''xml_report'']
/value/deadlock') AS deadlock_xml,
target_data_XML.query('/event/data[@name=''database_name'']
/value').value('(/value)[1]', 'nvarchar(100)') AS db_name
FROM CTE
```

##