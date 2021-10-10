---
layout: post
title: '[java] SPRING, AOP, JDBC를 활용해 쿼리 개수를 세어보자 (쿼리 스파이)'
description: >
  SPRING, AOP, JDBC를 활용해 쿼리 개수를 세어보자 (쿼리 스파이)
tags: [jojal-jojal]
author: nabom
---
# SPRING, AOP, JDBC를 활용해 쿼리 개수를 세어보자 (쿼리 스파이)
## 쿼리 개수를 세는 작업이 필요했던 이유

이번 팀 프로젝트에서는 JPA 를 사용하게 되었다. 자동으로 쿼리를 작성해 디비에 요청을 하다보니 생각치도 못한 곳에서 여러 개의 쿼리를 날리며 (n+1 문제) 서비스를 느리게 만들었다. 이 문제를 확인하기 위해 매번 날아가는 쿼리를 확인해가며 n+1 문제가 발생하는 지 확인해야했다. 이 부분을 자동으로 세어주는 '쿼리 스파이'를 만들어 쿼리 문제 개선의 발판을 만들었다.

## '쿼리 스파이' 고민의 시작

처음엔 많은 고민이 있었다. p6spy 라는 쿼리 로그를 사용하는 라이브러리를 커스터마이징 해 로그를 찍을 때마다 카운트를 올려줄까? JPA 에서 사용되는 EntityManager 를 이용해볼까? 이렇게되면 너무 한 기술(p6spy 혹은 JPA)에 의존적이라 생각하게 되어 가장 기술에 덜 의존적인 부분이 어디일까 고민하다 jdbc api의 흐름인 `Connection` 의 `PreparedStatement` 부분을 이용하자라고 생각하게 되었다.

## '쿼리 스파이' 의 흐름

먼저, JPA 라는 기술의 흐름을 알아보자. JPA 는 Hibernate 를 사용하며 Hibernate 는 jdbc api를 사용하게 된다.  그리고 필자는 이 jdbc api의 흐름을 이용할 것이다.

JdbcTemplate의 흐름은 다음과 같다.

1. `DataSource` 를 주입 받는다.
2. `DataSource` 에서 `getConnection()` 메서드를 이용해 `Connection` 을 획득한다.
3. `Connection` 의 `prepareStatement()` 메서드를 이용해 `PreparedStatement` 를 획득한다.
4. `PreparedStatement` 의 `executeQuery()` 메서드를 이용해 쿼리를 실행한다.

즉, `executeQuery()` 의 실행 횟수를 알아내게 되면 쿼리 개수를 알 수 있는 것이다. 목표는 `PreparedStatement` 의 프록시를 만들어 `executeQuery()` 를 호출할 때마다 카운팅을 해주는 작업을 한다.

## 만들어보기

위에서 이야기했듯이 우리에게 필요한 건 `PreparedStatement` 의 프록시를 대신 사용하게 하는 것이다. `PreparedStatement` 의 프록시를 사용하려면 `Connection` 의 `preparedStatement()` 를 조작해야 하며 이는 `DataSource` 도 해당된다. 꽤나 복잡해 보이지만 간단하게 그림으로 살펴보자!

![Untitled](https://user-images.githubusercontent.com/63535027/136681171-1db1d070-f394-4aa8-881b-b4d4ae3da18e.png)

스프링 부트에서는 `DataSource` 를 빈으로 만든다면 해당 `DataSource` 를 이용해 위의 흐름을 진행한다. (기본 `DataSource` 빈은 h2 디비를 이용한 `DataSource` 이다.) 그렇기에 빈으로 등록된 부분을 Proxy로 만들 수 있는 스프링 AOP를 이용해 `DataSource` 를 해결할 것이고 다른 부분들은 다이나믹 프록시를 이용해 해결할 것이다.(일반 프록시를 만들어 해결할 수 있지만 각각의 인터페이스의 메서드가 너무 많아서 다이나믹 프록시로 구현했다.)

먼저, `QueryCounter` 라는 쿼리를 카운트하기 편한 객체를 만들어보자. (테스트 용도로만 사용하기에 `QueryCounter` 를 빈으로 등록해서 사용하게 되었다. 실제 서버에서도 사용하려면 쓰레드 세이프하게 다시 구성해야한다.) 간단한 흐름은 다음과 같다.

```java
public class QueryCounter {

    private QueryResult queryResult;
    private Count count;
    private boolean countable;

    public QueryCounter() {
        this.countable = false;
    }

    public void startCount() {
        this.countable = true;
        this.count = new Count(0);
        this.queryResult = new QueryResult();
    }

    public void upCount(String statement) {
        this.count = count.upCount();
        this.queryResult.save(count, statement);
    }

    public QueryResult endCount() {
        final QueryResult result = this.queryResult;
        this.countable = false;
        return result;
    }

    public boolean isCountable() {
        return countable;
    }
}
```

다음은, `PreparedStatement` 의 다이나믹 프록시를 만들어 위에서 만든 쿼리카운터를 넣어주자. jdk dynamic proxy는 다음과 같이 만들 수 있다.(`Connection` 과 `PreparedStatement` 모두 인터페이스이기 때문에 jdk 로 dynamic proxy를 만들 수 있다. 만일 클래스라면 CGLib 를 알아보자.)

**Dynamic proxy 만드는 코드**

```java
Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                "InvocationHandler")
```

이제 `InvocationHandler` 부분에 들어갈 부분을 구현한다. (프록시의 모든 요청은 `InvocationHandler` 의 `invoke(...)` 로 들어오게된다.)

**ProxyPreparedStatementHandler**

```java
public class ProxyPreparedStatementHandler implements InvocationHandler {

    private final Object preparedStatement;    //target
    private final QueryCounter queryCounter;

    public ProxyPreparedStatementHandler(Object preparedStatement,
                                         QueryCounter queryCounter) {
        this.preparedStatement = preparedStatement;
        this.queryCounter = queryCounter;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if(method.getName().equals("executeQuery")) {
						if(queryCounter.isCountable()) {
		            queryCounter.countUp();
						}
        }
        return method.invoke(preparedStatement, args);
    }
}
```

`executeQuery` 라는 메서드가 호출 될 때만 쿼리카운터에서 카운트가 되게끔 코드를 구현했다. 이제 `Connection` 이 `prepareStatement()` 메서드를 호출할 때 위의 프록시를 리턴하게끔 만들면 된다.

**ProxyConnectionHandler**

```java
public class ProxyConnectionHandler implements InvocationHandler {

    private final Object connection;
    private final QueryCounter queryCounter;

    public ProxyConnectionHandler(Object connection, PerformanceLoggingForm loggingForm) {
        this.connection = connection;
        this.queryCounter = queryCounter;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        final Object returnValue = method.invoke(connection, args);
        if (method.getName().equals("prepareStatement")) {
            return Proxy.newProxyInstance(
                returnValue.getClass().getClassLoader(),
                returnValue.getClass().getInterfaces(),
                new ProxyPreparedStatementHandler(returnValue, queryCounter));
        }
        return returnValue;
    }
}
```

이제 마지막으로 `DataSource` 가 `getConnection` 을 호출할 때 위에서 만든 `Proxy Connection` 을 리턴하도록 만들어보자. 이 부분은 스프링 AOP 를 이용해보자!

**Spring AOP 코드**

```java
@Component
@Aspect
@RequiredArgsConstructor
public class QueryCounterAop {
    
    private final QueryCounter queryCounter;

    @Around("execution(* javax.sql.DataSource.getConnection())")
    public Object datasource(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        final Object proceed = proceedingJoinPoint.proceed();
        return Proxy.newProxyInstance(
            proceed.getClass().getClassLoader(),
            proceed.getClass().getInterfaces(),
            new ProxyConnectionHandler(proceed, queryCounter));
    }
}
```

## QueryCounter 로 쿼리 세기

이제 `QueryCounter` 를 주입받아 쿼리를 세어 주면 된다.

```java
@SpringBootTest
@ActiveProfiles("test")
public class TestQueryCounter {

    @Autowired
    private QueryCounter queryCounter;
    @Autowired
    private DrinkService drinkService;

    @Test
    @DisplayName("query counter test")
    public void queryCounter() {
        queryCounter.startCount();
        drinkService.showAllDrinksByPage(Pageable.ofSize(8), LoginMember.anonymous());
        final QueryResult queryResult = queryCounter.endCount();
        System.out.println("쿼리 개수 : " + queryResult.queryCount());
    }
}
```

이것을 활용해 생각보다 쿼리 개수가 많은 로직에서 n+1 을 의심해볼 수 있다. 또한 모든 로직에 쿼리 개수를 셀 수 있게 하고 5개 이상 쿼리 개수가 세어진다면 WARN level 로 로깅해 다시 한 번 확인할 수 있게 만들어 의도치 않은 쿼리를 방지할 수 있다.

다음 글은 이 기술을 활용해 편하게 AWS의 CloudWatch 에서 각 api 별 실제 트랜잭션 시간, 쿼리 개수, 쿼리 실행 시간 등 편하게 퍼포먼스를 확인할 수 있는 방법을 소개할 것이다.
