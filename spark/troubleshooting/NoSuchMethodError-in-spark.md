## NoSuchMethodError in Spark
 
---
### Problem
DW 에서 Query 한 데이터를 ClickHouse 에 저장하기 위해 Spark Job 개발했다.  
Local 에서 테스트 후 Prod 에서 실행해보니 NoSuchMethodError 가 발생한다.

 ```
 24/06/28 15:46:50 INFO yarn.ApplicationMaster: Final app status: FAILED, exitCode: 15, (reason: User class threw exception: java.lang.NoSuchMethodError: com.clickhouse.client.http.ApacheHttpConnectionImpl$HttpConnectionManager.setDefaultConnectionConfig(Lorg/apache/hc/client5/http/config/ConnectionConfig;)V
     at com.clickhouse.client.http.ApacheHttpConnectionImpl$HttpConnectionManager.<init>(ApacheHttpConnectionImpl.java:362)
     at com.clickhouse.client.http.ApacheHttpConnectionImpl.newConnection(ApacheHttpConnectionImpl.java:93)
     at com.clickhouse.client.http.ApacheHttpConnectionImpl.<init>(ApacheHttpConnectionImpl.java:80)
     at com.clickhouse.client.http.ClickHouseHttpConnectionFactory.createConnection(ClickHouseHttpConnectionFactory.java:22)
     at com.clickhouse.client.http.ClickHouseHttpClient.newConnection(ClickHouseHttpClient.java:56)
     at com.clickhouse.client.http.ClickHouseHttpClient.newConnection(ClickHouseHttpClient.java:26)
     at com.clickhouse.client.AbstractClient.getConnection(AbstractClient.java:198)
     at com.clickhouse.client.http.ClickHouseHttpClient.send(ClickHouseHttpClient.java:90)
     at com.clickhouse.client.AbstractClient.execute(AbstractClient.java:280)
     at com.clickhouse.client.ClickHouseClientBuilder$Agent.sendOnce(ClickHouseClientBuilder.java:282)
     at com.clickhouse.client.ClickHouseClientBuilder$Agent.send(ClickHouseClientBuilder.java:294)
     at com.clickhouse.client.ClickHouseClientBuilder$Agent.execute(ClickHouseClientBuilder.java:349)
     at com.clickhouse.client.ClickHouseClient.executeAndWait(ClickHouseClient.java:878)
     at com.clickhouse.client.ClickHouseRequest.executeAndWait(ClickHouseRequest.java:2154)
     at com.clickhouse.jdbc.internal.ClickHouseConnectionImpl.getServerInfo(ClickHouseConnectionImpl.java:128)
     at com.clickhouse.jdbc.internal.ClickHouseConnectionImpl.<init>(ClickHouseConnectionImpl.java:335)
     at com.clickhouse.jdbc.ClickHouseDataSource.getConnection(ClickHouseDataSource.java:46)
     ...
 ```
Log 를 보면 ClickHouse client 는 apache httpclient5 를 사용하는데, httpclient5 class 들을 못찾는다.
<br/>
<br/>

### Cause
다른 곳에서 httpclient4 를 사용하고 있어 충돌된것으로 보인다. httpclient5 class를 못찾는 것으로 보인다.
<br/>
<br/>

### Solution
다른 곳에서 사용하는 httpclient4를 5로 올리는 것이 좋으나,  
모든 부분을 테스트해야 하므로 httpclient5 를 relocate 하는 방법으로 해결했다.  

shadowJar 시 httpclient5 pacakge relocate
 ```groovy
 shadowJar {
     ...
     relocate 'org.apache.hc.client5', 'shadow.org.apache.hc.client5'
     relocate 'org.apache.hc.core5', 'shadow.org.apache.hc.core5'
 }
 ```
주의할 점은 groupId 가 아닌 실제 package 명을 입력해야 한다.
<br/>
<br/>
<br/>