### Problem
전사 OKR 인 취약성 제거의 일환으로 Redis Version 업그레이드 Task 를 진행하게 되었다.
가이드에 따라 Redis SDK 를 최신버전으로 적용한 후 Migration 당일 Switchover 후 RedisCommandTimeout Error 가 발생했고,    
이로인해 서비스 전면 장애가 발생했다.
<br/>
<br/>

### Cause
#### Root Cause
Redis SDK 는 내부적으로 Lettuce 를 사용하고 있고, 최신버전에선 pub/sub 으로 Switchover 이벤트를 리스닝하고 있다.      
Switchover 이벤트를 받으면 AS-IS 서버 용으로 생성된 clientResources 를 shutdown 하고 TO-BE 서버를 위한 clientResources 를 생성해 사용하게끔 구현되어 있다.  
([clientResource](https://github.com/redis/lettuce/wiki/Configuring-Client-resources) 는 쓰레드 풀이라고 보면 된다.)  

문제는 우리 팀에선 이 SDK 를 비 정상적인 방법으로 사용하고 있었다.

```kotlin
val redisCluster = RedisCluster("my-redis", "PROD") // SDK Class

...

val factory = LettuceConnectionFactory(clusterConfig, clientConfig)
factory.clientResources = redisCluster.clientResources
```
SDK 를 사용할 경우 제공된 RedisCluster 클래스만 이용하면 되는데,  
위와 같이 별도의 LettuceClient 를 따로 생성해 사용하고 있었던 것이다.  
또한 이렇게 생성한 LettuceClient 는 SDK 가 생성한 clientResource 를 같이 사용한다.  

그러다보니 Switchover 이벤트가 발생하면 SDK 내부에선 clientResources 를 shutdown 하고,  
이를 같이 사용하던 LettuceClient 에서 RedisCommandTimeout 이 발생했다.  

* 별도의 LettuceClient 를 사용하는 것까지는, Switchover 가 정상적으로 되지 않을 뿐 장애로 이어지진 않았을 것이다.
<br/>
<br/>

#### 전면 장애까지 발생한 이유 
1. CommandTimeout 설정이 없어 default 값인 60s 로 설정되었고,  
   웹에서 Redis 를 사용하는 API 를 호출했을 때, 60s 를 기다리는 동안 client (browser)에서 timeout 을 발생시키며 정상적인 이용이 불가했다.
2. 메인 화면에서 Redis 를 사용하고 있어 위의 이유로 접속 자체가 불가했다.
3. 임팩트가 적은 TW 리전에서 Switchover 를 먼저 진행했으나,  
   당시 TW 사용자들은 퇴근한 이후라 사용자가 없어 오류를 발견할 수 없었다.
4. Dev 환경에선 Switchover 단계가 없다. 단순히 New Version Redis 와의 연동 테스트만 할수있다.
<br/>
<br/>

### Solution
ShortTerm 으로 clientResources 를 set 하는 코드를 제거했고, CommandTimeout 설정도 500ms 로 설정했다.  
MiddleTerm 으로는 비정상적인 SDK 사용 코드를 수정한다.
<br/>
<br/>

### Lesson Learned
* 기존 코드를 제대로 분석하지 못했다.  
* SDK 를 비정상적으로 사용하는 것을 인지했음에도, Switchover 가 정상적으로 될것이라 오판했다.
* TW 리전에서 Switchover 후 직접 접속 해봤다면 발견했을 것이다.
<br/>
<br/>
<br/>