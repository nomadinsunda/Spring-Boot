## ✅ Refresh Token 운용 개요

| 항목    | 설명                                                              |
| ----- | --------------------------------------------------------------- |
| 목적    | Access Token이 만료된 경우, 다시 로그인하지 않고 새로운 Access Token을 발급받기 위함     |
| 특성    | 일반적으로 \*\*긴 유효시간(7\~30일)\*\*을 가짐                                |
| 보안    | 탈취 시 위험 크므로, DB 또는 Redis 등에 **서버 측 저장 및 검증 필요**                 |
| 발급 시점 | 최초 로그인 시 Access Token과 함께 발급                                    |
| 저장 위치 | **클라이언트: 보안 쿠키 또는 HttpOnly Cookie** / **서버: DB, Redis, Memory** |
| 인증 여부 | Refresh Token 자체로는 인증되지 않으며, **검증 후 Access Token 재발급 용도로만 사용**  |

---

## 🧩 기본 구조

```text
[로그인 요청]
   ↓
[access_token + refresh_token 발급]
   ↓
클라이언트 저장 (e.g., Cookie, LocalStorage)
   ↓
[access_token 만료]
   ↓
[refresh_token으로 토큰 재발급 요청]
   ↓
서버는 refresh_token 검증 후
→ access_token 재발급 + (옵션) refresh_token도 재발급
```

---

## ✅ 실전 운용 방식 (Spring Security 기반)

### 1. **Refresh Token 발급 시 저장**

```java
public String createAndPersistRefreshTokenForUser(Authentication authentication) {
    String refreshToken = createToken(authentication, false); // long expiry
    String username = authentication.getName();
    
    RefreshToken tokenEntity = new RefreshToken(username, refreshToken, expirationTime);
    refreshTokenRepository.save(tokenEntity);
    
    return refreshToken;
}
```

* **DB 컬럼**: `username`, `refresh_token`, `expiry`, `revoked`, `issued_at`, `ip_address`, `device_info`

### 2. **Refresh 요청 처리 (예: /api/refresh-token)**

```java
@PostMapping("/api/refresh-token")
public ResponseEntity<?> refresh(@RequestBody String refreshToken) {
    if (!tokenProvider.validateToken(refreshToken)) {
        throw new InvalidTokenException("Invalid refresh token");
    }

    String username = tokenProvider.getUsernameFromToken(refreshToken);
    
    RefreshToken savedToken = refreshTokenRepository.findByUsername(username)
        .orElseThrow(() -> new TokenNotFoundException("No token saved"));

    if (!savedToken.getToken().equals(refreshToken) || savedToken.isExpired()) {
        throw new InvalidTokenException("Token mismatch or expired");
    }

    Authentication auth = tokenProvider.getAuthentication(refreshToken);
    String newAccessToken = tokenProvider.createToken(auth, true);
    
    return ResponseEntity.ok(new AccessTokenResponse(newAccessToken));
}
```

---

## 🚨 보안 고려 사항

| 항목              | 권장 사항                                              |
| --------------- | -------------------------------------------------- |
| 저장 위치           | 서버 측 저장 (DB/Redis 등) 필수                            |
| 토큰 회전(Rotation) | 재발급 시 **기존 RefreshToken 무효화 후 새 RefreshToken 발급**  |
| 탈취 방지           | 클라이언트에는 HttpOnly, Secure Cookie로 저장                |
| IP/Device 제한    | refresh 요청 시 클라이언트 IP, User-Agent 비교               |
| 로그아웃 처리         | 로그아웃 시 Refresh Token DB에서 제거                       |
| 동시 로그인 제어       | 사용자 1명당 1개의 RefreshToken만 허용하거나, 여러 기기 구분 가능하게 구조화 |

---

## ✅ Refresh Token Rotation (선택적 강화)

> 🔁 "새로운 Access Token과 함께 **새로운 Refresh Token도** 재발급하고, 기존 Refresh Token은 폐기하는 방식"

* **장점**: 탈취 시 피해 최소화
* **단점**: 클라이언트가 오래된 토큰을 쓰면 에러 발생 → UX 고려 필요

---

## 🛠️ 실무에서 자주 쓰는 구조 요약

| 종류                | 설명                                                          |
| ----------------- | ----------------------------------------------------------- |
| **Access Token**  | 5\~15분 유효, JWT, 인증/인가에 사용, 서버 저장 안함                         |
| **Refresh Token** | 7\~30일 유효, JWT 또는 UUID, DB/Redis에 저장, Access Token 재발급에만 사용 |
| **쿠키 저장 방식**      | RefreshToken을 HttpOnly Cookie에 저장하여 JS 접근 차단                |

---

## 🔐 보안 팁

* refresh token 요청을 처리하는 `/api/refresh-token`은 **IP/Device 로그 기록**
* refresh token 탈취 대비를 위해 **발급 시마다 UUID 세션 키와 매핑하여 rotation**
* 만료되지 않은 refresh token이 여러 개 존재하지 않도록 조치

---

## ❓예상 질문에 대한 답변

| 질문                                       | 답변                                                  |
| ---------------------------------------- | --------------------------------------------------- |
| 클라이언트에 refresh token 저장해도 괜찮은가?          | Yes, 단 HttpOnly + Secure Cookie 필수. LocalStorage는 X |
| AccessToken이 만료되었는데 refresh token도 만료되면? | 401 응답 → 로그인 다시 유도                                  |
| JWT 자체를 refresh token으로 써도 되나?           | 가능하나 DB 저장과 검증을 병행해야 안전                             |

---

## 📌 결론

Refresh Token은 **인증 상태를 연장하면서도 보안을 유지하기 위한 핵심 장치**입니다.
Spring Security에서는 반드시 다음을 지켜야 합니다:

* ✅ 서버 저장 (DB, Redis)
* ✅ 유효성 검증
* ✅ 회전(Rotation) 전략 고려
* ✅ 만료 및 무효화 처리
* ✅ 로그아웃 및 비정상 사용 탐지


