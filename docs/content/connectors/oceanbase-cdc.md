# OceanBase CDC Connector

The OceanBase CDC connector allows for reading snapshot data and incremental data from OceanBase. This document describes how to set up the OceanBase CDC connector to run SQL queries against OceanBase.

Dependencies
------------

In order to set up the OceanBase CDC connector, the following table provides dependency information for both projects using a build automation tool (such as Maven or SBT) and SQL Client with SQL JAR bundles.

```xml
<dependency>
  <groupId>com.ververica</groupId>
  <artifactId>flink-connector-oceanbase-cdc</artifactId>
  <!-- The dependency is available only for stable releases, SNAPSHOT dependency need build by yourself. -->
  <version>2.5-SNAPSHOT</version>
</dependency>
```

If you want to use OceanBase JDBC driver to connect to the enterprise edition database, you should also include the following dependency in your class path.

```xml
<dependency>
   <groupId>com.oceanbase</groupId>
   <artifactId>oceanbase-client</artifactId>
   <version>2.4.2</version>
</dependency>
```

### SQL Client JAR

```Download link is available only for stable releases.```

Download [flink-sql-connector-oceanbase-cdc-2.5-SNAPSHOT.jar](https://repo1.maven.org/maven2/com/ververica/flink-sql-connector-oceanbase-cdc/2.5-SNAPSHOT/flink-sql-connector-oceanbase-cdc-2.5-SNAPSHOT.jar) and put it under `<FLINK_HOME>/lib/`.

**Note:** flink-sql-connector-oceanbase-cdc-XXX-SNAPSHOT version is the code corresponding to the development branch. Users need to download the source code and compile the corresponding jar. Users should use the released version, such as [flink-sql-connector-oceanbase-cdc-2.2.1.jar](https://mvnrepository.com/artifact/com.ververica/flink-sql-connector-oceanbase-cdc), the released version will be available in the Maven central warehouse.

For JDBC driver, the cdc jar above already contains MySQL JDBC driver 5.1.47, which is our recommended version. Due to the license issue, we can not include the OceanBase JDBC driver in the cdc jar. If you need to use it, you can download it from [here](https://repo1.maven.org/maven2/com/oceanbase/oceanbase-client/2.4.2/oceanbase-client-2.4.2.jar) and put it under `<FLINK_HOME>/lib/`, you also need to set the start option `jdbc.driver` to `com.oceanbase.jdbc.Driver`.

Setup OceanBase and LogProxy Server
----------------------

1. Set up the OceanBase cluster following the [doc](https://github.com/oceanbase/oceanbase#quick-start).

2. Create a user with password in `sys` tenant, this user is used in OceanBase LogProxy.

   ```shell
   mysql -h${host} -P${port} -uroot
   
   mysql> SHOW TENANT;
   mysql> CREATE USER ${sys_username} IDENTIFIED BY '${sys_password}';
   mysql> GRANT ALL PRIVILEGES ON *.* TO ${sys_username} WITH GRANT OPTION;
   ```

3. Create a user in the tenant you want to monitor, this is used to read data for snapshot and change event.

4. For users of OceanBase Community Edition, you need to get the `rootserver-list`. You can use the following command to get the value:

    ```shell
    mysql> show parameters like 'rootservice_list';
    ```

   For users of OceanBase Enterprise Edition, you need to get the `config-url`. You can use the following command to get the value:

    ```shell
    mysql> show parameters like 'obconfig_url';
    ```

5. Setup OceanBase LogProxy. For users of OceanBase Community Edition, you can follow the [quick start](https://github.com/oceanbase/oblogproxy#getting-started).

How to create a OceanBase CDC table
----------------

The OceanBase CDC table can be defined as following:

```sql
-- checkpoint every 3000 milliseconds                       
Flink SQL> SET 'execution.checkpointing.interval' = '3s';

-- register a OceanBase table 'orders' in Flink SQL
Flink SQL> CREATE TABLE orders (
    order_id     INT,
    order_date   TIMESTAMP(0),
    customer_name STRING,
    price        DECIMAL(10, 5),
    product_id   INT,
    order_status BOOLEAN,
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector' = 'oceanbase-cdc',
    'scan.startup.mode' = 'initial',
    'username' = 'user@test_tenant#cluster_name',
    'password' = 'pswd',
    'tenant-name' = 'test_tenant',
    'database-name' = '^test_db$',
    'table-name' = '^orders$',
    'hostname' = '127.0.0.1',
    'port' = '2881',
    'rootserver-list' = '127.0.0.1:2882:2881',
    'logproxy.host' = '127.0.0.1',
    'logproxy.port' = '2983',
    'working-mode' = 'memory'
);

-- read snapshot and binlogs from orders table
Flink SQL> SELECT * FROM orders;
```

If you want to use OceanBase Oracle mode, you need to add the OceanBase jdbc jar file to Flink and set up the enterprise edition of oblogproxy, then you can create a table in Flink as following:

```sql
Flink SQL> CREATE TABLE orders (
    order_id     INT,
    order_date   TIMESTAMP(0),
    customer_name STRING,
    price        DECIMAL(10, 5),
    product_id   INT,
    order_status BOOLEAN,
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector' = 'oceanbase-cdc',
    'scan.startup.mode' = 'initial',
    'username' = 'user@test_tenant#cluster_name',
    'password' = 'pswd',
    'tenant-name' = 'test_tenant',
    'database-name' = '^test_db$',
    'table-name' = '^orders$',
    'hostname' = '127.0.0.1',
    'port' = '2881',
    'compatible-mode' = 'oracle',
    'jdbc.driver' = 'com.oceanbase.jdbc.Driver',
    'config-url' = 'http://127.0.0.1:8080/services?Action=ObRootServiceInfo&User_ID=xxx&UID=xxx&ObRegion=xxx',
    'logproxy.host' = '127.0.0.1',
    'logproxy.port' = '2983',
    'working-mode' = 'memory'
);
```

You can also try the quickstart tutorial that sync data from OceanBase to Elasticsearch, please refer [Flink CDC Tutorial](https://ververica.github.io/flink-cdc-connectors/release-2.3//content/quickstart/oceanbase-tutorial.html) for more information.

Connector Options
----------------

The OceanBase CDC Connector contains some options for both sql and stream api as the following sheet. 

*Note*: The connector supports two ways to specify the table list to listen to, and will get the union of the results when both way are used at the same time.
1. Use `database-name` and `table-name` to match database and table names in regex. As the `obcdc` (former `liboblog`) only supports `fnmatch` now, we can't use regex directly to filter change events, so these two options can only be used in `initial` startup mode.
2. Use `table-list` to match the exact value of database and table names.

<div class="highlight">
    <table class="colwidths-auto docutils">
        <thead>
            <tr>
                <th class="text-left" style="width: 10%">Option</th>
                <th class="text-left" style="width: 8%">Required</th>
                <th class="text-left" style="width: 7%">Default</th>
                <th class="text-left" style="width: 10%">Type</th>
                <th class="text-left" style="width: 65%">Description</th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>connector</td>
                <td>required</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>Specify what connector to use, here should be <code>'oceanbase-cdc'</code>.</td>
            </tr>
            <tr>
                <td>scan.startup.mode</td>
                <td>required</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>Specify the startup mode for OceanBase CDC consumer, valid enumerations are
                    <code>'initial'</code>,<code>'latest-offset'</code> or <code>'timestamp'</code>.
                </td>
            </tr>
            <tr>
                <td>scan.startup.timestamp</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>Long</td>
                <td>Timestamp in seconds of the start point, only used for <code>'timestamp'</code> startup mode.</td>
            </tr>
            <tr>
                <td>username</td>
                <td>required</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>Username to be used when connecting to OceanBase.</td>
            </tr>
            <tr>
                <td>password</td>
                <td>required</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>Password to be used when connecting to OceanBase.</td>
            </tr>
            <tr>
                <td>tenant-name</td>
                <td>required</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>Tenant name of OceanBase to monitor, should be exact value.</td>
            </tr>
            <tr>
                <td>database-name</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>Database name of OceanBase to monitor, should be regular expression. Only can be used with 'initial' mode.</td>
            </tr>
            <tr>
                <td>table-name</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>Table name of OceanBase to monitor, should be regular expression. Only can be used with 'initial' mode.</td>
            </tr>
            <tr>
                <td>table-list</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>List of full names of tables, separated by commas, e.g. "db1.table1, db2.table2".</td>
            </tr>
            <tr>
                <td>hostname</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>IP address or hostname of the OceanBase database server or OceanBase Proxy server.</td>
            </tr>
            <tr>
                <td>port</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>Integer</td>
                <td>Integer port number to connect to OceanBase. It can be the SQL port of OceanBase server, which is 2881 by default, or the port of OceanBase proxy service, which is 2883 by default.</td>
            </tr>
            <tr>
                <td>connect.timeout</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">30s</td>
                <td>Duration</td>
                <td>The maximum time that the connector should wait after trying to connect to the OceanBase database server before timing out.</td>
            </tr>
            <tr>
                <td>server-time-zone</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">+00:00</td>
                <td>String</td>
                <td>The session timezone which controls how temporal types are converted to STRING in OceanBase. Can be UTC offset in format "±hh:mm", or named time zones if the time zone information tables in the mysql database have been created and populated.</td>
            </tr>
            <tr>
                <td>logproxy.host</td>
                <td>required</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>Hostname or IP address of OceanBase log proxy service.</td>
            </tr>
            <tr>
                <td>logproxy.port</td>
                <td>required</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>Integer</td>
                <td>Port number of OceanBase log proxy service.</td>
            </tr>
            <tr>
                <td>logproxy.client.id</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">By rule.</td>
                <td>String</td>
                <td>Id of a log proxy client connection, will be in format {flink_ip}_{process_id}_{timestamp}_{thread_id}_{tenant} by default.</td>
            </tr>
            <tr>
                <td>rootserver-list</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>The semicolon-separated list of OceanBase root servers in format `ip:rpc_port:sql_port`, required for OceanBase CE.</td>
            </tr>
            <tr>
                <td>config-url</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>The url to get the server info from the config server, required for OceanBase EE.</td>
            </tr>
            <tr>
                <td>working-mode</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">storage</td>
                <td>String</td>
                <td>Working mode of `obcdc` in LogProxy, can be `storage` or `memory`.</td>
            </tr>
            <tr>
                <td>compatible-mode</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">mysql</td>
                <td>String</td>
                <td>Compatible mode of OceanBase, can be `mysql` or `oracle`.</td>
            </tr>
            <tr>
                <td>jdbc.driver</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">com.mysql.jdbc.Driver</td>
                <td>String</td>
                <td>JDBC driver class for snapshot reading.</td>
            </tr>
            <tr>
                <td>jdbc.properties.*</td>
                <td>optional</td>
                <td style="word-wrap: break-word;">(none)</td>
                <td>String</td>
                <td>Option to pass custom JDBC URL properties. User can pass custom properties like 'jdbc.properties.useSSL' = 'false'.</td>
            </tr>
        </tbody>
    </table>
</div>

Available Metadata
----------------

The following format metadata can be exposed as read-only (VIRTUAL) columns in a table definition.

<table class="colwidths-auto docutils">
    <thead>
        <tr>
            <th class="text-left" style="width: 15%">Key</th>
            <th class="text-left" style="width: 30%">DataType</th>
            <th class="text-left" style="width: 55%">Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>tenant_name</td>
            <td>STRING NOT NULL</td>
            <td>Name of the tenant that contains the row.</td>
        </tr>
        <tr>
            <td>database_name</td>
            <td>STRING NOT NULL</td>
            <td>Name of the database that contains the row.</td>
        </tr>
        <tr>
            <td>table_name</td>
            <td>STRING NOT NULL</td>
            <td>Name of the table that contains the row.</td>
        </tr>
        <tr>
            <td>op_ts</td>
            <td>TIMESTAMP_LTZ(3) NOT NULL</td>
            <td>It indicates the time that the change was made in the database. <br>
                If the record is read from snapshot of the table instead of the change stream, the value is always 0.</td>
        </tr>
    </tbody>
</table>

The extended CREATE TABLE example demonstrates the syntax for exposing these metadata fields:

```sql
CREATE TABLE products (
    tenant_name STRING METADATA FROM 'tenant_name' VIRTUAL,
    db_name STRING METADATA FROM 'database_name' VIRTUAL,
    table_name STRING METADATA  FROM 'table_name' VIRTUAL,
    operation_ts TIMESTAMP_LTZ(3) METADATA FROM 'op_ts' VIRTUAL,
    order_id INT,
    order_date TIMESTAMP(0),
    customer_name STRING,
    price DECIMAL(10, 5),
    product_id INT,
    order_status BOOLEAN,
    PRIMARY KEY(order_id) NOT ENFORCED
) WITH (
   'connector' = 'oceanbase-cdc',
   'scan.startup.mode' = 'initial',
   'username' = 'user@test_tenant',
   'password' = 'pswd',
   'tenant-name' = 'test_tenant',
   'database-name' = '^test_db$',
   'table-name' = '^orders$',
   'hostname' = '127.0.0.1',
   'port' = '2881',
   'rootserver-list' = '127.0.0.1:2882:2881',
   'logproxy.host' = '127.0.0.1',
   'logproxy.port' = '2983',
   'working-mode' = 'memory'
);
```

Features
--------

### At-Least-Once Processing

The OceanBase CDC connector is a Flink Source connector which will read database snapshot first and then continues to read change events with **at-least-once processing**.

OceanBase is a kind of distributed database whose log files are distributed on different servers. As there is no position information like MySQL binlog offset, we can only use timestamp as the position mark. In order to ensure the completeness of reading data, `liboblog` (a C++ library to read OceanBase log record) might read some log data before the given timestamp. So in this way we may read duplicate data whose timestamp is around the start point, and only 'at-least-once' can be guaranteed.

### Startup Reading Position

The config option `scan.startup.mode` specifies the startup mode for OceanBase CDC consumer. The valid enumerations are:

- `initial`: Performs an initial snapshot on the monitored table upon first startup, and continue to read the latest commit log.
- `latest-offset`: Never to perform snapshot on the monitored table upon first startup and just read the latest commit log since the connector is started.
- `timestamp`: Never to perform snapshot on the monitored table upon first startup and just read the commit log from the given `scan.startup.timestamp`.

### Consume Commit Log

The OceanBase CDC Connector using [oblogclient](https://github.com/oceanbase/oblogclient) to consume commit log from OceanBase LogProxy.

### DataStream Source

The OceanBase CDC connector can also be a DataStream source. You can create a SourceFunction as the following shows:

```java
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.SourceFunction;
import org.apache.flink.table.api.DataTypes;
import org.apache.flink.table.catalog.Column;
import org.apache.flink.table.catalog.ResolvedSchema;
import org.apache.flink.table.catalog.UniqueConstraint;
import org.apache.flink.table.data.RowData;
import org.apache.flink.table.runtime.typeutils.InternalTypeInfo;
import org.apache.flink.table.types.logical.RowType;

import com.ververica.cdc.connectors.oceanbase.OceanBaseSource;
import com.ververica.cdc.connectors.oceanbase.source.RowDataOceanBaseDeserializationSchema;
import com.ververica.cdc.connectors.oceanbase.table.OceanBaseDeserializationSchema;
import com.ververica.cdc.connectors.oceanbase.table.StartupMode;

import java.time.ZoneId;
import java.util.Arrays;
import java.util.Collections;

public class OceanBaseSourceExample {
   public static void main(String[] args) throws Exception {
      ResolvedSchema resolvedSchema =
              new ResolvedSchema(
                      Arrays.asList(
                              Column.physical("id", DataTypes.INT().notNull()),
                              Column.physical("name", DataTypes.STRING().notNull())),
                      Collections.emptyList(),
                      UniqueConstraint.primaryKey("pk", Collections.singletonList("id")));

      RowType physicalDataType =
              (RowType) resolvedSchema.toPhysicalRowDataType().getLogicalType();
      TypeInformation<RowData> resultTypeInfo = InternalTypeInfo.of(physicalDataType);
      String serverTimeZone = "+00:00";

      OceanBaseDeserializationSchema<RowData> deserializer =
              RowDataOceanBaseDeserializationSchema.newBuilder()
                      .setPhysicalRowType(physicalDataType)
                      .setResultTypeInfo(resultTypeInfo)
                      .setServerTimeZone(ZoneId.of(serverTimeZone))
                      .build();

      SourceFunction<RowData> oceanBaseSource =
              OceanBaseSource.<RowData>builder()
                      .rsList("127.0.0.1:2882:2881")
                      .startupMode(StartupMode.INITIAL)
                      .username("user@test_tenant")
                      .password("pswd")
                      .tenantName("test_tenant")
                      .databaseName("^test_db$")
                      .tableName("^test_table$")
                      .hostname("127.0.0.1")
                      .port(2881)
                      .compatibleMode("mysql")
                      .jdbcDriver("com.mysql.jdbc.Driver")
                      .logProxyHost("127.0.0.1")
                      .logProxyPort(2983)
                      .serverTimeZone(serverTimeZone)
                      .deserializer(deserializer)
                      .build();

      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

      // enable checkpoint
      env.enableCheckpointing(3000);

      env.addSource(oceanBaseSource).print().setParallelism(1);
      env.execute("Print OceanBase Snapshot + Change Events");
   }
}
```
Data Type Mapping
----------------

### Mysql Mode

<div class="wy-table-responsive">
    <table class="colwidths-auto docutils">
        <thead>
            <tr>
                <th class="text-left">OceanBase type</th>
                <th class="text-left">Flink SQL type</th>
                <th class="text-left">NOTE</th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>BOOLEAN<br>
                    TINYINT(1)<br>
                    BIT(1)</td>
                <td>BOOLEAN</td>
                <td></td>
            </tr>
            <tr>
                <td>TINYINT</td>
                <td>TINYINT</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    SMALLINT<br>
                    TINYINT UNSIGNED</td>
                <td>SMALLINT</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    INT<br>
                    MEDIUMINT<br>
                    SMALLINT UNSIGNED</td>
                <td>INT</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    BIGINT<br>
                    INT UNSIGNED</td>
                <td>BIGINT</td>
                <td></td>
            </tr>
            <tr>
                <td>BIGINT UNSIGNED</td>
                <td>DECIMAL(20, 0)</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    REAL<br>
                    FLOAT<br>
                </td>
                <td>FLOAT</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    DOUBLE
                </td>
                <td>DOUBLE</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    NUMERIC(p, s)<br>
                    DECIMAL(p, s)<br>
                    where p <= 38<br>
                </td>
                <td>DECIMAL(p, s)</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    NUMERIC(p, s)<br>
                    DECIMAL(p, s)<br>
                    where 38 < p <=65<br>
                </td>
                <td>STRING</td>
                <td>DECIMAL is equivalent to NUMERIC. The precision for DECIMAL data type is up to 65 in OceanBase, but
                    the precision for DECIMAL is limited to 38 in Flink.
                    So if you define a decimal column whose precision is greater than 38, you should map it to STRING to
                    avoid precision loss.</td>
            </tr>
            <tr>
                <td>DATE</td>
                <td>DATE</td>
                <td></td>
            </tr>
            <tr>
                <td>TIME [(p)]</td>
                <td>TIME [(p)]</td>
                <td></td>
            </tr>
            <tr>
                <td>DATETIME [(p)]</td>
                <td>TIMESTAMP [(p)]</td>
                <td></td>
            </tr>
            <tr>
                <td>TIMESTAMP [(p)]</td>
                <td>TIMESTAMP_LTZ [(p)]</td>
                <td></td>
            </tr>
            <tr>
                <td>CHAR(n)</td>
                <td>CHAR(n)</td>
                <td></td>
            </tr>
            <tr>
                <td>VARCHAR(n)</td>
                <td>VARCHAR(n)</td>
                <td></td>
            </tr>
            <tr>
                <td>BIT(n)</td>
                <td>BINARY(⌈n/8⌉)</td>
                <td></td>
            </tr>
            <tr>
                <td>BINARY(n)</td>
                <td>BINARY(n)</td>
                <td></td>
            </tr>
            <tr>
                <td>VARBINARY(N)</td>
                <td>VARBINARY(N)</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    TINYTEXT<br>
                    TEXT<br>
                    MEDIUMTEXT<br>
                    LONGTEXT<br>
                </td>
                <td>STRING</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    TINYBLOB<br>
                    BLOB<br>
                    MEDIUMBLOB<br>
                    LONGBLOB<br>
                </td>
                <td>BYTES</td>
                <td></td>
            </tr>
            <tr>
                <td>YEAR</td>
                <td>INT</td>
                <td></td>
            </tr>
            <tr>
                <td>ENUM</td>
                <td>STRING</td>
                <td></td>
            </tr>
            <tr>
                <td>SET</td>
                <td>ARRAY&lt;STRING&gt;</td>
                <td>As the SET data type in OceanBase is a string object that can have zero or more values, it should always be mapped to an array of string</td>
            </tr>
            <tr>
                <td>JSON</td>
                <td>STRING</td>
                <td>The JSON data type  will be converted into STRING with JSON format in Flink.</td>
            </tr>
        </tbody>
    </table>
</div>

### Oracle Mode

<div class="wy-table-responsive">
    <table class="colwidths-auto docutils">
        <thead>
            <tr>
                <th class="text-left">OceanBase type</th>
                <th class="text-left">Flink SQL type</th>
                <th class="text-left">NOTE</th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>NUMBER(1)</td>
                <td>BOOLEAN</td>
                <td></td>
            </tr>
            <tr>
                <td>NUMBER(p, s <= 0), p - s < 3 </td>
                <td>TINYINT</td>
                <td></td>
            </tr>
            <tr>
                <td>NUMBER(p, s <= 0), p - s < 5 </td>
                <td>SMALLINT</td>
                <td></td>
            </tr>
            <tr>
                <td>NUMBER(p, s <= 0), p - s < 10 </td>
                <td>INT</td>
                <td></td>
            </tr>
            <tr>
                <td>NUMBER(p, s <= 0), p - s < 19 </td>
                <td>BIGINT</td>
                <td></td>
            </tr>
            <tr>
                <td>NUMBER(p, s <= 0), 19 <=p - s <=38</td>
                <td>DECIMAL(p - s, 0)</td>
                <td></td>
            </tr>
            <tr>
                <td>NUMBER(p, s > 0)</td>
                <td>DECIMAL(p, s)</td>
            </tr>
            <tr>
                <td>NUMBER(p, s <= 0), p - s> 38 </td>
                <td>STRING</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    FLOAT<br>
                    BINARY_FLOAT
                </td>
                <td>FLOAT</td>
                <td></td>
            </tr>
            <tr>
                <td>BINARY_DOUBLE</td>
                <td>DOUBLE</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    DATE<br>
                    TIMESTAMP [(p)]
                </td>
                <td>TIMESTAMP [(p)]</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    CHAR(n)<br>
                    NCHAR(n)<br>
                    VARCHAR(n)<br>
                    VARCHAR2(n)<br>
                    NVARCHAR2(n)<br>
                    CLOB<br>
                </td>
                <td>STRING</td>
                <td></td>
            </tr>
            <tr>
                <td>
                    RAW<br>
                    BLOB<br>
                    ROWID
                </td>
                <td>BYTES</td>
                <td></td>
            </tr>
        </tbody>
    </table>
</div>