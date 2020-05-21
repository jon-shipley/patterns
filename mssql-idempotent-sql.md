# MSSQL Patterns

## Table of contents
1. [Columns](#Columns)
2. [Constraints](#Constraints)
3. [Indexes](#Indexes)
4. [Tables](#Tables)
5. [Triggers](#Triggers)
5. [Procedures](#Procedures)

## Indexes

### Drop Index

```tsql
DROP INDEX IF EXISTS [schema].[table].[index_name];
```

### Create Index

```tsql
IF EXISTS(SELECT * FROM sys.indexes WHERE object_id = object_id('schema.table') AND NAME ='index_name')
BEGIN
    DROP INDEX index_name ON schema.table;
END
CREATE INDEX index_name ON schema.table([column_list]);
```

## Constraints

### Drop a default value

```tsql
    IF EXISTS (
            select *
              from sys.all_columns c
                   join sys.tables t on t.object_id = c.object_id
                   join sys.schemas s on s.schema_id = t.schema_id
                   join sys.default_constraints d on c.default_object_id = d.object_id
             where t.name = 'table'
               and c.name = 'columnName'
               and s.name = 'schema')
        BEGIN
            ALTER TABLE [schema].[table]
            DROP CONSTRAINT [DF_<name>];
        END
```

### Add a default value

```tsql
    IF NOT EXISTS (
            select *
              from sys.all_columns c
                   join sys.tables t on t.object_id = c.object_id
                   join sys.schemas s on s.schema_id = t.schema_id
                   join sys.default_constraints d on c.default_object_id = d.object_id
             where t.name = 'table'
               and c.name = 'columnName'
               and s.name = 'schema')
        ALTER TABLE [schema].[table]
        ADD CONSTRAINT [DF_<name>]
        DEFAULT <defaultValue> FOR [columnName];
```

### Drop a Foreign Key constraint

```tsql
IF EXISTS(SELECT *
            FROM INFORMATION_SCHEMA.CONSTRAINT_COLUMN_USAGE
           WHERE CONSTRAINT_COLUMN_USAGE.TABLE_SCHEMA = 'schema'
             AND CONSTRAINT_COLUMN_USAGE.TABLE_NAME = 'table'
             AND CONSTRAINT_COLUMN_USAGE.COLUMN_NAME = 'columnName'
             AND CONSTRAINT_NAME = 'FK_<name>')
    BEGIN
        ALTER TABLE [schema].[table]
            DROP CONSTRAINT [FK_<name>];
    END
```

### Add a Foreign Key Constraint 

```tsql
IF NOT EXISTS(SELECT *
                FROM INFORMATION_SCHEMA.CONSTRAINT_COLUMN_USAGE
               WHERE CONSTRAINT_COLUMN_USAGE.TABLE_SCHEMA = 'schema'
                 AND CONSTRAINT_COLUMN_USAGE.TABLE_NAME = 'table'
                 AND CONSTRAINT_COLUMN_USAGE.COLUMN_NAME = 'columnName'
                 AND CONSTRAINT_NAME = 'FK_<name>')
    BEGIN
        ALTER TABLE [schema].[table]
            ADD CONSTRAINT [FK_<name>]
                FOREIGN KEY (colName) REFERENCES [schema].[table] (columnName);
    END
```

## Columns
### Drop a column

```tsql
IF EXISTS(
        SELECT * FROM sys.columns
         WHERE object_ID=object_id('schema.table')
           AND col_name(object_ID,column_Id)='columnName'
    )
    BEGIN
        ALTER TABLE [schema].[table]
            DROP COLUMN [columnName];
    END;
```

Or, more simply

```tsql
 ALTER TABLE [schema].[table] DROP COLUMN IF EXISTS  [columnName], [columnName], ...;
```

### Add a column
```tsql
IF NOT EXISTS(
                SELECT *
                FROM sys.columns
               WHERE object_ID = object_id('schema.table')
                 AND col_name(object_ID, column_Id) = 'columnName')
    BEGIN
        ALTER TABLE [schema].[table]
            ADD [columnName] <type>;               
    END
```

## Tables
## Drop table (SQL Server 2019)
```tsql
DROP TABLE IF EXISTS [schema].[table];
```

# Create a table
```tsql
IF NOT EXISTS(SELECT *
                FROM INFORMATION_SCHEMA.TABLES
               WHERE TABLE_NAME = 'tableName'
                 AND TABLE_SCHEMA = 'schema')
BEGIN
    <table definition>
END 
```

## Triggers

You can find out if a trigger exists using something like:
```tsql
SELECT * FROM sys.triggers
WHERE [parent_id] = OBJECT_ID('schema.table')
AND [name] = 'triggerName';
```

Note: SQL Server 2019 - dropping a table automatically drops any associated triggers.

### Drop a Trigger

```tsql
DROP TRIGGER IF EXISTS [schema].[triggerName];
```

### Create a trigger

You probably want to make sure the table for the trigger exists and then ensure that the 
latest trigger definition is applied.  We have to use `EXEC` to run the `CREATE TRIGGER` command 
as otherwise we will get a syntax error.  It must be the first statement in the batch.  Using 
`CREATE OR ALTER TRIGGER` means any old definition will be updated with the latest. 

```tsql
IF EXISTS(SELECT *
            FROM INFORMATION_SCHEMA.TABLES
           WHERE TABLE_NAME = 'tableName'
             AND TABLE_SCHEMA = 'schema')
EXEC('CREATE OR ALTER TRIGGER [schema].[triggerName]
    ON [schema].[tableName]
    FOR UPDATE AS
BEGIN
    <trigger definition>
END')

```

## Procedures

### Drop a stored procedure

```tsql 
DROP PROCEDURE IF EXISTS [schema].<procedureName>;
```

### Create a stored procedure

```tsql 
CREATE OR REPLACE <procedureName>
```
