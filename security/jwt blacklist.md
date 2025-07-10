JWT의 **Blacklist**란, 이미 발급된 JWT 토큰을 \*\*만료 시간 전에 강제로 무효화(폐기)\*\*하기 위해 사용하는 방법입니다.
JWT는 기본적으로 **stateless**(서버가 상태를 저장하지 않음)하기 때문에, **발급 후에는 서버가 토큰을 '잊어버리는' 구조**입니다.
따라서 클라이언트가 유효한 JWT를 갖고 있으면, 그 누구든 인증에 성공할 수 있습니다.

> 그런데, 로그아웃이나 강제 차단 등의 이유로 **특정 JWT를 더 이상 유효하지 않게 만들어야 할 경우**,
> 서버는 별도의 **Blacklist 목록**을 만들어 **해당 토큰을 거부**하도록 처리합니다.

---

## ✅ 왜 JWT에는 Blacklist가 필요한가?

### 🔒 JWT는 기본적으로 "취소할 수 없다"

JWT는 다음과 같은 구조로 작동합니다:

* 서버가 JWT를 발급하고
* 클라이언트가 그것을 계속 사용
* 서버는 토큰만 검증하고 아무것도 기억하지 않음 (Stateless)

➡️ 그래서 **로그아웃해도 토큰은 여전히 유효**하며,
**누군가가 그 토큰을 탈취했다면 계속 사용할 수 있습니다.**

---

## ✅ Blacklist의 필요 사례

| 상황            | 설명                                      |
| ------------- | --------------------------------------- |
| 로그아웃          | 사용자가 로그아웃하면, 해당 토큰을 더 이상 사용하지 못하게 하고 싶음 |
| 관리자에 의한 계정 정지 | 발급된 토큰이 있더라도 즉시 효력을 무효화해야 함             |
| 보안 사고 발생      | 토큰이 탈취되었을 가능성이 있는 경우                    |

---

## ✅ 동작 방식

1. 사용자가 로그아웃 시 JWT를 \*\*Blacklist 저장소(예: Redis, DB 등)\*\*에 저장
2. 이후 요청에서 토큰이 전달되면,

   * **서버는 먼저 이 토큰이 Blacklist에 있는지 확인**
   * 있으면 인증 거부

```java
if (tokenProvider.validateToken(jwt) && !blacklistService.isBlacklisted(jwt)) {
    Authentication auth = tokenProvider.getAuthentication(jwt);
    SecurityContextHolder.getContext().setAuthentication(auth);
}
```

---

## ✅ 구현 방식

### 📌 저장소 예시

| 저장소           | 특징                                  |
| ------------- | ----------------------------------- |
| Redis         | TTL(만료 시간) 지원 → 토큰 만료 시간까지 자동 보관 가능 |
| RDB           | 관리 편리하지만 TTL 기능이 없어 주기적 정리가 필요      |
| In-memory Map | 단일 서버 개발 환경에서는 간단하게 사용 가능           |

### 📌 Redis 예시

```bash
SET blacklist:<token> true EX <token_expiry_in_seconds>
```

이렇게 하면 Redis가 만료 시간까지 해당 키를 유지한 뒤 자동 삭제합니다.

---

## ✅ 한계점

| 문제점              | 설명                                                      |
| ---------------- | ------------------------------------------------------- |
| Stateless의 장점 상실 | 서버가 상태(Blacklist)를 관리해야 하므로 본래 JWT의 stateless 설계 취지와 상충 |
| 성능 이슈            | 매 요청마다 Blacklist를 조회해야 하므로 부하 발생 가능                     |
| 확장성              | 서버가 여러 대일 경우 공유 저장소(예: Redis)가 필요함                      |

---

## ✅ 대안: Refresh Token Rotation + Access Token 단기화

Blacklist의 부담을 줄이기 위해 다음과 같은 전략을 함께 사용합니다:

* **Access Token은 매우 짧게(5분 이내)**
* **Refresh Token은 길게 설정**
* Refresh Token 재발급 시 **기존 Refresh Token 폐기** (rotation)

---

## ✅ 요약

| 항목 | 설명                                      |
| -- | --------------------------------------- |
| 정의 | JWT를 강제로 무효화시키기 위한 서버 측 저장 목록           |
| 목적 | 로그아웃, 강제 차단, 보안 사고 대응                   |
| 방식 | Redis 등에 JWT 문자열 저장 후 차단 확인             |
| 한계 | Stateless 설계에 반함, 성능 이슈 있음              |
| 대안 | 토큰 만료 시간 짧게 + Refresh Token Rotation 사용 |

---

## ✅ Spring Boot 3 + Redis 기반의 **JWT 블랙리스트** 구현 예제

---

## ✅ 1. Redis 설정

```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

---

## ✅ 2. Blacklist 저장 서비스

```java
@Service
public class TokenBlacklistService {

    private final RedisTemplate<String, String> redisTemplate;

    public TokenBlacklistService(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void blacklistToken(String token, long expirationMillis) {
        redisTemplate.opsForValue().set(token, "blacklisted", expirationMillis, TimeUnit.MILLISECONDS);
    }

    public boolean isBlacklisted(String token) {
        return redisTemplate.hasKey(token);
    }
}
```

---

## ✅ 3. 로그아웃 기능 추가

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api")
public class AuthController {

    private final TokenBlacklistService tokenBlacklistService;
    private final TokenProvider tokenProvider;

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(HttpServletRequest request) {
        String bearer = request.getHeader("Authorization");

        if (bearer != null && bearer.startsWith("Bearer ")) {
            String token = bearer.substring(7);

            if (tokenProvider.validateToken(token)) {
                long expiration = tokenProvider.getExpiration(token);
                tokenBlacklistService.blacklistToken(token, expiration);
                return ResponseEntity.ok().build();
            }
        }
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).build();
    }
}
```

---

## ✅ 4. JwtAuthenticationFilter에서 블랙리스트 검사

```java
if (StringUtils.hasText(jwt) 
    && tokenProvider.validateToken(jwt)
    && !tokenBlacklistService.isBlacklisted(jwt)) {

    Authentication authentication = tokenProvider.getAuthentication(jwt);
    SecurityContextHolder.getContext().setAuthentication(authentication);
}
```

---

## ✅ 5. TokenProvider에 `getExpiration()` 메서드 추가

```java
public long getExpiration(String token) {
    Date expiration = Jwts.parserBuilder()
        .setSigningKey(key)
        .build()
        .parseClaimsJws(token)
        .getBody()
        .getExpiration();

    return expiration.getTime() - System.currentTimeMillis();
}
```


