# SPRING_CONVENTION.md — Spring Boot 코딩 규칙

> 이 파일은 모든 Spring Boot 프로젝트에 반드시 준수해야 하는 코딩 규칙이다.
> 규칙을 위반하는 코드는 생성하지 않는다. 애매한 경우에는 가장 보수적인 방향으로 설계한다.
> 기술 스택, 도메인 설명 등 프로젝트별 정보는 별도 PROJECT.md를 참고하라.

---

## 1. 디렉토리 구조

패키지는 반드시 **도메인형**으로 구성한다. 레이어 기반 최상위 분리(`controller/`, `service/` 등)는 금지한다.
공통 로직은 반드시 `global` 패키지에 위치한다.

```
src/main/java/com/example/
├── global/
│   ├── entity/
│   │   └── BaseEntity.java
│   ├── exception/
│   │   ├── GlobalExceptionHandler.java   # @RestControllerAdvice
│   │   ├── ErrorCode.java                # 예외 코드 인터페이스
│   │   └── GlobalException.java          # 최상위 커스텀 예외
│   ├── response/
│   │   └── ApiResponse.java              # 공통 API 응답 포맷
│   └── client/
│       └── {xxx}/                        # 여러 도메인에서 공유하는 외부 API (예: coolsms)
│           ├── XxxClient.java
│           ├── XxxProperties.java        # @ConfigurationProperties 설정 클래스
│           └── exception/
├── domain/
│   ├── user/
│   │   ├── controller/
│   │   │   ├── request/    # Controller 요청 DTO (XxxRequest)
│   │   │   └── response/   # Controller 응답 DTO (XxxResponse)
│   │   ├── service/
│   │   │   ├── request/    # Service 요청 DTO (XxxServiceRequest)
│   │   │   └── response/   # Service 응답 DTO (XxxServiceResponse)
│   │   ├── client/
│   │   │   └── {xxx}/      # 해당 도메인 전용 외부 API (예: kakao)
│   │   │       ├── XxxClient.java
│   │   │       ├── XxxProperties.java
│   │   │       └── exception/
│   │   ├── repository/
│   │   ├── entity/
│   │   └── exception/      # UserException, UserErrorCode
│   └── order/
│       ├── controller/
│       │   ├── request/
│       │   └── response/
│       ├── service/
│       │   ├── request/
│       │   └── response/
│       ├── repository/
│       ├── entity/
│       └── exception/
```

---

## 2. 레이어별 역할 및 규칙

### Controller

- Controller는 HTTP 요청/응답 처리만 담당한다. 비즈니스 로직 작성은 금지한다.
- 반드시 `@Valid`를 사용해 요청 DTO 유효성 검사를 Controller 레이어에서 처리한다.
- `XxxRequest`를 `XxxServiceRequest`로 변환한 뒤 Service를 호출한다. Controller DTO를 Service에 직접 넘기는 것은 금지한다.
- 반드시 아래 형태로 반환한다. Service는 절대 `ResponseEntity`를 반환하지 않는다.

```java
@PostMapping
public ResponseEntity<ApiResponse<UserServiceResponse>> createUser(
    @RequestBody @Valid UserCreateRequest request
) {
    UserServiceRequest serviceRequest = request.toServiceRequest();
    UserServiceResponse serviceResponse = userService.createUser(serviceRequest);
    return ResponseEntity.ok(ApiResponse.success(serviceResponse));
}
```

### Service

- 클래스 레벨에 반드시 `@Transactional(readOnly = true)`를 선언한다.
- 데이터를 변경하는 메서드에만 `@Transactional`을 개별 선언한다.
- Entity를 직접 반환하는 것은 금지한다. 반드시 Service Response DTO로 변환해서 반환한다.
- Controller DTO(`XxxRequest`)를 파라미터로 받는 것은 금지한다. 반드시 Service DTO(`XxxServiceRequest`)를 사용한다.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserService {

    private final UserRepository userRepository;

    @Transactional
    public UserServiceResponse createUser(UserServiceRequest request) { ... }

    public UserServiceResponse findUser(Long id) { ... }
}
```

### Repository

- 반드시 `JpaRepository`를 상속한다.
- 단순 조회는 Spring Data JPA 메서드 네이밍으로 처리한다.
- 복잡한 동적 쿼리가 필요한 경우에만 QueryDSL을 도입한다.

---

## 3. 외부 API 연동 규칙

외부 API 호출 로직은 반드시 `XxxClient` 클래스로 캡슐화한다.
Service에서 외부 API를 직접 호출하는 것은 금지한다.

### 패키지 위치 규칙

외부 API Client의 위치는 사용 범위에 따라 결정한다.

| 사용 범위 | 위치 | 예시 |
|---|---|---|
| 여러 도메인에서 공유 | `global/client/{xxx}/` | CoolSMS 문자 발송 |
| 특정 도메인 전용 | `domain/{domain}/client/{xxx}/` | Kakao 로그인 (user 도메인만 사용) |

각 Client 패키지 구조는 동일하다.

```
{위치}/{xxx}/
├── XxxClient.java        # 외부 API 호출 캡슐화
├── XxxProperties.java    # @ConfigurationProperties 설정 클래스
└── exception/
    ├── XxxException.java
    └── XxxErrorCode.java
```

### XxxClient 작성 규칙

- 클래스명은 반드시 `XxxClient`로 명명한다. (예: `KakaoClient`, `CoolSmsClient`)
- 외부 API 설정값은 반드시 `@ConfigurationProperties` 설정 클래스로 분리한다. `@Value` 직접 사용은 금지한다.
- 외부 API 호출 중 발생하는 예외는 반드시 `exception/` 하위의 전용 예외로 변환해서 던진다.
- 외부 API의 에러 응답 코드, 네트워크 오류, 타임아웃 등 각 케이스를 구체적으로 구분해서 처리한다.
- `XxxClient`는 오직 외부 API 호출과 예외 변환만 담당한다. 비즈니스 로직을 작성하지 않는다.

```java
// domain/user/client/kakao/KakaoProperties.java
@ConfigurationProperties(prefix = "kakao")
public record KakaoProperties(
    String apiKey,
    String apiUrl
) {}
```

```java
// domain/user/client/kakao/KakaoClient.java
@Component
@RequiredArgsConstructor
public class KakaoClient {

    private final RestClient restClient;
    private final KakaoProperties kakaoProperties; // @Value 대신 Properties 클래스 주입

    public KakaoUserInfo fetchUserInfo(String accessToken) {
        try {
            return restClient.get()
                .uri(kakaoProperties.apiUrl() + "/v2/user/me")
                .header("Authorization", "Bearer " + accessToken)
                .retrieve()
                .body(KakaoUserInfo.class);
        } catch (HttpClientErrorException e) {
            if (e.getStatusCode() == HttpStatus.UNAUTHORIZED) {
                throw new KakaoException(KakaoErrorCode.INVALID_TOKEN);
            }
            throw new KakaoException(KakaoErrorCode.API_CALL_FAILED);
        } catch (ResourceAccessException e) {
            // 네트워크 오류, 타임아웃
            throw new KakaoException(KakaoErrorCode.CONNECTION_TIMEOUT);
        }
    }
}
```

```java
// domain/user/client/kakao/exception/KakaoErrorCode.java
@Getter
@RequiredArgsConstructor
public enum KakaoErrorCode implements ErrorCode {
    INVALID_TOKEN("유효하지 않은 카카오 토큰입니다.", HttpStatus.UNAUTHORIZED),
    API_CALL_FAILED("카카오 API 호출에 실패했습니다.", HttpStatus.BAD_GATEWAY),
    CONNECTION_TIMEOUT("카카오 서버 연결에 실패했습니다.", HttpStatus.GATEWAY_TIMEOUT);

    private final String message;
    private final HttpStatusCode statusCode;
}

// domain/user/client/kakao/exception/KakaoException.java
public class KakaoException extends GlobalException {
    public KakaoException(KakaoErrorCode errorCode) {
        super(errorCode);
    }
}
```

### Service에서 사용 방법

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserService {

    private final UserRepository userRepository;
    private final KakaoClient kakaoClient;  // Client 주입

    @Transactional
    public UserServiceResponse loginWithKakao(String accessToken) {
        KakaoUserInfo kakaoUserInfo = kakaoClient.fetchUserInfo(accessToken); // 외부 API 호출은 Client에 위임
        // 이후 비즈니스 로직 처리
        ...
    }
}
```

---

## 4. Entity 설계 규칙

### BaseEntity

모든 Entity는 반드시 `global/entity/BaseEntity.java`를 상속한다.
`createdAt`, `updatedAt`은 BaseEntity에서만 관리하며 각 Entity에 중복 선언하는 것은 금지한다.

```java
// global/entity/BaseEntity.java
@MappedSuperclass
@Getter
@EntityListeners(AuditingEntityListener.class)
public class BaseEntity {

    @CreatedDate
    @Column(updatable = false, nullable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private LocalDateTime updatedAt;
}
```

`@EnableJpaAuditing`을 메인 클래스 또는 별도 설정 클래스에 반드시 선언한다.

### Entity 기본 구조

```java
@Entity
@Getter
@Table(name = "missions")                          // 테이블명은 복수형 snake_case
@NoArgsConstructor(access = AccessLevel.PROTECTED) // 기본 생성자는 반드시 PROTECTED
public class Mission extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "mission_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)             // 연관관계는 반드시 LAZY
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Enumerated(EnumType.STRING)                   // Enum은 반드시 STRING
    @Column(nullable = false)
    private MissionStatus missionStatus;

    @Builder                                       // @Builder는 반드시 생성자 레벨에 선언
    private Mission(User user, MissionStatus missionStatus) {
        this.user = user;
        this.missionStatus = missionStatus;
    }

    // 상태 변경은 반드시 의도가 명확한 메서드로 작성
    public void assignRobot(Robot robot) {
        this.robot = robot;
        this.missionStatus = MissionStatus.ASSIGNED;
    }

    public void arrive() {
        this.missionStatus = MissionStatus.ARRIVED;
    }
}
```

### Entity 규칙 요약

- `@NoArgsConstructor(access = AccessLevel.PROTECTED)` — 외부에서 기본 생성자로 직접 생성 금지
- `@Builder`는 클래스가 아닌 **생성자 레벨**에 선언하고, 생성자 접근제어자는 반드시 `private`
- 연관관계 fetch 전략은 반드시 **`FetchType.LAZY`** (`EAGER` 금지)
- `@Enumerated`는 반드시 **`EnumType.STRING`** (`ORDINAL` 금지)
- 연관관계 편의 메서드는 **연관관계 주인 쪽 Entity**에 작성
- 필드 변경은 `@Setter` 대신 반드시 **의도가 드러나는 메서드명**으로 작성 (예: `assignRobot()`, `arrive()`)

---

## 5. DTO 변환 규칙

변환 책임을 레이어별로 반드시 아래 규칙에 따라 분리한다.

| 변환 방향 | 책임 위치 | 메서드 |
|---|---|---|
| `XxxRequest` → `XxxServiceRequest` | `XxxRequest` 내부 | `toServiceRequest()` |
| `Entity` → `XxxServiceResponse` | `XxxServiceResponse` 내부 | `from(Entity)` |

### XxxRequest → XxxServiceRequest

변환 메서드는 반드시 `XxxRequest`에 선언하고, Controller에서 호출한다.
Controller에서 직접 `new XxxServiceRequest(...)` 생성하는 것은 금지한다.

```java
// controller/request/UserCreateRequest.java
@Getter
@NoArgsConstructor
public class UserCreateRequest {

    @NotBlank
    private String email;

    @NotBlank
    private String name;

    public UserServiceRequest toServiceRequest() {
        return UserServiceRequest.builder()
            .email(this.email)
            .name(this.name)
            .build();
    }
}

// Controller 호출부
UserServiceRequest serviceRequest = request.toServiceRequest();
UserServiceResponse serviceResponse = userService.createUser(serviceRequest);
```

### Entity → XxxServiceResponse

변환 메서드는 반드시 `XxxServiceResponse`에 정적 팩토리 메서드 `from()`으로 선언하고, Service에서 호출한다.
Service에서 직접 `new XxxServiceResponse(entity.getXxx(), ...)` 형태로 생성하는 것은 금지한다.

```java
// service/response/UserServiceResponse.java
@Getter
@Builder
public class UserServiceResponse {

    private Long id;
    private String email;
    private String name;

    public static UserServiceResponse from(User user) {
        return UserServiceResponse.builder()
            .id(user.getId())
            .email(user.getEmail())
            .name(user.getName())
            .build();
    }
}

// Service 호출부
User user = userRepository.findById(id)
    .orElseThrow(() -> new UserException(UserErrorCode.USER_NOT_FOUND));
return UserServiceResponse.from(user);
```

---

## 6. 예외 처리 전략

### 예외 클래스 구조

```
global/exception/
├── GlobalExceptionHandler.java   # @RestControllerAdvice
├── ErrorCode.java                # 예외 코드 인터페이스
└── GlobalException.java          # 최상위 커스텀 예외

domain/user/exception/
├── UserException.java            # GlobalException 상속
└── UserErrorCode.java            # ErrorCode 구현
```

**ErrorCode 인터페이스**

```java
public interface ErrorCode {
    String getMessage();
    HttpStatusCode getStatusCode();
}
```

**GlobalException**

```java
@Getter
public class GlobalException extends RuntimeException {

    private final ErrorCode errorCode;

    public GlobalException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }
}
```

**도메인별 예외 (예시: User)**

```java
// UserErrorCode.java
@Getter
@RequiredArgsConstructor
public enum UserErrorCode implements ErrorCode {
    USER_NOT_FOUND("존재하지 않는 사용자입니다.", HttpStatus.NOT_FOUND),
    DUPLICATE_EMAIL("이미 사용 중인 이메일입니다.", HttpStatus.CONFLICT);

    private final String message;
    private final HttpStatusCode statusCode;
}

// UserException.java
public class UserException extends GlobalException {
    public UserException(UserErrorCode errorCode) {
        super(errorCode);
    }
}
```

**사용 방법**

모든 예외는 반드시 아래 형태로 던진다.

```java
userRepository.findById(id)
    .orElseThrow(() -> new UserException(UserErrorCode.USER_NOT_FOUND));
```

### 공통 API 응답 포맷

모든 API 응답은 반드시 아래 3필드 포맷을 따른다.
HTTP 상태코드는 `ResponseEntity`가 담당하므로 `ApiResponse`에는 포함하지 않는다.
에러 응답은 별도의 `ErrorResponse`를 사용하지 않는다. 성공/에러 모든 응답은 `ApiResponse`로 통일한다.

```json
{
  "code": "SUCCESS",
  "message": "요청이 성공했습니다.",
  "data": { ... }
}
```

```java
@Getter
@RequiredArgsConstructor
public class ApiResponse<T> {
    private final String code;    // 비즈니스 결과 코드 (예: "SUCCESS", "USER_NOT_FOUND") — ErrorCode enum name 또는 "SUCCESS" 고정값
    private final String message; // 사람이 읽을 수 있는 결과 설명 (예: "요청이 성공했습니다.", "존재하지 않는 사용자입니다.")
    private final T data;         // 실제 응답 바디. 반환할 데이터가 없으면 null

    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>("SUCCESS", "요청이 성공했습니다.", data);
    }

    public static <T> ApiResponse<T> success(String message, T data) {
        return new ApiResponse<>("SUCCESS", message, data);
    }
}
```

### GlobalExceptionHandler

`global/exception/GlobalExceptionHandler.java`에 위치한다.
`ApiResponse`는 `global/response/ApiResponse.java`에 위치한다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(GlobalException.class)
    public ResponseEntity<ApiResponse<Void>> handleGlobalException(GlobalException e) {
        ErrorCode errorCode = e.getErrorCode();
        return ResponseEntity
            .status(errorCode.getStatusCode())
            .body(new ApiResponse<>(
                ((Enum<?>) errorCode).name(), // "UserErrorCode"가 아닌 "USER_NOT_FOUND" 형태로 반환
                errorCode.getMessage(),
                null
            ));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidationException(
        MethodArgumentNotValidException e
    ) {
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(FieldError::getDefaultMessage)
            .findFirst()
            .orElse("입력값이 올바르지 않습니다.");
        return ResponseEntity.badRequest()
            .body(new ApiResponse<>("VALIDATION_ERROR", message, null));
    }
}
```

---

## 7. 테스트 전략

### 테스트 지원 클래스 구조

모든 테스트는 반드시 아래 두 지원 클래스 중 하나를 상속받아 작성한다.

```
src/test/java/com/example/
├── IntegrationTestSupport.java    # Service, Repository 테스트 베이스
└── RestControllerTestSupport.java # Controller 테스트 베이스
```

| 테스트 대상 | 상속 클래스 |
|---|---|
| Service, Repository | `IntegrationTestSupport` |
| Controller | `RestControllerTestSupport` |

---

### IntegrationTestSupport

Service, Repository 테스트의 베이스 클래스다.
Testcontainers 설정을 담당하며, 외부 의존성(DB 등)을 Docker 컨테이너로 띄운다.
컨테이너는 반드시 `static` 영역에 싱글톤으로 선언해 전체 테스트 suite에서 컨테이너를 한 번만 실행한다.

```java
@SpringBootTest
@Transactional
public abstract class IntegrationTestSupport {

    static final MySQLContainer<?> MY_SQL_CONTAINER;

    static {
        MY_SQL_CONTAINER = new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
        MY_SQL_CONTAINER.start();
    }

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", MY_SQL_CONTAINER::getJdbcUrl);
        registry.add("spring.datasource.username", MY_SQL_CONTAINER::getUsername);
        registry.add("spring.datasource.password", MY_SQL_CONTAINER::getPassword);
    }
}
```

Service, Repository 테스트는 반드시 `IntegrationTestSupport`를 상속받아 작성한다.

```java
class UserServiceTest extends IntegrationTestSupport {

    @Autowired
    private UserService userService;

    @Test
    void 유저_생성_성공() {
        UserServiceRequest request = new UserServiceRequest("test@email.com", "홍길동");
        UserServiceResponse response = userService.createUser(request);

        assertThat(response.getEmail()).isEqualTo("test@email.com");
    }
}
```

```java
class UserRepositoryTest extends IntegrationTestSupport {

    @Autowired
    private UserRepository userRepository;

    @Test
    void 이메일로_유저_조회_성공() {
        User user = User.builder().email("test@email.com").build();
        userRepository.save(user);

        Optional<User> found = userRepository.findByEmail("test@email.com");

        assertThat(found).isPresent();
    }
}
```

---

### RestControllerTestSupport

Controller 테스트의 베이스 클래스다.
`@WebMvcTest` 설정과 Controller 레이어에 필요한 모든 의존성(`MockMvc`, `MockitoBean` 등)을 한 곳에서 관리한다.
Controller 테스트는 반드시 `RestControllerTestSupport`를 상속받아 작성한다.

```java
@WebMvcTest(controllers = {
    UserController.class,
    OrderController.class
    // 새 Controller 추가 시 여기에 등록
})
public abstract class RestControllerTestSupport {

    @Autowired
    protected MockMvc mockMvc;

    @Autowired
    protected ObjectMapper objectMapper;

    @MockitoBean
    protected UserService userService;

    @MockitoBean
    protected OrderService orderService;
    // 새 Service 추가 시 여기에 등록
}
```

```java
class UserControllerTest extends RestControllerTestSupport {

    @Test
    void 유저_조회_성공() throws Exception {
        given(userService.findUser(1L)).willReturn(new UserServiceResponse(...));

        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.id").value(1L));
    }
}
```

---

## 8. 코딩 컨벤션

### 메서드 네이밍 컨벤션

Service 메서드명은 반드시 아래 접두사 규칙을 따른다.

| 동작 | 접두사 | 예시 |
|---|---|---|
| 조회 | `find` | `findUser()`, `findOrders()` |
| 생성 | `create` | `createUser()`, `createOrder()` |
| 수정 | `update` | `updateUser()`, `updateOrderStatus()` |
| 삭제 | `delete` | `deleteUser()`, `deleteOrder()` |

Repository 메서드명은 Spring Data JPA 네이밍 규칙을 따른다.

```java
// ✅ 올바른 예시
Optional<User> findByEmail(String email);
List<User> findAllByStatus(UserStatus status);
boolean existsByEmail(String email);

// ❌ 금지 — 접두사 혼용
User getUser(Long id);       // get 대신 find
User searchUser(Long id);    // search 대신 find
User fetchUser(Long id);     // fetch 대신 find
```

### 의존성 주입

- **프로덕션 코드**: 반드시 `@RequiredArgsConstructor` + `final` 필드로 생성자 주입만 사용한다. `@Autowired` 필드 주입은 금지한다.
- **테스트 코드**: `@Autowired` 필드 주입을 허용한다.

### Entity 수정

Entity에 `@Setter`를 선언하는 것은 금지한다.
필드 변경이 필요한 경우 반드시 Entity 내부에 의도가 드러나는 메서드를 작성한다.

```java
// ✅ 올바른 방법
public void updateEmail(String email) {
    this.email = email;
}

// ❌ 금지
@Setter
private String email;
```

### Optional 처리

`Optional.get()`을 직접 호출하는 것은 금지한다.
반드시 `orElseThrow()`를 사용해 예외로 처리한다.

```java
// ✅ 올바른 방법
User user = userRepository.findById(id)
    .orElseThrow(() -> new UserException(UserErrorCode.USER_NOT_FOUND));

// ❌ 금지
User user = userRepository.findById(id).get();
```

---

## 9. 금지 사항

아래 항목은 절대 위반하지 않는다.

| 금지 항목 | 대안 |
|---|---|
| `@Autowired` 프로덕션 코드 필드 주입 | `@RequiredArgsConstructor` + `final` |
| Entity `@Setter` 선언 | Entity 내부 의도가 드러나는 메서드 |
| `Optional.get()` 직접 호출 | `orElseThrow()` |
| Service에서 Entity 직접 반환 | `XxxServiceResponse.from(entity)` |
| Service에서 `ResponseEntity` 반환 | Controller에서 감싸기 |
| Controller에 비즈니스 로직 작성 | Service 레이어로 이동 |
| Controller DTO를 Service에 직접 전달 | `request.toServiceRequest()` 변환 후 전달 |
| Service에서 직접 `new XxxServiceResponse(...)` 생성 | `XxxServiceResponse.from(entity)` |
| 레이어 기반 최상위 패키지 구조 | 도메인형 패키지 구조 |
| Service 메서드에 `get`, `search`, `fetch` 접두사 | `find`, `create`, `update`, `delete` 접두사 |
| 연관관계 `FetchType.EAGER` | `FetchType.LAZY` |
| `@Enumerated(EnumType.ORDINAL)` | `@Enumerated(EnumType.STRING)` |
| Service에서 외부 API 직접 호출 | `XxxClient`로 캡슐화 |
| 외부 API 예외를 그대로 전파 | `XxxException`으로 변환 후 던지기 |
| Client에 `@Value`로 설정값 직접 주입 | `@ConfigurationProperties` 설정 클래스로 분리 |

---

## 10. 코드 작성 전 최종 체크리스트

코드를 작성하거나 수정하기 전에 반드시 아래 항목을 확인한다.
하나라도 위반 시 코드를 다시 작성한다.

- [ ] 레이어 규칙을 위반하지 않는가? (Controller → Service → Repository 방향 준수)
- [ ] Entity를 외부로 노출하지 않았는가? (`XxxServiceResponse.from(entity)` 변환 여부)
- [ ] DTO 변환이 올바르게 이루어졌는가? (`toServiceRequest()`, `from()` 사용 여부)
- [ ] 금지된 패턴을 사용하지 않았는가? (8번 금지 사항 참고)
- [ ] Service 메서드 접두사 규칙을 따르는가? (`find`, `create`, `update`, `delete`)
- [ ] 새 도메인 예외를 추가했는가? (`XxxException`, `XxxErrorCode` 생성 여부)
- [ ] 외부 API 호출을 `XxxClient`로 캡슐화했는가?
- [ ] 외부 API 예외를 케이스별로 구체적으로 처리했는가?