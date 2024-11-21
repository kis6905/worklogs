## ClickHouse - Database does not exist
 
---
### Problem
Prod 환경에서 data insert 중 Database 를 못 찾는 에러가 발생했다.  
추가로 insert query 의 50% 정도만 발생했다.
 ```
 java.sql.SQLException: Code: 81. DB::Exception: Database andon does not exist. (UNKNOWN_DATABASE) (version 24.4.1.2088 (official build))
 , server ClickHouseNode [uri=http://andon-rest.ch.coupangdb.net:3306/andon, options={driver=com.clickhouse.jdbc.ClickHouseDriver}]@507994551
 at com.clickhouse.jdbc.SqlExceptionUtils.handle(SqlExceptionUtils.java:85)
 at com.clickhouse.jdbc.SqlExceptionUtils.create(SqlExceptionUtils.java:31)
 at com.clickhouse.jdbc.SqlExceptionUtils.handle(SqlExceptionUtils.java:90)
 at com.clickhouse.jdbc.internal.ClickHouseConnectionImpl.getTableColumns(ClickHouseConnectionImpl.java:267)
 at com.clickhouse.jdbc.internal.ClickHouseConnectionImpl.prepareStatement(ClickHouseConnectionImpl.java:843)
 at com.clickhouse.jdbc.ClickHouseConnection.prepareStatement(ClickHouseConnection.java:121)
 at com.coupang.csdp.spark.etl.common.clickhouse.ClickHouseJdbcConnector$.insert(ClickHouseJdbcConnector.scala:40)
 ```
 <br/>

### Cause
Prod 에서만 발생하는 원인을 찾다보니 node 에 차이가 있었다.(Dev node = 1개, Prod node = 2개)  
ClickHouse cluster 에 node 가 2개 있었고 table 생성 시 한쪽 node 에만 생성해 발생했다.
* shard_num: 1 / replica_num: 1 / host_name: ck-0-0 / host_address: 10.1.1.1
* shard_num: 1 / replica_num: 2 / host_name: ck-0-1 / host_address: 10.1.1.2
  <br/>
  <br/>

### Solution
database 와 table 생성 시 'ON CLUSTER {cluster_name}' 를 붙여준다.
 ```SQL
 -- database
 CREATE DATABASE mart ON CLUSTER 'mart-ck';
 
 -- table
 CREATE TABLE mart.daily_mart ON CLUSTER 'mart-ck' (
   ...
 ) ENGINE = MergeTree;
 ```
 <br/>
 <br/>
 <br/>

참고  
https://clickhouse.com/docs/en/architecture/cluster-deployment  
https://clickhouse.com/docs/en/operations/system-tables/clusters  
https://alwayscalm.tistory.com/3  