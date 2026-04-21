---
name: be-add-es-field
description: 도메인별 ES IndexField enum과 QueryBuilderRegistry 2개를 생성한다. ES 인덱스에 대한 필드 스펙과 쿼리 빌더 매핑을 정의할 때 사용한다. "ES 필드 스펙 만들어줘", "인덱스 필드 정의해줘" 등의 요청에 트리거한다. be-add-es-core가 선행되어야 한다.
---

# be-add-es-field

도메인별 ES IndexField enum + QueryBuilderRegistry 2개(TERM/MATCH, RANGE)를 생성한다.

## Phase 0. 프로젝트 컨텍스트 해석

`{esModule}`, `{esPackage}` 플레이스홀더 사용.
실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

## 선행 조건

- `be-add-es-core` 실행 완료 (공통 인프라 존재 확인)
- `be-add-es-doc` 실행 완료 (해당 도메인의 Document 클래스 존재)

## 입력

유저에게 아래를 확인한다:

1. **대상 도메인 Document 클래스** — 해당 Doc의 필드 목록을 읽어옴
2. **각 필드의 쿼리 타입** — TERM, MATCH, RANGE 중 하나
3. **각 필드의 Bool 타입** — MUST, FILTER, SHOULD 중 하나

판단 기준 (유저가 지정하지 않으면 아래 기본 규칙 적용):

| 필드 성격 | QueryType | BoolType | 예시 |
|---|---|---|---|
| ID, URL | TERM | FILTER | docId, setupId, url |
| enum, keyword 분류 | TERM | FILTER | categories, tags, type |
| deleted 플래그 | TERM | MUST | deleted |
| 텍스트 검색 대상 | MATCH | SHOULD | title, summary, searchWord |
| 숫자 통계 | RANGE | FILTER | viewCount, ratingCount |
| 날짜 | RANGE | FILTER | createdAt, updatedAt |

## 생성 결과물 구조

```
{esModule}/src/main/java/{esPackage-as-path}/
├── es/
│   └── fieldSpec/
│       ├── DocFieldName.java                    ← 공통 인터페이스 (최초 1회)
│       ├── DocQueryType.java                    ← 공통 enum (최초 1회)
│       └── {domain}/
│           └── {Domain}IndexField.java          ← 도메인별 필드 enum
└── es/
    └── query/
        └── {domain}/
            ├── {Domain}IndexQueryBuilderRegistry.java      ← TERM/MATCH 레지스트리
            └── {Domain}IndexRangeQueryBuilderRegistry.java  ← RANGE 레지스트리
```

## 생성 규칙

### 1. DocFieldName / DocQueryType (최초 1회)

이미 존재하면 스킵.

```java
public interface DocFieldName {
    String getFieldName();
}
```

```java
public enum DocQueryType {
    TERM, MATCH, RANGE, EXISTS, NESTED
}
```

### 2. IndexField enum

- `DocFieldName` + `FieldName` 두 인터페이스를 implements
- 각 enum 값은 `(fieldName, DocQueryType)` 쌍
- fieldName은 Document 클래스의 `@JsonProperty` 값과 일치해야 함

```java
public enum {Domain}IndexField implements DocFieldName, FieldName {

    DOC_ID("doc_id", DocQueryType.TERM),
    // ... Document의 모든 검색 가능 필드

    SEARCH_WORD("search_word", DocQueryType.MATCH);

    private final String fieldName;
    private final DocQueryType queryType;

    {Domain}IndexField(String fieldName, DocQueryType queryType) {
        this.fieldName = fieldName;
        this.queryType = queryType;
    }

    @Override
    public String getFieldName() { return fieldName; }

    public DocQueryType getQueryType() { return queryType; }
}
```

### 3. QueryBuilderRegistry (TERM/MATCH)

- `EnumMap` 기반 정적 레지스트리
- TERM 필드 → `TermQueryBuilder(BoolType.XXX)`
- MATCH 필드 → `MatchQueryBuilder(BoolType.XXX)`
- `LOOKUP` 상수를 `Function<F, Optional<FieldQueryBuilder>>` 로 노출
- RANGE 필드는 여기에 등록하지 않음

```java
public class {Domain}IndexQueryBuilderRegistry {
    private static final Map<{Domain}IndexField, FieldQueryBuilder> builderMap =
            new EnumMap<>({Domain}IndexField.class);

    static {
        // TERM 필드
        register({Domain}IndexField.DOC_ID, new TermQueryBuilder(BoolType.FILTER));
        register({Domain}IndexField.DELETED, new TermQueryBuilder(BoolType.MUST));
        // MATCH 필드
        register({Domain}IndexField.TITLE, new MatchQueryBuilder(BoolType.SHOULD));
        register({Domain}IndexField.SEARCH_WORD, new MatchQueryBuilder(BoolType.SHOULD));
    }

    private static void register({Domain}IndexField field, FieldQueryBuilder builder) {
        builderMap.put(field, builder);
    }

    public static Optional<FieldQueryBuilder> getBuilder({Domain}IndexField field) {
        return Optional.ofNullable(builderMap.get(field));
    }

    public static final Function<{Domain}IndexField, Optional<FieldQueryBuilder>> LOOKUP =
            {Domain}IndexQueryBuilderRegistry::getBuilder;
}
```

### 4. RangeQueryBuilderRegistry

- RANGE 타입 필드만 등록
- 동일한 EnumMap + LOOKUP 패턴

```java
public class {Domain}IndexRangeQueryBuilderRegistry {
    private static final Map<{Domain}IndexField, RangeQueryBuilder> builderMap =
            new EnumMap<>({Domain}IndexField.class);

    static {
        register({Domain}IndexField.VIEW_COUNT, new RangeQueryBuilder(BoolType.FILTER));
        register({Domain}IndexField.CREATED_AT, new RangeQueryBuilder(BoolType.FILTER));
        register({Domain}IndexField.UPDATED_AT, new RangeQueryBuilder(BoolType.FILTER));
    }

    private static void register({Domain}IndexField field, RangeQueryBuilder builder) {
        builderMap.put(field, builder);
    }

    public static Optional<RangeQueryBuilder> getBuilder({Domain}IndexField field) {
        return Optional.ofNullable(builderMap.get(field));
    }

    public static final Function<{Domain}IndexField, Optional<RangeQueryBuilder>> LOOKUP =
            {Domain}IndexRangeQueryBuilderRegistry::getBuilder;
}
```

## 검증

- IndexField의 모든 필드가 QueryBuilderRegistry 또는 RangeQueryBuilderRegistry 둘 중 하나에 등록되어야 함
- 누락된 필드가 있으면 `GenericSearchQueryBuilder`에서 `SearchFieldNotFoundException` 발생
- vector, vectorText 필드는 검색 조건으로 사용하지 않으므로 등록하지 않아도 됨

## 산출물

- 생성된 파일 목록
- 필드별 QueryType + BoolType 매핑 테이블 출력
