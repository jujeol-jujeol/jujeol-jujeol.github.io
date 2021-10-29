---
layout: post
title: '[spring] Dirties Context 제거를 통한 테스트 최적화'
description: >
   Dirties Context 제거를 통한 테스트 최적화
tags: [jojal-jojal]
author: wedge
---

# 문제상황

인수테스트가 많아지며 테스트 시간이 증가하기 시작했습니다. 주절주절이 구축한 CI/CD는 PR을 보내면 테스트 코드 실행이 포함된 빌드 테스트를 거칩니다. 또한 빌드시에도 다시 한번 운영환경에서 테스트를 거치고 배포를 수행하게 되는데, 2분 이상이 되자 CI/CD 전 과정에 걸쳐서 10분 이상의 시간을 소요하게 되었습니다. 이는 서버 인프라 테스트나 다양한 기능이 비슷한 타이밍에 주절주절 베이스 코드에  합쳐지게 되었을 때 생산성을 저하하는 것으로 이어졌기 때문에, 팀 회의를 거쳐 테스트 최적화를 진행하기로 결정하였습니다.

# 사전정보

SpringBoot는 SpringBoot가 run 될 때마다 새롭게 어플리케이션 컨텍스트를 막기 위해 context caching 기능을 지원합니다.

컨텍스트 캐싱 기능이 없다면 매 테스트를 실행할 때마다 스프링의 Application Context를 로딩해주어야 합니다. 테스트가 2~3개 이면 크게 상관 없겠지만, 100개, 200개가 되면 상황이 다릅니다.

Spring TestContext 프레임워크는 한 번 ApplicationContext가 만들어지면 캐시에 저장합니다.

그리고 다른 테스트를 실행할 때 가능한 경우 재사용 하는데요, 여기서 가능한 경우란

1. 같은 bean의 조합을 필요로 하고
2. 이전 테스트에서 ApplicationContext가 오염되지 않은 경우

를 말합니다.

### 컨텍스트 캐싱의 조건

이 중 1번, 같은 bean의 조합은 Context Caching에서 cache key로 어떤 것을 쓰는지와 동일합니다. Spring TestContext에선 여러 configuration으로 이 key를 구성합니다. 공식 문서에 나타나 있는 설정 종류는 아래와 같습니다.

- `locations` (from `@ContextConfiguration`)
- `classes` (from `@ContextConfiguration`)
- `contextInitializerClasses` (from `@ContextConfiguration`)
- `contextCustomizers` (from `ContextCustomizerFactory`)
- `contextLoader` (from `@ContextConfiguration`)
- `parent` (from `@ContextHierarchy`)
- `activeProfiles` (from `@ActiveProfiles`)
- `propertySourceLocations` (from `@TestPropertySource`)
- `propertySourceProperties` (from `@TestPropertySource`)
- `resourceBasePath` (from `@WebAppConfiguration`)

여기에 더하여, TestStub 라이브러리인 Mockito를 사용했을 때, 특정 빈을 @MockBean으로 교체하면 contextCustomizers에 MockitoContextCustomizer가 추가 되어서 MockBean 처리한 bean의 조합이 달라질 경우 cache key가 달라지게 됩니다. 그러면 새로운 컨텍스트를 로딩하게 되므로 주의해야 합니다.

### 컨텍스트가 오염되었을 때(Dirties Context)

2번, 테스트에서 사용하는 컨텍스트가 오염되었다는 신호는 @DirtiesContext 어노테이션을 통해 줄 수 있습니다.

주절주절 프로젝트에서 인수테스트 시 상속받아 활용하고 있는 AcceptanceTest의 코드 입니다.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
@ExtendWith(RestDocumentationExtension.class)
@ActiveProfiles("test")
public class AcceptanceTest {
...
}
```

classMode 옵션을 주어 매 테스트 이후(after each test) 새롭게 어플리케이션 컨텍스트를 띄우게 됩니다.

상황에 맞게 사용하면 훌륭한 어노테이션이지만, 주절주절에서는 인수테스트에서 테스트 DB(h2)를 초기화 하기 위해 매 메소드마다 활용하게 되었습니다.

### 왜 @Transactional, @Rollback을 사용하지 않고 DirtiesContext를?

위에서 DB 롤백을 위해 DirtiesContext를 활용하였다고 말씀 드렸는데요, 스프링 테스트에서는 DB 롤백을 위해 @Transactioal 과 @Rollback을 지원하고 있습니다. 테스트 메소드에 해당 어노테이션이 있는 경우, TransactionSyncManager에서 테스트 메소드가 종료된 후 rollback 명령을 날려 테스트 동안 진행된 데이터를 롤백할 수 있습니다.

하지만 주절주절에서는 인수테스트 도구로 RestAssured를 활용하고 있었는데, RestAssured에선 Transactional 어노테이션이 동작하지 않습니다.

그 이유는 RestAssured와 MockMVC의 차이점 때문입니다.

MockMVC는 슬라이스 테스트 도구입니다. 스프링 서버를 별도로 띄우지 않고도 복잡한 스프링의 설정을 다 불러들이기 위하는 것에 사용 목적이 있습니다. 그래서 서블릿 컨테이너를 새롭게 생성하지 않고, 내부적으로 DispatcherServlet을 모킹하여 사용합니다. 같은 스레드에서 코드가 수행되기 때문에 Rollback이나 Transactional 어노테이션이 동작합니다.

그와 달리 RestAssured는 스프링 서버를 별도 포트(SpringBootTest의 webEnvironment 설정에 의해)에 띄워놓고, 유저 요청(HTTP request)를 모방하여 인수테스트를 진행하게 됩니다. 테스트가 진행되는 스레드와 실제 서버가 띄워져 있는 스레드가 다르기 때문에, RestAssured를 활용하는 코드에 Transactional이나 Rollback 코드를 붙여도 서버 쪽에는 영향을 주지 못 합니다. 도식화 하면 다음과 같습니다.

![1](https://user-images.githubusercontent.com/51393021/139366575-91965776-0b7d-459a-8b4d-c0004050edd7.png)


이때문에 DirtiesContext를 사용하여, 어플리케이션 컨텍스트를 새로 띄워 데이터베이스를 초기화 하는 방식으로 사용하였습니다.

이는 인수테스트가 추가 될 때마다 테스트 시간이 산술급수적으로 증가하는 결과로 나타나게 됩니다.

매번 DML (Truncate)을 날려 초기화하는 방법도 있지만 삭제와 관련된 DML은 누군가의 실수로 인해 테스트용 데이터를 미리 적용해놓은 DB에 영향을 줄 수 있어 기각하였습니다.

실제로 어떤 내용을 진행하였는지 아래에 서술하겠습니다.

# 개선 목표

1. DirtiesContext를 제거하여 test시간을 n초*수정 요망*으로 줄인다.

# 방법 논의

1. RestAssuered를 MockMVC로 변환한다.
   1. 기존에 활용하던 RequestBuilder를 인터페이스화 하여 MockMVC 구현체를 활용할 수 있도록 수정한다.

# 진행

1. 기존에는 RequestBuilder 라고 하는 인수테스트 툴을 만들어 사용하고 있었습니다.  [링크](https://github.com/woowacourse-teams/2021-jujeol-jujeol/blob/fa9f4ed9553f30fe5167f54ead01cb2907b75338/backend/src/test/java/com/jujeol/RequestBuilder.java)

    ```java
    @Component
    @ActiveProfiles("test")
    public class RequestBuilder {
    
        private final ObjectMapper objectMapper;
        private final LoginService loginService;
        private final QueryCounter queryCounter;
    
        private RestDocumentationContextProvider restDocumentation;
    
        ...
    		
    		// 각 메소드별 요청의 추상화
    		public class Function {
    				//Option Chaining을 통해 선택적으로 필요한 내용 구현 
            public Option get(String path, Object... pathParams) {
                return new Option(new GetRequest(path, pathParams));
            }
    				...
    		}
    
    		// 각 옵션을 활성화하거나 비활성화 할 수 있도록 추상화
    		public class Option {
    
            private final RestAssuredRequest request;
            private boolean logFlag;
            private DocumentHelper documentHelper;
            private UserHelper userHelper;
            private MultipartHelper multipartHelper
    
    				// Option chaining 
    				public Option withDocument(String identifier) {
                documentHelper.createDocument(identifier);
                return this;
            }
    				...		
    		}
    
    		// RestAssured.then()의 결과를 변환하는 역할을 하는 클래스 
    		public class HttpResponse {
    				...
    		}
    
    		public HttpResponse build() {
    				// RestAssured를 의존하여 테스트를 수행함 
    				RequestSpecification requestSpec = documentHelper.startRequest();
    				...
            return new HttpResponse(validatableResponse.extract(), queryResult);
    		}
    }
    ```

   해당 클래스의 역할은 중복되는 Dto 변환 로직, 인증 관련 처리, Content-type 설정 등 주절주절에서 기본 설정에 가까운 코드들을 캡슐화 해둔 클래스입니다.

   또한 RestDocs, logging, multipart 처리, 인증, 테스트 도중 몇번의 쿼리가 나가는지 확인하는 QueryCounte의 동작을 RequestBuilder 사용하는 코드에서 결정할 수 있도록 코드를 작성했습니다.

    ```java
    @Test
    void 카테고리_전체조회_테스트(){
    	...
    	// 활용 예시
    	new RequestBuilder().builder()
    				.get("/categories")
    				.withUser() // 인증을 활용하고 싶을 때
            .withDocument("category/show/all") // Restdocs snipet 생성
            .build()
            .convertBodyToList(CategoryResponse.class); // 주절주절 공용 DTO에 맞는 형변환 제공
    }
    ```

2. 기존에 작성한 인수테스트 코드를 고치지 않고 RestAssuered에서 MockMVC로 변경하기 위하여, RequestBuilder의 내부 테스트 실행부를 인터페이스화 하여 유연하게 변경할 수 있도록 수정하였습니다.

   <img width="730" alt="스크린샷 2021-10-29 오전 11 44 59" src="https://user-images.githubusercontent.com/51393021/139366785-34c9a406-1823-4552-a45e-fd2982ac59cb.png">

3. 이후 RestAssured 구현체 사용 & DirtiesContext를 제거하니 M1 Mac, 16G 메모리 팀원 기준 150s 걸리던 테스트가 15s 로 개선되었습니다.
4. PR결과, github action으로 빌드테스트를 하는 구간에서 4분 17초에서 1분53초로 감소한 것을 확인하여 CI과정에서 걸리는 병목을 의미있게 감소시켰습니다.

   <img width="1409" alt="스크린샷 2021-10-29 오전 11 49 06" src="https://user-images.githubusercontent.com/51393021/139366598-036d711d-4061-45c4-8f43-4ca366e4f377.png">
   <img width="1411" alt="스크린샷 2021-10-29 오전 11 49 35" src="https://user-images.githubusercontent.com/51393021/139366603-cafcf588-624f-4373-bf64-8a7c5726f112.png">

# 결론

PR을 보내고 빌드 테스트를 하는 동안 '세상에서 가장 오래 기다려야하는 5분'이라는 느낌으로 대기 했는데, 2분 내외로 줄어드니 살 맛납니다. 각종 최적화 기법을 적용해 테스트 시간을 줄여보는 것이 어떨지요!

# 참고

- [https://www.youtube.com/watch?v=jdlBu2vFv58&t=463s](https://www.youtube.com/watch?v=jdlBu2vFv58&t=463s)
- [https://bperhaps.tistory.com/entry/테스트-코드-최적화-여행기-1](https://bperhaps.tistory.com/entry/%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%BD%94%EB%93%9C-%EC%B5%9C%EC%A0%81%ED%99%94-%EC%97%AC%ED%96%89%EA%B8%B0-1)
- [https://docs.spring.io/spring-framework/docs/current/reference/html/testing.html](https://docs.spring.io/spring-framework/docs/current/reference/html/testing.html)