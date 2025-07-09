### 🔐 `access()` 메서드란?

Spring Security에서는 어떤 URL에 누가 접근할 수 있는지를 설정할 수 있습니다. 보통은 다음과 같은 방식으로 설정하죠:

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin").hasRole("ADMIN")
);
```

이처럼 `hasRole()`, `hasAuthority()`, `hasAnyAuthority()` 등의 메서드를 사용하면 **간단하고 직관적**입니다.

그런데 **좀 더 복잡하고 유연한 권한 부여 로직이 필요할 때** 사용할 수 있는 메서드가 바로 `access()`입니다.

---

### ⚙️ `access()`는 어떤 역할을 하나요?

`access()`는 사용자의 요청에 대해 접근을 허용할지 여부를 판단하는 **AuthorizationManager**를 지정할 수 있는 메서드입니다.

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin").access(new CustomAuthorizationManager())
);
```

즉, `access()`를 사용하면 권한 검사 방식 자체를 커스터마이징할 수 있고, 그 안에서 여러 조건들을 조합하거나 외부 시스템과 연동해서 권한 여부를 판단할 수도 있습니다.

---

### 💡 `AuthorizationManager`는 무엇인가요?

Spring Security 6부터 도입된 새로운 개념으로,

* 요청에 대한 정보 (예: 어떤 URI, 어떤 HTTP 메서드 등)와
* 현재 인증된 사용자 정보

를 기반으로 "이 사용자가 이 요청을 허용받을 수 있는가?"를 결정하는 객체입니다.

`AuthorizationManager`는 인터페이스이고, Spring에서는 다음과 같은 구현체를 제공합니다:

* `WebExpressionAuthorizationManager` : SpEL (Spring Expression Language)를 사용하여 권한을 표현하는 방식
  예: `"hasRole('ADMIN') or hasAuthority('SOME_PRIVILEGE')"`

```java
.access(new WebExpressionAuthorizationManager("hasRole('ADMIN')"))
```

---

### 🧠 그런데 왜 `access()` 사용을 신중하게 하라고 하나요?

* `hasRole()`, `hasAuthority()` 같은 메서드는 **짧고 직관적**이며, 코드를 읽는 사람에게 바로 의미가 전달됩니다.
* 반면 `access()`를 사용하면 내부에 SpEL이나 커스텀 로직이 들어가기 때문에, **코드를 읽고 해석하기가 어려워질 수 있습니다.**

  * 특히 여러 권한 조건을 복잡하게 조합할수록 가독성이 떨어집니다.
  * 테스트나 유지보수도 어려워질 수 있습니다.

---

### ✅ 결론: 언제 `access()`를 사용해야 할까?

* 단순한 권한 체크: `hasRole()`, `hasAuthority()` 등 사용 (권장)
* 복잡한 로직이 필요한 경우:

  * DB 조회 기반 권한 부여
  * 외부 API 호출
  * 사용자 세션 정보에 따른 조건 분기

이럴 때만 `access()` + `AuthorizationManager` 조합을 사용하는 것이 좋습니다.

---

### 예시 코드

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/secure").access((authentication, context) -> {
        boolean allowed = checkSomeCondition(authentication.get());
        return new AuthorizationDecision(allowed);
    })
);
```

위 코드는 `access()`에 직접 람다로 권한 판단 로직을 구현한 것입니다.

---


## ✅ CustomAuthorizationManager 구현

특정 조건 (예: `username`이 "admin"인 경우에만 접근 허용)을 만족할 때만 URL에 접근할 수 있게 하는 `CustomAuthorizationManager`를 만들어보겠습니다.

---

## 1. 기본 구조

`AuthorizationManager<RequestAuthorizationContext>` 인터페이스를 구현합니다:

```java
import org.springframework.security.authorization.AuthorizationDecision;
import org.springframework.security.authorization.AuthorizationManager;
import org.springframework.security.core.Authentication;
import org.springframework.security.web.access.intercept.RequestAuthorizationContext;

import java.util.function.Supplier;

public class CustomAuthorizationManager implements AuthorizationManager<RequestAuthorizationContext> {

    @Override
    public AuthorizationDecision check(Supplier<Authentication> authenticationSupplier,
                                       RequestAuthorizationContext context) {

        Authentication authentication = authenticationSupplier.get();

        // 인증되지 않았다면 접근 거부
        if (authentication == null || !authentication.isAuthenticated()) {
            return new AuthorizationDecision(false);
        }

        // 사용자 이름이 "admin"일 때만 허용
        String username = authentication.getName();
        boolean granted = "admin".equals(username);

        return new AuthorizationDecision(granted);
    }
}
```

---

## 2. Spring Security 설정에 적용

이제 이 커스텀 매니저를 `access()`에 적용합니다:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/admin").access(new CustomAuthorizationManager())  // 적용
            .anyRequest().authenticated()
        )
        .formLogin();

    return http.build();
}
```

---

## 3. 결과

* `/admin` 요청이 들어올 때,
* 현재 인증된 사용자의 `username`이 `"admin"`인 경우에만 접근 허용됩니다.
* 그 외 사용자나 인증되지 않은 경우는 **403 Forbidden** 발생.

---

## 📌 확장 예시

필요하다면 다음과 같이 더 복잡한 조건도 넣을 수 있습니다:

```java
boolean isAdmin = authentication.getAuthorities().stream()
    .anyMatch(grantedAuthority -> grantedAuthority.getAuthority().equals("ROLE_ADMIN"));

boolean fromKorea = context.getRequest().getRemoteAddr().startsWith("203."); // 예시

return new AuthorizationDecision(isAdmin && fromKorea);
```

---

## 🧠 참고

* `Supplier<Authentication>` 을 쓰는 이유는 Lazy Evaluation (지연 평가)을 통해 필요할 때만 인증 정보를 얻기 위함입니다.
* `RequestAuthorizationContext`는 `HttpServletRequest`에 접근할 수 있게 해줍니다:

  ```java
  context.getRequest().getMethod();
  context.getRequest().getHeader("X-Custom-Header");
  ```



