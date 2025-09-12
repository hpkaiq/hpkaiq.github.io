# 封装mysql Jdbc驱动使datagrip支持doris多catalog


希望通过datagrip展示多catalog，在sql编辑器中多catalog跨表查询时也能提示多catalog表中的元数据。

<!--more-->

官方关于datagrip使用mysql jdbc连接doris的说明如下：Database 可以用于区别 internal catalog 和 external catalog，如仅填写 Database 名称，则当前数据源默认连接 internal catalog，如填写格式为 catalog.db，则当前数据源默认连接 Database 中所填写的 catalog，DataGrip 中展示的库表也为所连接 catalog 中的库表，以此可以使用 DataGrip 的 MySQL 数据源来创建多个 Doris 数据源来管理 Doris 中不同的 Catalog。



主要思路是重写 Connection 中 getMetaData()方法，返回多catalog的元信息，setCatalog()时调用switch catalog切换catalog上下文。

在DatabaseMetaData中重写 getCatalogs()、getSchemas(String catalog, String schemaPattern)、getTables(String catalog, String schemaPattern, String tableNamePattern, String[] types)、getColumns(String catalog, String schemaPattern, String tableNamePattern, String columnNamePattern)等方法。

部分代码如下，源码请查看[hpkaiq/doris-jdbc-wrapper: 封装mysql jdbc驱动，使datagrip能显示多catalog](https://github.com/hpkaiq/doris-jdbc-wrapper)。

从github release下载jar包或下载源码打jar包，在datagrip中新建驱动程序，自定义jar包，使用`io.github.hpkaiq.dorisjdbc.DorisDriver`主类，再新建数据源，填写连接形如`jdbc:doris://127.0.0.1:9030`。

**注意，本驱动仅在datagrip中测试使用，其他环境未经测试，请不要在其他环境使用，以免造成不必要的数据损失。**

## 1. DorisDriver

```java
package io.github.hpkaiq.dorisjdbc;

import java.sql.*;
import java.util.Properties;
import java.util.logging.Logger;

public final class DorisDriver implements Driver {
    private static final String URL_PREFIX = "jdbc:doris://";
    private static final Driver MYSQL;

    static {
        try {
            MYSQL = new com.mysql.cj.jdbc.Driver();
            DriverManager.registerDriver(new DorisDriver());
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public Connection connect(String url, Properties info) throws SQLException {
        if (!acceptsURL(url)) return null;
        String mysqlUrl = "jdbc:mysql://" + url.substring(URL_PREFIX.length());
        Connection raw = MYSQL.connect(mysqlUrl, info);
        return new DorisConnection(raw);
    }

    @Override public boolean acceptsURL(String url) { return url != null && url.startsWith(URL_PREFIX); }
    @Override public DriverPropertyInfo[] getPropertyInfo(String url, Properties info) { return new DriverPropertyInfo[0]; }
    @Override public int getMajorVersion() { return 1; }
    @Override public int getMinorVersion() { return 0; }
    @Override public boolean jdbcCompliant() { return false; }
    @Override public Logger getParentLogger() { return Logger.getGlobal(); }
}

```

## 2. DorisConnection

```java
package io.github.hpkaiq.dorisjdbc;


import java.sql.*;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.Executor;

public class DorisConnection implements Connection {
    private final Connection delegate;
    private volatile String currentCatalog = "internal";

    DorisConnection(Connection delegate) {
        this.delegate = delegate;
    }

    @Override
    public void setCatalog(String catalog) throws SQLException {
        checkOpen();
        if (catalog != null && !catalog.equalsIgnoreCase(this.currentCatalog)) {
            try (Statement stmt = delegate.createStatement()) {
                stmt.execute("SWITCH `" + catalog + "`");
            }
            this.currentCatalog = catalog;
        }
    }

    private void checkOpen()
            throws SQLException {
        if (isClosed()) {
            throw new SQLException("Connection is closed");
        }
    }

    @Override
    public String getCatalog() {
        return currentCatalog;
    }

    @Override
    public void setTransactionIsolation(int level) throws SQLException {
        delegate.setTransactionIsolation(level);
    }

    @Override
    public int getTransactionIsolation() throws SQLException {
        return delegate.getTransactionIsolation();
    }

    @Override
    public SQLWarning getWarnings() throws SQLException {
        return delegate.getWarnings();
    }

    @Override
    public void clearWarnings() throws SQLException {
        delegate.clearWarnings();
    }

    @Override
    public Statement createStatement(int resultSetType, int resultSetConcurrency) throws SQLException {
        return delegate.createStatement(resultSetType, resultSetConcurrency);
    }

    @Override
    public PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency) throws SQLException {
        return delegate.prepareStatement(sql, resultSetType, resultSetConcurrency);
    }

    @Override
    public CallableStatement prepareCall(String sql, int resultSetType, int resultSetConcurrency) throws SQLException {
        return delegate.prepareCall(sql, resultSetType, resultSetConcurrency);
    }

    @Override
    public Map<String, Class<?>> getTypeMap() throws SQLException {
        return delegate.getTypeMap();
    }

    @Override
    public void setTypeMap(Map<String, Class<?>> map) throws SQLException {
        delegate.setTypeMap(map);
    }

    @Override
    public void setHoldability(int holdability) throws SQLException {
        delegate.setHoldability(holdability);
    }

    @Override
    public int getHoldability() throws SQLException {
        return delegate.getHoldability();
    }

    @Override
    public Savepoint setSavepoint() throws SQLException {
        return delegate.setSavepoint();
    }

    @Override
    public Savepoint setSavepoint(String name) throws SQLException {
        return delegate.setSavepoint(name);
    }

    @Override
    public void rollback(Savepoint savepoint) throws SQLException {
        delegate.rollback(savepoint);
    }

    @Override
    public void releaseSavepoint(Savepoint savepoint) throws SQLException {
        delegate.releaseSavepoint(savepoint);
    }

    @Override
    public Statement createStatement(int resultSetType, int resultSetConcurrency, int resultSetHoldability) throws SQLException {
        return delegate.createStatement(resultSetType, resultSetConcurrency, resultSetHoldability);
    }

    @Override
    public PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency, int resultSetHoldability) throws SQLException {
        return delegate.prepareStatement(sql, resultSetType, resultSetConcurrency, resultSetHoldability);
    }

    @Override
    public CallableStatement prepareCall(String sql, int resultSetType, int resultSetConcurrency, int resultSetHoldability) throws SQLException {
        return delegate.prepareCall(sql, resultSetType, resultSetConcurrency, resultSetHoldability);
    }

    @Override
    public PreparedStatement prepareStatement(String sql, int autoGeneratedKeys) throws SQLException {
        return delegate.prepareStatement(sql, autoGeneratedKeys);
    }

    @Override
    public PreparedStatement prepareStatement(String sql, int[] columnIndexes) throws SQLException {
        return delegate.prepareStatement(sql, columnIndexes);
    }

    @Override
    public PreparedStatement prepareStatement(String sql, String[] columnNames) throws SQLException {
        return delegate.prepareStatement(sql, columnNames);
    }

    @Override
    public Clob createClob() throws SQLException {
        return delegate.createClob();
    }

    @Override
    public Blob createBlob() throws SQLException {
        return delegate.createBlob();
    }

    @Override
    public NClob createNClob() throws SQLException {
        return delegate.createNClob();
    }

    @Override
    public SQLXML createSQLXML() throws SQLException {
        return delegate.createSQLXML();
    }

    @Override
    public boolean isValid(int timeout) throws SQLException {
        return delegate.isValid(timeout);
    }

    @Override
    public void setClientInfo(String name, String value) throws SQLClientInfoException {
        delegate.setClientInfo(name, value);
    }

    @Override
    public void setClientInfo(Properties properties) throws SQLClientInfoException {
        delegate.setClientInfo(properties);
    }

    @Override
    public String getClientInfo(String name) throws SQLException {
        return delegate.getClientInfo(name);
    }

    @Override
    public Properties getClientInfo() throws SQLException {
        return delegate.getClientInfo();
    }

    @Override
    public Array createArrayOf(String typeName, Object[] elements) throws SQLException {
        return delegate.createArrayOf(typeName, elements);
    }

    @Override
    public Struct createStruct(String typeName, Object[] attributes) throws SQLException {
        return delegate.createStruct(typeName, attributes);
    }

    @Override
    public void setSchema(String schema) throws SQLException {
        delegate.setSchema(schema);
    }

    @Override
    public String getSchema() throws SQLException {
        return delegate.getSchema();
    }

    @Override
    public void abort(Executor executor) throws SQLException {
        delegate.abort(executor);
    }

    @Override
    public void setNetworkTimeout(Executor executor, int milliseconds) throws SQLException {
        delegate.setNetworkTimeout(executor, milliseconds);
    }

    @Override
    public int getNetworkTimeout() throws SQLException {
        return delegate.getNetworkTimeout();
    }

    @Override
    public DatabaseMetaData getMetaData() throws SQLException {
        return new DorisDatabaseMetaData(this, delegate.getMetaData());
    }

    @Override
    public void setReadOnly(boolean readOnly) throws SQLException {
        delegate.setReadOnly(readOnly);
    }

    @Override
    public boolean isReadOnly() throws SQLException {
        return delegate.isReadOnly();
    }

    @Override
    public Statement createStatement() throws SQLException {
        return new DorisStatement(delegate.createStatement(),this);
    }

    @Override
    public PreparedStatement prepareStatement(String sql) throws SQLException {
        return new DorisPreparedStatement(delegate.prepareStatement(sql),this);
    }

    @Override
    public CallableStatement prepareCall(String sql) throws SQLException {
        return delegate.prepareCall(sql);
    }

    @Override
    public String nativeSQL(String sql) throws SQLException {
        return delegate.nativeSQL(sql);
    }

    @Override
    public void setAutoCommit(boolean autoCommit) throws SQLException {
        delegate.setAutoCommit(autoCommit);
    }

    @Override
    public boolean getAutoCommit() throws SQLException {
        return delegate.getAutoCommit();
    }

    @Override
    public void commit() throws SQLException {
        delegate.commit();
    }

    @Override
    public void rollback() throws SQLException {
        delegate.rollback();
    }

    @Override
    public void close() throws SQLException {
        delegate.close();
    }

    @Override
    public boolean isClosed() throws SQLException {
        return delegate.isClosed();
    }

    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException {
        return delegate.unwrap(iface);
    }

    @Override
    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        return delegate.isWrapperFor(iface);
    }

}

```

## 3. DorisDatabaseMetaData

```java
package io.github.hpkaiq.dorisjdbc;

import javax.sql.rowset.CachedRowSet;
import javax.sql.rowset.RowSetMetaDataImpl;
import javax.sql.rowset.RowSetProvider;
import java.sql.*;

class DorisDatabaseMetaData implements DatabaseMetaData {
    private final DorisConnection conn;
    private final DatabaseMetaData delegate;

    DorisDatabaseMetaData(DorisConnection conn, DatabaseMetaData delegate) {
        this.conn = conn;
        this.delegate = delegate;
    }

    @Override
    public ResultSet getCatalogs() throws SQLException {
        CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
        crs.setMetaData(buildMeta("TABLE_CAT"));

        Statement st = conn.createStatement();
        ResultSet showCatalogs = st.executeQuery("SHOW CATALOGS");
        while (showCatalogs.next()) {
            String cat = showCatalogs.getString("CatalogName");
            crs.moveToInsertRow();
            crs.updateString("TABLE_CAT", cat);
            crs.insertRow();
            crs.moveToCurrentRow();
        }
        crs.beforeFirst();
        return crs;
    }

    private static RowSetMetaDataImpl buildMeta(String... cols) throws SQLException {
        RowSetMetaDataImpl meta = new RowSetMetaDataImpl();
        meta.setColumnCount(cols.length);
        for (int i = 0; i < cols.length; i++) {
            int colIndex = i + 1;
            meta.setColumnName(colIndex, cols[i]);
            meta.setColumnType(colIndex, java.sql.Types.VARCHAR); // 默认给 VARCHAR，避免 NPE
            meta.setNullable(colIndex, ResultSetMetaData.columnNullable);
        }
        return meta;
    }

    @Override
    public ResultSet getTableTypes() throws SQLException {
        return delegate.getTableTypes();
    }

    @Override
    public ResultSet getSchemas(String catalog, String schemaPattern) throws SQLException {
        ResultSet catalogs = getCatalogs();
        Statement st = conn.createStatement();
        CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
        crs.setMetaData(buildMeta("TABLE_SCHEM", "TABLE_CATALOG"));
        while (catalogs.next()) {
            String cl = catalogs.getString("TABLE_CAT");
            if (catalog != null && !catalog.isEmpty() && !cl.equalsIgnoreCase(catalog)) {
                continue;
            }
            ResultSet rs = st.executeQuery("SHOW DATABASES FROM `" + cl + "`");
            while (rs.next()) {
                String db = rs.getString(1);
                crs.moveToInsertRow();
                crs.updateString("TABLE_SCHEM", db);
                crs.updateString("TABLE_CATALOG", cl);
                crs.insertRow();
                crs.moveToCurrentRow();
            }
        }
        crs.beforeFirst();
        return crs;
    }

    @Override
    public ResultSet getTables(String catalog, String schemaPattern, String tableNamePattern, String[] types) throws SQLException {
        if (schemaPattern.contains("\\")) {
            schemaPattern = schemaPattern.replace("\\", "");
        }
        final String useCatalog = (catalog != null && !catalog.isEmpty()) ? catalog : conn.getCatalog();
        Statement st = conn.createStatement();
        String query = String.format("SHOW FULL TABLES FROM `%s`.`%s`", useCatalog, schemaPattern);
        ResultSet rs = st.executeQuery(query);

        CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
        crs.setMetaData(buildMeta("TABLE_CAT", "TABLE_SCHEM", "TABLE_NAME", "TABLE_TYPE", "REMARKS"));

        while (rs.next()) {
            String tableName = rs.getString(1);
            String tableType = rs.getString(2);
            crs.moveToInsertRow();
            crs.updateString("TABLE_CAT", useCatalog);
            crs.updateString("TABLE_SCHEM", schemaPattern);
            crs.updateString("TABLE_NAME", tableName);
            crs.updateString("TABLE_TYPE", tableType);
            crs.updateString("REMARKS", "");
            crs.insertRow();
            crs.moveToCurrentRow();
        }
        crs.beforeFirst();
        return crs;
    }

    @Override
    public ResultSet getSchemas() throws SQLException {
        return getSchemas(null, null);
    }

    @Override
    public ResultSet getColumns(String catalog, String schemaPattern, String tableNamePattern, String columnNamePattern) throws SQLException {
        String useCatalog = (catalog != null && !catalog.isEmpty()) ? catalog : conn.getCatalog();
        if (schemaPattern.contains("\\")) {
            schemaPattern = schemaPattern.replace("\\", "");
        }
        if (tableNamePattern.contains("\\")) {
            tableNamePattern = tableNamePattern.replace("\\", "");
        }
        if (schemaPattern.contains(".")) {
            schemaPattern = schemaPattern.split("\\.")[1];
        }
        Statement st = conn.createStatement();
        String query = String.format("show full columns from `%s`.`%s`.`%s`", useCatalog, schemaPattern, tableNamePattern);
        ResultSet rs = st.executeQuery(query);
        CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
        crs.setMetaData(buildMeta(
                "TABLE_CAT", "TABLE_SCHEM", "TABLE_NAME",
                "COLUMN_NAME", "DATA_TYPE", "TYPE_NAME", "COLUMN_SIZE", "DECIMAL_DIGITS", "COLUMN_DEF",
                "IS_NULLABLE", "NULLABLE", "REMARKS", "ORDINAL_POSITION"
        ));
        int pos = 1;
        while (rs.next()) {
            String colName = rs.getString("Field");
            String dorisType = rs.getString("Type");
            String nullable = rs.getString("Null");
            String defVal = rs.getString("Default");
            String comment = rs.getString("Comment");

            int jdbcType = TypeMapper.toJdbcType(dorisType);
            int columnSize = TypeMapper.extractLength(dorisType);
            int scale = TypeMapper.extractScale(dorisType);
            int nullableType = TypeMapper.extractNullAble(nullable);
            String typeName = TypeMapper.baseTypeName(dorisType);
            crs.moveToInsertRow();
            crs.updateString("TABLE_CAT", useCatalog);
            crs.updateString("TABLE_SCHEM", schemaPattern);
            crs.updateString("TABLE_NAME", tableNamePattern);
            crs.updateString("COLUMN_NAME", colName);
            crs.updateInt("DATA_TYPE", jdbcType);
            crs.updateString("TYPE_NAME", typeName);
            crs.updateInt("COLUMN_SIZE", columnSize);
            crs.updateInt("DECIMAL_DIGITS", scale);
            crs.updateString("COLUMN_DEF", defVal);
            crs.updateString("IS_NULLABLE", nullable);
            crs.updateInt("NULLABLE", nullableType);
            crs.updateString("REMARKS", comment);
            crs.updateInt("ORDINAL_POSITION", pos++);
            crs.insertRow();
            crs.moveToCurrentRow();
        }
        crs.beforeFirst();
        return crs;
    }

    @Override
    public boolean supportsStoredFunctionsUsingCallSyntax() throws SQLException {
        return delegate.supportsStoredFunctionsUsingCallSyntax();
    }

    @Override
    public boolean autoCommitFailureClosesAllResultSets() throws SQLException {
        return delegate.autoCommitFailureClosesAllResultSets();
    }

    @Override
    public ResultSet getClientInfoProperties() throws SQLException {
        return delegate.getClientInfoProperties();
    }

    @Override
    public ResultSet getFunctions(String catalog, String schemaPattern, String functionNamePattern) throws SQLException {
        return delegate.getFunctions(catalog, schemaPattern, functionNamePattern);
    }

    @Override
    public ResultSet getFunctionColumns(String catalog, String schemaPattern, String functionNamePattern, String columnNamePattern) throws SQLException {
        CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
        crs.setMetaData(buildMeta(
                "FUNCTION_CAT", "FUNCTION_SCHEM", "FUNCTION_NAME",
                "COLUMN_NAME", "COLUMN_TYPE", "DATA_TYPE",
                "TYPE_NAME", "PRECISION", "LENGTH", "SCALE",
                "RADIX", "NULLABLE", "REMARKS"
        ));
        crs.beforeFirst();
        return crs;
    }

    @Override
    public ResultSet getPseudoColumns(String catalog, String schemaPattern, String tableNamePattern, String columnNamePattern) throws SQLException {
        CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
        crs.setMetaData(buildMeta(
                "TABLE_CAT", "TABLE_SCHEM", "TABLE_NAME", "COLUMN_NAME",
                "DATA_TYPE", "COLUMN_SIZE", "DECIMAL_DIGITS", "NUM_PREC_RADIX",
                "COLUMN_USAGE", "REMARKS", "CHAR_OCTET_LENGTH", "IS_NULLABLE"
        ));
        crs.beforeFirst();
        return crs;
    }

    @Override
    public boolean generatedKeyAlwaysReturned() throws SQLException {
        return delegate.generatedKeyAlwaysReturned();
    }

    @Override
    public boolean allProceduresAreCallable() throws SQLException {
        return delegate.allProceduresAreCallable();
    }

    @Override
    public boolean allTablesAreSelectable() throws SQLException {
        return delegate.allTablesAreSelectable();
    }

    @Override
    public String getURL() throws SQLException {
        return delegate.getURL();
    }

    @Override
    public String getUserName() throws SQLException {
        return delegate.getUserName();
    }

    @Override
    public boolean isReadOnly() throws SQLException {
        return delegate.isReadOnly();
    }

    @Override
    public boolean nullsAreSortedHigh() throws SQLException {
        return delegate.nullsAreSortedHigh();
    }

    @Override
    public boolean nullsAreSortedLow() throws SQLException {
        return delegate.nullsAreSortedLow();
    }

    @Override
    public boolean nullsAreSortedAtStart() throws SQLException {
        return delegate.nullsAreSortedAtStart();
    }

    @Override
    public boolean nullsAreSortedAtEnd() throws SQLException {
        return delegate.nullsAreSortedAtEnd();
    }

    @Override
    public String getDatabaseProductName() throws SQLException {
        return "Doris";
    }

    @Override
    public String getDatabaseProductVersion() throws SQLException {
        return delegate.getDatabaseProductVersion();
    }

    @Override
    public String getDriverName() throws SQLException {
        return delegate.getDriverName();
    }

    @Override
    public String getDriverVersion() throws SQLException {
        return delegate.getDriverVersion();
    }

    @Override
    public int getDriverMajorVersion() {
        return delegate.getDriverMajorVersion();
    }

    @Override
    public int getDriverMinorVersion() {
        return delegate.getDriverMinorVersion();
    }

    @Override
    public boolean usesLocalFiles() throws SQLException {
        return delegate.usesLocalFiles();
    }

    @Override
    public boolean usesLocalFilePerTable() throws SQLException {
        return delegate.usesLocalFilePerTable();
    }

    @Override
    public boolean supportsMixedCaseIdentifiers() throws SQLException {
        return delegate.supportsMixedCaseIdentifiers();
    }

    @Override
    public boolean storesUpperCaseIdentifiers() throws SQLException {
        return delegate.storesUpperCaseIdentifiers();
    }

    @Override
    public boolean storesLowerCaseIdentifiers() throws SQLException {
        return delegate.storesLowerCaseIdentifiers();
    }

    @Override
    public boolean storesMixedCaseIdentifiers() throws SQLException {
        return delegate.storesMixedCaseIdentifiers();
    }

    @Override
    public boolean supportsMixedCaseQuotedIdentifiers() throws SQLException {
        return delegate.supportsMixedCaseQuotedIdentifiers();
    }

    @Override
    public boolean storesUpperCaseQuotedIdentifiers() throws SQLException {
        return delegate.storesUpperCaseQuotedIdentifiers();
    }

    @Override
    public boolean storesLowerCaseQuotedIdentifiers() throws SQLException {
        return delegate.storesLowerCaseQuotedIdentifiers();
    }

    @Override
    public boolean storesMixedCaseQuotedIdentifiers() throws SQLException {
        return delegate.storesMixedCaseQuotedIdentifiers();
    }

    @Override
    public String getIdentifierQuoteString() throws SQLException {
        return delegate.getIdentifierQuoteString();
    }

    @Override
    public String getSQLKeywords() throws SQLException {
        return delegate.getSQLKeywords();
    }

    @Override
    public String getNumericFunctions() throws SQLException {
        return delegate.getNumericFunctions();
    }

    @Override
    public String getStringFunctions() throws SQLException {
        return delegate.getStringFunctions();
    }

    @Override
    public String getSystemFunctions() throws SQLException {
        return delegate.getSystemFunctions();
    }

    @Override
    public String getTimeDateFunctions() throws SQLException {
        return delegate.getTimeDateFunctions();
    }

    @Override
    public String getSearchStringEscape() throws SQLException {
        return delegate.getSearchStringEscape();
    }

    @Override
    public String getExtraNameCharacters() throws SQLException {
        return delegate.getExtraNameCharacters();
    }

    @Override
    public boolean supportsAlterTableWithAddColumn() throws SQLException {
        return delegate.supportsAlterTableWithAddColumn();
    }

    @Override
    public boolean supportsAlterTableWithDropColumn() throws SQLException {
        return delegate.supportsAlterTableWithDropColumn();
    }

    @Override
    public boolean supportsColumnAliasing() throws SQLException {
        return delegate.supportsColumnAliasing();
    }

    @Override
    public boolean nullPlusNonNullIsNull() throws SQLException {
        return delegate.nullPlusNonNullIsNull();
    }

    @Override
    public boolean supportsConvert() throws SQLException {
        return delegate.supportsConvert();
    }

    @Override
    public boolean supportsConvert(int fromType, int toType) throws SQLException {
        return delegate.supportsConvert(fromType, toType);
    }

    @Override
    public boolean supportsTableCorrelationNames() throws SQLException {
        return delegate.supportsTableCorrelationNames();
    }

    @Override
    public boolean supportsDifferentTableCorrelationNames() throws SQLException {
        return delegate.supportsDifferentTableCorrelationNames();
    }

    @Override
    public boolean supportsExpressionsInOrderBy() throws SQLException {
        return delegate.supportsExpressionsInOrderBy();
    }

    @Override
    public boolean supportsOrderByUnrelated() throws SQLException {
        return delegate.supportsOrderByUnrelated();
    }

    @Override
    public boolean supportsGroupBy() throws SQLException {
        return delegate.supportsGroupBy();
    }

    @Override
    public boolean supportsGroupByUnrelated() throws SQLException {
        return delegate.supportsGroupByUnrelated();
    }

    @Override
    public boolean supportsGroupByBeyondSelect() throws SQLException {
        return delegate.supportsGroupByBeyondSelect();
    }

    @Override
    public boolean supportsLikeEscapeClause() throws SQLException {
        return delegate.supportsLikeEscapeClause();
    }

    @Override
    public boolean supportsMultipleResultSets() throws SQLException {
        return delegate.supportsMultipleResultSets();
    }

    @Override
    public boolean supportsMultipleTransactions() throws SQLException {
        return delegate.supportsMultipleTransactions();
    }

    @Override
    public boolean supportsNonNullableColumns() throws SQLException {
        return delegate.supportsNonNullableColumns();
    }

    @Override
    public boolean supportsMinimumSQLGrammar() throws SQLException {
        return delegate.supportsMinimumSQLGrammar();
    }

    @Override
    public boolean supportsCoreSQLGrammar() throws SQLException {
        return delegate.supportsCoreSQLGrammar();
    }

    @Override
    public boolean supportsExtendedSQLGrammar() throws SQLException {
        return delegate.supportsExtendedSQLGrammar();
    }

    @Override
    public boolean supportsANSI92EntryLevelSQL() throws SQLException {
        return delegate.supportsANSI92EntryLevelSQL();
    }

    @Override
    public boolean supportsANSI92IntermediateSQL() throws SQLException {
        return delegate.supportsANSI92IntermediateSQL();
    }

    @Override
    public boolean supportsANSI92FullSQL() throws SQLException {
        return delegate.supportsANSI92FullSQL();
    }

    @Override
    public boolean supportsIntegrityEnhancementFacility() throws SQLException {
        return delegate.supportsIntegrityEnhancementFacility();
    }

    @Override
    public boolean supportsOuterJoins() throws SQLException {
        return delegate.supportsOuterJoins();
    }

    @Override
    public boolean supportsFullOuterJoins() throws SQLException {
        return true;
    }

    @Override
    public boolean supportsLimitedOuterJoins() throws SQLException {
        return delegate.supportsLimitedOuterJoins();
    }

    @Override
    public String getSchemaTerm() throws SQLException {
        return "schema";
        //return delegate.getSchemaTerm();
    }

    @Override
    public String getProcedureTerm() throws SQLException {
        return "procedure";
        //return delegate.getProcedureTerm();
    }

    @Override
    public String getCatalogTerm() throws SQLException {
        return "catalog";
        //return delegate.getCatalogTerm();
    }

    @Override
    public boolean isCatalogAtStart() throws SQLException {
        return delegate.isCatalogAtStart();
    }

    @Override
    public String getCatalogSeparator() throws SQLException {
        return delegate.getCatalogSeparator();
    }

    @Override
    public boolean supportsSchemasInDataManipulation() throws SQLException {
        return delegate.supportsSchemasInDataManipulation();
    }

    @Override
    public boolean supportsSchemasInProcedureCalls() throws SQLException {
        return delegate.supportsSchemasInProcedureCalls();
    }

    @Override
    public boolean supportsSchemasInTableDefinitions() throws SQLException {
        return true;
        //return delegate.supportsSchemasInTableDefinitions();
    }

    @Override
    public boolean supportsSchemasInIndexDefinitions() throws SQLException {
        return delegate.supportsSchemasInIndexDefinitions();
    }

    @Override
    public boolean supportsSchemasInPrivilegeDefinitions() throws SQLException {
        return delegate.supportsSchemasInPrivilegeDefinitions();
    }

    @Override
    public boolean supportsCatalogsInDataManipulation() throws SQLException {
        return delegate.supportsCatalogsInDataManipulation();
    }

    @Override
    public boolean supportsCatalogsInProcedureCalls() throws SQLException {
        return delegate.supportsCatalogsInProcedureCalls();
    }

    @Override
    public boolean supportsCatalogsInTableDefinitions() throws SQLException {
        return true;
        //return delegate.supportsCatalogsInTableDefinitions();
    }

    @Override
    public boolean supportsCatalogsInIndexDefinitions() throws SQLException {
        return delegate.supportsCatalogsInIndexDefinitions();
    }

    @Override
    public boolean supportsCatalogsInPrivilegeDefinitions() throws SQLException {
        return delegate.supportsCatalogsInPrivilegeDefinitions();
    }

    @Override
    public boolean supportsPositionedDelete() throws SQLException {
        return delegate.supportsPositionedDelete();
    }

    @Override
    public boolean supportsPositionedUpdate() throws SQLException {
        return delegate.supportsPositionedUpdate();
    }

    @Override
    public boolean supportsSelectForUpdate() throws SQLException {
        return delegate.supportsSelectForUpdate();
    }

    @Override
    public boolean supportsStoredProcedures() throws SQLException {
        return delegate.supportsStoredProcedures();
    }

    @Override
    public boolean supportsSubqueriesInComparisons() throws SQLException {
        return delegate.supportsSubqueriesInComparisons();
    }

    @Override
    public boolean supportsSubqueriesInExists() throws SQLException {
        return delegate.supportsSubqueriesInExists();
    }

    @Override
    public boolean supportsSubqueriesInIns() throws SQLException {
        return delegate.supportsSubqueriesInIns();
    }

    @Override
    public boolean supportsSubqueriesInQuantifieds() throws SQLException {
        return delegate.supportsSubqueriesInQuantifieds();
    }

    @Override
    public boolean supportsCorrelatedSubqueries() throws SQLException {
        return delegate.supportsCorrelatedSubqueries();
    }

    @Override
    public boolean supportsUnion() throws SQLException {
        return delegate.supportsUnion();
    }

    @Override
    public boolean supportsUnionAll() throws SQLException {
        return delegate.supportsUnionAll();
    }

    @Override
    public boolean supportsOpenCursorsAcrossCommit() throws SQLException {
        return delegate.supportsOpenCursorsAcrossCommit();
    }

    @Override
    public boolean supportsOpenCursorsAcrossRollback() throws SQLException {
        return delegate.supportsOpenCursorsAcrossRollback();
    }

    @Override
    public boolean supportsOpenStatementsAcrossCommit() throws SQLException {
        return delegate.supportsOpenStatementsAcrossCommit();
    }

    @Override
    public boolean supportsOpenStatementsAcrossRollback() throws SQLException {
        return delegate.supportsOpenStatementsAcrossRollback();
    }

    @Override
    public int getMaxBinaryLiteralLength() throws SQLException {
        return delegate.getMaxBinaryLiteralLength();
    }

    @Override
    public int getMaxCharLiteralLength() throws SQLException {
        return delegate.getMaxCharLiteralLength();
    }

    @Override
    public int getMaxColumnNameLength() throws SQLException {
        return delegate.getMaxColumnNameLength();
    }

    @Override
    public int getMaxColumnsInGroupBy() throws SQLException {
        return delegate.getMaxColumnsInGroupBy();
    }

    @Override
    public int getMaxColumnsInIndex() throws SQLException {
        return delegate.getMaxColumnsInIndex();
    }

    @Override
    public int getMaxColumnsInOrderBy() throws SQLException {
        return delegate.getMaxColumnsInOrderBy();
    }

    @Override
    public int getMaxColumnsInSelect() throws SQLException {
        return delegate.getMaxColumnsInSelect();
    }

    @Override
    public int getMaxColumnsInTable() throws SQLException {
        return delegate.getMaxColumnsInTable();
    }

    @Override
    public int getMaxConnections() throws SQLException {
        return delegate.getMaxConnections();
    }

    @Override
    public int getMaxCursorNameLength() throws SQLException {
        return delegate.getMaxCursorNameLength();
    }

    @Override
    public int getMaxIndexLength() throws SQLException {
        return delegate.getMaxIndexLength();
    }

    @Override
    public int getMaxSchemaNameLength() throws SQLException {
        return delegate.getMaxSchemaNameLength();
    }

    @Override
    public int getMaxProcedureNameLength() throws SQLException {
        return delegate.getMaxProcedureNameLength();
    }

    @Override
    public int getMaxCatalogNameLength() throws SQLException {
        return delegate.getMaxCatalogNameLength();
    }

    @Override
    public int getMaxRowSize() throws SQLException {
        return delegate.getMaxRowSize();
    }

    @Override
    public boolean doesMaxRowSizeIncludeBlobs() throws SQLException {
        return delegate.doesMaxRowSizeIncludeBlobs();
    }

    @Override
    public int getMaxStatementLength() throws SQLException {
        return delegate.getMaxStatementLength();
    }

    @Override
    public int getMaxStatements() throws SQLException {
        return delegate.getMaxStatements();
    }

    @Override
    public int getMaxTableNameLength() throws SQLException {
        return delegate.getMaxTableNameLength();
    }

    @Override
    public int getMaxTablesInSelect() throws SQLException {
        return delegate.getMaxTablesInSelect();
    }

    @Override
    public int getMaxUserNameLength() throws SQLException {
        return delegate.getMaxUserNameLength();
    }

    @Override
    public int getDefaultTransactionIsolation() throws SQLException {
        return delegate.getDefaultTransactionIsolation();
    }

    @Override
    public boolean supportsTransactions() throws SQLException {
        return delegate.supportsTransactions();
    }

    @Override
    public boolean supportsTransactionIsolationLevel(int level) throws SQLException {
        return delegate.supportsTransactionIsolationLevel(level);
    }

    @Override
    public boolean supportsDataDefinitionAndDataManipulationTransactions() throws SQLException {
        return delegate.supportsDataDefinitionAndDataManipulationTransactions();
    }

    @Override
    public boolean supportsDataManipulationTransactionsOnly() throws SQLException {
        return delegate.supportsDataManipulationTransactionsOnly();
    }

    @Override
    public boolean dataDefinitionCausesTransactionCommit() throws SQLException {
        return delegate.dataDefinitionCausesTransactionCommit();
    }

    @Override
    public boolean dataDefinitionIgnoredInTransactions() throws SQLException {
        return delegate.dataDefinitionIgnoredInTransactions();
    }

    @Override
    public ResultSet getProcedures(String catalog, String schemaPattern, String procedureNamePattern) throws SQLException {
        //return delegate.getProcedures(catalog, schemaPattern, procedureNamePattern);
        CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
        crs.setMetaData(buildMeta("PROCEDURE_CAT", "PROCEDURE_SCHEM", "PROCEDURE_NAME",
                "REMARKS", "PROCEDURE_TYPE", "SPECIFIC_NAME"));
        crs.beforeFirst();
        return crs;
    }

    @Override
    public ResultSet getProcedureColumns(String catalog, String schemaPattern, String procedureNamePattern, String columnNamePattern) throws SQLException {
        //return delegate.getProcedureColumns(catalog, schemaPattern, procedureNamePattern, columnNamePattern);
        CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
        crs.setMetaData(buildMeta("PROCEDURE_CAT", "PROCEDURE_SCHEM", "PROCEDURE_NAME",
                "COLUMN_NAME", "COLUMN_TYPE", "DATA_TYPE", "TYPE_NAME",
                "PRECISION", "LENGTH", "SCALE", "RADIX", "NULLABLE", "REMARKS"));
        crs.beforeFirst();
        return crs;
    }

    @Override
    public ResultSet getColumnPrivileges(String catalog, String schema, String table, String columnNamePattern) throws SQLException {
        return delegate.getColumnPrivileges(catalog, schema, table, columnNamePattern);
    }

    @Override
    public ResultSet getTablePrivileges(String catalog, String schemaPattern, String tableNamePattern) throws SQLException {
        return delegate.getTablePrivileges(catalog, schemaPattern, tableNamePattern);
    }

    @Override
    public ResultSet getBestRowIdentifier(String catalog, String schema, String table, int scope, boolean nullable) throws SQLException {
        return delegate.getBestRowIdentifier(catalog, schema, table, scope, nullable);
    }

    @Override
    public ResultSet getVersionColumns(String catalog, String schema, String table) throws SQLException {
        return delegate.getVersionColumns(catalog, schema, table);
    }

    @Override
    public ResultSet getPrimaryKeys(String catalog, String schema, String table) throws SQLException {
        return delegate.getPrimaryKeys(catalog, schema, table);
    }

    @Override
    public ResultSet getImportedKeys(String catalog, String schema, String table) throws SQLException {
        return delegate.getImportedKeys(catalog, schema, table);
    }

    @Override
    public ResultSet getExportedKeys(String catalog, String schema, String table) throws SQLException {
        return delegate.getExportedKeys(catalog, schema, table);
    }

    @Override
    public ResultSet getCrossReference(String parentCatalog, String parentSchema, String parentTable, String foreignCatalog, String foreignSchema, String foreignTable) throws SQLException {
        return delegate.getCrossReference(parentCatalog, parentSchema, parentTable, foreignCatalog, foreignSchema, foreignTable);
    }

    @Override
    public ResultSet getTypeInfo() throws SQLException {
        return delegate.getTypeInfo();
    }

    @Override
    public ResultSet getIndexInfo(String catalog, String schema, String table, boolean unique, boolean approximate) throws SQLException {
        return delegate.getIndexInfo(catalog, schema, table, unique, approximate);
    }

    @Override
    public boolean supportsResultSetType(int type) throws SQLException {
        return delegate.supportsResultSetType(type);
    }

    @Override
    public boolean supportsResultSetConcurrency(int type, int concurrency) throws SQLException {
        return delegate.supportsResultSetConcurrency(type, concurrency);
    }

    @Override
    public boolean ownUpdatesAreVisible(int type) throws SQLException {
        return delegate.ownUpdatesAreVisible(type);
    }

    @Override
    public boolean ownDeletesAreVisible(int type) throws SQLException {
        return delegate.ownDeletesAreVisible(type);
    }

    @Override
    public boolean ownInsertsAreVisible(int type) throws SQLException {
        return delegate.ownInsertsAreVisible(type);
    }

    @Override
    public boolean othersUpdatesAreVisible(int type) throws SQLException {
        return delegate.othersUpdatesAreVisible(type);
    }

    @Override
    public boolean othersDeletesAreVisible(int type) throws SQLException {
        return delegate.othersDeletesAreVisible(type);
    }

    @Override
    public boolean othersInsertsAreVisible(int type) throws SQLException {
        return delegate.othersInsertsAreVisible(type);
    }

    @Override
    public boolean updatesAreDetected(int type) throws SQLException {
        return delegate.updatesAreDetected(type);
    }

    @Override
    public boolean deletesAreDetected(int type) throws SQLException {
        return delegate.deletesAreDetected(type);
    }

    @Override
    public boolean insertsAreDetected(int type) throws SQLException {
        return delegate.insertsAreDetected(type);
    }

    @Override
    public boolean supportsBatchUpdates() throws SQLException {
        return delegate.supportsBatchUpdates();
    }

    @Override
    public ResultSet getUDTs(String catalog, String schemaPattern, String typeNamePattern, int[] types) throws SQLException {
        //return delegate.getUDTs(catalog, schemaPattern, typeNamePattern, types);
        CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
        crs.setMetaData(buildMeta(
                "TYPE_CAT", "TYPE_SCHEM", "TYPE_NAME", "CLASS_NAME", "DATA_TYPE",
                "REMARKS", "BASE_TYPE"
        ));
        crs.beforeFirst();
        return crs;
    }

    @Override
    public Connection getConnection() throws SQLException {
        return delegate.getConnection();
    }

    @Override
    public boolean supportsSavepoints() throws SQLException {
        return delegate.supportsSavepoints();
    }

    @Override
    public boolean supportsNamedParameters() throws SQLException {
        return delegate.supportsNamedParameters();
    }

    @Override
    public boolean supportsMultipleOpenResults() throws SQLException {
        return delegate.supportsMultipleOpenResults();
    }

    @Override
    public boolean supportsGetGeneratedKeys() throws SQLException {
        return delegate.supportsGetGeneratedKeys();
    }

    @Override
    public ResultSet getSuperTypes(String catalog, String schemaPattern, String typeNamePattern) throws SQLException {
        //return delegate.getSuperTypes(catalog, schemaPattern, typeNamePattern);
        CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
        crs.setMetaData(buildMeta(
                "TYPE_CAT", "TYPE_SCHEM", "TYPE_NAME",
                "SUPERTYPE_CAT", "SUPERTYPE_SCHEM", "SUPERTYPE_NAME"
        ));
        crs.beforeFirst();
        return crs;
    }

    @Override
    public ResultSet getSuperTables(String catalog, String schemaPattern, String tableNamePattern) throws SQLException {
        //return delegate.getSuperTables(catalog, schemaPattern, tableNamePattern);
        CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
        crs.setMetaData(buildMeta(
                "TABLE_CAT", "TABLE_SCHEM", "TABLE_NAME",
                "SUPERTABLE_NAME"
        ));
        crs.beforeFirst();
        return crs;
    }

    @Override
    public ResultSet getAttributes(String catalog, String schemaPattern, String typeNamePattern, String attributeNamePattern) throws SQLException {
        //return delegate.getAttributes(catalog, schemaPattern, typeNamePattern, attributeNamePattern);
        CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
        crs.setMetaData(buildMeta(
                "TYPE_CAT", "TYPE_SCHEM", "TYPE_NAME", "ATTR_NAME",
                "DATA_TYPE", "ATTR_TYPE_NAME", "ATTR_SIZE", "DECIMAL_DIGITS",
                "NUM_PREC_RADIX", "NULLABLE", "REMARKS", "ATTR_DEF", "SQL_DATA_TYPE",
                "SQL_DATETIME_SUB", "CHAR_OCTET_LENGTH", "ORDINAL_POSITION", "IS_NULLABLE",
                "SCOPE_CATALOG", "SCOPE_SCHEMA", "SCOPE_TABLE", "SOURCE_DATA_TYPE"
        ));
        crs.beforeFirst();
        return crs;
    }

    @Override
    public boolean supportsResultSetHoldability(int holdability) throws SQLException {
        return delegate.supportsResultSetHoldability(holdability);
    }

    @Override
    public int getResultSetHoldability() throws SQLException {
        return delegate.getResultSetHoldability();
    }

    @Override
    public int getDatabaseMajorVersion() throws SQLException {
        return delegate.getDatabaseMajorVersion();
    }

    @Override
    public int getDatabaseMinorVersion() throws SQLException {
        return delegate.getDatabaseMinorVersion();
    }

    @Override
    public int getJDBCMajorVersion() throws SQLException {
        return delegate.getJDBCMajorVersion();
    }

    @Override
    public int getJDBCMinorVersion() throws SQLException {
        return delegate.getJDBCMinorVersion();
    }

    @Override
    public int getSQLStateType() throws SQLException {
        return delegate.getSQLStateType();
    }

    @Override
    public boolean locatorsUpdateCopy() throws SQLException {
        return delegate.locatorsUpdateCopy();
    }

    @Override
    public boolean supportsStatementPooling() throws SQLException {
        return delegate.supportsStatementPooling();
    }

    @Override
    public RowIdLifetime getRowIdLifetime() throws SQLException {
        return delegate.getRowIdLifetime();
    }

    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException {
        return delegate.unwrap(iface);
    }

    @Override
    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        return delegate.isWrapperFor(iface);
    }
}

```



---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/doris-datagrip-jdbc-driver/  

