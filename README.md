# Notes: Azure Dedicated SQL Pool optimization

# 1. Data distribution

- Control node (master) and compute nodes (workers)
- Control node comes up with a distributed query plan and asks each of the control nodes to execute their part
- DMS moves data between these nodes
- 60 distributions (buckets) are distributed among compute nodes
- distributed table can be round Robin (default) or based on a deterministic/consistent hash key

## Round Robin table -
- when there is no obvious distribution key
- can be faster for temparary staging tables

## Hash distributed table -
- For large fact tables > 2TB
- Hash distribution column should have high cardinality, even distribution of values
- Avoid distribution on date column (as it can result in under utilisation of compute resources) or column with high numbers of null values
- Pick distribution column that minimizes data movement and avoids data skewness
- Distribution column - column that is frequently used for join, group by, distinct, over, and having clause, and not for where clause

## Replicated table -
- Has full copy on each compute node
- if update occurs, tables need to be rebuilt on each node
- ideal for small dimension tables < 2GB, and if the data is static even bigger tables
- may not work well in cases where there are large number of columns but only a few columns are ever queried

# 2. Index Options (Work in Progress)

- Row Store
  - Heap
  - Clustered Index
  - Non-Clustered Index
- Clustered Columnstore Index (default)

## Clustered Columnstore Index

Segment quality is most optimal where there are at least 100 K rows per compressed row group and gain in performance as
the number of rows per row group approach 1,048,576 rows, which is the most rows a row group can contain.

clustered columnstore may not be a good option in following cases:

- Columnstore tables do not support varchar(max), nvarchar(max), and varbinary(max). Consider heap or clustered index instead.
- Columnstore tables may be less efficient for transient data. Consider heap and perhaps even temporary tables.
- Small tables with less than 60 million rows. Consider heap tables.
  - Cluster columnstore tables begin to achieve optimal compression once there is more than 60 million rows. For small lookup tables, less than 60 million rows, consider using HEAP or clustered index for faster query performance.

## Heap Tables

If you are loading data only to stage it before running more transformations, loading the table to heap table is much faster than loading the data to a clustered columnstore table.

## Clustered indexes

- Clustered indexes may outperform clustered columnstore tables when a single row needs to be quickly retrieved.
- The disadvantage to using a clustered index is that only queries that benefit are the ones that use a highly selective filter on the clustered index column. To improve filter on other columns, a nonclustered index can be added to other columns. However, each index that is added to a table adds both space and processing time to loads.

# 3. Table partitioning

- Dividing data into partitions, can only be done on one column
- Typically done on the date column (column that is typically filtered by in queries) - tied to the order of data loading
- Can improve query performance by eliminating partitions that are not necessary
- Makes data loading simple by switch-in/switch-out methods, and can make lifecycle management efficient

## Range partitioning

- we define boundary points by which the partitions will be created
- Range Right - includes boundary point on the partition right to the point
- Range Left - includes boundary point on the partition left to the point

So if boundary points are 1, 2, 3
- Range Right partitions will be <1 | >=1 & <2 | >=2 & <3 | >=3
- Range Left partitions will be <=1 | >1 & <=2 | >2 & <=3 | >3
- Number of partitions = number of boundary points + 1

## Partitioning Tips

- Be careful of over-partitioning as
  - data is already spread across 60 distributions
  - columnstore index row groups upto ~1M rows (1,048,576) for optimal compression
  - so ideally we should have 60M rows per partition (1M rows per partition per distribution)
- Partition where lifecycle management is needed
  - Sliding window deployment - incoming data being added to one side of the table
  - Targeted index rebuilds - can rebuild indexes for specific partition as needed

## Operations


### Setup

```azure
/*
	Sample Contoso database can be installed from here:
	https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-load-from-azure-blob-storage-with-polybase
	https://github.com/microsoft/sql-server-samples/blob/master/samples/databases/contoso-data-warehouse/load-contoso-data-warehouse-to-sql-data-warehouse.sql
*/

/*
	-- A: Create a master key.
	-- Only necessary if one does not already exist.
	-- Required to encrypt the credential secret in the next step.

	CREATE MASTER KEY;

	-- Create an external data source
	-- TYPE: HADOOP - PolyBase uses Hadoop APIs to access data in Azure blob storage.
	-- LOCATION: Provide Azure storage account name and blob container name.
	-- CREDENTIAL: Provide the credential created in the previous step.

	CREATE EXTERNAL DATA SOURCE AzureStorage_west_public
	WITH 
	(  
		TYPE = Hadoop 
	,   LOCATION = 'wasbs://contosoretaildw-tables@contosoretaildw.blob.core.windows.net/'
	); 
	GO

	-- The data is stored in text files in Azure blob storage, and each field is separated with a delimiter. 
	-- Run this [CREATE EXTERNAL FILE FORMAT][] command to specify the format of the data in the text files. 
	-- he Contoso data is uncompressed and pipe delimited.

	CREATE EXTERNAL FILE FORMAT TextFileFormatContoso 
	WITH 
	(   FORMAT_TYPE = DELIMITEDTEXT
	,	FORMAT_OPTIONS	(   FIELD_TERMINATOR = '|'
						,	STRING_DELIMITER = ''
						,	DATE_FORMAT		 = 'yyyy-MM-dd HH:mm:ss.fff'
						,	USE_TYPE_DEFAULT = FALSE 
						)
	);
	GO
*/
/*
	-- To create a place to store the Contoso data in your database, create a schema
	-- for external tables and a schema for internal tables.
	CREATE SCHEMA [asb]
	GO
	CREATE SCHEMA [cso]
	GO
*/

```

### Creating an un-partitioned table

```azure
/*
--DROP EXTERNAL TABLE [asb].[FactOnlineSales];
CREATE EXTERNAL TABLE [asb].[FactOnlineSales]
(
    [OnlineSalesKey] [int]  NOT NULL,
    [DateKey] [datetime] NOT NULL,
    [StoreKey] [int] NOT NULL,
    [ProductKey] [int] NOT NULL,
    [PromotionKey] [int] NOT NULL,
    [CurrencyKey] [int] NOT NULL,
    [CustomerKey] [int] NOT NULL,
    [SalesOrderNumber] [nvarchar](20) NOT NULL,
    [SalesOrderLineNumber] [int] NULL,
    [SalesQuantity] [int] NOT NULL,
    [SalesAmount] [money] NOT NULL,
    [ReturnQuantity] [int] NOT NULL,
    [ReturnAmount] [money] NULL,
    [DiscountQuantity] [int] NULL,
    [DiscountAmount] [money] NULL,
    [TotalCost] [money] NOT NULL,
    [UnitCost] [money] NULL,
    [UnitPrice] [money] NULL,
    [ETLLoadID] [int] NULL,
    [LoadDate] [datetime] NULL,
    [UpdateDate] [datetime] NULL
)
WITH
(
    LOCATION='/FactOnlineSales/'
,   DATA_SOURCE = AzureStorage_west_public
,   FILE_FORMAT = TextFileFormatContoso
,   REJECT_TYPE = VALUE
,   REJECT_VALUE = 0
);

--DROP TABLE [cso].[FactOnlineSales];
CREATE TABLE [cso].[FactOnlineSales]       
WITH (DISTRIBUTION = HASH([ProductKey])) --Hash Distributed
AS 
SELECT * FROM [asb].[FactOnlineSales]        
OPTION (LABEL = 'CTAS : Load [cso].[FactOnlineSales]');

*/
```

```azure
SELECT min(datekey), max(datekey) 
FROM [cso].[FactOnlineSales];

SELECT DISTINCT year(datekey)
FROM [cso].[FactOnlineSales];

SELECT year(datekey), count(*)
FROM [cso].[FactOnlineSales]
GROUP BY year(datekey);
```


### Creating an empty partitioned table

```azure
--DROP TABLE [cso].[FactOnlineSales_PExample]
CREATE TABLE [cso].[FactOnlineSales_PExample] 
(   
	[OnlineSalesKey] [int]  NOT NULL,
    [DateKey] [datetime] NOT NULL,
    [StoreKey] [int] NOT NULL,
    [ProductKey] [int] NOT NULL,
    [PromotionKey] [int] NOT NULL,
    [CurrencyKey] [int] NOT NULL,
    [CustomerKey] [int] NOT NULL,
    [SalesOrderNumber] [nvarchar](20) NOT NULL,
    [SalesOrderLineNumber] [int] NULL,
    [SalesQuantity] [int] NOT NULL,
    [SalesAmount] [money] NOT NULL,
    [ReturnQuantity] [int] NOT NULL,
    [ReturnAmount] [money] NULL,
    [DiscountQuantity] [int] NULL,
    [DiscountAmount] [money] NULL,
    [TotalCost] [money] NOT NULL,
    [UnitCost] [money] NULL,
    [UnitPrice] [money] NULL,
    [ETLLoadID] [int] NULL,
    [LoadDate] [datetime] NULL,
    [UpdateDate] [datetime] NULL
)
WITH 
(   CLUSTERED COLUMNSTORE INDEX
,   DISTRIBUTION = HASH([ProductKey])
,   PARTITION
    (
        [DateKey] RANGE RIGHT FOR VALUES
        (
            '2007-01-01 00:00:00.000','2008-01-01 00:00:00.000'
        ,   '2009-01-01 00:00:00.000','2010-01-01 00:00:00.000'
        )
    )
);
```

### Creating a partitioned table with CTAS

```azure
--DROP TABLE [cso].[FactOnlineSales_Partitioned]
CREATE TABLE [cso].[FactOnlineSales_Partitioned]
WITH
(   CLUSTERED COLUMNSTORE INDEX
,   DISTRIBUTION = HASH([ProductKey])
,   PARTITION
    (
        [DateKey] RANGE RIGHT FOR VALUES
        (
        	'2007-01-01 00:00:00.000','2008-01-01 00:00:00.000'
		,	'2009-01-01 00:00:00.000','2010-01-01 00:00:00.000'
        )
    )
)
AS
SELECT * FROM [cso].[FactOnlineSales];
```

#### Recommended to create statistics on this table for optimal performance

```azure
--UPDATE statistics [cso].[FactOnlineSales_Partitioned]
```

```azure
SELECT year(datekey), count(*)
FROM [cso].[FactOnlineSales_Partitioned]
GROUP BY year(datekey)
ORDER BY year(datekey);
```

#### To check partitions on this table - sys.dm_pdw_nodes_db_partition_stats

```azure
SELECT  pnp.partition_number, sum(nps.[row_count]) AS Row_Count
FROM
   sys.tables t
INNER JOIN sys.indexes i
    ON  t.[object_id] = i.[object_id]
    AND i.[index_id] <= 1 /* HEAP = 0, CLUSTERED or CLUSTERED_COLUMNSTORE =1 */
INNER JOIN sys.pdw_table_mappings tm
    ON t.[object_id] = tm.[object_id]
INNER JOIN sys.pdw_nodes_tables nt
    ON tm.[physical_name] = nt.[name]
INNER JOIN sys.pdw_nodes_partitions pnp 
    ON nt.[object_id]=pnp.[object_id] 
    AND nt.[pdw_node_id]=pnp.[pdw_node_id] 
    AND nt.[distribution_id] = pnp.[distribution_id]
INNER JOIN sys.dm_pdw_nodes_db_partition_stats nps
    ON nt.[object_id] = nps.[object_id]
    AND nt.[pdw_node_id] = nps.[pdw_node_id]
    AND nt.[distribution_id] = nps.[distribution_id]
    AND pnp.[partition_id]=nps.[partition_id]
WHERE t.name='FactOnlineSales_Partitioned'
GROUP BY pnp.partition_number;
```

### Data deletion or archival - Partition Switch Out

#### Create empty table

Boundary point is defined for the range for which data needs to be switched out. for Switching-out this can be skipped
and the data will be switched out to a non-partition table, but is necessary for Switching in

```azure
--DROP TABLE [cso].[FactOnlineSales_out]
CREATE TABLE [cso].[FactOnlineSales_out]
WITH 
(   DISTRIBUTION=HASH ([ProductKey])
,   CLUSTERED COLUMNSTORE INDEX
,   PARTITION ([DateKey] 
    RANGE RIGHT FOR VALUES 	
	(
		'2007-01-01 00:00:00.000'
    ))
)
AS 
SELECT * FROM [cso].[FactOnlineSales_Partitioned] WHERE 1=2;
```

#### Check partition info for the above table
this will show 0 rows in each of the 2 partitions as there is no data
```azure
--Data deletion or archival - Partition Switch Out
SELECT  pnp.partition_number, sum(nps.[row_count]) AS Row_Count
FROM
   sys.tables t
INNER JOIN sys.indexes i
    ON  t.[object_id] = i.[object_id]
    AND i.[index_id] <= 1 /* HEAP = 0, CLUSTERED or CLUSTERED_COLUMNSTORE =1 */
INNER JOIN sys.pdw_table_mappings tm
    ON t.[object_id] = tm.[object_id]
INNER JOIN sys.pdw_nodes_tables nt
    ON tm.[physical_name] = nt.[name]
INNER JOIN sys.pdw_nodes_partitions pnp 
    ON nt.[object_id]=pnp.[object_id] 
    AND nt.[pdw_node_id]=pnp.[pdw_node_id] 
    AND nt.[distribution_id] = pnp.[distribution_id]
INNER JOIN sys.dm_pdw_nodes_db_partition_stats nps
    ON nt.[object_id] = nps.[object_id]
    AND nt.[pdw_node_id] = nps.[pdw_node_id]
    AND nt.[distribution_id] = nps.[distribution_id]
    AND pnp.[partition_id]=nps.[partition_id]
WHERE t.name='FactOnlineSales_out'
GROUP BY pnp.partition_number;
```

#### Switch partition 2 from original table to the out table

This is a meta-data operation, so it will run quickly irrespective of how big the partition is

```azure
ALTER TABLE [cso].[FactOnlineSales_Partitioned] 
SWITCH PARTITION 2 
TO [cso].[FactOnlineSales_out] PARTITION 2;
```

```azure
SELECT min(datekey), max(datekey) 
FROM [cso].[FactOnlineSales_Partitioned];

SELECT  min(datekey), max(datekey) 
FROM [cso].[FactOnlineSales_out];
```

#### Drop/Archive the out table

```azure
--Validate row count for both main and archive tables

DROP TABLE [cso].[FactOnlineSales_out];
```

### Partition Split - create new partition

https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-tables-partition#how-to-split-a-partition-that-contains-data

A clustered columnstore table, the table partition must be
empty before it can be split.
1. The most efficient method to split a partition that already contains data is to use a CTAS statement.
2. Consider disabling the columnstore index before issuing the ALTER PARTITION statement,
   then rebuilding the columnstore index after ALTER PARTITION is complete.
3. Use CTAS to create a new table to hold the data (to empty the partition) and the split
   and then finally switch data back in.

#### Split range

This will work in this case as the partition we are splitting does not have any data, would not work otherwise

```azure
ALTER TABLE [cso].[FactOnlineSales_Partitioned] 
SPLIT RANGE ('2011-01-01 00:00:00.000');
```

#### Checking the partition count

This will show one more partition

```azure
SELECT  pnp.partition_number, sum(nps.[row_count]) AS Row_Count
FROM
   sys.tables t
INNER JOIN sys.indexes i
    ON  t.[object_id] = i.[object_id]
    AND i.[index_id] <= 1 /* HEAP = 0, CLUSTERED or CLUSTERED_COLUMNSTORE =1 */
INNER JOIN sys.pdw_table_mappings tm
    ON t.[object_id] = tm.[object_id]
INNER JOIN sys.pdw_nodes_tables nt
    ON tm.[physical_name] = nt.[name]
INNER JOIN sys.pdw_nodes_partitions pnp 
    ON nt.[object_id]=pnp.[object_id] 
    AND nt.[pdw_node_id]=pnp.[pdw_node_id] 
    AND nt.[distribution_id] = pnp.[distribution_id]
INNER JOIN sys.dm_pdw_nodes_db_partition_stats nps
    ON nt.[object_id] = nps.[object_id]
    AND nt.[pdw_node_id] = nps.[pdw_node_id]
    AND nt.[distribution_id] = nps.[distribution_id]
    AND pnp.[partition_id]=nps.[partition_id]
WHERE t.name='FactOnlineSales_Partitioned'
GROUP BY pnp.partition_number;
```

### Incremental data load - Partition Switch In : Scenario 1

Adding an entire new year's worth of data

```azure
--ensure the table definitions match and that the partitions align 
--on their respective boundaries, i.e. the source table must contain 
--the same partition boundaries as the target table 
--Scenario 1
--We are artificially adding data for 2010 by adding 1 year to all data of 2009 
--DROP TABLE [cso].[FactOnlineSales_in]
CREATE TABLE [cso].[FactOnlineSales_in]
WITH
(   CLUSTERED COLUMNSTORE INDEX
,   DISTRIBUTION = HASH([ProductKey])
,   PARTITION
    (
        [DateKey] RANGE RIGHT FOR VALUES
        (
        	'2007-01-01 00:00:00.000','2008-01-01 00:00:00.000'
		,	'2009-01-01 00:00:00.000','2010-01-01 00:00:00.000'
		,	'2011-01-01 00:00:00.000'
        )
    )
)
AS
SELECT [OnlineSalesKey]
	  ,ISNULL(DATEADD(year, 1, [DateKey]), GETDATE()) AS [DateKey]
      ,[StoreKey],[ProductKey],[PromotionKey],[CurrencyKey],[CustomerKey]
      ,[SalesOrderNumber],[SalesOrderLineNumber],[SalesQuantity]
      ,[SalesAmount],[ReturnQuantity],[ReturnAmount],[DiscountQuantity]
      ,[DiscountAmount],[TotalCost],[UnitCost],[UnitPrice]
      ,[ETLLoadID],[LoadDate],[UpdateDate]
FROM   [cso].[FactOnlineSales] stg
WHERE stg.[DateKey] >= '2009-01-01 00:00:00.000'
AND   stg.[DateKey] <  '2010-01-01 00:00:00.000'
```

```azure
SELECT  min(datekey), max(datekey) 
FROM [cso].[FactOnlineSales_in];
```

#### Check partition info for staging table

This will only show data in 2010 partition

```azure
SELECT  pnp.partition_number, sum(nps.[row_count]) AS Row_Count
FROM
   sys.tables t
INNER JOIN sys.indexes i
    ON  t.[object_id] = i.[object_id]
    AND i.[index_id] <= 1 /* HEAP = 0, CLUSTERED or CLUSTERED_COLUMNSTORE =1 */
INNER JOIN sys.pdw_table_mappings tm
    ON t.[object_id] = tm.[object_id]
INNER JOIN sys.pdw_nodes_tables nt
    ON tm.[physical_name] = nt.[name]
INNER JOIN sys.pdw_nodes_partitions pnp 
    ON nt.[object_id]=pnp.[object_id] 
    AND nt.[pdw_node_id]=pnp.[pdw_node_id] 
    AND nt.[distribution_id] = pnp.[distribution_id]
INNER JOIN sys.dm_pdw_nodes_db_partition_stats nps
    ON nt.[object_id] = nps.[object_id]
    AND nt.[pdw_node_id] = nps.[pdw_node_id]
    AND nt.[distribution_id] = nps.[distribution_id]
    AND pnp.[partition_id]=nps.[partition_id]
WHERE t.name='FactOnlineSales_in'
GROUP BY pnp.partition_number;
```

#### Switch partition 5 from staging table to original

```azure
ALTER TABLE [cso].[FactOnlineSales_in] 
SWITCH PARTITION 5 
TO [cso].[FactOnlineSales_Partitioned] PARTITION 5;
```

#### Check partition info for original table

```azure
SELECT  pnp.partition_number, sum(nps.[row_count]) AS Row_Count
FROM
   sys.tables t
INNER JOIN sys.indexes i
    ON  t.[object_id] = i.[object_id]
    AND i.[index_id] <= 1 /* HEAP = 0, CLUSTERED or CLUSTERED_COLUMNSTORE =1 */
INNER JOIN sys.pdw_table_mappings tm
    ON t.[object_id] = tm.[object_id]
INNER JOIN sys.pdw_nodes_tables nt
    ON tm.[physical_name] = nt.[name]
INNER JOIN sys.pdw_nodes_partitions pnp 
    ON nt.[object_id]=pnp.[object_id] 
    AND nt.[pdw_node_id]=pnp.[pdw_node_id] 
    AND nt.[distribution_id] = pnp.[distribution_id]
INNER JOIN sys.dm_pdw_nodes_db_partition_stats nps
    ON nt.[object_id] = nps.[object_id]
    AND nt.[pdw_node_id] = nps.[pdw_node_id]
    AND nt.[distribution_id] = nps.[distribution_id]
    AND pnp.[partition_id]=nps.[partition_id]
WHERE t.name='FactOnlineSales_Partitioned'
GROUP BY pnp.partition_number;
```

### Incremental data load - Partition Switch In : Scenario 2

Adding data for a specific month

```azure
--Scenario 2
CREATE TABLE [cso].[FactOnlineSales_in2]
WITH 
(   DISTRIBUTION=HASH ([ProductKey])
,   CLUSTERED COLUMNSTORE INDEX
PARTITION
    (
        [DateKey] RANGE RIGHT FOR VALUES
        (
        	'2007-01-01 00:00:00.000','2008-01-01 00:00:00.000'
		,	'2009-01-01 00:00:00.000','2010-01-01 00:00:00.000'
		,	'2011-01-01 00:00:00.000'
        )
    )
)
AS
SELECT [OnlineSalesKey]
	  ,[DateKey] AS [DateKey]
      ,[StoreKey],[ProductKey],[PromotionKey],[CurrencyKey],[CustomerKey]
      ,[SalesOrderNumber],[SalesOrderLineNumber],[SalesQuantity]
      ,[SalesAmount],[ReturnQuantity],[ReturnAmount],[DiscountQuantity]
      ,[DiscountAmount],[TotalCost],[UnitCost],[UnitPrice]
      ,[ETLLoadID],[LoadDate],[UpdateDate]
FROM   [cso].[FactOnlineSales] stg
WHERE stg.[DateKey] >= '2010-05-01 00:00:00.000'
AND   stg.[DateKey] <  '2010-05-31 00:00:00.000'
UNION ALL
SELECT *
FROM   [cso].[FactOnlineSales_Partitioned] tgt
WHERE tgt.[DateKey] >= '2010-01-01 00:00:00.000'
AND   tgt.[DateKey] <  '2010-04-30 00:00:00.000'
```

### Check distribution and partition info for the table

- Each distribution will have 6 partitions, 3 of them empty for each distribution, with skewed number of rows

```azure
--Before partitions are created, dedicated SQL pool already divides 
--each table into 60 distributed databases. For optimal compression and 
--performance of clustered columnstore tables, 1 million rows 
--per distribution and partition is recommended. 
--Distribution - partitions - rowcounts
SELECT  t.name, nt.distribution_id, 
	pnp.partition_number, nps.[row_count]
FROM
   sys.tables t
INNER JOIN sys.indexes i
    ON  t.[object_id] = i.[object_id]
    AND i.[index_id] <= 1 /* HEAP = 0, CLUSTERED or CLUSTERED_COLUMNSTORE =1 */
INNER JOIN sys.pdw_table_mappings tm
    ON t.[object_id] = tm.[object_id]
INNER JOIN sys.pdw_nodes_tables nt
    ON tm.[physical_name] = nt.[name]
INNER JOIN sys.pdw_nodes_partitions pnp 
    ON nt.[object_id]=pnp.[object_id] 
    AND nt.[pdw_node_id]=pnp.[pdw_node_id] 
    AND nt.[distribution_id] = pnp.[distribution_id]
INNER JOIN sys.dm_pdw_nodes_db_partition_stats nps
    ON nt.[object_id] = nps.[object_id]
    AND nt.[pdw_node_id] = nps.[pdw_node_id]
    AND nt.[distribution_id] = nps.[distribution_id]
    AND pnp.[partition_id]=nps.[partition_id]
WHERE t.name='FactOnlineSales_Partitioned'
ORDER BY nt.distribution_id;

```
