---
name: be-add-es-search
description: 도메인별 ES Search 컴포넌트와 SearchResult를 생성한다. 텍스트 검색, 벡터 검색, 필터 검색을 수행하는 도메인별 검색 클래스를 만들 때 사용한다. "ES 검색 기능 만들어줘", "벡터 검색 구현해줘" 등의 요청에 트리거한다. be-add-es-core, be-add-es-field가 선행되어야 한다.
---

# be-add-es-search

도메인별 ES Search 컴포넌트 + SearchResult를 생성한다.

## Phase 0. 프로젝트 컨텍스트 해석

`{esModule}`, `{esPackage}` 플레이스홀더 사용.
실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

## 선행 조건

- `be-add-es-core` 실행 완료 (SearchQueryExecutor 등 공통 인프라 존재)
- `be-add-es-field` 실행 완료 (IndexField + Registry 존재)
- `be-add-es-doc` 실행 완료 (Document 클래스 존재)

## 입력

유저에게 아래를 확인한다:

1. **대상 도메인** — IndexField, Registry, Document가 이미 존재해야 함
2. **검색 유형** — 아래 중 필요한 것만 선택
   - 텍스트 검색 (BM25, relevance score 정렬)
   - 벡터 검색 (KNN similarity)
   - 정렬 검색 (created_at desc 등)
   - 단건 조회 (getById)
3. **자동 필터 조건** — 항상 추가되어야 하는 조건 (예: deleted=false)
4. **MATCH 판단 필드** — relevance score 정렬 여부를 결정하는 필드 목록 (예: TITLE, SEARCH_WORD)
5. **기본 페이지 사이즈**

## 생성 결과물 구조

```
{esModule}/src/main/java/{esPackage-as-path}/
└── es/
    └── search/
        └── {domain}/
            ├── {Domain}Search.java          ← 검색 컴포넌트
            └── {Domain}SearchResult.java    ← 결과 record
```

## 생성 규칙

### 1. SearchResult record

```java
public record {Domain}SearchResult(
        List<{Domain}Doc> docs,
        boolean hasNext,
        long totalHits
) {
    // 벡터 검색용 간편 생성자 (totalHits 불필요)
    public {Domain}SearchResult(List<{Domain}Doc> docs, boolean hasNext) {
        this(docs, hasNext, -1L);
    }
}
```

### 2. Search 컴포넌트

`@Component` + `@RequiredArgsConstructor`. 핵심 패턴:

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class {Domain}Search {

    @Value("${elasticsearch.index.{domain}}")
    private String INDEX_NAME;

    private final SearchQueryExecutor executor;
    private static final Integer DEFAULT_PAGE_SIZE = 20;

    /** 텍스트 + 필터 검색 */
    public {Domain}SearchResult search(SearchCommand<{Domain}IndexField> command) {
        var cmd = ensureDefaultConditions(command);

        var query = GenericSearchQueryBuilder.build(cmd,
                {Domain}IndexQueryBuilderRegistry.LOOKUP,
                {Domain}IndexRangeQueryBuilderRegistry.LOOKUP);

        var pagination = PaginationUtils.calculatePaginationInfo(
                cmd.from(), cmd.to(), DEFAULT_PAGE_SIZE);

        SearchResult<{Domain}Doc> result;

        if (hasMatchQuery(cmd)) {
            // MATCH 쿼리 있으면 → relevance score 정렬
            result = executor.search(INDEX_NAME, query,
                    pagination.from(), pagination.searchSize(), {Domain}Doc.class);
        } else {
            // 없으면 → created_at desc
            result = executor.searchWithSort(INDEX_NAME, query,
                    pagination.from(), pagination.searchSize(),
                    SortOption.desc("created_at"), {Domain}Doc.class);
        }

        var page = PaginationUtils.paginate(result.docs(), pagination.requestedSize());
        return new {Domain}SearchResult(page.data(), page.hasNext(), result.totalHits());
    }

    /** 벡터(KNN) 검색 */
    public {Domain}SearchResult searchByVector(
            List<Float> queryVector,
            SearchCommand<{Domain}IndexField> command,
            int size) {
        var cmd = ensureDefaultConditions(command);

        var filterQuery = GenericSearchQueryBuilder.build(cmd,
                {Domain}IndexQueryBuilderRegistry.LOOKUP,
                {Domain}IndexRangeQueryBuilderRegistry.LOOKUP);

        var docs = executor.searchByVector(
                INDEX_NAME, queryVector, filterQuery, size, {Domain}Doc.class);

        return new {Domain}SearchResult(docs, false);
    }

    /** 단건 조회 */
    public {Domain}Doc getById(String docId) {
        return executor.getById(INDEX_NAME, docId, {Domain}Doc.class);
    }

    /** MATCH 쿼리 존재 여부 판단 */
    private boolean hasMatchQuery(SearchCommand<{Domain}IndexField> command) {
        return command.conditions().stream()
                .anyMatch(e -> e.getField() == {Domain}IndexField.SEARCH_WORD
                        || e.getField() == {Domain}IndexField.TITLE);
    }

    /** 기본 조건 자동 추가 (deleted=false 등) */
    private SearchCommand<{Domain}IndexField> ensureDefaultConditions(
            SearchCommand<{Domain}IndexField> command) {
        boolean hasDeleted = command.conditions().stream()
                .anyMatch(e -> e.getField() == {Domain}IndexField.DELETED);

        if (hasDeleted) return command;

        var newConditions = new ArrayList<>(command.conditions());
        newConditions.add(new SearchElement<>({Domain}IndexField.DELETED, false));
        return new SearchCommand<>(newConditions, command.from(), command.to());
    }
}
```

### 3. 검색 유형별 메서드 포함 여부

유저가 선택한 검색 유형에 따라 메서드를 포함/제외:

| 메서드 | 텍스트 검색 | 벡터 검색 | 단건 조회 |
|---|---|---|---|
| `search()` | O | - | - |
| `searchByVector()` | - | O | - |
| `getById()` | - | - | O |

기본적으로 전부 포함. 유저가 명시적으로 빼달라고 하면 제외.

### 4. 정렬 전략

- MATCH 쿼리(텍스트 검색어)가 있으면 → ES relevance score 정렬 (기본)
- MATCH 쿼리 없이 TERM/RANGE 필터만 있으면 → `created_at desc` 정렬
- `hasMatchQuery()` 에서 판단할 필드 목록은 유저에게 확인

## 참고 코드

- `references/techblog-search-example.md` — TechBlogSearch 전체 코드

## 산출물

- 생성된 파일 목록
- 지원 검색 유형 테이블
- hasMatchQuery 판단 필드 목록
