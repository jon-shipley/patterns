# MSSQL Patterns

## Table of contents
1. [Indexes](#Indexes)
2. [Constraints](#Constraints)
3. [Columns](#Columns)

## Indexes

### Drop Index

```
DROP INDEX IF EXISTS [schema].[table].[index_name];
```

### Create Index

```
IF NOT EXISTS (SELECT * FROM sys.indexes i
               WHERE i.object_ID=object_id('schema.table')
                 AND name ='idx_name')
    BEGIN
        --then the index doesn’t exist
        CREATE INDEX idx_azure_recommended_pupil_school
            ON [schema].[table] (colName);
    END;
```

## Constraints

### Drop a default value

```
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

```
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

```
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

```
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

```
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

### Add a column
```
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