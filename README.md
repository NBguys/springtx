# 스프링 트랜잭션

### 트랜잭션 AOP 주의 사항 - 프록시 내부 호출

![screenshot](/img/screen1.jpg)
1. 클라이언트인 테스트 코드는 callService.internal() 을 호출한다. 여기서 callService 는 트랜       
   잭션 프록시이다.
2. callService 의 트랜잭션 프록시가 호출된다.
3. internal() 메서드에 @Transactional 이 붙어 있으므로 트랜잭션 프록시는 트랜잭션을 적용한다.
4. 트랜잭션 적용 후 실제 callService 객체 인스턴스의 internal() 을 호출한다.
   실제 callService 가 처리를 완료하면 응답이 트랜잭션 프록시로 돌아오고, 트랜잭션 프록시는 트랜잭션을 완   
   료한다.


### public 메서드만 트랜잭션 적용
스프링의 트랜잭션 AOP 기능은 public 메서드에만 트랜잭션을 적용하도록 기본 설정이 되어있다. 그래서    
protected , private , package-visible 에는 트랜잭션이 적용되지 않는다. 생각해보면 protected ,    
package-visible 도 외부에서 호출이 가능하다. 따라서 이 부분은 앞서 설명한 프록시의 내부 호출과는 무관하고,    
스프링이 막아둔 것이다.     

> 참고: 스프링 부트 3.0
> 스프링 부트 3.0 부터는 protected , package-visible (default 접근제한자)에도 트랜잭션이 적용된다.

### 트랜잭션 옵션 소개
@Transactional - 코드, 설명 순서에 따라 약간 수정했음 
```java
public @interface Transactional {
   String value() default "";
   String transactionManager() default "";
   Class<? extends Throwable>[] rollbackFor() default {};
   Class<? extends Throwable>[] noRollbackFor() default {};
   Propagation propagation() default Propagation.REQUIRED;
   Isolation isolation() default Isolation.DEFAULT;
   int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
   boolean readOnly() default false;
   String[] label() default {};
}
```

#### value, transactionManager
트랜잭션을 사용하려면 먼저 스프링 빈에 등록된 어떤 트랜잭션 매니저를 사용할지 알아야 한다. 생각해보면 코드로 직
접 트랜잭션을 사용할 때 분명 트랜잭션 매니저를 주입 받아서 사용했다. @Transactional 에서도 트랜잭션 프록시
가 사용할 트랜잭션 매니저를 지정해주어야 한다.    
사용할 트랜잭션 매니저를 지정할 때는 value , transactionManager 둘 중 하나에 트랜잭션 매니저의 스프링 빈
의 이름을 적어주면 된다.    
이 값을 생략하면 기본으로 등록된 트랜잭션 매니저를 사용하기 때문에 대부분 생략한다. 그런데 사용하는 트랜잭션 매
니저가 둘 이상이라면 다음과 같이 트랜잭션 매니저의 이름을 지정해서 구분하면 된다.     

#### rollbackFor
예외 발생시 스프링 트랜잭션의 기본 정책은 다음과 같다.     
언체크 예외인 RuntimeException , Error 와 그 하위 예외가 발생하면 롤백한다.      
체크 예외인 Exception 과 그 하위 예외들은 커밋한다.     
이 옵션을 사용하면 기본 정책에 추가로 어떤 예외가 발생할 때 롤백할 지 지정할 수 있다. 
```java
@Transactional(rollbackFor = Exception.class) 
```
예를 들어서 이렇게 지정하면 체크 예외인 Exception 이 발생해도 롤백하게 된다. (하위 예외들도 대상에 포함된다.)    
rollbackForClassName 도 있는데, rollbackFor 는 예외 클래스를 직접 지정하고,     
rollbackForClassName 는 예외 이름을 문자로 넣으면 된다.    

#### noRollbackFor
앞서 설명한 rollbackFor 와 반대이다. 기본 정책에 추가로 어떤 예외가 발생했을 때 롤백하면 안되는지 지정할 수 있다.
예외 이름을 문자로 넣을 수 있는 noRollbackForClassName 도 있다.    
롤백 관련 옵션에 대한 더 자세한 내용은 뒤에서 더 자세히 설명한다.    

#### propagation
트랜잭션 전파에 대한 옵션이다. 

#### isolation
트랜잭션 격리 수준을 지정할 수 있다. 기본 값은 데이터베이스에서 설정한 트랜잭션 격리 수준을 사용하는 DEFAULT이다.    
대부분 데이터베이스에서 설정한 기준을 따른다. 애플리케이션 개발자가 트랜잭션 격리 수준을 직접 지정하는 경우는 드물다.      

* DEFAULT : 데이터베이스에서 설정한 격리 수준을 따른다.
* READ_UNCOMMITTED : 커밋되지 않은 읽기
* READ_COMMITTED : 커밋된 읽기
* REPEATABLE_READ : 반복 가능한 읽기
* SERIALIZABLE : 직렬화 가능
> 참고: 강의에서는 일반적으로 많이 사용하는 READ COMMITTED(커밋된 읽기) 트랜잭션 격리 수준을 기준으로 설명한다.
> 트랜잭션 격리 수준은 데이터베이스에 자체에 관한 부분이어서 이 강의 내용을 넘어선다. 트랜잭션 격리 수준에
> 대한 더 자세한 내용은 데이터베이스 메뉴얼이나, JPA 책 16.1 트랜잭션과 락을 참고하자

#### timeout
트랜잭션 수행 시간에 대한 타임아웃을 초 단위로 지정한다. 기본 값은 트랜잭션 시스템의 타임아웃을 사용한다. 운영      
환경에 따라 동작하는 경우도 있고 그렇지 않은 경우도 있기 때문에 꼭 확인하고 사용해야 한다.     
timeoutString 도 있는데, 숫자 대신 문자 값으로 지정할 수 있다.     

#### label
트랜잭션 애노테이션에 있는 값을 직접 읽어서 어떤 동작을 하고 싶을 때 사용할 수 있다. 일반적으로 사용하지 않는다.    

#### readOnly
트랜잭션은 기본적으로 읽기 쓰기가 모두 가능한 트랜잭션이 생성된다.
readOnly=true 옵션을 사용하면 읽기 전용 트랜잭션이 생성된다. 이 경우 등록, 수정, 삭제가 안되고 읽기 기능만    
작동한다. (드라이버나 데이터베이스에 따라 정상 동작하지 않는 경우도 있다.) 그리고 readOnly 옵션을 사용하면 읽     
기에서 다양한 성능 최적화가 발생할 수 있다.     
readOnly 옵션은 크게 3곳에서 적용된다.    

* 프레임워크
   * JdbcTemplate은 읽기 전용 트랜잭션 안에서 변경 기능을 실행하면 예외를 던진다.
   * JPA(하이버네이트)는 읽기 전용 트랜잭션의 경우 커밋 시점에 플러시를 호출하지 않는다. 읽기 전용이니 변   
     경에 사용되는 플러시를 호출할 필요가 없다. 추가로 변경이 필요 없으니 변경 감지를 위한 스냅샷 객체도
     생성하지 않는다. 이렇게 JPA에서는 다양한 최적화가 발생한다.
     * JPA 관련 내용은 JPA를 더 학습해야 이해할 수 있으므로 지금은 이런 것이 있다 정도만 알아두자.

* JDBC 드라이버
   * 참고로 여기서 설명하는 내용들은 DB와 드라이버 버전에 따라서 다르게 동작하기 때문에 사전에 확인이 필
     요하다.
   * 읽기 전용 트랜잭션에서 변경 쿼리가 발생하면 예외를 던진다.
   * 읽기, 쓰기(마스터, 슬레이브) 데이터베이스를 구분해서 요청한다. 읽기 전용 트랜잭션의 경우 읽기(슬레이
     브) 데이터베이스의 커넥션을 획득해서 사용한다.
     * 예) https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-source-replica-replication-connection.html

* 데이터베이스
   * 데이터베이스에 따라 읽기 전용 트랜잭션의 경우 읽기만 하면 되므로, 내부에서 성능 최적화가 발생한다.

### 예외와 트랜잭션 커밋, 롤백 - 기본
예외가 발생했는데, 내부에서 예외를 처리하지 못하고, 트랜잭션 범위( @Transactional가 적용된 AOP ) 밖으로 예     
외를 던지면 어떻게 될까?
예외 발생시 스프링 트랜잭션 AOP는 예외의 종류에 따라 트랜잭션을 커밋하거나 롤백한다.     
언체크 예외인 RuntimeException , Error 와 그 하위 예외가 발생하면 트랜잭션을 롤백한다.      
체크 예외인 Exception 과 그 하위 예외가 발생하면 트랜잭션을 커밋한다.    
물론 정상 응답(리턴)하면 트랜잭션을 커밋한다.    

#### 트랜잭션 로그 설정
application.properties
```properties
logging.level.org.springframework.transaction.interceptor=TRACE
logging.level.org.springframework.jdbc.datasource.DataSourceTransactionManager=DEBUG
#JPA log
logging.level.org.springframework.orm.jpa.JpaTransactionManager=DEBUG
logging.level.org.hibernate.resource.transaction=DEBUG
```

### 스프링 트랜잭션 전파
트랜잭션을 각각 사용하는 것이 아니라, 트랜잭션이 이미 진행중인데, 여기에 추가로 트랜잭션을 수행하면 어떻게 될까?        
기존 트랜잭션과 별도의 트랜잭션을 진행해야 할까? 아니면 기존 트랜잭션을 그대로 이어 받아서 트랜잭션을 수행해야 할까?
이런 경우 어떻게 동작할지 결정하는 것을 트랜잭션 전파(propagation)라 한다.    

* 원칙
    * 모든 논리 트랜잭션이 커밋되어야 물리 트랜잭션이 커밋된다.
    * 하나의 논리 트랜잭션이라도 롤백되면 물리 트랜잭션은 롤백된다.

### 스프링 트랜잭션 전파 - 다양한 전파 옵션
스프링은 다양한 트랜잭션 전파 옵션을 제공한다. 전파 옵션에 별도의 설정을 하지 않으면 REQUIRED 가 기본으로 사용된다.      
참고로 실무에서는 대부분 REQUIRED 옵션을 사용한다. 그리고 아주 가끔 REQUIRES_NEW 을 사용하고, 나머지는 거      
의 사용하지 않는다. 그래서 나머지 옵션은 이런 것이 있다는 정도로만 알아두고 필요할 때 찾아보자.   

#### REQUIRED
가장 많이 사용하는 기본 설정이다. 기존 트랜잭션이 없으면 생성하고, 있으면 참여한다.
트랜잭션이 필수라는 의미로 이해하면 된다. (필수이기 때문에 없으면 만들고, 있으면 참여한다.)
* 기존 트랜잭션 없음: 새로운 트랜잭션을 생성한다.
* 기존 트랜잭션 있음: 기존 트랜잭션에 참여한다.

#### REQUIRES_NEW
항상 새로운 트랜잭션을 생성한다.
* 기존 트랜잭션 없음: 새로운 트랜잭션을 생성한다.
* 기존 트랜잭션 있음: 새로운 트랜잭션을 생성한다.

#### SUPPORT
트랜잭션을 지원한다는 뜻이다. 기존 트랜잭션이 없으면, 없는대로 진행하고, 있으면 참여한다.
* 기존 트랜잭션 없음: 트랜잭션 없이 진행한다.
* 기존 트랜잭션 있음: 기존 트랜잭션에 참여한다.

#### NOT_SUPPORT
트랜잭션을 지원하지 않는다는 의미이다.
* 기존 트랜잭션 없음: 트랜잭션 없이 진행한다.
* 기존 트랜잭션 있음: 트랜잭션 없이 진행한다. (기존 트랜잭션은 보류한다)

#### MANDATORY
의무사항이다. 트랜잭션이 반드시 있어야 한다. 기존 트랜잭션이 없으면 예외가 발생한다.
* 기존 트랜잭션 없음: IllegalTransactionStateException 예외 발생
* 기존 트랜잭션 있음: 기존 트랜잭션에 참여한다.

#### NEVER
트랜잭션을 사용하지 않는다는 의미이다. 기존 트랜잭션이 있으면 예외가 발생한다. 기존 트랜잭션도 허용하지 않는 강
한 부정의 의미로 이해하면 된다.
* 기존 트랜잭션 없음: 트랜잭션 없이 진행한다.
* 기존 트랜잭션 있음: IllegalTransactionStateException 예외 발생

#### NESTED
* 기존 트랜잭션 없음: 새로운 트랜잭션을 생성한다.
* 기존 트랜잭션 있음: 중첩 트랜잭션을 만든다.
    * 중첩 트랜잭션은 외부 트랜잭션의 영향을 받지만, 중첩 트랜잭션은 외부에 영향을 주지 않는다.
    * 중첩 트랜잭션이 롤백 되어도 외부 트랜잭션은 커밋할 수 있다.
    * 외부 트랜잭션이 롤백 되면 중첩 트랜잭션도 함께 롤백된다.

* 참고
    * JDBC savepoint 기능을 사용한다. DB 드라이버에서 해당 기능을 지원하는지 확인이 필요하다.     
    * 중첩 트랜잭션은 JPA에서는 사용할 수 없다.   

### 트랜잭션 전파와 옵션
isolation , timeout , readOnly 는 트랜잭션이 처음 시작될 때만 적용된다. 트랜잭션에 참여하는 경우에는 적용 되지 않는다.       
예를 들어서 REQUIRED 를 통한 트랜잭션 시작, REQUIRES_NEW 를 통한 트랜잭션 시작 시점에만 적용된다
