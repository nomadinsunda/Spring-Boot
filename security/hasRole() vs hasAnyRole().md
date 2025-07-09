## ✅ `hasRole()` vs `hasAnyRole()` 비교표

| 항목                  | `hasRole("X")`                          | `hasAnyRole("X", "Y", ...)`                                               |
| ------------------- | --------------------------------------- | ------------------------------------------------------------------------- |
| **의미**              | 지정된 단일 역할만 있는지 검사                       | 여러 역할 중 **하나라도** 있는지 검사                                                   |
| **접근 허용 조건**        | `"ROLE_X"` 권한이 **정확히 있어야 함**            | `"ROLE_X"`, `"ROLE_Y"` 등 중 하나라도 있으면 허용                                    |
| **권한 prefix 자동 추가** | `"ROLE_"` 자동으로 붙여짐                      | `"ROLE_"` 자동으로 붙여짐                                                        |
| **복수 검사 여부**        | ❌ 단일 역할만 검사                             | ✅ 여러 역할을 OR 조건으로 검사                                                       |
| **예시**              | `.hasRole("ADMIN")` → `"ROLE_ADMIN"` 검사 | `.hasAnyRole("ADMIN", "MANAGER")` → `"ROLE_ADMIN"` 또는 `"ROLE_MANAGER"` 검사 |

---

## 🔍 예제

### 사용자 권한:

```java
[ "ROLE_USER", "ROLE_MANAGER" ]
```

### 코드 결과:

| 코드                                | 결과   | 설명                  |
| --------------------------------- | ---- | ------------------- |
| `.hasRole("USER")`                | ✅ 허용 | `"ROLE_USER"` 있음    |
| `.hasRole("ADMIN")`               | ❌ 거부 | `"ROLE_ADMIN"` 없음   |
| `.hasAnyRole("ADMIN", "MANAGER")` | ✅ 허용 | `"ROLE_MANAGER"` 있음 |
| `.hasAnyRole("ADMIN", "DEV")`     | ❌ 거부 | 둘 다 없음              |

---

## ⚠️ 주의사항

* \*\*`"ROLE_"는 직접 쓰지 마세요!**  
  `hasRole("ADMIN")`→`"ROLE\_ADMIN"`을 내부에서 자동으로 검사합니다.  
  따라서 `hasRole("ROLE\_ADMIN")`처럼 쓰면 `"ROLE\_ROLE\_ADMIN"\`이 되어 실패합니다 ❌

