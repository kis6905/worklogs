## Background
Java LTS 버전 단위로 추가된 기능 정리

### Java 8
* 람다
* Stream API
* 메소드 참조
* Interface default method, static method
* Optional
* LocalDate, LocalDateTime

### Java 11
* Interface private method
* Reactive Streams
  * java.util.concurrent.Flow 를 통해 Reactive Streams 를 지원한다.
* Default gc 가 G1GC 이다.
* var 키워드
* Http Client
  ```java
  HttpClient client = HttpClient.newHttpClient();

  HttpRequest request = HttpRequest.newBuilder()
  .uri(URI.create("https://www.naver.com"))
  .build();
  
  HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
  String responseBody = response.body();
  ```
* 새로운 String methods
  * isBlank(): 문자열이 공백인지 확인
  * lines(): 문자열을 라인 스트림으로 변환
  * strip(), stripLeading(), stripTrailing(): 공백을 제거
  * repeat(int count): 문자열을 반복
* 새로운 Files methods
  * writeString, readString  
    ```java
    Path path = Paths.get("example.txt");
    Files.writeString(path, "Hello, World");

    String content = Files.readString(path);
    System.out.println(content);
    ```

### Java 17
* 패턴 매칭 for switch  
  ```java
  switch (obj) {
    case Integer i -> System.out.println("Integer: " + i);
    case String s -> System.out.println("String: " + s);
    default -> System.out.println("Unknown type");
  }
  ```
* Sealed Classes - 상속 제한  
  ```java
  public abstract sealed class Person permits Member, Manager {}
  ```
* Record Patterns - 불변 데이터 객체를 간편하게 만들 수 있다.
  ```java
  public record Param(String key, String value) {}
  ```

### Java 21
* Sequenced Collections
  * 순서를 가진 Collection을 표현하는 새로운 인터페이스
  * 첫 번째 요소와 마지막 요소에 접근하고 해당 요소를 역순으로 처리하기 위한 일관된 API를 제공
* Virtual Threads
* String Templates
  ```java
  String name = "Java";
  String message = STR."Hello, \{name}!";
  ```
* Key Encapsulation Mechanism API
  * 공개 키 암호화를 사용하여 대칭 키를 보호하는 암호화 기술인 Key Encapsulation Mechanism(KEM)을 위한 API
* Generational ZGC
  * Z Garbage Collector(ZGC)를 확장하여 young and old object의 세대를 분리하여 유지함으로써 애플리케이션 성능을 개선
  * ZGC가 대체로 빨리 죽는 young object를 더 자주 수집할 수 있다
