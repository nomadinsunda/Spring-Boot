# ⚙️ @EnableAutoConfiguration 완전 해부 — `AutoConfigurationImportSelector`, `DeferredImportSelector`, `@Conditional`

---

## 🔰 개요: Spring Boot 자동 구성의 정체

Spring Boot는 의존성을 추가하는 것만으로도 Bean을 자동 등록해 줍니다. 예를 들어, `spring-boot-starter-web`을 의존성에 추가하면, 다음과 같은 컴포넌트들이 자동으로 구성됩니다:

* Embedded Tomcat
* DispatcherServlet
* WebMvcConfigurer
* Jackson (ObjectMapper)
* 오류 핸들러 (BasicErrorController)
* Spring MVC 설정

그 비결은 바로 다음 3가지 구성요소에 있습니다:

| 구성 요소                             | 역할                                 |
| --------------------------------- | ---------------------------------- |
| `@EnableAutoConfiguration`        | 자동 구성 클래스들을 Import 하도록 Spring에게 알림 |
| `AutoConfigurationImportSelector` | 어떤 구성 클래스들을 Import할지 결정            |
| `@Conditional` 계열 애너테이션           | 특정 조건이 충족될 때만 해당 설정 클래스를 적용함       |

---

## 1️⃣ @EnableAutoConfiguration

```java
@Target(...)
@Retention(...)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    ...
}
```

### 핵심: `@Import(AutoConfigurationImportSelector.class)`

* Spring은 `@Import`를 만나면 해당 클래스의 `selectImports()`를 호출해서 **추가로 등록할 Bean 설정 클래스**들을 가져옵니다.
* 이때 `AutoConfigurationImportSelector`가 등장합니다.

---

## 2️⃣ AutoConfigurationImportSelector

```java
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware, EnvironmentAware, ResourceLoaderAware, BeanFactoryAware {
    ...
}
```

### 🔍 핵심 메서드: `selectImports(AnnotationMetadata)`

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

### 내부 메서드 호출 흐름

1. **`getAutoConfigurationEntry()`**

   * `spring.factories`에서 `EnableAutoConfiguration` 항목을 읽어옴

```java
List<String> configurations = SpringFactoriesLoader.loadFactoryNames(EnableAutoConfiguration.class, classLoader);
```

2. **Filter 적용**

   * 중복 제거, 제외 필터(@EnableAutoConfiguration(exclude=...)), 조건부 설정 등 적용

3. 최종적으로 import할 구성 클래스들 반환

---

## 3️⃣ spring.factories 파일

위 과정에서 다음 파일이 참조됩니다:

📄 `META-INF/spring.factories`
예시:

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
...
```

이 파일은 모듈별로 정의되어 있으며, 각 자동 구성 클래스를 나열합니다. 이들은 모두 `@Configuration` 클래스입니다.

---

## 4️⃣ DeferredImportSelector란?

### 일반적인 ImportSelector vs DeferredImportSelector

| 인터페이스                    | 실행 시점                  | 사용 목적                            |
| ------------------------ | ---------------------- | -------------------------------- |
| `ImportSelector`         | @Configuration 파싱 시점   | 조기 구성 (early configuration)      |
| `DeferredImportSelector` | 모든 @Configuration 등록 후 | 자동 구성처럼 **우선순위가 낮은 등록** 필요할 때 사용 |

> 자동 구성은 사용자가 등록한 설정보다 **우선순위가 낮아야 하므로**, `DeferredImportSelector`를 사용합니다.

---

## 5️⃣ 자동 구성 클래스의 구조

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class })
@ConditionalOnWebApplication(type = SERVLET)
@Import(EnableWebMvcConfiguration.class)
public class WebMvcAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public RequestMappingHandlerMapping requestMappingHandlerMapping() {
        ...
    }

    @Bean
    @ConditionalOnClass(ObjectMapper.class)
    public MappingJackson2HttpMessageConverter converter() {
        ...
    }
}
```

### 특징:

* 조건부 애너테이션 `@ConditionalOnClass`, `@ConditionalOnMissingBean` 등으로 빈 등록 여부 결정
* 내장된 WebMvc 설정 빈을 필요 시 등록

---

## 6️⃣ @Conditional과 하위 애너테이션

Spring Boot는 `@Conditional`을 통해 구성 클래스를 **동적으로 적용하거나 무시**합니다.

### 대표적인 확장 애너테이션:

| 애너테이션                          | 조건 설명                                |
| ------------------------------ | ------------------------------------ |
| `@ConditionalOnClass`          | 특정 클래스가 classpath에 존재할 경우            |
| `@ConditionalOnMissingBean`    | Bean이 존재하지 않을 경우                     |
| `@ConditionalOnProperty`       | application.yml 속성 값이 존재하거나 특정 값일 경우 |
| `@ConditionalOnWebApplication` | 웹 애플리케이션일 경우                         |

### 예시: `@ConditionalOnClass`

```java
@ConditionalOnClass(name = "javax.servlet.Servlet")
```

> `javax.servlet.Servlet` 클래스가 classpath에 있을 경우 해당 설정이 유효하게 동작함.

---

## 7️⃣ 자동 구성 우선순위 조정

자동 구성 클래스에는 우선순위를 지정할 수 있습니다:

* `@AutoConfigureOrder`
* `@AutoConfigureBefore`, `@AutoConfigureAfter`

예:

```java
@AutoConfigureBefore(JacksonAutoConfiguration.class)
public class MyCustomJsonAutoConfiguration {
    ...
}
```

---

## 8️⃣ 요약 흐름도

```
@SpringBootApplication
    └── @EnableAutoConfiguration
            └── @Import(AutoConfigurationImportSelector)
                    └── implements DeferredImportSelector
                            └── selectImports()
                                ├─ spring.factories에서 자동 구성 클래스 로딩
                                ├─ @Conditional을 통해 필터링
                                └─ ApplicationContext에 설정 클래스 등록
```

---

## ✅ 마무리 요약

| 핵심 컴포넌트                           | 역할 요약                             |
| --------------------------------- | --------------------------------- |
| `@EnableAutoConfiguration`        | 자동 설정 클래스들을 import하도록 Spring에 지시  |
| `AutoConfigurationImportSelector` | spring.factories에서 설정 클래스 목록을 로딩  |
| `DeferredImportSelector`          | @Configuration 등록이 끝난 후 import 수행 |
| `@Conditional` 계열                 | 설정 클래스 및 Bean 등록을 조건에 따라 제어       |
| `spring.factories`                | 자동 구성 클래스들을 정의한 메타데이터 파일          |

