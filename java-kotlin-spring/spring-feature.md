
### Spring 5 (Boot 2.x)
* WebFlux
* Java 8+
* HTTP/2 지원
* Kotlin 지원

### Spring 6 (Boot 3.x)
* Java 17+
* Kotlin 1.7+
* Gradle 7.5+
* AOT(Ahead-Of-Time) 엔진 지원
* Servlet, JPA 네임스페이스 -> Jakarta 로 변경
  javax.* -> jakarta.*
* URL 마지막 "/" 지원 X
  * 아래 두 URL 모두 각각 처리
    * "/hello"
    * "/hello/"
* Spring Boot 3.2 부터 Virtual Thread 지원
