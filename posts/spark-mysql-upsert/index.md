# spark 实现 mysql upsert


**实现 spark dataframe/dataset 根据mysql表唯一键实现有则更新，无则插入功能。**

&lt;!--more--&gt;



基于 **spark2.4.3 scala2.11.8**

## 工具类 DataFrameWriterEnhance
```java
package com.xxx.utils

import org.apache.spark.sql.catalyst.plans.logical.LogicalPlan
import org.apache.spark.sql.execution.SQLExecution
import org.apache.spark.sql.execution.datasources.DataSource
import org.apache.spark.sql.execution.datasources.jdbc.{JDBCOptions, JdbcOptionsInWrite, JdbcRelationProvider, JdbcUtils}
import org.apache.spark.sql.jdbc.{JdbcDialect, JdbcDialects}
import org.apache.spark.sql.sources.BaseRelation
import org.apache.spark.sql.types.StructType
import org.apache.spark.sql._

import java.sql.Connection

object DataFrameWriterEnhance {

  implicit class DataFrameWriterMysqlUpdateEnhance(writer: DataFrameWriter[Row]) {
    def update(): Unit = {
      val extraOptionsField = writer.getClass.getDeclaredField(&#34;org$apache$spark$sql$DataFrameWriter$$extraOptions&#34;)
      val dfField = writer.getClass.getDeclaredField(&#34;df&#34;)
      val sourceField = writer.getClass.getDeclaredField(&#34;source&#34;)
      val partitioningColumnsField = writer.getClass.getDeclaredField(&#34;partitioningColumns&#34;)
      extraOptionsField.setAccessible(true)
      dfField.setAccessible(true)
      sourceField.setAccessible(true)
      partitioningColumnsField.setAccessible(true)
      val extraOptions = extraOptionsField.get(writer).asInstanceOf[scala.collection.Map[String, String]]
      val df = dfField.get(writer).asInstanceOf[DataFrame]
      val partitioningColumns = partitioningColumnsField.get(writer).asInstanceOf[Option[Seq[String]]]
      val logicalPlanField = df.getClass.getDeclaredField(&#34;logicalPlan&#34;)
      logicalPlanField.setAccessible(true)
      var logicalPlan = logicalPlanField.get(df).asInstanceOf[LogicalPlan]
      val session = df.sparkSession
      val dataSource = DataSource(
        sparkSession = session,
        className = s&#34;${DataFrameWriterEnhance.getClass.getName}MysqlUpdateRelationProvider&#34;,
        partitionColumns = partitioningColumns.getOrElse(Nil),
        options = extraOptions.toMap)
      logicalPlan = dataSource.planForWriting(SaveMode.Append, logicalPlan)
      val qe = session.sessionState.executePlan(logicalPlan)
      SQLExecution.withNewExecutionId(session, qe)(qe.toRdd)
    }
  }

  class MysqlUpdateRelationProvider extends JdbcRelationProvider {
    override def createRelation(sqlContext: SQLContext, mode: SaveMode, parameters: Map[String, String], df: DataFrame): BaseRelation = {
      val options = new JdbcOptionsInWrite(parameters)
      val isCaseSensitive = sqlContext.sparkSession.sessionState.conf.caseSensitiveAnalysis
      val conn = JdbcUtils.createConnectionFactory(options)()
      try {
        val tableExists = JdbcUtils.tableExists(conn, options)
        if (tableExists) {
          mode match {
            case SaveMode.Overwrite =&gt;
              if (options.isTruncate &amp;&amp; JdbcUtils.isCascadingTruncateTable(options.url).contains(false)) {
                // In this case, we should truncate table and then load.
                JdbcUtils.truncateTable(conn, options)
                val tableSchema = JdbcUtils.getSchemaOption(conn, options)
                updateTable(df, tableSchema, isCaseSensitive, options)
              } else {
                // Otherwise, do not truncate the table, instead drop and recreate it
                JdbcUtils.dropTable(conn, options.table, options)
                JdbcUtils.createTable(conn, df, options)
                updateTable(df, Some(df.schema), isCaseSensitive, options)
              }

            case SaveMode.Append =&gt;
              val tableSchema = JdbcUtils.getSchemaOption(conn, options)
              updateTable(df, tableSchema, isCaseSensitive, options)

            case SaveMode.ErrorIfExists =&gt;
              throw new Exception(
                s&#34;Table or view &#39;${options.table}&#39; already exists. &#34; &#43;
                  s&#34;SaveMode: ErrorIfExists.&#34;)

            case SaveMode.Ignore =&gt;
            // With `SaveMode.Ignore` mode, if table already exists, the save operation is expected
            // to not save the contents of the DataFrame and to not change the existing data.
            // Therefore, it is okay to do nothing here and then just return the relation below.
          }
        } else {
          JdbcUtils.createTable(conn, df, options)
          updateTable(df, Some(df.schema), isCaseSensitive, options)
        }
      } finally {
        conn.close()
      }

      createRelation(sqlContext, parameters)
    }

    def updateTable(df: DataFrame,
                    tableSchema: Option[StructType],
                    isCaseSensitive: Boolean,
                    options: JdbcOptionsInWrite): Unit = {
      val url = options.url
      val table = options.table
      val dialect = JdbcDialects.get(url)
      val rddSchema = df.schema
      val getConnection: () =&gt; Connection = JdbcUtils.createConnectionFactory(options)
      val batchSize = options.batchSize
      val isolationLevel = options.isolationLevel

      val updateStmt = getUpdateStatement(table, rddSchema, tableSchema, isCaseSensitive, dialect)
      println(updateStmt)
      val repartitionedDF = options.numPartitions match {
        case Some(n) if n &lt;= 0 =&gt; throw new IllegalArgumentException(
          s&#34;Invalid value `$n` for parameter `${JDBCOptions.JDBC_NUM_PARTITIONS}` in table writing &#34; &#43;
            &#34;via JDBC. The minimum value is 1.&#34;)
        case Some(n) if n &lt; df.rdd.partitions.length =&gt; df.coalesce(n)
        case _ =&gt; df
      }
      repartitionedDF.rdd.foreachPartition(iterator =&gt; JdbcUtils.savePartition(
        getConnection, table, iterator, rddSchema, updateStmt, batchSize, dialect, isolationLevel,
        options)
      )
    }

    def getUpdateStatement(table: String,
                           rddSchema: StructType,
                           tableSchema: Option[StructType],
                           isCaseSensitive: Boolean,
                           dialect: JdbcDialect): String = {
      val columns = if (tableSchema.isEmpty) {
        rddSchema.fields.map(x =&gt; dialect.quoteIdentifier(x.name)).mkString(&#34;,&#34;)
      } else {
        val columnNameEquality = if (isCaseSensitive) {
          org.apache.spark.sql.catalyst.analysis.caseSensitiveResolution
        } else {
          org.apache.spark.sql.catalyst.analysis.caseInsensitiveResolution
        }
        // The generated insert statement needs to follow rddSchema&#39;s column sequence and
        // tableSchema&#39;s column names. When appending data into some case-sensitive DBMSs like
        // PostgreSQL/Oracle, we need to respect the existing case-sensitive column names instead of
        // RDD column names for user convenience.
        val tableColumnNames = tableSchema.get.fieldNames
        rddSchema.fields.map { col =&gt;
          val normalizedName = tableColumnNames.find(f =&gt; columnNameEquality(f, col.name)).getOrElse {
            throw new Exception(s&#34;&#34;&#34;Column &#34;${col.name}&#34; not found in schema $tableSchema&#34;&#34;&#34;)
          }
          dialect.quoteIdentifier(normalizedName)
        }.mkString(&#34;,&#34;)
      }
      val placeholders = rddSchema.fields.map(_ =&gt; &#34;?&#34;).mkString(&#34;,&#34;)
      s&#34;&#34;&#34;INSERT INTO $table ($columns) VALUES ($placeholders)
         |ON DUPLICATE KEY UPDATE
         |${columns.split(&#34;,&#34;).map(col =&gt; s&#34;$col=VALUES($col)&#34;).mkString(&#34;,&#34;)}
         |&#34;&#34;&#34;.stripMargin
    }
  }
}

```
## 工具类 MysqlUtils
```java
package com.xxx.utils

import com.xxx.utils.DataFrameWriterEnhance.DataFrameWriterMysqlUpdateEnhance
import org.apache.spark.sql.functions.col
import org.apache.spark.sql.types.{NullType, ShortType}
import org.apache.spark.sql.{DataFrame, SaveMode, SparkSession}

object MysqlUtils {

  def upsert(rawDF: DataFrame, database: String, tableName: String)(implicit spark: SparkSession): Unit = {
    var df = rawDF
    for (elem &lt;- df.schema.fields) {
      if (elem.dataType == NullType) {
        df = df.withColumn(elem.name, col(elem.name).cast(ShortType))
      }
    }

    df.write
      .format(&#34;jdbc&#34;)
      .mode(SaveMode.Append)
      .option(&#34;driver&#34;, &#34;com.mysql.jdbc.Driver&#34;)
      .option(&#34;url&#34;, spark.conf.get(s&#34;spark.job.mysql.${database}.url&#34;))
      .option(&#34;user&#34;, spark.conf.get(s&#34;spark.job.mysql.${database}.username&#34;))
      .option(&#34;password&#34;, spark.conf.get(s&#34;spark.job.mysql.${database}.password&#34;))
      .option(&#34;dbtable&#34;, tableName)
      .option(&#34;useSSL&#34;, &#34;false&#34;)
      .option(&#34;showSql&#34;, &#34;false&#34;)
      .option(&#34;numPartitions&#34;, &#34;1&#34;)
      .update()
  }


}

```
## 使用
spark启动脚本加入mysql配置
```bash
spark-submit \
--master yarn \
--deploy-mode cluster \
--executor-memory 3G \
--num-executors 5 \
--executor-cores 4 \
--driver-memory 3G \
--conf spark.job.mysql.test.url=${jdbc_url} \
--conf spark.job.mysql.test.username=${jdbc_username} \
--conf spark.job.mysql.test.password=${jdbc_password} \
```
使用范例
```java
import utils.MysqlUtils

object TestMysqlUpsert {
  def main(args: Array[String]): Unit = {
    implicit val spark = SparkSession.builder().enableHiveSupport().getOrCreate()
    import spark.implicits._

    val database = &#34;test&#34;
    val arr = Array((1,11,&#34;name1&#34;,11111),(2,22,&#34;name2&#34;,22222))
    val df = spark.sparkContext.parallelize(arr)
      .toDF(&#34;key_one&#34;, &#34;key_two&#34;, &#34;val_one&#34;, &#34;val_two&#34;)

    MysqlUtils.upsert(df, database, &#34;test_unique_key&#34;)
    spark.close()

  }
}
```
test_unique_key表结构
```sql
CREATE TABLE `test_unique_key` (
  `key_one` int(11) NOT NULL DEFAULT &#39;0&#39;,
  `key_two` int(11) NOT NULL DEFAULT &#39;0&#39;,
  `val_one` varchar(50) DEFAULT NULL,
  `val_two` int(11) NOT NULL DEFAULT &#39;0&#39;,
  UNIQUE KEY `uk` (`key_one`,`key_two`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT=&#39;test&#39;;
```
## 参考
[csdn_Spark Upsert写入Mysql(scala增强) 无需依赖](https://blog.csdn.net/qq_18453581/article/details/125907861 &#34;csdn_Spark Upsert写入Mysql(scala增强) 无需依赖&#34;)


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/spark-mysql-upsert/  

