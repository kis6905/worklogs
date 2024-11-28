### Problem
비동기로 처리하기 위해 사용중인 Kafka Topic 에서 Lag 가 발생했다.  
Consumer 가 메시지를 처리하지 못하고 있고, 이로인해 비지니스에 영향을 미쳤다.  
또한 한 개의 원천 데이터에 대해 계속해서 새로운 데이터가 생겨났다.  
<br/>

### Cause
Consumer 에선 데이터 가공 후 마지막에 외부 API 를 호출한다.  
이 외부 API 에 문제가 생겨 API Latency 가 증가했고,  
Kafka 메시지 처리에 대한 Timeout 이 발생해 Offset 을 Commit 하지 못해 Lag 가 발생했다.  
* Timeout 이 발생한 이유
  * max.poll.records 옵션과 max.poll.interval.ms 옵션이 default 값으로 설정되어 있었다.
    * max.poll.records: 500
    * max.poll.interval.ms: 300,000ms(5분)
  * 외부 시스템 문제로 API 가 3초정도 소요되면서 500 개 메시지를 처리하는데 1500s(25분)가 소요되면서 Timeout 이 발생했다.   

엎친데 덮친격으로, 컨슈머는 멱등하지 않아 동일한 메시지에 대해 계속 새로운 데이터를 만들어 냈다.    
<br/>
<br/>

### Solution
우선 Hotfix 로 멱등성을 보장하도록 수정해 배포했고, Lag 가 서서히 줄어갔다.
* Lag 가 줄어든 이유는, 멱등성을 보장하게 되면서 한번 처리된 메시지는 외부 API 를 호출하지 않기때문이다.  

하지만 드라마틱하게 줄어들지 않아 Polling 옵션을 수정했다.
* max.poll.records: 500 -> 50
  * 외부 API 가 3초 소요되므로 50 * 3s = 150s (2분30초) 가 소요된다.  
    이는 max.poll.interval.ms 인 5분의 절반이므로 메시지를 모두 처리 할때까지 Timeout 이 발생하지 않을 것이로 판단했다.

Polling 옵션 수정 후 Lag 가 줄어드는 속도가 굉장히 빨라졌다.  
또한 중복으로 생성된 데이터를 제거하는 배치 잡을 급하게 개발해 처리했고, 장애는 종료되었다.  
<br/>
<br/>
<br/>
