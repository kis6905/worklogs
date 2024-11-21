## ClickHouse - System table query cache
 
---
### Problem
Spark 에서 ClickHouse jdbc 를 사용해 ClickHouse 연동 시 Dev 에선 발생하지 않던 Exception 이 Prod 에서 발생한다.
 ```
 Caused by: java.sql.SQLException: Code: 719. DB::Exception: The query result was not cached because the query contains a system table. Use setting `query_cache_system_table_handling = 'save'` or `= 'ignore'` to cache the query result regardless or to omit caching. (QUERY_CACHE_USED_WITH_SYSTEM_TABLE) (version 24.4.1.2088 (official build))
 , server ClickHouseNode [uri=http://andon-rest.ch.coupangdb.net:3306/andon, options={driver=com.clickhouse.jdbc.ClickHouseDriver}]@-449390932
 at com.clickhouse.jdbc.SqlExceptionUtils.handle(SqlExceptionUtils.java:85)
 at com.clickhouse.jdbc.SqlExceptionUtils.create(SqlExceptionUtils.java:31)
 at com.clickhouse.jdbc.SqlExceptionUtils.handle(SqlExceptionUtils.java:90)
 at com.clickhouse.jdbc.internal.ClickHouseConnectionImpl.getServerInfo(ClickHouseConnectionImpl.java:131)
 at com.clickhouse.jdbc.internal.ClickHouseConnectionImpl.<init>(ClickHouseConnectionImpl.java:335)
 at com.clickhouse.jdbc.ClickHouseDataSource.getConnection(ClickHouseDataSource.java:46)
 ```
 <br/>

### Cause
ClickHouse jdbc 에서 Connection 을 생성할때 ClickHouse Server Info 를 조회하기위한 query 를 실행한다.  
https://github.com/ClickHouse/clickhouse-java/blob/main/clickhouse-jdbc/src/main/java/com/clickhouse/jdbc/internal/ClickHouseConnectionImpl.java#L81

 ```SQL
 select
     currentUser() user,
     timezone() timezone,
     version() version,
     toUInt8(ifnull((select value from system.settings where name = 'readonly'), '0')) as readonly,
     toInt8(ifnull((select value from system.settings where name = 'throw_on_unsupported_query_inside_transaction'), '-1')) as throw_on_unsupported_query_inside_transaction,
     (ifnull((select value from system.settings where name = 'wait_changes_become_visible_after_commit_mode'), '')) as wait_changes_become_visible_after_commit_mode,
     toInt8(ifnull((select value from system.settings where name = 'implicit_transaction'), '-1')) as implicit_transaction,
     toUInt64(ifnull((select value from system.settings where name = 'max_insert_block_size'), '0')) as max_insert_block_size,
     toInt8(ifnull((select value from system.settings where name = 'allow_experimental_lightweight_delete'), '-1')) as allow_experimental_lightweight_delete,
     (ifnull((select value from system.settings where name = 'custom_jdbc_config'), '')) as custom_jdbc_config
 FORMAT RowBinaryWithNamesAndTypes
 ```
Query 를 보면 system 테이블을 조회하는데,  
Error log 를 보면 DB Server 설정 중 query_cache_system_table_handling 을 'save' 또는 'ignore' 로 변경하라고 한다.  
query_cache_system_table_handling 의 default 값은 'throw' 다.  
https://clickhouse.com/docs/en/operations/settings/settings#query-cache-system-table-handling
<br/>
<br/>

### Solution
DB Server 설정 변경 및 재시작이 필요하기 때문에 DB에 요청했고, 설정 변경 후 정상 동작한다.
<br/>
<br/>
<br/>