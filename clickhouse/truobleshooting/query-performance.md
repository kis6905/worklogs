## ClickHouse - Query performance
 
---
### Problem
Dev ClickHouse 에 데이터를 밀어넣고 query 테스트를 해보니 생각보다 성능이 좋지 않았다.

Table create DDL 은 아래와 같고, condition 에 맞게 index 를 생성해주었다.
 ```SQL
 CREATE TABLE mart.daily_mart (
     `id` String(50) NOT NULL,
     `itemId` BIGINT NOT NULL,
     `date` String(10) NOT NULL,
     `categoryId` BIGINT,
     `reasonCode` String(100),
     `uniqueReceiptCount` BIGINT,
     `targetReturnUnitQuantity` BIGINT,
     `shippedUnitQuantity` BIGINT,
     INDEX index_1 `date` TYPE minmax,
     INDEX index_2 (`categoryId`, `date`) TYPE minmax,
     INDEX index_3 `reasonCode` TYPE minmax
 ) ENGINE = MergeTree primary key (`itemId`, `date`)
 ```
테이블 생성 후 실제 운영 데이터 1개월치(약 23,000,000 rows)를 밀어넣고 테스트 해보았다.

 <table>
     <thead>
         <tr>
             <th>Case</th>
             <th>Query</th>
             <th>Result</th>
         </tr>
     </thead>
     <tbody>
         <tr>
             <td>
                 where: itemId + date<br/>
                 group by: itemId + date
             </td>
             <td>

 ```SQL
 select
     `itemId`,
     `date`,
     sum(uniqueReceiptCount),
     sum(targetReturnUnitQuantity),
     max(shippedUnitQuantity)
 from mart.daily_mart
 where `itemId` = 111
   and `date` between '2024-05-01' and '2024-05-31'
 group by `itemId`, `date`
 ```
 </td>
             <td>
                 <b>Execution time:</b> 0.07s<br/>
                 <b>Explain Estimate:</b><br/>

| part | rows   | marks |
 |------|--------|-------|
| 9    | 73,749 | 10    |
 </td>
         </tr>
         <tr>
             <td>
                 where: itemId + date<br/>
                 group by: itemId + reasonCode
             </td>
             <td>

 ```SQL
 select
     `itemId`,
     `reasonCode`,
     sum(uniqueReceiptCount),
     sum(targetReturnUnitQuantity),
     sum(shippedUnitQuantity)
 from mart.daily_mart
 where `itemId` = 20098313
   and `date` between '2024-05-01' and '2024-05-31'
 group by `itemId`, `reasonCode`
 ```
 </td>
             <td>
                 <b>Execution time:</b> 0.062s<br/>
                 <b>Explain Estimate:</b><br/>

| part | rows   | marks |
 |------|--------|-------|
| 7    | 40,978 | 7     |
 </td>
         </tr>
         <tr>
             <td>
                 where: kanCategory5 + date
             </td>
             <td>

 ```SQL
 select
     `categoryId`,
     `date`,
     sum(uniqueReceiptCount) as uniqueReceiptCount,
     sum(targetReturnUnitQuantity) as targetReturnUnitQuantity,
     sum(shippedUnitQuantity) as shippedUnitQuantity
 from (
          select
              `categoryId`,
              `date`,
              `itemId`,
              sum(uniqueReceiptCount) as uniqueReceiptCount,
              sum(targetReturnUnitQuantity) as targetReturnUnitQuantity,
              max(shippedUnitQuantity) as shippedUnitQuantity
          from mart.daily_mart dm
          where `categoryId` = 2296
            and `date` between '2024-05-01' and '2024-05-31'
          group by `categoryId`, `date`, `itemId`
      )
 group by `categoryId`, `date`
 ```
 </td>
             <td>
                 <b>Execution time:</b> <span style="color: red;">2s</span><br/>
                 <b>Explain Estimate:</b><br/>

| part | rows                                      | marks                               |
 |------|-------------------------------------------|-------------------------------------|
| 8    | <span style="color:red">23,149,463</span> | <span style="color:red">2939</span> |
 </td>
         </tr>
     </tbody>
 </table>

3번 Case 에서 2s 나 걸린다.  
rows 와 marks 를 보면 거의 풀스캔이다.
<br/>
<br/>

### Cause
primary key 는 itemId 이기 때문에 granule 은 itemId 순으로 생성 되었을 것이다.  
그런데 categoryId 로 조회하게 되면 분산되어 있는 granule 을 모두 가져올 것이기 때문에 모든 marks 찾게될 것이고 그 결과 2939개나 된다.
<br/>
<br/>

### Solution
이런 경우, ClickHouse index 특성 상 보조 index 가 아닌 query에 맞는 Primary Key를 각각 선언해야 한다.  
따라서, 사용하고자 하는 query에 따라 Primary Key가 달려져야 하며, ClickHouse는 이를 위해 세 가지 대안을 제시한다.

1. Primary Key가 다른 Copy Table 생성
2. Primary Key가 다른 Materialize View 생성
3. Primary Key가 다른 Projection을 추가

이 중 권장하는 3번째 방법인 Projection 을 추가해본다.

 ```SQL
 ALTER TABLE mart.daily_mart ADD PROJECTION `categoryId_date_proejction`
 ( SELECT * ORDER BY `categoryId`, `date`)
 ```

Projection을 추가해서 사용하는 방법은 내부적으로 Table이 생성되고 자동으로 Sync가 맞춰지는 것까지는 Materialized View와 같다.  
그러나 같은 이름의 Table을 참조하더라도 사용되는 Index에 따라 ClickHouse Optimizer에 의해 자동으로 최적의 Table을 선택한다는 큰 장점이 있다.

Projection 을 추가한 후 다시 테스트 해보니 Execution Time 이 2s -> 0.35s 로 줄었고,  
Query Explain Estimate 해보면 rows 와 marks 모두 줄어든 것을 볼 수있다.

| part | rows    | marks |
 |------|---------|-------|
| 7    | 363,511 | 45    |

추가로,  
Projection 을 추가할때 테이블에 이미 데이터가 있다면 Projection 을 생성하고 데이터를 다시 쌓아줘야 한다.  
즉 테이블을 날리고 다시 생성해야 한다.  
이미 production 에서 운영중이라면 적절한 migration 방법을 고민해봐야할것 같다.  
<br/>
<br/>
<br/>


참고:  
https://alwayscalm.tistory.com/7  
https://clickhouse.com/docs/en/sql-reference/statements/alter/projection