# 🔐 HMAC-SHA 알고리즘 완벽 해설

## 🧩 1. HMAC이란?

**HMAC**은 “**Hash-based Message Authentication Code**”의 약자로,
\*\*해시 함수(SHA-256 등)\*\*를 활용한 \*\*메시지 인증 코드(MAC)\*\*입니다.

> 즉, 메시지와 비밀 키를 조합해 고유한 해시를 만들어, 메시지가 위조되지 않았음을 검증하는 방식입니다.

---

## 🔁 2. 왜 HMAC이 필요한가?

단순히 해시 함수만 사용하는 것은 보안상 취약합니다:

```plaintext
SHA256("message") → anyone can generate it
```

✅ 그래서 등장한 방식이 **HMAC**, 즉:

```plaintext
HMAC_SHA256(secret, message) → Only someone with secret can generate this!
```

HMAC은 메시지를 **비밀키로 서명**하는 것과 유사합니다.
메시지가 변경되면 HMAC 값도 바뀌므로 **무결성**을 검증할 수 있습니다.

---

## ⚙️ 3. HMAC-SHA의 내부 구조

HMAC은 다음 수식으로 정의됩니다:

```plaintext
HMAC(K, m) = H((K' ⊕ opad) || H((K' ⊕ ipad) || m))
```

| 기호     | 의미                                 |    |                        |
| ------ | ---------------------------------- | -- | ---------------------- |
| `K`    | 비밀 키                               |    |                        |
| `K'`   | 키를 블록 크기에 맞게 패딩한 값 (SHA256은 64바이트) |    |                        |
| `H()`  | 해시 함수 (예: SHA-256, SHA-512)        |    |                        |
| `m`    | 메시지 (payload)                      |    |                        |
| `⊕`    | XOR 연산                             |    |                        |
| \`     |                                    | \` | 문자열 연결 (concatenation) |
| `opad` | 0x5c 반복 (외부 패딩, outer pad)         |    |                        |
| `ipad` | 0x36 반복 (내부 패딩, inner pad)         |    |                        |

### 🔁 작동 과정:

1. 키 K를 64바이트로 패딩 → `K'`
2. `K'`와 `ipad`를 XOR → `Si`
3. `Si || m`을 해시 → `Hi`
4. `K'`와 `opad`를 XOR → `So`
5. `So || Hi`를 다시 해시 → 결과값이 **HMAC**

---

## 🔑 4. SHA 종류별 HMAC 알고리즘

| 알고리즘          | 내부 해시 함수 | 출력 길이         |
| ------------- | -------- | ------------- |
| `HMAC-SHA256` | SHA-256  | 256비트 (32바이트) |
| `HMAC-SHA384` | SHA-384  | 384비트 (48바이트) |
| `HMAC-SHA512` | SHA-512  | 512비트 (64바이트) |

HMAC의 보안성은 사용되는 해시 함수에 따라 결정됩니다.

---

## 📦 5. Java에서의 HMAC-SHA 예제

```java
public static String hmacSha256(String secret, String message) {
    SecretKeySpec keySpec = new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
    Mac mac = Mac.getInstance("HmacSHA256");
    mac.init(keySpec);
    byte[] hmac = mac.doFinal(message.getBytes(StandardCharsets.UTF_8));
    return Base64.getEncoder().encodeToString(hmac);
}
```

---

## 🔐 6. JWT에서 HMAC-SHA는 어떻게 사용되나?

JWT는 다음 세 부분으로 구성됩니다:

```plaintext
<Header>.<Payload>.<Signature>
```

HMAC-SHA를 사용하는 경우:

```plaintext
Signature = HMAC-SHA256(secret, Base64UrlEncode(header) + "." + Base64UrlEncode(payload))
```

* 서명은 서버만 알고 있는 **비밀 키(secret)** 로 생성됨
* 수신자는 secret을 가지고 같은 방식으로 서명하여, 서명이 일치하면 **신뢰 가능**

---

## 🧱 7. HMAC의 보안성

* **충돌 회피성(Collision Resistance)**: 두 메시지가 같은 해시를 갖지 않도록 설계
* **2차 프리이미지 저항(Second pre-image resistance)**: 해시값이 주어졌을 때 원래 메시지를 추측하기 어려움
* **MAC 유효성 판단**: 비밀 키 없이 같은 HMAC 값을 생성하는 건 불가능

**주의**:

* HMAC의 보안성은 해시 함수의 보안성과 비밀 키의 길이에 의존합니다.

  * 예: `HS256`을 사용할 경우 secret 키는 **32바이트 이상** 권장

---

## ❌ 잘못된 사용 예

* **비밀 키를 짧게** 주면 공격 가능성 증가 (`"mysecret"` 같은 키는 위험)
* **JWT에서 `alg: none` 사용**은 치명적 보안 위협

---

## ✅ 요약

| 항목       | 설명                        |   |              |   |       |
| -------- | ------------------------- | - | ------------ | - | ----- |
| HMAC     | 키 기반 해시 인증 방식             |   |              |   |       |
| 구조       | \`H((K ⊕ opad)            |   | H((K ⊕ ipad) |   | m))\` |
| SHA 종류   | SHA256, SHA384, SHA512    |   |              |   |       |
| JWT에서 사용 | `alg: HS256`, `HS512` 등   |   |              |   |       |
| 장점       | 빠르고 간단, 고속 인증             |   |              |   |       |
| 단점       | 대칭키 방식이라 키 유출 시 전체 시스템 위협 |   |              |   |       |
