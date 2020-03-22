Title: Deleting large amounts of data from MS SQL Server database
Description: An alternative approach for deleting data and script generation
Published: 27/2/2020
Tags: [SQL, PowerShell, SMO]
---
<!--- # Deleting large amounts of data from MS SQL Server database --->

Deleting large amount of rows (like millions of them) from a table has a downside of being slow and making a transaction log file to explode in terms of size. Here is an approach that worked for me overcome these obstacles.

## Background

My problem was that in a multi-tenant database a tenant has grown too much - to the point that it had to be moved to it's own database. The plan was to restore a backup of the database and delete all other tenant's data. The database had around 150 tables - for most of them deletions were not a problem - but for around 20 of them more than 10 million rows were to be deleted and for particular 5 tables more than 600 million.

## First iteration: Standard deletions

For the first version of the deletion I used standard delete statements similar to:  

``` sql
DELETE x FROM a_table x JOIN @TenantsToDelete t ON t.Id=x.TenantId; 
```  
It worked fine on a small data set and was useful to confirm the deletion order and flush out some bad data and design oddities. Also it has no problem for few thousand to even tens of thousands of rows so it worked fine for the majority of the cases.  
Where millions of rows were to be deleted it was taking long time and transaction log was growing rapidly. The full deletion script actually never ran to completion on production-grade data.  

## Second iteration: Batched deletions  

For the large tables I tried batched deletes similar to:  

```sql
DECLARE @batch_size INT=5000;
delete_top:
  DELETE TOP(@batch_size) x FROM a_table x JOIN @TenantsToDelete t ON t.Id=x.TenantId;
  IF @batch_size = @@ROWCOUNT GOTO delete_top;
```  

This version of the script was able to complete deletion of a single small-sized tenant for around 30 minutes, for a large one it took more than 15 hours - and I needed to remove around 40 of them. It was still no good.  

## Third iteration: Replace DELETE with INSERT

So `DELETE` is slow and there is no much room for improvement. There is no _"bulk delete"_ kind of operation in MS SQL Server to boost the performance and I was not in a position to use `TRUNCATE` or `DROP TABLE` since I needed to preserve part of the data.  

While searching for solution I found this article about optimizing loading of data and the impact of minimally logged operations on I/O: [The Data Loading Performance Guide](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008/dd425070(v=sql.100)?redirectedfrom=MSDN)  

In essence we can insert data fast (with minimal logging) when these conditions are met:
 - database is in `SIMPLE` or `BULK_LOGGED` recovery mode,
 - target table is a _heap table_ (without clustered index),
 - a `WITH (TABLOCK)` hint is used with the insert (allows exclusive lock on the target table).

There is additional performance boost using the latter in SQL Server 2016 and higher - `INSERT...SELECT WITH (TABLOCK)` may use parallel inserts.  


So back to my deletion problem - what if instead of delete I do the following:
1. Create a heap table with exactly the same structure as the one I want to delete from
2. Move to the heap table data I want to **keep** using `INSERT...SELECT WITH (TABLOCK)`
3. Drop the original table (metadata-only operation, fast)
4. Rename the heap table to original one's name
5. Create indexes, constraints, references etc.
   
_**TL;DR;**_ Yes, it worked and it was way faster without growing the transaction log since all operations were minimally logged.  
Replacing deletion of my around 30 problematic tables with this technique lead to deletion of everything but a mid-sized tenant data to complete in 15 minutes.  
Deletion of everything but the largest tenant (moving the most data) completed in 1 hour.  

### Generating the SQL  

Although I was quite happy with the performance, there was another problem - scripting the heap tables, moving the data and re-creating the indexes by hand is tedious and error prone (imagine doing it for 30 tables).  

SQL Server Management Studio has a lot of scripting capabilities and fortunately they are available via the [SQL Management Objects](https://docs.microsoft.com/en-us/sql/relational-databases/server-management-objects-smo/sql-server-management-objects-smo-programming-guide?view=sql-server-ver15) (SMO). It is a set of .NET Framework assemblies meaning that they can be used from PowerShell also.  

Loading the assemblies and creating `Microsoft.SqlServer.Management.Smo.Server` instance is the first step:  

```powershell
[System.Reflection.Assembly]::LoadWithPartialName("Microsoft.SqlServer.Smo")
[System.Reflection.Assembly]::LoadWithPartialName("Microsoft.SqlServer.ConnectionInfo")

$connection = New-Object System.Data.SqlClient.SqlConnection $sqlConnectionString
$serverConnection = New-Object Microsoft.SqlServer.Management.Common.ServerConnection $connection

[Microsoft.SqlServer.Management.Smo.Server] $server = New-Object Microsoft.SqlServer.Management.Smo.Server $serverConnection
```

To create a heap table:

```powershell
$scripter = New-Object Microsoft.SqlServer.Management.Smo.Scripter $server

$script = $scripter.Script(($server.Databases[$connection.Database].Tables[$table]))

# scripter can only generate scripts for existing objects so we need to change the table name 
$script[2] = $script[2].Replace("CREATE TABLE [dbo].[$table]", "CREATE TABLE [dbo].[$heapTable]")
$script
```

For moving the data, we can generate the columns list to use in the insert statement:

```powershell
$columnNames = $server.Databases[$connection.Database].Tables[$table].Columns | Select-Object $_ | ForEach-Object {$_.Name}
$columns = [System.String]::Join(', ', $columnNames)

$sql = New-Object System.Collections.Specialized.StringCollection
$sql.Add("INSERT INTO $heapTable WITH(TABLOCK)") > $null
$sql.Add("($columns)") > $null
$sql.Add("SELECT") > $null
$sql.Add("$columns FROM $table ") > $null
$sql.Add("$whereClause") > $null

$sql
```

There are some pitfalls to consider:  
- beware of `TIMESTAMP` columns: you cannot insert them
- beware of `IDENTITY` columns: I found it hard to change column to identity so create it as an identity and use `SET IDENTITY_INSERT ON/OFF` 
- again for `IDENTITY` columns: the scripter will capture the current identity seed when generating the script but it will be executed at some later point and the seed will be different. Thus in my script I capture the seed from the original table column after the data is moved and reseed the column of the heap table. To find the identity column you can use something like:
```powershell
$server.Databases[$connection.Database].Tables[$table].Columns | Where-Object { $_.Identity -eq $true } | Select-Object -First 1
```
- depending on your `$whereClause` you may need to prefix the columns in the select list, e.g. when using something like `SELECT x.ID FROM x_Table x JOIN y_Table y on y.Id=x.Id...`  

Before dropping the original table we need to remove all references to it's Primary Key:
```powershell
$inboundForeignKeys = $server.Databases[$connection.Database].Tables[$table].EnumForeignKeys()
foreach($foreignKey in $inboundForeignKeys)
{
    Write-Output "ALTER TABLE [dbo].[$($foreignKey['Table_Name'])] DROP CONSTRAINT [$($foreignKey['Name'])];"
}
```
If you have triggers or schema-bound objects or anything preventing table to be drop should be taken care of - luckily in my case there was not. 
Then we can drop the original table and rename the heap table. Nothing special required - `DROP TABLE` and `sp_rename`.  

The last step is to create primary and foreign keys, defaults, constraints and indexes:
```powershell
# indexes, including primary key
$scripter = New-Object Microsoft.SqlServer.Management.Smo.Scripter $server
$scripter.Options.NoCollation = $true
$scripter.Options.DriPrimaryKey = $true
$scripter.Options.DriUniqueKeys = $true
$scripter.Options.ClusteredIndexes = $true
$scripter.Options.NonClusteredIndexes = $true
$scripter.Options.Indexes = $true
$scripter.Options.DriAll = $true

[Microsoft.SqlServer.Management.Smo.SqlSmoObject[]]$indexes = $server.Databases$connection.Database].Tables[$table].Indexes

$scripter.Script($indexes)

# default constraints
$columns = $server.Databases[$connection.Database].Tables[$table].Columns
foreach($column in $columns)
{
    if($column.DefaultConstraint)
    {
        $column.DefaultConstraint.Script()
    }
}

# foreign keys
[Microsoft.SqlServer.Management.Smo.SqlSmoObject[]]$foreignKeys = $server.Databases[$connection.Database].Tables[$table].ForeignKeys

$scripter.Script($foreignKeys)

# foreign keys referencing the table
$inboundForeignKeys = $server.Databases[$connection.Database].Tables[$table].EnumForeignKeys()
foreach($foreignKey in $inboundForeignKeys)
{
    [Microsoft.SqlServer.Management.Smo.SqlSmoObject[]]$key = $server.Databases[$connection.Database].Tables[$foreignKey['Table_Name']].ForeignKeys[$foreignKey['Name']]

    $scripter.Script($key)
}
```  

These are the most important the pieces I needed to generate the SQL.  

In practice I turned the initial deletion SQL script into a kind of template containing the standard deletion statements and occasionally a palceholder with the table name and where clause defining the data to be kept. Then a PowerShell script will read the template and replace the placeholders with SQL statements generated using the steps described above. 

Having automated SQL script generation based on the actual metadata has two main advantages:  
- removes the repetitive (and boring) manual work
- when the schema changes (e.g. a table is added or altered, column is changed or dropped, index is created, etc.) the SQL script can be easily generated again   

## Conclusion  

Replacing a regular SQL deletes with inserting in heap table and dropping the original one has achieved a satisfying performance boost. It is a combination of SQL Server's data loading capabilities and minimally logged operations. In my case it worked good since I needed to keep (move) 20-30% of the table's data.  

Implementation is more complex than regular deletes so an automated SQL script generation comes handy.  

The downside of the solution is it's complexity so I would recommend it only as a last resort.
