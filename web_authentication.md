다음은  **Form Login**, **Basic 인증**, **JWT 인증** 방식을 **항목별로 비교 정리**한 표입니다. 

---

### 🔐 인증 방식 비교

| 항목                               | **Form Login**                  | **Basic 인증**                     | **JWT 인증**              |
| -------------------------------- | ------------------------------- | -------------------------------- | ----------------------- |
| **세션 저장 위치**                     | `HttpSession`을 서블릿 컨텍스트에 저장     | `HttpSession` 생성 안 함             | `HttpSession` 생성 안 함    |
| **인증 방식 (Authorization Header)** | 사용 안 함 (쿠키 기반)                  | `ID:Password` → Base64 인코딩       | `Bearer Token` (JWT)    |
| **세션 유지 방식**                     | `HttpSession`에 상태 저장 (stateful) | 상태 저장 안 함 (`stateless`)          | 상태 저장 안 함 (`stateless`) |
| **세션 만료/로그아웃 시 처리**              | `HttpSession` 삭제                | 해당 없음                            | 해당 없음                   |
| **클라이언트와 세션 연동**                 | `Set-Cookie`로 `Session ID` 전달   | 매 요청마다 `Authorization Header` 포함 | 매 요청마다 `JWT` 포함         |
| **SecurityContext 저장 위치**        | `HttpSession`에 저장               | 스레드 `LocalStorage`에 저장           | 스레드 `LocalStorage`에 저장  |
| **주 사용 환경**                      | 전통적 웹 애플리케이션 (웹 페이지 중심)         | RESTful API 기반 애플리케이션            | RESTful API 기반 애플리케이션   |

---

### ✅ 요약 포인트

* **Form Login**: 서버에 세션을 저장하며 상태를 유지하는 전통적인 웹 인증 방식.
* **Basic 인증**: 매우 간단하지만 보안에 취약할 수 있어 HTTPS와 함께 사용 권장.
* **JWT 인증**: 토큰 기반 인증으로 클라이언트에 인증 상태를 저장하여 서버 확장성에 유리.

