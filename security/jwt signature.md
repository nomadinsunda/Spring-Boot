# 🔐 JWT 서명(Signature)이란?

## 1️⃣ JWT란?

JWT는 **JSON 기반의 토큰**으로, 클라이언트와 서버 간에 \*\*사용자 인증 정보 또는 클레임(claim)\*\*을 안전하게 전달하기 위한 표준입니다.

형식은 다음과 같습니다:

```
xxxxx.yyyyy.zzzzz
```

* `xxxxx`: Header (메타정보)
* `yyyyy`: Payload (클레임)
* `zzzzz`: Signature (서명)

---

## 2️⃣ Signature는 왜 필요한가?

JWT는 기본적으로 **Header**와 **Payload**를 **Base64로 인코딩**하여 전달합니다.
👉 하지만 **Base64는 인코딩일 뿐, 암호화가 아닙니다!**

즉, **누구나 Header, Payload를 쉽게 디코딩할 수 있음!**

그래서 필요한 것이 바로 \*\*Signature (서명)\*\*입니다.

### 🛡 Signature의 목적

| 목적     | 설명                                     |
| ------ | -------------------------------------- |
| 무결성 보장 | Payload가 중간에 **변조되지 않았는지 검증** 가능       |
| 신원 인증  | 이 JWT를 **서명한 주체가 누구인지 식별** 가능 (서명자 검증) |
| 위변조 방지 | 서명이 맞지 않으면 "이 토큰은 위조된 것!"이라고 판단 가능     |

---

## 3️⃣ JWT Signature는 어떻게 만들어지나?

JWT의 서명은 다음의 내용을 바탕으로 생성됩니다:

```
HMACSHA( base64UrlEncode(header) + "." + base64UrlEncode(payload), secret )
```

또는

```
RSASHA( base64UrlEncode(header) + "." + base64UrlEncode(payload), privateKey )
```

### 🔹 대칭 키 방식 (HMAC-SHA256 등)

```plaintext
Signature = HMAC-SHA256(
  data = base64UrlEncode(header) + "." + base64UrlEncode(payload),
  key = secret
)
```

* 사용하는 알고리즘: `HS256`, `HS384`, `HS512` 등
* 특징: 같은 **비밀 키(secret)** 를 알고 있는 주체만 서명을 생성/검증 가능

### 🔹 비대칭 키 방식 (RSA, ECDSA 등)

```plaintext
Signature = Sign(
  data = base64UrlEncode(header) + "." + base64UrlEncode(payload),
  privateKey
)
```

* 사용하는 알고리즘: `RS256`, `RS512`, `ES256`, `EdDSA` 등
* 특징: **개인키로 서명**, **공개키로 검증** → 송신자 검증 가능

---

## 4️⃣ Signature는 어떻게 검증되나?

### ① 클라이언트 → JWT를 서버에 전송

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### ② 서버는 다음 절차로 검증:

1. JWT를 `.`으로 분리하여 Header, Payload, Signature를 나눔
2. Header와 Payload를 Base64 디코딩
3. **Header에서 alg와 typ를 확인**
4. Header + Payload를 **서버에서 다시 서명 계산**
5. 클라이언트가 보낸 Signature와 비교
6. 같으면 → **유효한 토큰**, 다르면 → **위조된 토큰**

```java
String data = base64Url(header) + "." + base64Url(payload);
String expectedSignature = HMACSHA(data, secret);

if (expectedSignature.equals(clientSignature)) {
    // ✅ Valid JWT
} else {
    // ❌ Invalid JWT (tampered or fake)
}
```

---

## 5️⃣ JWT 서명 관련 실무 주의점

### ✅ secret 키는 충분히 길고 복잡해야 함

* `Keys.hmacShaKeyFor(byte[])`는 길이 체크를 하며, **HS512**는 **64byte 이상 권장**

### ✅ secret이 노출되면 위조 가능

* 대칭 방식(HMAC)을 사용할 경우, secret 키를 알고 있으면 **누구나 서명 생성 가능**
* 따라서 **서버 측만 보관**하고 절대 노출 금지

### ✅ Signature는 토큰 위조 방지 수단이지, 데이터 암호화가 아님

* Payload는 여전히 **Base64-encoded JSON** → 노출됨
* 민감한 정보는 담지 말 것

---

## 6️⃣ Spring Security와의 통합 예시

```java
// JWT 검증 코드
String username = jwtProvider.getUsernameFromToken(token);
Collection<? extends GrantedAuthority> authorities = jwtProvider.getAuthoritiesFromToken(token);

Authentication authentication = new UsernamePasswordAuthenticationToken(
        username, null, authorities);

SecurityContextHolder.getContext().setAuthentication(authentication);
```

---

## 📌 결론 요약

| 항목           | 설명                                  |
| ------------ | ----------------------------------- |
| Signature 역할 | Payload 변조 여부 검증, 송신자 인증            |
| 포함 방식        | JWT의 마지막 부분 (xxxxx.yyyyy.**zzzzz**) |
| 생성 방식        | HMAC (대칭), RSA/ECDSA (비대칭)          |
| 검증 방식        | Header + Payload를 다시 서명하고 비교        |
| 보안 포인트       | Payload는 암호화되지 않으므로 민감 정보는 담지 말 것   |
| 일반적 사용       | HS512 (비밀 키), RS256 (공개/개인 키)       |

