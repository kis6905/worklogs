## ClickHouse - Select from only one replica
 
---
### Problem
Prod ClickHouse 에 테이블 생성 후 데이터를 insert 했다.  
이 후 조회를 해보면 하나의 replica 에서만 조회된다.
* 저장된 총 데이터 row 수: 792,206
* Query 시 조회되는 row 수
    * Replica1(mart-ck-0-0): 394,782
    * Replica2(mart-ck-0-1): 397,424

Distributed Table 을 사용해봐도 동일하다.
<br/>
<br/>

### Cause
현재 prod 클러스터는 shard: 1, node: 2, replica: 2(1 per node)로 구성되어 있다.  
shard 에 replica 가 여러개 있으면 MergeTree 테이블도 keeper 가 데이터를 복제해주는 것으로 알았는데, 이 부분을 잘 못알고 있었다.  
<< 그림 추가 >>  
각 replica 에 데이터를 복제하려면 ReplicatedMergeTree 를 사용해야 한다.
<br/>
<br/>

### Solution
테이블을 ReplicatedMegerTree 로 다시 생성한다.

 ```SQL
 CREATE TABLE mart.daily_mart ON CLUSTER 'mart-ck' (
   ...
 )
 ENGINE = ReplicatedMergeTree
 PRIMARY KEY (`itemId`, `date`, reasonCode)
 PARTITION BY substring(`date`, 1, 7)
 ```
 <br/>
 <br/>
 <br/>

참고  
https://alwayscalm.tistory.com/3  
https://clickhouse.com/docs/en/guides/sre/keeper/clickhouse-keeper  
https://clickhouse.com/docs/en/engines/table-engines/special/distributed  
https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replication