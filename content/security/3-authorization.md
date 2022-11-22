---
title: "3. Authorization"
metaTitle: "Authorization Description"
metaDescription: "This is the meta description for this page"
---

## Authorization

- 인증 처리과정에서 리소스를 요청한 클라이언트의 권한 목록을 얻을 수 있었다. 인증 처리과정에서 얻은 권한 목록은 요청 리소스에 대한 접근 권한을 판별할 때 사용된다. `Authorization` 섹션에서는 클라이언트의 인증이 정상적으로 완료된 이후, `Spring Security`가 어떻게 특정 리소스에 대한 접근 권한을 부여하는지 알아본다.

## Authorization Flow

- Spring Security 5.5 버전 이전까지 `FilterSecurityInterceptor`를 통해 권한 부여를 처리했다면, 5.5 버전부터 `AuthorizationFilter`가 `FilterSecurityInterceptor`를 대체하여 보다 간결한 권한 부여 처리가 가능해졌다. 따라서, `AuthorizationFilter`를 통한 인가 처리과정을 알아본다. 

### a. AuthorizationFilter
- 인증된 사용자의 요청에 대한 인가처리를 하는 필터이다.
- 해당 필터는 `SeuciryContextHolder`에서 `Authentication`을 얻고, 실질적인 인가 처리는 `AuthorizationManager`에게 위임한다.
- 예외
  - 인증객체가 없는 경우: `AuthenticationCredentialsNotFoundException`
  - 인가되지 않은 경우: `AccessDeniedException`

### b. AuthorizationManager
- 실질적인 인가 처리를 담당하는 함수형 인터페이스이다.
- `AuthorizationFilter`가 초기화될 때, 구현체 중 `RequestMatcherDelegatingAuthorizationManager`가 DI된다.
```java
@FunctionalInterface
public interface AuthorizationManager<T> {
    default void verify(Supplier<Authentication> authentication, T object) {
        AuthorizationDecision decision = check(authentication, object);
        if (decision != null && !decision.isGranted()) {
            throw new AccessDeniedException("Access Denied");
        }
    }

    @Nullable
    AuthorizationDecision check(Supplier<Authentication> authentication, T object);
}
```

### c. RequestMatcherDelegatingAuthorizationManager
- `AuthorizationManager` 구현체 중에서 가장 먼저 호출된다.
- 요청 리소스에 부여된 권한들을 `RequestMatcher`를 사용하여 매치되는 `AuthorizationManager` 구현체에게 인가 처리를 위임한다.
  - `RequestMatcher`는 Security를 설정할 때 `.antMatchers("/resources/**").hasRole("{ROLE}")`과 같은 메서드 체인 정보를 기반으로 생성된다.
- 평가식과 매치되는 `AuthorizationManager`가 없다면 `null`을 반환한다.
```java
@Override
public AuthorizationDecision check(Supplier<Authentication> authentication, HttpServletRequest request) {
    for (RequestMatcherEntry<AuthorizationManager<RequestAuthorizationContext>> mapping : this.mappings) {
        RequestMatcher matcher = mapping.getRequestMatcher();
        MatchResult matchResult = matcher.matcher(request);
        if (matchResult.isMatch()) {
            AuthorizationManager<RequestAuthorizationContext> manager = mapping.getEntry();
            return manager.check(authentication,
                    new RequestAuthorizationContext(request, matchResult.getVariables()));
        }
    }
    return null;
}
```

### Spring Security EL
| Expression Language | Description |
| --- | --- |
| `permitAll()` | 항상 `ture`로 평가한다. | 
| `denyAll()` | 항상 `false`로 평가한다. |
| `access(AuthorizationManager<RequestAuthorizationContext>)` | 커스텀한 `AuthorizationManager`로 평가한다. |
| `authenticated()` |  인증된 사용자에 대해서 `true`로 평가한다. |
| `hasRole(String role)` | - `Principal`이 `{role}`을 가지고 있는지 검증한다. |
| `hasAnyRole(String... roles)` | - `{roles}`에 존재하는 역할 중 `Principal`이 하나라도 가지고 있는지 검증한다. |
| `hasAuthority(String authority)` | - `Principal`이 `{authority}`를 가지고 있는지 검증한다. |
| `hasAnyAuthority(String... authorities)` | - `{authorities}`에 존재하는 권한 중 `Principal`이 하나라도 가지고 있는지 검증한다. |

> `hasRole("ROLE_admin")`처럼 권한이 `ROLE_`로 시작하지 않으면 `ROLE_`을 추가한다. 
> `DefaultWebSecurityExpressionHandler`의 `defaultRolePrefix`를 수정하면 변경할 수 있다. 