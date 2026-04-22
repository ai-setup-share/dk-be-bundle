---
name: be-add-api
description: api 모듈에 Controller와 Request/Response DTO를 생성/변경한다.
---

# add-api

api 모듈에 Controller와 Request/Response DTO를 생성/변경한다.

## 입력

### 지시
- assess-api 산출물 (API 스펙, 호출 대상 Reader/Writer/Usecase)에 따라 작업한다.
- 개별 API 추가 요청에도 아래의 공통 규칙을 따르시오.

### 참고 문서 (읽어올 것)
- `${CLAUDE_PLUGIN_ROOT}/references/be-refs/api-references.md` — 4 패턴 (Foo/Bar/Baz/Qux) + 공통 요소 (Request/Response DTO, 인증, 로깅, enum 파싱)
- specs/api.md — 기존 API 엔드포인트 정보 (프로젝트 루트 기준, 없으면 소스에서 추론)
- specs/usecase.md — 호출 대상 인터페이스 시그니처 + DTO 스펙 (프로젝트 루트 기준)
- specs/models.md — 도메인 모델 (Response 필드 참조) (프로젝트 루트 기준)

## 패턴 선택 (먼저 결정)

api-references.md §"패턴 선택 가이드" 로 엔드포인트 특성에 맞는 패턴을 먼저 고른다:

| 엔드포인트 특성 | Pattern |
|---|---|
| 리소스 명사형, 인증, 전체 CRUD | **Foo** (주력) |
| 동작형 리소스 (bookmark/rating) | **Bar** |
| 공개 조회 전용 | **Baz** |
| 여러 도메인 집계 (마이페이지/대시보드) | **Qux** |

## 작업 순서

1. **참고 문서를 읽는다.**
2. **개발 가능 여부를 판단한다.**
   - 요구 스펙을 현재 usecase 인터페이스와 모델로 구현할 수 있는지 확인한다.
3. **불가하거나 스펙 변화가 필요한 경우:**
   - 최소 변화로 스펙을 달성할 수 있는 방안을 유저에게 제시하고 확인받는다.
4. **가능하다면 제공된 데이터 스펙과 인터페이스에 맞춰 개발한다.**

## 생성 결과물 구조

### 기본

```
api/src/main/java/{packagePath}/api/{domain}/
├── {Domain}ApiController.java
└── dto/
    ├── {Domain}Request.java
    └── {Domain}Response.java
```

### 복수 DTO

```
api/src/main/java/{packagePath}/api/{domain}/
├── {Domain}ApiController.java
└── dto/
    ├── {Domain}Request.java
    ├── {Domain}Response.java
    ├── {Domain}ListResponse.java       ← 목록 + 페이징
    └── {Domain}{Action}Response.java   ← 특수 응답
```

## 생성 규칙

### 1. Controller

```java
@RestController
@RequestMapping("/api/{domain}")
@RequiredArgsConstructor
@Slf4j
@Tag(name = "{Domain}", description = "{Domain} API")
public class {Domain}ApiController {

    private final {Domain}Reader {domain}Reader;
    private final {Domain}Writer {domain}Writer;
}
```

- `@RestController`, `@RequestMapping`, `@RequiredArgsConstructor`, `@Slf4j`
- `@Tag` (Swagger 그룹)
- Reader/Writer 인터페이스만 의존
- 비즈니스 로직 없음 — 변환과 위임만

### 2. 엔드포인트 패턴

**조회 (GET):**

```java
@Operation(summary = "...")
@GetMapping("/{id}")
public ResponseEntity<{Domain}Response> get{Domain}(@PathVariable Long id) {
    {Domain}Read read = {domain}Reader.getById(new {Domain}Identity(id));
    {Domain}Response response = {Domain}Response.from(read);
    return ResponseEntity.ok(response);
}
```

**생성/수정 (POST):**

```java
@Operation(summary = "...")
@PostMapping
public ResponseEntity<{Domain}Response> create{Domain}(
        @Valid @RequestBody {Domain}Request request) {
    // Request → Command 변환
    // Writer 호출
    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}
```

**삭제 (DELETE):**

```java
@Operation(summary = "...")
@DeleteMapping("/{id}")
public ResponseEntity<Void> delete{Domain}(@PathVariable Long id) {
    // Writer 호출
    return ResponseEntity.noContent().build();
}
```

### 3. Request DTO

```java
@Getter
@NoArgsConstructor
@AllArgsConstructor
public class {Domain}Request {
    @NotNull
    private {Type} requiredField;

    @NotBlank
    private String title;

    private String optionalField;  // optional은 validation 없음
}
```

- `@Getter`, `@NoArgsConstructor`, `@AllArgsConstructor`
- 필수 필드에 `@NotNull`, `@NotBlank` 등 validation
- optional 필드는 어노테이션 없음

### 4. Response DTO

```java
@Getter
@AllArgsConstructor
@Schema(description = "{Domain} response")
public class {Domain}Response {
    @Schema(description = "ID", example = "1")
    private final Long {domain}Id;
    // ...

    public static {Domain}Response from({Domain}Read read) {
        return new {Domain}Response(
                read.get{Domain}Id(),
                // ...
        );
    }
}
```

- `@Getter`, `@AllArgsConstructor`, `@Schema`
- **`static from()` 팩토리 메서드**로 Domain → Response 변환
- 필드에 `@Schema` (Swagger 문서화)

### 5. 로깅

- 요청 시: `log.info("{METHOD} {path} - params...")`
- 응답 시: `log.info("결과 요약")`

## 기존 변경 시

- as-is/to-be를 유저에게 제시하고 확인받는다
- 엔드포인트 추가는 하위 호환
- Request/Response 스펙 변경은 유저 확인 필수
- 기존 엔드포인트와 유사한 요청이 오면 병합 여부를 유저에게 문의

## 산출물

- 생성/변경된 Controller, Request, Response 파일 목록
- specs/api.md 업데이트
- 추가/변경된 스펙 정리본 출력
