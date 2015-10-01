# Spark-HBase Connector

Spark-HBase Connector is a library to support Spark accessing HBase table as external data source or sink. With it, user can operate HBase with Spark-SQL on data frame level. 

With the data frame support, the lib leverages all the optimization techniques in catalyst, and achieves data locality, partition pruning, predicate pushdown, Scanning and BulkGet, etc. 

## Catalog
For each table, a catalog has to be provided,  which includes the row key, and the columns with data type with predefined column families, and defines the mapping between hbase column and table schema. The catalog is user defined json format.

## Datatype conversion
Java primitive types is supported. In the future, other data types will be supported, which relies on user specified sedes. Take avro as an example. User defined sedes will be responsible to convert byte array to avro object, and connector will be responsible to convert avro object to catalyst supported datatypes. 

Note that if user want dataframe to only handle byte array, the binary type can be specified. Then user can get the catalyst row with each column as a byte array. User can further deserialize it with customized deserializer, or operate on the RDD of the data frame directly.

## Data locality
When the spark work node co-located with hbase region servers, data locality is achieved by identifying the region server location, and co-locate the executor with the region server. Each executor will only perform Scan/BulkGet on the part of the data that co-locates on the same host. 

## Predicate pushdown
The lib use existing standard HBase filter provided by HBase and does not operate on the coprocessor. 

## Partition Pruning
By extracting the row key from the predicates, we split the scan/BulkGet into multiple non-overlapping regions, only the region servers that have the requested data will perform scan/BulkGet. Currently, the partition pruning is performed on the first dimension of the row keys. Note that the WHERE conditions need to be defined carefully. Otherwise, the result scanning may includes a region larger than user expectd. For example, following condition will result in a full scan (rowkey1 is the first dimension of the rowkey, and column is a regular hbase column).
         WHERE rowkey1 > "abc" OR column = "xyz"

## Scanning and BulkGet
Both are exposed to users by specifying WHERE CLAUSE, e.g., where column > x and column < y for scan and where column = x for get. All the operations are performed in the executors, and driver only constructs these operations. Internally we will convert them to scan or get or combination of both, which return Iterator[Row] to catalyst engine. 

## 
Creatable DataSource  The libary support both read/write from/to HBase.

##Application API/Usage: 

### Defined the HBase catalog
   def catalog = s"""{
            |"table":{"namespace":"default", "name":"table1"},
            |"rowkey":"key",
            |"columns":{
              |"col0":{"cf":"rowkey", "col":"key", "type":"string"},
              |"col1":{"cf":"cf1", "col":"col1", "type":"boolean"},
              |"col2":{"cf":"cf2", "col":"col2", "type":"double"},
              |"col3":{"cf":"cf3", "col":"col3", "type":"float"},
              |"col4":{"cf":"cf4", "col":"col4", "type":"int"},
              |"col5":{"cf":"cf5", "col":"col5", "type":"bigint"},
              |"col6":{"cf":"cf6", "col":"col6", "type":"smallint"},
              |"col7":{"cf":"cf7", "col":"col7", "type":"string"},
              |"col8":{"cf":"cf8", "col":"col8", "type":"tinyint"}
            |}
          |}""".stripMargin

### Write to HBase table to populate data.
    sc.parallelize(data).toDF.write.options(
      Map(HBaseTableCatalog.tableCatalog -> catalog, HBaseTableCatalog.newTable -> "5"))
      .format("org.apache.spark.sql.execution.datasources.hbase")
      .save()

### Perform data frame operation on top of HBase table
   def withCatalog(cat: String): DataFrame = {
      sqlContext
      .read
      .options(Map(HBaseTableCatalog.tableCatalog->cat))
      .format("org.apache.spark.sql.execution.datasources.hbase")
      .load()
  }
  1 complicated query
    val df = withCatalog(catalog)
    val s = df.filter((($"col0" <= "row050" && $"col0" > "row040") ||
      $"col0" === "row005" ||
      $"col0" === "row020" ||
      $"col0" ===  "r20" ||
      $"col0" <= "row005") &&
      ($"col4" === 1 ||
      $"col4" === 42))
      .select("col0", "col1", "col4")
    s.show
  2 SQL support
   // Load the dataframe
   val df = withCatalog(catalog)
   //SQL example
   df.registerTempTable("table")
   sqlContext.sql("select count(col1) from table").show
