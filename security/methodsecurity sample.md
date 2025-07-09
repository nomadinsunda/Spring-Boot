
## ✅ 예제 코드 내역

* 인증: **HTTP Basic**
* 사용자 관리: **MySQL + Spring Data JPA**
* 인증 제공자: **Custom `AuthenticationProvider`**
* 인가 처리:

  * `ROLE_ADMIN`, `ROLE_MANAGER`, `ROLE_USER`
  * 권한: `READ`, `WRITE`, `DELETE`
* URL 접근 제어:

  * `/public/**`: 모두 허용
  * `/admin/**`: `ROLE_ADMIN`만
  * `/management/**`: `ROLE_MANAGER` 이상
  * `/secure/read`: `READ` 권한 필요
  * `/secure/write`: `WRITE` 권한 필요
  * `/secure/delete`: `DELETE` 권한 필요
* `@PreAuthorize`와 `@Secured` 혼용
* 비밀번호 암호화: **BCrypt**
* `AuthenticationManager` 주입 방식: **Spring Boot 3.x 방식**

---

## 📁 패키지 구성

```
com.example.advancedsecurity
├── SecurityApplication.java
├── config
│   └── SecurityConfig.java
│   └── CustomAuthenticationProvider.java
├── controller
│   └── AccessController.java
├── entity
│   └── AppUser.java
├── repository
│   └── UserRepository.java
├── service
│   └── JpaUserDetailsService.java
```

---

## 🔒 1. SecurityConfig.java

```java
@Configuration
@EnableMethodSecurity(securedEnabled = true)
public class SecurityConfig {

    private final CustomAuthenticationProvider authProvider;

    public SecurityConfig(CustomAuthenticationProvider authProvider) {
        this.authProvider = authProvider;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .httpBasic(Customizer.withDefaults())
            .csrf(AbstractHttpConfigurer::disable)
            .authenticationProvider(authProvider)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/management/**").hasAnyRole("MANAGER", "ADMIN")
                .requestMatchers("/secure/read").hasAuthority("READ")
                .requestMatchers("/secure/write").hasAuthority("WRITE")
                .requestMatchers("/secure/delete").hasAuthority("DELETE")
                .anyRequest().denyAll()
            );

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

## 🔐 2. CustomAuthenticationProvider.java

```java
@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {

    private final JpaUserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;

    public CustomAuthenticationProvider(JpaUserDetailsService service, PasswordEncoder encoder) {
        this.userDetailsService = service;
        this.passwordEncoder = encoder;
    }

    @Override
    public Authentication authenticate(Authentication auth) throws AuthenticationException {
        String username = auth.getName();
        String password = auth.getCredentials().toString();

        UserDetails user = userDetailsService.loadUserByUsername(username);

        if (!passwordEncoder.matches(password, user.getPassword())) {
            throw new BadCredentialsException("Invalid password");
        }

        return new UsernamePasswordAuthenticationToken(user, password, user.getAuthorities());
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

---

## 👤 3. AppUser.java (JPA Entity)

```java
@Entity
@Table(name = "users")
public class AppUser {

    @Id @GeneratedValue
    private Long id;

    private String username;
    private String password;
    private boolean enabled;

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "user_authorities", joinColumns = @JoinColumn(name = "user_id"))
    private Set<String> authorities = new HashSet<>();

    // getters/setters
}
```

---

## 📦 4. UserRepository.java

```java
public interface UserRepository extends JpaRepository<AppUser, Long> {
    Optional<AppUser> findByUsername(String username);
}
```

---

## 👥 5. JpaUserDetailsService.java

```java
@Service
public class JpaUserDetailsService implements UserDetailsService {

    private final UserRepository repository;

    public JpaUserDetailsService(UserRepository repository) {
        this.repository = repository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        AppUser user = repository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return User.builder()
                .username(user.getUsername())
                .password(user.getPassword())
                .disabled(!user.isEnabled())
                .authorities(user.getAuthorities().toArray(new String[0]))
                .build();
    }
}
```

---

## 🌐 6. AccessController.java

```java
@RestController
public class AccessController {

    @GetMapping("/public/info")
    public String publicEndpoint() {
        return "Accessible by anyone";
    }

    @GetMapping("/admin/area")
    public String adminOnly() {
        return "Admin Area";
    }

    @GetMapping("/management/dashboard")
    @Secured("ROLE_MANAGER")
    public String managerOnly() {
        return "Manager Dashboard";
    }

    @GetMapping("/secure/read")
    @PreAuthorize("hasAuthority('READ')")
    public String readSecure() {
        return "Read Access";
    }

    @GetMapping("/secure/write")
    @PreAuthorize("hasAuthority('WRITE')")
    public String writeSecure() {
        return "Write Access";
    }

    @GetMapping("/secure/delete")
    @PreAuthorize("hasAuthority('DELETE')")
    public String deleteSecure() {
        return "Delete Access";
    }
}
```

---

## 🧪 application.yml

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/security_db
    username: root
    password: your_password
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    defer-datasource-initialization: true

  sql:
    init:
      mode: always
```

---

## ✅ 요약

| 구분       | 구성                                                           |
| -------- | ------------------------------------------------------------ |
| 인증 방식    | HTTP Basic (세션 X, 토큰 X)                                      |
| 인증 처리    | Custom `AuthenticationProvider`                              |
| 사용자 관리   | MySQL + Spring Data JPA                                      |
| 권한 모델    | `ROLE_xxx` + 세부 권한 (`READ`, `WRITE`, `DELETE`) 혼합            |
| 접근 제어 방식 | `SecurityFilterChain` + `@Secured` + `@PreAuthorize` 혼용      |
| 테스트      | `curl -u admin:1234 http://localhost:8080/admin/area` 등으로 가능 |

