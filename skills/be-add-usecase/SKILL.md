---
name: be-add-usecase
description: usecase 모듈에 Reader/Writer 또는 Usecase를 생성/변경한다.
---

# add-usecase

usecase 모듈에 Reader/Writer 또는 Usecase를 생성/변경한다.

## 입력

### 지시
- 기본적으로 도메인의 Reader/Writer를 생성한다.
- 단순 읽기/쓰기를 벗어나는 동작은 Usecase로 분리한다.
- Usecase 생성은 유저에게 허락을 받고 진행한다.

### 참고 문서 (읽어올 것)
- `${CLAUDE_PLUGIN_ROOT}/references/be-refs/service-references.md` — 5 패턴 (Foo/Bar/Baz/Qux/Quux) + 공통 요소 + 선택 가이드
- specs/usecase.md — 기존 Reader/Writer 정보 (프로젝트 루트 기준, 없으면 소스에서 직접 추론)
- specs/infrastructure.md — 의존할 Repository 인터페이스 (프로젝트 루트 기준)
- specs/models.md — 도메인 모델 (반환 타입, Command 필드 참조) (프로젝트 루트 기준)

## 패턴 선택 (먼저 결정)

service-references.md §"패턴 선택 가이드" 를 적용해 도메인 특성에 맞는 패턴을 먼저 고른다:

| 도메인 특성 | Pattern |
|---|---|
| 조회만 (정적 / reference 데이터) | **Foo** — Reader 단독 |
| 기본 CRUD | **Bar** — Reader + Writer 쌍 |
| API 단위 여러 도메인 + 부수효과 | **Baz** — Usecase (Bar 위에) |
| 집계/대시보드, 중간 Reader 불필요 | **Qux** — Usecase 단독 |
| 고빈도 이벤트 집계 (특수) | **Quux** — ViewMemory |

대부분 **Bar + Baz** 조합.

## 생성 결과물 구조

### 기본 (Reader/Writer)

```
service/src/main/java/{packagePath}/service/{domain}/
├── {Domain}Reader.java          ← 조회 인터페이스
├── {Domain}Writer.java          ← 명령 인터페이스
├── impl/
│   ├── Default{Domain}Reader.java
│   └── Default{Domain}Writer.java
└── dto/
    └── {Domain}{Action}Command.java  ← 필요 시
```

### Usecase (단순 Reader/Writer를 벗어나는 동작)

```
service/src/main/java/{packagePath}/service/{usecase}/
└── {domainOrActionName}/
    ├── {Name}Usecase.java
    └── dto/
        └── {requiredDto}.java
```

## 기본 원칙

1. **최소 스펙**으로 작업한다.
2. 도메인은 **Reader + Writer**를 기본으로 가진다.
3. Usecase 동작은 **유저에게 허락**을 받고 진행한다.
4. 기존 동작의 변형, DTO 스펙 변형은 **유저에게 허락**을 받고 진행한다.

## 생성 규칙

### 1. Reader (조회)

**인터페이스:**

```java
public interface {Domain}Reader {
    {Domain}Read getById({Domain}Identity identity);
    List<{Domain}Read> getByUserId(UserIdentity userIdentity, int page, int size);
}
```

**구현체:**

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class Default{Domain}Reader implements {Domain}Reader {
    private final {Domain}Repository {domain}Repository;

    @Override
    public {Domain}Read getById({Domain}Identity identity) {
        return {domain}Repository.findById(identity)
                .orElseThrow(() -> new {Domain}NotFoundException("{domain} not found"));
    }
}
```

- `@Service`, `@RequiredArgsConstructor`, `@Slf4j`
- infrastructure Repository 인터페이스만 의존
- 조회 실패 시 도메인 예외 throw
- `@Transactional` 없음

### 2. Writer (명령)

**인터페이스:**

```java
public interface {Domain}Writer {
    {Domain}Read create({Domain}CreateCommand command);
    void delete({Domain}Identity identity, Long requestUserId);
}
```

**구현체:**

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class Default{Domain}Writer implements {Domain}Writer {
    private final {Domain}Reader {domain}Reader;
    private final {Domain}Repository {domain}Repository;

    @Override
    @Transactional
    public {Domain}Read create({Domain}CreateCommand command) {
        {Domain} domain = {Domain}.create(command);
        {Domain} saved = {domain}Repository.save(domain);
        return saved.toRead(/* 추가 필드 */);
    }
}
```

- `@Service`, `@RequiredArgsConstructor`, `@Slf4j`
- **모든 public 메서드에 `@Transactional`**
- Reader를 주입하여 조회 활용
- 크로스 도메인 Repository 주입 가능 (카운터 증감 등)

### 3. Command DTO

```java
public record {Domain}{Action}Command(
        Long requestUserId,
        String title,
        String content
) {
    // 필요 시 팩토리 메서드
    public static {Domain}{Action}Command of(...) {
        return new {Domain}{Action}Command(...);
    }
}
```

- **record** 사용 (불변, 간결)
- `requestUserId`를 명시적으로 포함
- `dto/` 하위에 위치

### 4. Usecase (Reader/Writer를 벗어나는 동작)

여러 도메인을 조율하거나, 단순 CRUD가 아닌 비즈니스 흐름인 경우.

**유저에게 계획을 허락받을 것:**
- 어떤 동작인지
- 왜 Reader/Writer로 부족한지
- as-is/to-be (기존 변형인 경우)

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class {Name}Usecase {
    private final {Domain}Reader {domain}Reader;
    private final {Domain}Writer {domain}Writer;
    // 필요한 의존성
}
```

## 규칙

- Writer는 같은 도메인 Reader를 주입할 수 있다
- 다른 도메인 Reader/Writer 주입은 유저에게 문의
- 비즈니스 로직은 Writer 또는 Usecase에만 둔다
- Reader는 변환과 위임만

## 기존 변경 시

- as-is/to-be를 유저에게 제시하고 확인받는다
- 메서드 추가는 하위 호환
- 시그니처 변경, DTO 스펙 변경은 유저 확인 필수
- 기존 메서드와 유사한 요청이 오면 병합 여부를 유저에게 문의

## 산출물

- 생성/변경된 Reader, Writer, Usecase, Command 파일 목록
- specs/usecase.md 업데이트 — 인터페이스가 외부(api)에 노출되는 계약이므로, 인터페이스 시그니처와 DTO 스펙을 반드시 기록한다.
- 추가/변경된 스펙 정리본 출력
