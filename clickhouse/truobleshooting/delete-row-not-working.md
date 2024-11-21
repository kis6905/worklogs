## ClickHouse - Delete row not working
 
---
### Problem
Delete Query 가 동작하지 않는다.  
기존 Row 를 삭제할수 없다면 Backfill 시 기존 데이터와 중복되어 문제가 된다.  
또한 오래된 데이터를 삭제할 수도 없다.
<br/>
<br/>

### Cause
Projection 을 사용하면 성능 이슈로 Delete 를 지원하지 않는다.  
https://clickhouse.com/docs/en/sql-reference/statements/delete#lightweight-deletes-do-not-work-with-projections
<br/>
<br/>

### Solution
세 가지 정도 해결책이 있어보인다.

1. Projection 을 사용하지말고 테이블을 모두 나눈다.
    * Where 조건마다 테이블을 나누고 모두 Insert 해야한다.
    * 동일한 데이터를 각 테이블마다 저장하기때문에 볼륨이 증가하고 유지보수가 어렵다.
2. 월별로 Partition 을 나눠 Partition 채로 삭제하고 1달치를 백필한다.
    * Old 데이터 삭제 시 좋다.
    * 일별 Backfill 을 못한다.
3. Update 를 위한 Flag Table 을 추가해 join 한다.
    * 모든 집계 Query 에 Join 이 필요하다.
    * Flag 테이블에 대한 관리 포인트가 늘어난다.

3번이 가장 적절해보여 Flag 테이블을 추가해 Join 하도록 변경하고,  
Data 저장 시 Flag Table 에도 Insert 한다.
<br/>
<br/>
<br/>