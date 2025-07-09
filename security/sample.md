## ✅ Sample code 내

* 인증 방식: **HTTP Basic**
* 사용자 정보 저장: **MySQL 데이터베이스** 기반
* 사용자 조회: **Spring Data JPA**
* 인증 처리: **직접 구현한 `AuthenticationProvider`**
* 권한: `ROLE_ADMIN`, `ROLE_USER`, `READ`, `WRITE`, `DELETE`
* `/public/**`: 인증 없이 모두 허용
* `/admin/**`: `ROLE_ADMIN`만
* `/user/**`: `ROLE_USER` 이상
* `/secure/**`: `READ`, `WRITE`, `DELETE` 권한 기반

---

## 🧱 프로젝트 구조

```
com.example.securitymysql
├── SecurityMysqlApplication.java
├── config
│   └── SecurityConfig.java
│   └── CustomAuthenticationProvider.java
├── controller
│   └── TestController.java
├── entity
│   └── AppUser.java
│   └── Role.java
├── repository
│   └── UserRepository.java
├── service
│   └── JpaUserDetailsService.java
```

---

## 1️⃣ AppUser.java (엔티티)

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
    @CollectionTable(name = "user_roles")
    private Set<String> authorities = new HashSet<>();

    // getters, setters
}
```

---

## 2️⃣ UserRepository.java

```java
public interface UserRepository extends JpaRepository<AppUser, Long> {
    Optional<AppUser> findByUsername(String username);
}
```

---

## 3️⃣ JpaUserDetailsService.java

```java
@Service
public class JpaUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public JpaUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        AppUser user = userRepository.findByUsername(username)
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

## 4️⃣ CustomAuthenticationProvider.java

```java
@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {

    private final UserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;

    public CustomAuthenticationProvider(JpaUserDetailsService userDetailsService, PasswordEncoder passwordEncoder) {
        this.userDetailsService = userDetailsService;
        this.passwordEncoder = passwordEncoder;
    }

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = authentication.getName();
        String password = authentication.getCredentials().toString();

        UserDetails user = userDetailsService.loadUserByUsername(username);

        if (!passwordEncoder.matches(password, user.getPassword())) {
            throw new BadCredentialsException("Invalid credentials");
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

## 5️⃣ SecurityConfig.java

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    private final CustomAuthenticationProvider authProvider;

    public SecurityConfig(CustomAuthenticationProvider authProvider) {
        this.authProvider = authProvider;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authenticationProvider(authProvider)
            .csrf(AbstractHttpConfigurer::disable)
            .httpBasic(Customizer.withDefaults())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/user/**").hasAnyRole("ADMIN", "USER")
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

## 6️⃣ TestController.java

```java
@RestController
public class TestController {

    @GetMapping("/public/hello")
    public String publicHello() {
        return "Hello, world!";
    }

    @GetMapping("/admin/dashboard")
    public String adminOnly() {
        return "Admin dashboard";
    }

    @GetMapping("/user/profile")
    public String userProfile() {
        return "User profile";
    }

    @GetMapping("/secure/read")
    public String readResource() {
        return "Secured READ";
    }

    @GetMapping("/secure/write")
    public String writeResource() {
        return "Secured WRITE";
    }

    @GetMapping("/secure/delete")
    public String deleteResource() {
        return "Secured DELETE";
    }
}
```

---

## 7️⃣ application.yml (MySQL 연결 설정 예시)

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/security_db
    username: root
    password: your_password
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: none
    show-sql: true
    defer-datasource-initialization: true

  sql:
    init:
      mode: always

```
## schema.sql
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    enabled BOOLEAN NOT NULL
);

CREATE TABLE user_roles (
    app_user_id BIGINT NOT NULL,
    authorities VARCHAR(100) NOT NULL,
    FOREIGN KEY (app_user_id) REFERENCES users(id)
);
```

## data.sql
```sql
-- ADMIN 사용자: 비밀번호는 BCrypt로 암호화된 "admin123"
INSERT INTO users (id, username, password, enabled)
VALUES (1, 'admin', '$2a$10$5M9wH0DVLKx0PU8Z2.VURONFjcK7wdf9R7I0GhvUcxMIUClMfOc82', true);

-- 일반 사용자
INSERT INTO users (id, username, password, enabled)
VALUES (2, 'user', '$2a$10$7aOubTWUsR.QN/cJzWQeSevDbRH7mISvZ5fylbtZRVNQbK1PA2EeW', true);

-- writer 사용자
INSERT INTO users (id, username, password, enabled)
VALUES (3, 'writer', '$2a$10$2dfKzgkxQMQK0EYz6n0rHekCF6dEsPx6m5cgAWg9iVDpd0QY.L3FS', true);

-- 권한 매핑 (각 권한 문자열은 @ElementCollection 매핑용)
INSERT INTO user_roles (app_user_id, authorities) VALUES (1, 'ROLE_ADMIN');
INSERT INTO user_roles (app_user_id, authorities) VALUES (1, 'READ');
INSERT INTO user_roles (app_user_id, authorities) VALUES (1, 'WRITE');
INSERT INTO user_roles (app_user_id, authorities) VALUES (1, 'DELETE');

INSERT INTO user_roles (app_user_id, authorities) VALUES (2, 'ROLE_USER');
INSERT INTO user_roles (app_user_id, authorities) VALUES (2, 'READ');

INSERT INTO user_roles (app_user_id, authorities) VALUES (3, 'WRITE');
```

---

## 🧪 테스트 시나리오

* DB에 사용자 `admin` 추가 (`ROLE_ADMIN`, `READ`, `WRITE`, `DELETE`)
* 사용자 `user` 추가 (`ROLE_USER`, `READ`)
* 인증 없이 `/public/hello` 가능
* `GET /secure/read`는 `READ` 권한 필요
* `GET /admin/dashboard`는 `ROLE_ADMIN`만


