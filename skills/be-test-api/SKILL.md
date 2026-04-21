---
name: be-test-api
description: api 모듈의 Controller 단위 테스트(Mockito)를 평가하고 작성한다.
---

# be-test-api

api 모듈의 Controller 단위 테스트를 3단계로 수행한다.
직접 메서드 호출 방식 (MockMvc 사용하지 않음).

---

## 입력

```
대상 Controller 클래스 이름 (예: SetupApiController, CommentApiController)
```

입력이 없으면 "어떤 Controller를 테스트할까요?" 로 확인한다.

---

## Phase 0. 프로젝트 컨텍스트 해석

이 스킬은 `{packageRoot}`, `{modules.api}` 플레이스홀더를 사용한다.
실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md`의 절차로 해석한다:

1. 프로젝트 루트의 `dk-be-bundle.config.json` 확인
2. 없으면 `build.gradle.kts`의 `group = "..."` 에서 `{packageRoot}` 추출
3. `{modulesPath}/api` 존재 확인 → `{modules.api}` 확정
4. 탐지 실패 시 유저에게 질의

---

## Phase 1. assess-infra

| 항목 | 경로 | 확인 |
|------|------|------|
| build.gradle.kts | `{modules.api}/build.gradle.kts` | `spring-boot-starter-test` 있는지 |

있으면 PASS, 없으면 추가.

**불필요**: MockMvc, @WebMvcTest, application.yml — Spring 컨텍스트 없음.

---

## Phase 2. assess-scope

### 2-1. 대상 분석

1. Controller 소스를 Read로 읽는다
2. 각 엔드포인트 메서드를 추출:
   - HTTP 메서드 + 경로
   - 파라미터 (RequestParam, PathVariable, RequestBody, AuthenticationPrincipal)
   - 반환 타입 (`ResponseEntity<Xxx>`)
   - 의존 Usecase/Reader/Writer 호출

### 2-2. 의존성 추출 + specs lookup

1. Controller의 `final` 필드 전부 → `@Mock` 후보
2. 각 의존성의 API 시그니처를 확인:
   - `specs/usecase.md` — Usecase/Reader/Writer 메서드 시그니처
   - `specs/api.md` — 엔드포인트별 요청/응답 스펙
3. 실제 인터페이스 소스도 Read로 확인 (specs와 차이 있을 수 있음)

### 2-3. 테스트 케이스 도출

각 엔드포인트 메서드에 대해 **정상 경로 + 예외 경로**를 도출한다.

#### 정상 경로 TC

| HTTP 메서드 | 반환 | TC |
|------------|------|-----|
| GET (단건) | `ResponseEntity<XxxResponse>` | 200 OK + body 검증 |
| GET (목록) | `ResponseEntity<XxxListResponse>` | 200 OK + data/total 검증 |
| POST (생성) | `ResponseEntity<XxxResponse>` (201) | 201 CREATED + id 검증 |
| PUT (수정) | `ResponseEntity<XxxResponse>` (200) | 200 OK + id 검증 |
| DELETE | `ResponseEntity<Void>` (204) | 204 NO_CONTENT + verify |

#### 예외 경로 TC — **Controller 내부 분기 + Usecase 예외 전파**

| 소스 | 예외 타입 | TC |
|------|----------|-----|
| Controller 내부 파싱/변환 | `InvalidInputException` | 400 코드+메시지 검증 |
| Usecase → Optional.empty | `EntityNotFoundException` 하위 | 404 코드+메시지 검증 |
| Usecase → 소유권 불일치 | `ForbiddenException` | 403 코드+메시지 검증 |
| Usecase → 중복 | `ConflictException` | 409 코드+메시지 검증 |

#### 예외 검증 규칙 (핵심)

모든 예외 TC에서 `HttpException` 인터페이스의 값을 반드시 검증한다:

```java
assertThatThrownBy(() -> controller.xxx(...))
        .isInstanceOf(XxxException.class)
        .satisfies(ex -> {
            HttpException httpEx = (HttpException) ex;
            assertThat(httpEx.httpStatusCode()).isEqualTo(expectedCode);
            assertThat(httpEx.httpErrorMessage()).isEqualTo(expectedMessage);
        });
```

**GlobalExceptionHandler가 `httpStatusCode()`와 `httpErrorMessage()`를 그대로 사용하므로,
예외 객체에서 직접 검증하면 HTTP 응답 검증과 동일하다.**

#### 추가 TC 규칙

| 코드 패턴 | TC 추가 |
|----------|---------|
| Controller 내부 `if/else`, `?:` | 분기별 TC 각각 |
| Enum 파싱 (valueOf) | 유효/무효 입력 각각 |
| `@AuthenticationPrincipal` | sessionUser 기반 userId 전달 verify |
| void 반환 Usecase | `verify(usecase).execute(...)` |

### 2-4. 산출물

```
## 분석 결과: {ControllerName}
### Mock 의존성
| 의존성 | 역할 |
### 엔드포인트별 테스트 케이스
| # | 엔드포인트 | 테스트명 | 유형 | 검증 |
```

사용자 확인 후 Phase 3 진행.

---

## Phase 3. add-test

### 파일 위치

```
{modules.api}/src/test/java/{packagePath}/api/{domain}/
└── {Controller}Test.java
```

### 클래스 템플릿

```java
@ExtendWith(MockitoExtension.class)
class {Controller}Test {

    @Mock private XxxUsecase xxxUsecase;
    @Mock private XxxReader xxxReader;

    @InjectMocks private {Controller} controller;

    // fixture
    private final SessionUser sessionUser = new SessionUser(1L);
    private final Long userId = 1L;
}
```

### Request DTO 생성

Controller의 Request DTO가 기본 생성자 + setter 없이 `@Getter`만 있는 경우 Reflection 헬퍼 사용:

```java
private <T> T createRequest(Class<T> clazz, Map<String, Object> fields) {
    try {
        T instance = clazz.getDeclaredConstructor().newInstance();
        for (var entry : fields.entrySet()) {
            Field field = clazz.getDeclaredField(entry.getKey());
            field.setAccessible(true);
            field.set(instance, entry.getValue());
        }
        return instance;
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

생성자가 public이면 직접 호출. record이면 정규 생성자 사용.

### 작성 규칙

1. **네이밍**: `should_예상결과_when_조건`
2. **구조**: `// given` → `// when` → `// then`
3. **정상 검증**: `assertThat(response.getStatusCode()).isEqualTo(HttpStatus.XXX)`
4. **예외 검증**: `assertThatThrownBy` + `satisfies`로 httpStatusCode + httpErrorMessage 동시 검증
5. **부수효과**: `verify(usecase).execute(...)` / `verify(usecase, never()).execute(...)`
6. **Read 모델 mock**: `SetupRead` 등은 `mock()`으로 생성, `from()` 변환에 필요한 getter만 stub
7. **Spring 컨텍스트 금지**: `@ExtendWith(MockitoExtension.class)` 만 사용

### 예외 상태코드·메시지 매핑 참조

| 예외 클래스 | 기본 httpStatusCode | 기본 httpErrorMessage |
|-----------|--------------------|--------------------|
| `EntityNotFoundException` 하위 | 404 | "해당 리소스를 찾을 수 없습니다" |
| `ForbiddenException` | 403 | "접근 권한이 없습니다" |
| `InvalidInputException` | 400 | "잘못된 요청입니다" |
| `ConflictException` | 409 | "이미 존재하는 리소스입니다" |

커스텀 메시지로 생성된 예외도 있으므로, 실제 Usecase/Exception 소스를 확인하여 정확한 값으로 검증한다.

코드 예제는 `${CLAUDE_PLUGIN_ROOT}/references/be-test-refs/api-patterns.md`의 패턴 레터 참조.

### 실제 예시: BookmarkApiControllerTest

```java
@ExtendWith(MockitoExtension.class)
class BookmarkApiControllerTest {

    @Mock private BookmarkUsecase bookmarkUsecase;
    @Mock private UnbookmarkUsecase unbookmarkUsecase;

    @InjectMocks private BookmarkApiController controller;

    private final SessionUser sessionUser = SessionUser.of(1L, Instant.now(), Instant.now().plusSeconds(3600));
    private final Long userId = 1L;
    private final Long setupId = 10L;

    // --- 정상: POST 201 ---
    @Test
    void should_returnCreated_when_bookmarkSuccess() {
        doNothing().when(bookmarkUsecase).execute(userId, setupId);

        ResponseEntity<Void> response = controller.bookmark(setupId, sessionUser);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        verify(bookmarkUsecase).execute(userId, setupId);
    }

    // --- 예외: 404 + httpStatusCode/httpErrorMessage 검증 ---
    @Test
    void should_throw404_when_bookmarkUserNotFound() {
        doThrow(new UserNotFoundException("User not found: 999"))
                .when(bookmarkUsecase).execute(userId, setupId);

        assertThatThrownBy(() -> controller.bookmark(setupId, sessionUser))
                .isInstanceOf(UserNotFoundException.class)
                .satisfies(ex -> {
                    HttpException httpEx = (HttpException) ex;
                    assertThat(httpEx.httpStatusCode()).isEqualTo(404);
                    assertThat(httpEx.httpErrorMessage()).isEqualTo("사용자를 찾을 수 없습니다");
                });
    }

    // --- 예외: 409 ---
    @Test
    void should_throw409_when_alreadyBookmarked() {
        doThrow(new ConflictException("already bookmarked"))
                .when(bookmarkUsecase).execute(userId, setupId);

        assertThatThrownBy(() -> controller.bookmark(setupId, sessionUser))
                .isInstanceOf(ConflictException.class)
                .satisfies(ex -> {
                    HttpException httpEx = (HttpException) ex;
                    assertThat(httpEx.httpStatusCode()).isEqualTo(409);
                    assertThat(httpEx.httpErrorMessage()).isEqualTo("이미 존재하는 리소스입니다");
                });
    }

    // --- 정상: DELETE 200 ---
    @Test
    void should_returnOk_when_unbookmarkSuccess() {
        doNothing().when(unbookmarkUsecase).execute(userId, setupId);

        ResponseEntity<Void> response = controller.unbookmark(setupId, sessionUser);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        verify(unbookmarkUsecase).execute(userId, setupId);
    }

    // --- 예외: 400 ---
    @Test
    void should_throw400_when_notBookmarked() {
        doThrow(new InvalidInputException("not bookmarked"))
                .when(unbookmarkUsecase).execute(userId, setupId);

        assertThatThrownBy(() -> controller.unbookmark(setupId, sessionUser))
                .isInstanceOf(InvalidInputException.class)
                .satisfies(ex -> {
                    HttpException httpEx = (HttpException) ex;
                    assertThat(httpEx.httpStatusCode()).isEqualTo(400);
                    assertThat(httpEx.httpErrorMessage()).isEqualTo("잘못된 요청입니다");
                });
    }
}
```

> 참고: `UserNotFoundException`의 `httpErrorMessage`는 기본값 `"해당 리소스를 찾을 수 없습니다"`가 아니라
> 커스텀 `"사용자를 찾을 수 없습니다"`이다. **반드시 실제 Exception 소스를 확인할 것.**

### 검증

```bash
./gradlew :{modules.api.gradlePath}:test --tests "{packageRoot}.api.{domain}.{TestClass}"
```

---

## 공통 규칙

- Phase 1은 최초 1회만 실행
- Phase 2 산출물을 사용자에게 보여주고 확인 후 Phase 3 진행
- 테스트 코드 내에서 프로덕션 코드를 수정하지 않는다
- 기존 테스트 파일이 있으면 덮어쓰지 않고 추가/수정한다
