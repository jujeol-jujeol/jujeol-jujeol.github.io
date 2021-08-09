---
layout: post
title: '[java] 프록시 패턴으로 인터셉터 Path와 Method 설정하기'
description: >
  프록시 패턴으로 인터셉터 Path와 Method 설정하기
tags: [jojal-jojal]
author: nabom
---

# 프록시 패턴으로 인터셉터 Path와 Method 설정하기
안녕하세요! 이번 포스팅에서는 스프링 인터셉터에 Path와 Method를 검증할 수 있는 기능을 프록시 패턴으로 구현하며 프록시 패턴에 대해 알아볼게요!

먼저 프록시에 대해 알아볼게요! 프록시(Proxy)란 대리자라는 뜻이에요. 프록시 패턴 또한 원래 객체를 대신해 무언가를 일을 처리하는 역할이에요.

이제 우리 프로젝트에서 적용한 예제를 통해 프록시 패턴에 대해 더 자세히 알아볼게요!

```java
public class WebMvcConfig implements WebMvcConfigurer {

    private final JwtTokenProvider jwtTokenProvider;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor())
                .addPathPatterns("/api/**");
    }

		private HandlerInterceptor loginInterceptor() {
        return new LoginInterceptor();
    }
}
```

위 코드처럼 스프링에는 인터셉터에 원하는 요청 api에만 작동할 수 있는 기능을 제공해요.

하.지.만, `GET`, `POST`, `PUT`, `DELETE` 등과 같은 메서드를 구분하는 기능은 제공하지 않기 때문에 인터셉터 안에 다음과 같이 코드가 추가돼요!

```java
public class LoginInterceptor implements HandlerInterceptor {

    private final JwtTokenProvider jwtTokenProvider;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
            Object handler) {

        final String method = request.getMethod();
        if(method.equals(HttpMethod.POST) || method.equals(HttpMethod.PUT)){
            String token = AuthorizationExtractor.extract(request);
            if (jwtTokenProvider.validateToken(token)) {
                return true;
            }
            throw new UnauthorizedUserException();
        }
        return true;
    }
}
```

여기서 또 애매한 건 각 api 요청마다 적용되는 메서드가 달라요. 예를 들어 `/members/me` 같은 경우는 모든 메서드에 `LoginInterceptor` 가 적용이 되어야하는 반면 `/drinks` 는 `GET` 요청을 제외한 메서드에 `LoginInterceptor` 가 적용이 되어야 했죠!

이러한 상황을 프록시 패턴을 이용해 해결할 거에요. 구조를 간단하게 그림으로 표현하면 다음과 같아요!

<img src="https://jujeol-jujeol.github.io/assets/img/interceptor/loginInterceptor.png" height="90%" width="70%">

그림을 보면 로그인 인터셉터를 `PathMatcher Interceptor`가 감싸고 있어요. `PathMatcher Interceptor`는 로그인 인터셉터에 가는 요청의 흐름을 제어하게 되죠. 즉, `PathMatcher` 인터셉터가 로그인 인터셉터가 어떤 api + http method 에 적용이 될지 정해주는 프록시가 되는 거죠!

그럼 천천히 `PathMatcher Interceptor` 를 만들어볼게요!

먼저, `PathMatcher Interceptor` 도 스스로 인터셉터여야 해요.

```java
public class PathMatcherInterceptor implements HandlerInterceptor {
    
}
```

이 `PathMatcher Interceptor` 는 로그인 인터셉터뿐만이 아니라 어떠한 인터셉터이든 요청의 흐름을 제어가 가능한 인터셉터로 만들 것이기 때문에 인터셉터 자체를 필드로 받게 만들었어요!

```java
public class PathMatcherInterceptor implements HandlerInterceptor {

    private final HandlerInterceptor handlerInterceptor;

    public PathMatcherInterceptor(HandlerInterceptor handlerInterceptor) {
        this.handlerInterceptor = handlerInterceptor;
    }

}
```

다음은 `url pattern`과 `HttpMethod` 를 소지하고 `target path` 가 포함된 `path` 인지 확인 책임을 가진 `PathContainer` 라는 객체를 만들게요! `PathMatcher` 는 스프링에서 제공하는 `AntPathMatcher` 를 사용했어요.

```java
public class PathContainer {

    private final PathMatcher pathMatcher;
    private final List<RequestPath> includePathPattern;
    private final List<RequestPath> excludePathPattern;

    public PathContainer() {
        this.pathMatcher = new AntPathMatcher();
        this.includePathPattern = new ArrayList<>();
        this.excludePathPattern = new ArrayList<>();
    }

    public boolean notIncludedPath(String targetPath, String pathMethod) {
        boolean excludePattern = excludePathPattern.stream()
                .anyMatch(requestPath -> anyMatchPathPattern(targetPath, pathMethod, requestPath));

        boolean includePattern = includePathPattern.stream()
                .anyMatch(requestPath -> anyMatchPathPattern(targetPath, pathMethod, requestPath));

        return excludePattern || !includePattern;
    }

    private boolean anyMatchPathPattern(String targetPath, String pathMethod, RequestPath requestPath) {
        return pathMatcher.match(requestPath.getPathPattern(), targetPath) &&
                requestPath.matchesMethod(pathMethod);
    }

    public void includePathPattern(String targetPath, PathMethod pathMethod) {
        this.includePathPattern.add(new RequestPath(targetPath, pathMethod));
    }

    public void excludePathPattern(String targetPath, PathMethod pathMethod) {
        this.excludePathPattern.add(new RequestPath(targetPath, pathMethod));
    }
}
```

이제 이 객체를 `Path Interceptor` 에 적용만하면 끝이에요!

```java
public class PathMatcherInterceptor implements HandlerInterceptor {

    private final HandlerInterceptor handlerInterceptor;
    private final PathContainer pathContainer;

    public PathMatcherInterceptor(HandlerInterceptor handlerInterceptor) {
        this.handlerInterceptor = handlerInterceptor;
        this.pathContainer = new PathContainer();
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
            Object handler) throws Exception {

				// pathContainer에 해당 요청 url과 메서드가 포함되지 않다면 바로 서비스 로직으로 요청이 간다.
        if (pathContainer.notIncludedPath(request.getServletPath(), request.getMethod())) {
            return true;
        }
				
				// 해당 요청 url과 메서드가 포함이 되어있다면 본 인터셉터(로그인 인터셉터)에 요청을 위임한다. (인터셉터 기능 실행)
        return handlerInterceptor.preHandle(request, response, handler);
    }

		// 외부에서 적용 path 패턴을 추가할 때
    public PathMatcherInterceptor includePathPattern(String pathPattern, PathMethod pathMethod) {
        pathContainer.includePathPattern(pathPattern, pathMethod);
        return this;
    }

		// 외부에서 미적용 path 패턴을 추가할 때
    public PathMatcherInterceptor excludePathPattern(String pathPattern, PathMethod pathMethod) {
        pathContainer.excludePathPattern(pathPattern, pathMethod);
        return this;
    }
}
```

이제 로그인 인터셉터와 함께 인터셉터로 등록을 해볼까요?

```java
@Configuration
@RequiredArgsConstructor
public class WebMvcConfig implements WebMvcConfigurer {

    private final JwtTokenProvider jwtTokenProvider;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor())
                .addPathPatterns("/api/**");
    }

    private HandlerInterceptor loginInterceptor() {
        final PathMatcherInterceptor interceptor = 
								// PathMatcherInterceptor 에 로그인 인터셉터를 주입한다.
                new PathMatcherInterceptor(new LoginInterceptor(jwtTokenProvider));
        
				// PathMatcherInterceptor를 로그인 인터셉터인 것처럼 인터셉터로 등록한다.
				return interceptor
                .excludePathPattern("/**", PathMethod.OPTIONS)
                .includePathPattern("/members/me/**", PathMethod.ANY)
                .includePathPattern("/drinks/*/reviews/**", PathMethod.ANY)
                .excludePathPattern("/drinks/*/reviews", PathMethod.GET);
    }
}
```

위와 같은 `PathMatcherInterceptor` 를 만든다면 어떠한 인터셉터이든 저렇게 편하게 `PathPattern` 을 제어할 수 있어요!

## 마무리

이렇게 스프링 인터셉터에 Path와 Method를 검증할 수 있는 기능을 구현해봤어요! 프록시 패턴은 정말 많은 곳에서 사용이 가능한 유용한 패턴이에요! 객체의 정보를 늦게 가져올 수 있는 `Lazy Loading` 을 적용할 수도 있고 위처럼 실제 객체 사용에 접근에 대한 흐름제어도 가능하죠! (이것 말고도 다른 많은 활용법이 있어요!)

실제 객체 대신 무언가 일 처리가 필요하다면 프록시 패턴을 이용해보면 어떨까요~? =]
