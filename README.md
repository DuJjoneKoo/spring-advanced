# 🌱 Spring Advanced

## 📋 프로젝트 정보

| 항목 | 내용 |
|------|------|
| **프로젝트명** | Spring Advanced |
| **언어** | Java 17 |
| **프레임워크** | Spring Boot 3.3.3 |
| **빌드 도구** | Gradle 9.0 |
| **데이터베이스** | MySQL |
| **ORM** | JPA / Hibernate |

---

## 🛠 기술 스택

- **Java 17**
- **Spring Boot 3.3.3**
- **Spring MVC**
- **Spring Data JPA**
- **MySQL**
- **JWT (JSON Web Token)**
- **Lombok**
- **Spring Validation**

---

## ✅ 구현 목록

### Lv 0. 프로젝트 세팅 - 에러 분석

- `src/main/resources` 폴더 생성
- `application.properties` 파일 생성 및 설정
  - JWT 시크릿 키 설정 (Base64 인코딩, 32바이트 이상)
  - MySQL 데이터베이스 연결 설정
  - JPA Dialect 설정

```properties
jwt.secret.key=BASE64_ENCODED_SECRET_KEY
spring.datasource.url=jdbc:mysql://localhost:3306/advanced
spring.datasource.username=root
spring.datasource.password=비밀번호
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.jpa.database-platform=org.hibernate.dialect.MySQLDialect
```

---

### Lv 1. ArgumentResolver

- `AuthUserArgumentResolver`에 `@Component` 어노테이션 추가
- `WebConfig` 클래스 생성 (`WebMvcConfigurer` 구현)
  - `addArgumentResolvers()`에 `AuthUserArgumentResolver` 등록

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final AuthUserArgumentResolver authUserArgumentResolver;
    private final AdminInterceptor adminInterceptor;

    public WebConfig(AuthUserArgumentResolver authUserArgumentResolver, AdminInterceptor adminInterceptor) {
        this.authUserArgumentResolver = authUserArgumentResolver;
        this.adminInterceptor = adminInterceptor;
    }

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(authUserArgumentResolver);
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(adminInterceptor)
                .addPathPatterns("/admin/**");
    }
}
```

---

### Lv 2. 코드 개선

#### 1. Early Return
- `AuthService.signup()` 메서드에서 이메일 중복 체크를 비밀번호 인코딩보다 먼저 실행하도록 순서 변경
- 이메일이 중복일 때 불필요한 `passwordEncoder.encode()` 실행 방지

#### 2. 불필요한 if-else 제거
- `WeatherClient.getTodayWeather()` 메서드에서 불필요한 `else` 블록 제거
- `throw` 는 `return` 처럼 즉시 메서드를 종료하므로 `else` 불필요

#### 3. Validation
- `UserChangePasswordRequest` DTO에 유효성 검증 어노테이션 추가
  - `@Size(min = 8)` - 8자 이상
  - `@Pattern(regexp = "^(?=.*[A-Z])(?=.*\\d).+$")` - 숫자 및 대문자 포함
- `UserController.changePassword()`에 `@Valid` 어노테이션 추가

---

### Lv 3. N+1 문제 해결

- `TodoRepository`의 `findAllByOrderByModifiedAtDesc()` 메서드를
  `@Query + LEFT JOIN FETCH` 방식에서 `@EntityGraph` 방식으로 변경

```java
// Before
@Query("SELECT t FROM Todo t LEFT JOIN FETCH t.user u ORDER BY t.modifiedAt DESC")
Page<Todo> findAllByOrderByModifiedAtDesc(Pageable pageable);

// After
@EntityGraph(attributePaths = {"user"})
Page<Todo> findAllByOrderByModifiedAtDesc(Pageable pageable);
```

---

### Lv 4. 테스트 코드 수정

#### 1. PasswordEncoderTest
- `matches()` 메서드 파라미터 순서 수정
  - Before: `matches(encodedPassword, rawPassword)`
  - After: `matches(rawPassword, encodedPassword)`

#### 2. ManagerServiceTest - 케이스 1
- 테스트 메서드명 수정
  - Before: `manager_목록_조회_시_Todo가_없다면_NPE_에러를_던진다()`
  - After: `manager_목록_조회_시_Todo가_없다면_InvalidRequestException_에러를_던진다()`
- 에러 메시지 수정: `"Manager not found"` → `"Todo not found"`

#### 3. CommentServiceTest
- 예외 클래스 수정
  - Before: `ServerException`
  - After: `InvalidRequestException`

#### 4. ManagerServiceTest - 케이스 3
- `ManagerService.saveManager()`에 `todo.getUser() == null` 체크 추가

```java
// Before
if (!ObjectUtils.nullSafeEquals(user.getId(), todo.getUser().getId())) {

// After
if (todo.getUser() == null || !ObjectUtils.nullSafeEquals(user.getId(), todo.getUser().getId())) {
```

---

### Lv 5. API 로깅 - Interceptor

- `AdminInterceptor` 클래스 생성 (`HandlerInterceptor` 구현)
  - `preHandle()`에서 어드민 권한 확인
  - 어드민이 아닌 경우 `AuthException` 발생
  - 어드민인 경우 요청 시각과 URL 로깅

```java
@Slf4j
@Component
public class AdminInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String userRole = (String) request.getAttribute("userRole");

        if (!"ADMIN".equals(userRole)) {
            throw new AuthException("어드민만 접근할 수 있습니다.");
        }

        log.info("어드민 API 요청 - 시각: {}, URL: {}", LocalDateTime.now(), request.getRequestURI());

        return true;
    }
}
```

- `WebConfig`에 `AdminInterceptor` 등록
  - `/admin/**` 경로에만 적용

---

## 📁 프로젝트 구조

```
src
├── main
│   ├── java
│   │   └── org.example.expert
│   │       ├── client
│   │       │   └── WeatherClient.java
│   │       ├── config
│   │       │   ├── AdminInterceptor.java  ← 새로 추가
│   │       │   ├── AuthUserArgumentResolver.java
│   │       │   ├── FilterConfig.java
│   │       │   ├── GlobalExceptionHandler.java
│   │       │   ├── JwtFilter.java
│   │       │   ├── JwtUtil.java
│   │       │   ├── PasswordEncoder.java
│   │       │   ├── PersistenceConfig.java
│   │       │   └── WebConfig.java  ← 새로 추가
│   │       └── domain
│   │           ├── auth
│   │           ├── comment
│   │           ├── manager
│   │           ├── todo
│   │           └── user
│   └── resources
│       └── application.properties  ← 새로 추가
└── test
```

---

## ⚠️ 주의사항

> `application.properties` 파일은 민감한 정보(DB 비밀번호, JWT 시크릿 키)를 포함하므로
> `.gitignore`에 `*.properties`를 추가하여 GitHub에 업로드되지 않도록 설정했습니다.
