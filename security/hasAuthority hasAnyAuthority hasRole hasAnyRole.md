좋습니다! 그럼 `hasAuthority`, `hasAnyAuthority`, `hasRole`, `hasAnyRole`의 차이점을 **정확하게 비교하는 표**로 정리해드리겠습니다.

---

## ✅ 권한 검사 메서드 비교표

| 메서드                         | 내부 비교 대상                          | 내부 권한 prefix      | 의미                                     | 예시 코드                                                                       |
| --------------------------- | --------------------------------- | ----------------- | -------------------------------------- | --------------------------------------------------------------------------- |
| `hasAuthority("X")`         | `GrantedAuthority.getAuthority()` | 없음                | 정확히 `"X"` 문자열과 일치해야 함                  | `.hasAuthority("READ_PRIVILEGE")`                                           |
| `hasAnyAuthority("X", "Y")` | `GrantedAuthority.getAuthority()` | 없음                | `"X"` 또는 `"Y"` 중 하나 이상 일치              | `.hasAnyAuthority("ROLE_USER", "ROLE_ADMIN")`                               |
| `hasRole("X")`              | `GrantedAuthority.getAuthority()` | 자동으로 `"ROLE_"` 붙임 | `"ROLE_X"`가 존재하는지 확인                   | `.hasRole("ADMIN")` → `"ROLE_ADMIN"` 검사                                     |
| `hasAnyRole("X", "Y")`      | `GrantedAuthority.getAuthority()` | 자동으로 `"ROLE_"` 붙임 | `"ROLE_X"` 또는 `"ROLE_Y"` 중 하나라도 있으면 허용 | `.hasAnyRole("USER", "MODERATOR")` → `"ROLE_USER"` or `"ROLE_MODERATOR"` 검사 |

---

## 🔍 예제 상황

### 1. 현재 사용자의 권한 목록

```java
[ "ROLE_USER", "READ_PRIVILEGE", "WRITE_PRIVILEGE" ]
```

### 2. 다음 코드들의 의미

| 코드 예시                                                | 결과   | 설명                     |
| ---------------------------------------------------- | ---- | ---------------------- |
| `.hasAuthority("READ_PRIVILEGE")`                    | ✅ 허용 | 정확히 일치                 |
| `.hasAuthority("ROLE_USER")`                         | ✅ 허용 | 정확히 일치                 |
| `.hasRole("USER")`                                   | ✅ 허용 | 내부적으로 `"ROLE_USER"` 검사 |
| `.hasRole("ADMIN")`                                  | ❌ 거부 | `"ROLE_ADMIN"` 없음      |
| `.hasAnyRole("ADMIN", "USER")`                       | ✅ 허용 | `"ROLE_USER"` 있음       |
| `.hasAnyAuthority("ROLE_MANAGER", "READ_PRIVILEGE")` | ✅ 허용 | `"READ_PRIVILEGE"` 있음  |



