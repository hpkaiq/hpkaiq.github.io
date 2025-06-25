# Spark通过Jdbc读取Doris Varchar类型长度异常


spark内通过jdbc方式读取阿里云selectdb（doris）数据，varchar类型长度不足85时总是自动填充空格至85长度，例如表name varchar(20) 字段，name值为 'xxx'，spark内通过jdbc方式读出来显示'xxx'后补空格至总长度85位，在spark里length(name)也是85。

<!--more-->

## 1. 自定义 MySQL Dialect 类

版本：

```xml
  <scala.version>2.12.18</scala.version>
  <spark.version>3.5.3</spark.version>
```

自定义Dialect类：

```scala
import org.apache.spark.sql.execution.datasources.jdbc.{JDBCOptions, JdbcUtils}
import org.apache.spark.sql.jdbc.{JdbcDialect, JdbcDialects, JdbcType}
import org.apache.spark.sql.types._
import java.sql.{Connection, Types}
import scala.collection.mutable.ArrayBuilder

object MyDorisDialect {
  def useMyJdbcDialect(jdbcUrl: String): Unit = {
    // 将当前的 JdbcDialect 对象unregistered掉
    JdbcDialects.unregisterDialect(JdbcDialects.get(jdbcUrl))
    if (jdbcUrl.contains("jdbc:mysql")) {
      JdbcDialects.registerDialect(new CustomMySqlJdbcDialect)
    } else if (jdbcUrl.contains("jdbc:postgresql")) {
    } else if (jdbcUrl.contains("jdbc:sqlserver")) {
    } else if (jdbcUrl.contains("jdbc:oracle")) {
    } else if (jdbcUrl.contains("jdbc:informix")) {
    }
  }
}
class CustomMySqlJdbcDialect extends JdbcDialect {
  override def quoteIdentifier(colName: String): String = {
    s"`$colName`"
  }
  override def schemasExists(conn: Connection, options: JDBCOptions, schema: String): Boolean = {
    listSchemas(conn, options).exists(_.head == schema)
  }
  override def listSchemas(conn: Connection, options: JDBCOptions): Array[Array[String]] = {
    val schemaBuilder = ArrayBuilder.make[Array[String]]
    try {
      JdbcUtils.executeQuery(conn, options, "SHOW SCHEMAS") { rs =>
        while (rs.next()) {
          schemaBuilder += Array(rs.getString("Database"))
        }
      }
    } catch {
      case _: Exception =>
        logWarning("Cannot show schemas.")
    }
    schemaBuilder.result
  }
  override def isCascadingTruncateTable(): Option[Boolean] = Some(false)
  override def canHandle(url: String): Boolean = url.startsWith("jdbc:mysql")
  override def getCatalystType(
                                sqlType: Int, typeName: String, size: Int, md: MetadataBuilder): Option[DataType] = {
    if (sqlType == Types.VARBINARY && typeName.equals("BIT") && size != 1) {
      // MariaDB connector behaviour
      // This could instead be a BinaryType if we'd rather return bit-vectors of up to 64 bits as
      // byte arrays instead of longs.
      md.putLong("binarylong", 1)
      Option(LongType)
    } else if (sqlType == Types.BIT && size > 1) {
      // MySQL connector behaviour
      md.putLong("binarylong", 1)
      Option(LongType)
    } else if (sqlType == Types.BIT && typeName.equals("TINYINT")) {
      Option(BooleanType)
    } else if ("TINYTEXT".equalsIgnoreCase(typeName)) {
      // TINYTEXT is Types.VARCHAR(63) from mysql jdbc, but keep it AS-IS for historical reason
      Some(StringType)
    } else if (sqlType == Types.VARCHAR && typeName.equals("JSON")) {
      // Some MySQL JDBC drivers converts JSON type into Types.VARCHAR with a precision of -1.
      // Explicitly converts it into StringType here.
      Some(StringType)
    } else if (sqlType == Types.VARCHAR || sqlType == Types.CHAR) {
      Some(StringType)
    } else if (sqlType == Types.TINYINT) {
      if (md.build().getBoolean("isSigned")) {
        Some(ByteType)
      } else {
        Some(ShortType)
      }
    } else if (sqlType == Types.TIMESTAMP || sqlType == Types.DATE || sqlType == -101 || sqlType == -102) {
      // 将不支持的 Timestamp with local Timezone 以TimestampType形式返回
      // Some(TimestampType)
      Some(StringType)
    } else None
  }
  /**
   * 从 Spark(DataType) 到 MySQL(SQLType) 的数据类型映射
   *
   * @param dt
   * @return
   */
  override def getJDBCType(dt: DataType): Option[JdbcType] = {
    dt match {
      case IntegerType => Option(JdbcType("INTEGER", java.sql.Types.INTEGER))
      case LongType => Option(JdbcType("BIGINT", java.sql.Types.BIGINT))
      case DoubleType => Option(JdbcType("DOUBLE PRECISION", java.sql.Types.DOUBLE))
      case FloatType => Option(JdbcType("REAL", java.sql.Types.FLOAT))
      case ShortType => Option(JdbcType("INTEGER", java.sql.Types.SMALLINT))
      case ByteType => Option(JdbcType("BYTE", java.sql.Types.TINYINT))
      case BooleanType => Option(JdbcType("BIT(1)", java.sql.Types.BIT))
      case StringType => Option(JdbcType("TEXT", java.sql.Types.CLOB))
      case BinaryType => Option(JdbcType("BLOB", java.sql.Types.BLOB))
      case TimestampType => Option(JdbcType("TIMESTAMP", java.sql.Types.TIMESTAMP))
      case DateType => Option(JdbcType("DATE", java.sql.Types.DATE))
      case t: DecimalType => Option(JdbcType(s"DECIMAL(${t.precision},${t.scale})", java.sql.Types.DECIMAL))
      case _ => JdbcUtils.getCommonJDBCType(dt)
    }
  }
}
```

## 2.使用

```scala
    MyDorisDialect.useMyJdbcDialect(spark.conf.get(s"spark.job.doris.jdbcurl"))
    spark.read.format("jdbc")
      .option("driver", "com.mysql.jdbc.Driver")
      .option("url", spark.conf.get(s"spark.job.doris.jdbcurl"))
      .option("dbtable", s"($sql) as df")
      .option("user", spark.conf.get(s"spark.job.doris.username"))
      .option("password", spark.conf.get(s"spark.job.doris.password"))
      .load()
	  .show()
```



## 3.参考

1. [jdbc获取元数据类型问题 · apache/doris · Discussion #24809](https://github.com/apache/doris/discussions/24809)
2. [bigdata-learning-notes/note/spark/Spark读取JDBC数据Time类型字段异常解析.md at master · kinoxyz1/bigdata-learning-notes](https://github.com/kinoxyz1/bigdata-learning-notes/blob/master/note/spark/Spark读取JDBC数据Time类型字段异常解析.md)


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/spark-jdbc-doris-dialect/  

