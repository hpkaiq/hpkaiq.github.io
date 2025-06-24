# Spark 实现 Mysql Upsert，可忽略null值


**实现 spark dataframe/dataset 根据mysql表唯一键实现有则更新，无则插入功能。**

**2024.07.25更新：新增特性，忽略值为null的列，即当df里列值为null时，不更新mysql表数据，保留表原有的值。**

<!--more-->



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
      val extraOptionsField = writer.getClass.getDeclaredField("org$apache$spark$sql$DataFrameWriter$$extraOptions")
      val dfField = writer.getClass.getDeclaredField("df")
      val sourceField = writer.getClass.getDeclaredField("source")
      val partitioningColumnsField = writer.getClass.getDeclaredField("partitioningColumns")
      extraOptionsField.setAccessible(true)
      dfField.setAccessible(true)
      sourceField.setAccessible(true)
      partitioningColumnsField.setAccessible(true)
      val extraOptions = extraOptionsField.get(writer).asInstanceOf[scala.collection.Map[String, String]]
      val df = dfField.get(writer).asInstanceOf[DataFrame]
      val partitioningColumns = partitioningColumnsField.get(writer).asInstanceOf[Option[Seq[String]]]
      val logicalPlanField = df.getClass.getDeclaredField("logicalPlan")
      logicalPlanField.setAccessible(true)
      var logicalPlan = logicalPlanField.get(df).asInstanceOf[LogicalPlan]
      val session = df.sparkSession
      val dataSource = DataSource(
        sparkSession = session,
        className = s"${DataFrameWriterEnhance.getClass.getName}MysqlUpdateRelationProvider",
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
            case SaveMode.Overwrite =>
              if (options.isTruncate && JdbcUtils.isCascadingTruncateTable(options.url).contains(false)) {
                // In this case, we should truncate table and then load.
                JdbcUtils.truncateTable(conn, options)
                val tableSchema = JdbcUtils.getSchemaOption(conn, options)
                updateTable(df, tableSchema, isCaseSensitive, options, parameters.getOrElse("ignoreNull", "false").toBoolean)
              } else {
                // Otherwise, do not truncate the table, instead drop and recreate it
                JdbcUtils.dropTable(conn, options.table, options)
                JdbcUtils.createTable(conn, df, options)
                updateTable(df, Some(df.schema), isCaseSensitive, options, parameters.getOrElse("ignoreNull", "false").toBoolean)
              }

            case SaveMode.Append =>
              val tableSchema = JdbcUtils.getSchemaOption(conn, options)
              updateTable(df, tableSchema, isCaseSensitive, options, parameters.getOrElse("ignoreNull", "false").toBoolean)

            case SaveMode.ErrorIfExists =>
              throw new Exception(
                s"Table or view '${options.table}' already exists. " +
                  s"SaveMode: ErrorIfExists.")

            case SaveMode.Ignore =>
            // With `SaveMode.Ignore` mode, if table already exists, the save operation is expected
            // to not save the contents of the DataFrame and to not change the existing data.
            // Therefore, it is okay to do nothing here and then just return the relation below.
          }
        } else {
          JdbcUtils.createTable(conn, df, options)
          updateTable(df, Some(df.schema), isCaseSensitive, options, parameters.getOrElse("ignoreNull", "false").toBoolean)
        }
      } finally {
        conn.close()
      }

      createRelation(sqlContext, parameters)
    }

    def updateTable(df: DataFrame,
                    tableSchema: Option[StructType],
                    isCaseSensitive: Boolean,
                    options: JdbcOptionsInWrite,
                    ignoreNull: Boolean = false): Unit = {
      val url = options.url
      val table = options.table
      val dialect = JdbcDialects.get(url)
      val rddSchema = df.schema
      val getConnection: () => Connection = JdbcUtils.createConnectionFactory(options)
      val batchSize = options.batchSize
      val isolationLevel = options.isolationLevel

      val updateStmt = getUpdateStatement(table, rddSchema, tableSchema, isCaseSensitive, dialect, ignoreNull)
      println(updateStmt)
      val repartitionedDF = options.numPartitions match {
        case Some(n) if n <= 0 => throw new IllegalArgumentException(
          s"Invalid value `$n` for parameter `${JDBCOptions.JDBC_NUM_PARTITIONS}` in table writing " +
            "via JDBC. The minimum value is 1.")
        case Some(n) if n < df.rdd.partitions.length => df.coalesce(n)
        case _ => df
      }
      repartitionedDF.rdd.foreachPartition(iterator => JdbcUtils.savePartition(
        getConnection, table, iterator, rddSchema, updateStmt, batchSize, dialect, isolationLevel,
        options)
      )
    }

    def getUpdateStatement(table: String,
                           rddSchema: StructType,
                           tableSchema: Option[StructType],
                           isCaseSensitive: Boolean,
                           dialect: JdbcDialect,
                           ignoreNull: Boolean): String = {
      val columns = if (tableSchema.isEmpty) {
        rddSchema.fields.map(x => dialect.quoteIdentifier(x.name)).mkString(",")
      } else {
        val columnNameEquality = if (isCaseSensitive) {
          org.apache.spark.sql.catalyst.analysis.caseSensitiveResolution
        } else {
          org.apache.spark.sql.catalyst.analysis.caseInsensitiveResolution
        }
        // The generated insert statement needs to follow rddSchema's column sequence and
        // tableSchema's column names. When appending data into some case-sensitive DBMSs like
        // PostgreSQL/Oracle, we need to respect the existing case-sensitive column names instead of
        // RDD column names for user convenience.
        val tableColumnNames = tableSchema.get.fieldNames
        rddSchema.fields.map { col =>
          val normalizedName = tableColumnNames.find(f => columnNameEquality(f, col.name)).getOrElse {
            throw new Exception(s"""Column "${col.name}" not found in schema $tableSchema""")
          }
          dialect.quoteIdentifier(normalizedName)
        }.mkString(",")
      }
      val placeholders = rddSchema.fields.map(_ => "?").mkString(",")
      if (ignoreNull) {
        s"""INSERT INTO $table ($columns) VALUES ($placeholders) AS data_new
           |ON DUPLICATE KEY UPDATE
           |${columns.split(",").map(col => s"$col=IF(data_new.$col is null,$table.$col,data_new.$col)").mkString(",")}
           |""".stripMargin
      } else {
        s"""INSERT INTO $table ($columns) VALUES ($placeholders)
           |ON DUPLICATE KEY UPDATE
           |${columns.split(",").map(col => s"$col=VALUES($col)").mkString(",")}
           |""".stripMargin
      }
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

  def upsert(rawDF: DataFrame, database: String, tableName: String, ignoreNull: Boolean = false)(implicit spark: SparkSession): Unit = {
    var df = rawDF
    for (elem <- df.schema.fields) {
      if (elem.dataType == NullType) {
        df = df.withColumn(elem.name, col(elem.name).cast(ShortType))
      }
    }

    df.write
      .format("jdbc")
      .mode(SaveMode.Append)
      .option("driver", "com.mysql.jdbc.Driver")
      .option("url", spark.conf.get(s"spark.job.mysql.${database}.url"))
      .option("user", spark.conf.get(s"spark.job.mysql.${database}.username"))
      .option("password", spark.conf.get(s"spark.job.mysql.${database}.password"))
      .option("dbtable", tableName)
      .option("useSSL", "false")
      .option("showSql", "false")
      .option("numPartitions", "1")
      .option("ignoreNull", ignoreNull)
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

    val database = "test"
    val arr = Array((1,11,"name1",11111),(2,22,"name2",22222))
    val df = spark.sparkContext.parallelize(arr)
      .toDF("key_one", "key_two", "val_one", "val_two")

    MysqlUtils.upsert(df, database, "test_unique_key")
    //MysqlUtils.upsert(df, database, "test_unique_key", true) ignoreNull=true 忽略df里为null的列，不更新
    spark.close()

  }
}
```
test_unique_key表结构
```sql
CREATE TABLE `test_unique_key` (
  `key_one` int(11) NOT NULL DEFAULT '0',
  `key_two` int(11) NOT NULL DEFAULT '0',
  `val_one` varchar(50) DEFAULT NULL,
  `val_two` int(11) NOT NULL DEFAULT '0',
  UNIQUE KEY `uk` (`key_one`,`key_two`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='test';
```
## 参考
[csdn_Spark Upsert写入Mysql(scala增强) 无需依赖](https://blog.csdn.net/qq_18453581/article/details/125907861 "csdn_Spark Upsert写入Mysql(scala增强) 无需依赖")


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/spark-mysql-upsert/  

