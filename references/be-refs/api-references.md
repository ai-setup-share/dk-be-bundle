# API References

`be-add-api` 스킬이 참고하는 Controller + Request/Response DTO 작성 레퍼런스.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## 공통 요소

### Controller 클래스 선언

```java
@RestController
@RequestMapping("/api/{domain-plural}")
@RequiredArgsConstructor
@Slf4j
@Tag(name = "Foo", description = "Foo API")
public class FooApiController {
    private final FooReader fooReader;
    private final FooCreateUsecase fooCreateUsecase;
    // ...
}
```

- `/api/` prefix 필수 (be-init 컨벤션에서 노출된 경로)
- 도메인은 **복수형** (`/api/setups`, `/api/comments`, `/api/bookmarks`)
- `@Tag` — springdoc 에서 API 그룹 표시

### 메서드 어노테이션 셋

- `@Operation(summary = "...")` — springdoc 요약
- HTTP 매핑: `@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping` — **path 는 어노테이션 인자**
- 파라미터:
  - `@PathVariable Long id`
  - `@RequestParam(defaultValue = "...") String xxx`
  - `@Valid @RequestBody FooXxxRequest request`
  - `@AuthenticationPrincipal SessionUser sessionUser`

### HTTP 상태 코드와 반환 관례

| 상황 | 반환 |
|---|---|
| 조회 성공 | `ResponseEntity.ok(body)` — 200 |
| 생성 성공 | `ResponseEntity.status(HttpStatus.CREATED).body(body)` — 201 |
| 삭제 성공 (body 없음) | `ResponseEntity.noContent().build()` — 204 |
| 단순 완료 (body 없음) | `ResponseEntity.ok().build()` — 200 |

반환 타입: `ResponseEntity<{Domain}Response>` 또는 `ResponseEntity<Void>`.

### 의존 구성

- **Reader** + **Usecase** 중심
- Writer 를 Controller 에서 직접 주입하지 않는다 — 선검증·부수효과가 필요한 동작은 Usecase 를 거친다
- 인증 필요 없는 공개 API 는 `@AuthenticationPrincipal` 생략

### 로깅 관례

엔드포인트 진입 + 결과를 각각 한 번씩:

```java
log.info("POST /api/setups - userId={}, type={}, title={}", userId, req.getType(), req.getTitle());
// ... execute ...
log.info("Setup created: id={}", identity.getSetupId());
```

### Enum 파싱 helper

쿼리/request body 에서 들어오는 enum 스트링은 Controller private helper 로 변환.
실패 시 `InvalidInputException` (be-init C-6 / HttpException 400).

```java
private List<FooCategory> parseCategories(List<String> raw) {
    return raw.stream().map(s -> {
        try { return FooCategory.valueOf(s.toUpperCase()); }
        catch (IllegalArgumentException e) {
            throw new InvalidInputException("Invalid category: " + s);
        }
    }).toList();
}

private List<String> parseCommaSeparated(String value) {
    if (value == null || value.isBlank()) return List.of();
    return Arrays.stream(value.split(","))
            .map(String::trim)
            .filter(s -> !s.isEmpty())
            .toList();
}
```

### Request DTO 관례

```java
@Getter
@NoArgsConstructor
@AllArgsConstructor
public class FooCreateRequest {

    @NotBlank
    private String title;

    @NotBlank
    private String body;

    @NotNull
    private List<String> categories;

    private String link;    // optional
}
```

- **`class`** (record 아님) — `@NoArgsConstructor` + `@AllArgsConstructor` + `@Getter` 조합이 Jackson 역직렬화 · Bean Validation · 부분 패치 에 모두 편함
- Bean Validation 직접 부여: `@NotBlank`, `@NotNull`, `@Size`, `@Min`/`@Max`
- 파일 위치: `api/src/main/java/{packagePath}/api/{domain}/dto/{Domain}{Action}Request.java`

### Response DTO 관례

```java
@Getter
@AllArgsConstructor
@Schema(description = "Foo response")
public class FooResponse {

    @Schema(description = "ID", example = "1")
    private final Long id;

    @Schema(description = "제목")
    private final String title;

    @Schema(description = "작성자 닉네임")
    private final String author;

    @Schema(description = "카테고리")
    private final List<String> categories;

    @Schema(description = "작성 시각")
    private final Instant createdAt;

    public static FooResponse from(FooRead read) {
        return new FooResponse(
                read.getFooId(),
                read.getTitle(),
                read.getAuthor(),
                read.getCategories().stream().map(Enum::name).toList(),
                read.getCreatedAt()
        );
    }

    @Getter
    @AllArgsConstructor
    @Schema(description = "연관 VO")
    public static class RelatedVO {
        @Schema(description = "ID") private final Long id;
        @Schema(description = "이름") private final String name;
    }
}
```

- **`class`**, 모든 필드 `private final`
- `@Schema` 어노테이션으로 springdoc 설명
- **`static from({Domain}Read)` 팩토리** — Read 모델 → Response 변환. 이 팩토리가 Controller 본문을 간결하게 유지
- Inner static class 로 연관 VO 표현 가능 (`FooResponse.RelatedVO`)
- enum 은 `.name()` 으로 문자열 직렬화

### List + total envelope

페이지 조회용:

```java
@Getter
@AllArgsConstructor
@Schema(description = "Foo list response")
public class FooListResponse {
    @Schema(description = "결과 목록")
    private final List<FooResponse> data;

    @Schema(description = "총 건수")
    private final long total;
}
```

---

## Pattern Foo — 표준 RESTful Controller (리소스형 CRUD + 인증)

**언제 쓰나**: 리소스 명사형, 인증 필요, 전체 CRUD. 가장 흔한 경우.

```java
@RestController
@RequestMapping("/api/foos")
@RequiredArgsConstructor
@Slf4j
@Tag(name = "Foo", description = "Foo API")
public class FooApiController {

    private final FooReader fooReader;
    private final FooCreateUsecase fooCreateUsecase;
    private final FooUpdateUsecase fooUpdateUsecase;
    private final FooDeleteUsecase fooDeleteUsecase;
    private final FooDetailUsecase fooDetailUsecase;

    @Operation(summary = "Foo 목록 조회")
    @GetMapping
    public ResponseEntity<FooListResponse> getFoos(
            @RequestParam(defaultValue = "recent") String sort,
            @RequestParam(defaultValue = "") String q,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {

        log.info("GET /api/foos - sort={}, q={}, page={}, size={}", sort, q, page, size);

        List<FooRead> reads = fooReader.getAll(sort, q, page, size);
        long total = fooReader.countAll(q);

        List<FooResponse> data = reads.stream().map(FooResponse::from).toList();

        return ResponseEntity.ok(new FooListResponse(data, total));
    }

    @Operation(summary = "Foo 상세 조회")
    @GetMapping("/{id}")
    public ResponseEntity<FooDetailResponse> getFoo(@PathVariable Long id) {
        log.info("GET /api/foos/{}", id);
        FooRead read = fooDetailUsecase.execute(new FooIdentity(id));
        return ResponseEntity.ok(FooDetailResponse.from(read));
    }

    @Operation(summary = "Foo 생성")
    @PostMapping
    public ResponseEntity<FooCreateResponse> createFoo(
            @AuthenticationPrincipal SessionUser sessionUser,
            @Valid @RequestBody FooCreateRequest request) {

        Long userId = sessionUser.getUserId();
        log.info("POST /api/foos - userId={}, title={}", userId, request.getTitle());

        FooCreateCommand command = new FooCreateCommand(
                userId,
                request.getTitle(),
                request.getBody(),
                parseCategories(request.getCategories())
        );

        FooIdentity identity = fooCreateUsecase.execute(userId, command);

        log.info("Foo created: id={}", identity.getFooId());

        return ResponseEntity.status(HttpStatus.CREATED)
                .body(new FooCreateResponse(identity.getFooId()));
    }

    @Operation(summary = "Foo 수정")
    @PutMapping("/{id}")
    public ResponseEntity<FooCreateResponse> updateFoo(
            @PathVariable Long id,
            @AuthenticationPrincipal SessionUser sessionUser,
            @Valid @RequestBody FooUpdateRequest request) {

        Long userId = sessionUser.getUserId();
        log.info("PUT /api/foos/{} - userId={}", id, userId);

        FooUpdateCommand command = new FooUpdateCommand(
                userId, id,
                request.getTitle(), request.getBody(),
                parseCategories(request.getCategories())
        );

        Foo updated = fooUpdateUsecase.execute(userId, command);

        return ResponseEntity.ok(new FooCreateResponse(updated.getFooId()));
    }

    @Operation(summary = "Foo 삭제")
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteFoo(
            @PathVariable Long id,
            @AuthenticationPrincipal SessionUser sessionUser) {

        Long userId = sessionUser.getUserId();
        log.info("DELETE /api/foos/{} - userId={}", id, userId);

        fooDeleteUsecase.execute(userId, new FooIdentity(id));

        return ResponseEntity.noContent().build();
    }

    private List<FooCategory> parseCategories(List<String> raw) {
        return raw.stream().map(s -> {
            try { return FooCategory.valueOf(s.toUpperCase()); }
            catch (IllegalArgumentException e) {
                throw new InvalidInputException("Invalid category: " + s);
            }
        }).toList();
    }
}
```

**특징**:
- 엔드포인트 5개 (list/detail/create/update/delete) + 선택적 2개 (me/view 등)
- Reader 1개 + Usecase N개 (create/update/delete/detail)
- Request DTO 2~3개 (Create/Update, 필요 시 Query)
- Response DTO 3~4개 (Response/DetailResponse/ListResponse/CreateResponse)

---

## Pattern Bar — Action Controller (동작형 리소스)

**언제 쓰나**: "행동" 이 리소스인 경우. bookmark/rating 같이 **상태 전이** 중심.

```java
@RestController
@RequestMapping("/api/bookmarks")
@RequiredArgsConstructor
@Slf4j
@Tag(name = "Bookmark", description = "Bookmark API")
public class BookmarkApiController {

    private final BookmarkUsecase bookmarkUsecase;
    private final UnbookmarkUsecase unbookmarkUsecase;
    private final SavedPostReader savedPostReader;

    @Operation(summary = "북마크 추가")
    @PostMapping
    public ResponseEntity<Void> bookmark(
            @AuthenticationPrincipal SessionUser sessionUser,
            @Valid @RequestBody BookmarkRequest request) {

        Long userId = sessionUser.getUserId();
        log.info("POST /api/bookmarks - userId={}, setupId={}", userId, request.getSetupId());

        bookmarkUsecase.execute(userId, request.getSetupId());

        return ResponseEntity.status(HttpStatus.CREATED).build();
    }

    @Operation(summary = "북마크 취소")
    @DeleteMapping("/{setupId}")
    public ResponseEntity<Void> unbookmark(
            @PathVariable Long setupId,
            @AuthenticationPrincipal SessionUser sessionUser) {

        Long userId = sessionUser.getUserId();
        log.info("DELETE /api/bookmarks/{} - userId={}", setupId, userId);

        unbookmarkUsecase.execute(userId, setupId);

        return ResponseEntity.noContent().build();
    }

    @Operation(summary = "내 북마크 목록")
    @GetMapping
    public ResponseEntity<BookmarkListResponse> getMyBookmarks(
            @AuthenticationPrincipal SessionUser sessionUser,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        Long userId = sessionUser.getUserId();
        List<SavedPostRead> reads = savedPostReader.getByUserId(userId, page, size);
        return ResponseEntity.ok(BookmarkListResponse.from(reads));
    }
}
```

**특징**:
- CRUD 풀세트가 아니라 **주요 동작 2~4개**
- Usecase 위주 (BookmarkUsecase/UnbookmarkUsecase) — 동작 하나당 Usecase 하나
- Path variable 은 대상 리소스의 ID (여기선 setupId) — bookmark 자체 id 가 아님
- 예: BookmarkApiController, RatingApiController, CommentHideController(비슷하면)

---

## Pattern Baz — Read-only Public Controller (인증 없음)

**언제 쓰나**: 정적·reference 데이터. `@AuthenticationPrincipal` 사용 안 함.

```java
@RestController
@RequestMapping("/api/ai-doc-series")
@RequiredArgsConstructor
@Slf4j
@Tag(name = "AiDocSeries", description = "AI Doc Series API")
public class AiDocSeriesApiController {

    private final AiDocSeriesReader aiDocSeriesReader;

    @Operation(summary = "시리즈 목록")
    @GetMapping
    public ResponseEntity<AiDocSeriesListResponse> getAll() {
        List<AiDocSeriesRead> reads = aiDocSeriesReader.getAll();
        List<AiDocSeriesResponse> data = reads.stream().map(AiDocSeriesResponse::from).toList();
        return ResponseEntity.ok(new AiDocSeriesListResponse(data));
    }

    @Operation(summary = "시리즈 상세")
    @GetMapping("/{id}")
    public ResponseEntity<AiDocSeriesResponse> getById(@PathVariable Long id) {
        AiDocSeriesRead read = aiDocSeriesReader.getById(new AiDocSeriesIdentity(id));
        return ResponseEntity.ok(AiDocSeriesResponse.from(read));
    }
}
```

**특징**:
- GET 만
- `@AuthenticationPrincipal` 없음
- Reader 만 의존
- 공개 캐싱 전략 가능 (헤더 설정은 필요할 때 추가)

---

## Pattern Qux — 집계·마이페이지 Controller (Usecase 중심)

**언제 쓰나**: 여러 도메인을 한 엔드포인트로 조합 반환. 대시보드·마이페이지.

```java
@RestController
@RequestMapping("/api/mypage")
@RequiredArgsConstructor
@Slf4j
@Tag(name = "Mypage", description = "Mypage API")
public class MypageApiController {

    private final MypageUsecase mypageUsecase;
    private final MypageSetupUsecase mypageSetupUsecase;
    private final MypageCommentUsecase mypageCommentUsecase;

    @Operation(summary = "마이페이지 대시보드")
    @GetMapping
    public ResponseEntity<MypageResponse> getMypage(
            @AuthenticationPrincipal SessionUser sessionUser) {
        Long userId = sessionUser.getUserId();
        MypageRead read = mypageUsecase.execute(userId);
        return ResponseEntity.ok(MypageResponse.from(read));
    }

    @Operation(summary = "내 글 목록")
    @GetMapping("/setups")
    public ResponseEntity<SetupListResponse> getMySetups(
            @AuthenticationPrincipal SessionUser sessionUser,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        Long userId = sessionUser.getUserId();
        MypageSetupList list = mypageSetupUsecase.execute(userId, page, size);
        return ResponseEntity.ok(SetupListResponse.from(list));
    }

    @Operation(summary = "내 댓글 목록")
    @GetMapping("/comments")
    public ResponseEntity<CommentListResponse> getMyComments(
            @AuthenticationPrincipal SessionUser sessionUser,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        Long userId = sessionUser.getUserId();
        MypageCommentList list = mypageCommentUsecase.execute(userId, page, size);
        return ResponseEntity.ok(CommentListResponse.from(list));
    }
}
```

**특징**:
- Reader 미주입 (있어도 되지만 주력 로직은 Usecase)
- `@AuthenticationPrincipal` 필수 (내 데이터이므로)
- Usecase 가 여러 도메인 Repository 를 직접 조립 (service-references.md §Qux 참조)

---

## 패턴 선택 가이드

| 엔드포인트 특성 | Pattern |
|---|---|
| 리소스 명사형, 인증, 전체 CRUD | **Foo** (주력) |
| 동작형 리소스 (bookmark/rating) | **Bar** |
| 공개 조회 전용 | **Baz** |
| 여러 도메인 집계 (마이페이지/대시보드) | **Qux** |

---

## 💡 팁 — 위 패턴들로 풀리지 않을 때

1. **파일 분리 먼저 의심** — 한 Controller 에 메서드가 7~8개 이상이면 도메인 경계 재검토. Foo + Bar 혼합인 경우 분리.
2. **Controller 에 로직 들어가지 않게** — enum 파싱 외에는 Controller 본문이 request → command → usecase.execute → response 의 4줄 이상이면 Usecase 로 밀어내기.
3. **Response DTO 는 Read 모델과 1:N 관계일 수 있음** — 같은 Read 로 DetailResponse/ListResponse/MeResponse 등 여러 뷰 투영이 자연스러움.
4. 그래도 표현이 어색하면 **유저에게 질의** — GraphQL / BFF / 특수 endpoint 스펙 등 구조적 선택지.

---

## 파일 구성 요약

```
api/src/main/java/{packagePath}/api/{domain}/
├── {Domain}ApiController.java          ← Pattern Foo / Bar / Baz / Qux
└── dto/
    ├── {Domain}CreateRequest.java      ← class + validation
    ├── {Domain}UpdateRequest.java
    ├── {Domain}Response.java           ← class + @Schema + static from(...)
    ├── {Domain}DetailResponse.java
    ├── {Domain}ListResponse.java       ← List + total envelope
    ├── {Domain}CreateResponse.java     ← id 만 담는 생성 응답
    └── {Domain}MeResponse.java         ← 인증된 사용자 상태 (bookmark/myRating 등, 선택)
```

---

## 공통 금지 사항

- `@Controller` 단독 사용 금지 — 항상 `@RestController`
- `@RequestMapping` 을 메서드 레벨에서 쓰지 않기 — HTTP method 어노테이션 (`@GetMapping` 등) 사용
- Response DTO 에 도메인 모델 (`{Domain}Read`) 직접 노출 금지 — 항상 Response 로 투영
- Writer 를 Controller 에 직접 주입 금지 — Usecase 경유
- Controller 에서 `@Transactional` 금지 — 트랜잭션 경계는 service 레이어
- Request DTO 에 비즈니스 로직 메서드 금지 — 순수 데이터 홀더
- Response DTO 에 `from({Domain})` (Write 모델) 금지 — `from({Domain}Read)` 만 (Read 투영 규칙 유지)
- 세션·쿠키 직접 조작 금지 — `@AuthenticationPrincipal` 로 받음
