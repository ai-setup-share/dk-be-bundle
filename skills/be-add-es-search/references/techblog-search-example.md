# TechBlogSearch 참고 예시

## TechBlogSearch

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class TechBlogSearch {

    @Value("${elasticsearch.index.techblog}")
    private String TECH_BLOG_INDEX;

    private final SearchQueryExecutor executor;
    private static final Integer DEFAULT_PAGE_SIZE = 30;

    /** 텍스트 + 필터 검색 */
    public TechBlogSearchResult search(SearchCommand<TechBlogIndexField> command) {
        var commandWithDeleted = ensureDeletedFalseCondition(command);

        var q = GenericSearchQueryBuilder.build(commandWithDeleted,
                TechBlogIndexQueryBuilderRegistry.LOOKUP,
                TechBlogIndexRangeQueryBuilderRegistry.LOOKUP);

        var pagination = PaginationUtils.calculatePaginationInfo(
                commandWithDeleted.from(), commandWithDeleted.to(), DEFAULT_PAGE_SIZE);

        SearchResult<TechBlogDoc> searchResult;

        if (hasMatchQuery(commandWithDeleted)) {
            searchResult = executor.search(TECH_BLOG_INDEX, q,
                    pagination.from(), pagination.searchSize(), TechBlogDoc.class);
        } else {
            searchResult = executor.searchWithSort(TECH_BLOG_INDEX, q,
                    pagination.from(), pagination.searchSize(),
                    SortOption.desc("created_at"), TechBlogDoc.class);
        }

        var result = PaginationUtils.paginate(searchResult.docs(), pagination.requestedSize());
        return new TechBlogSearchResult(result.data(), result.hasNext(), searchResult.totalHits());
    }

    /** 벡터(KNN) 검색 */
    public TechBlogSearchResult searchByVector(
            List<Float> queryVector,
            SearchCommand<TechBlogIndexField> command,
            int size) {
        var commandWithDeleted = ensureDeletedFalseCondition(command);

        var filterQuery = GenericSearchQueryBuilder.build(commandWithDeleted,
                TechBlogIndexQueryBuilderRegistry.LOOKUP,
                TechBlogIndexRangeQueryBuilderRegistry.LOOKUP);

        var docs = executor.searchByVector(
                TECH_BLOG_INDEX, queryVector, filterQuery, size, TechBlogDoc.class);

        return new TechBlogSearchResult(docs, false);
    }

    /** 단건 조회 */
    public TechBlogDoc getById(String docId) {
        return executor.getById(TECH_BLOG_INDEX, docId, TechBlogDoc.class);
    }

    /** MATCH 쿼리 존재 여부 */
    private boolean hasMatchQuery(SearchCommand<TechBlogIndexField> command) {
        return command.conditions().stream()
                .anyMatch(element ->
                        element.getField() == TechBlogIndexField.SEARCH_WORD
                        || element.getField() == TechBlogIndexField.TITLE
                        || element.getField() == TechBlogIndexField.SUMMARY);
    }

    /** deleted=false 자동 추가 */
    private SearchCommand<TechBlogIndexField> ensureDeletedFalseCondition(
            SearchCommand<TechBlogIndexField> command) {
        boolean hasDeletedCondition = command.conditions().stream()
                .anyMatch(element -> element.getField() == TechBlogIndexField.DELETED);

        if (hasDeletedCondition) return command;

        List<SearchElement<TechBlogIndexField>> newConditions =
                new ArrayList<>(command.conditions());
        newConditions.add(new SearchElement<>(TechBlogIndexField.DELETED, false));

        return new SearchCommand<>(newConditions, command.from(), command.to());
    }
}
```

## TechBlogSearchResult

```java
public record TechBlogSearchResult(
        List<TechBlogDoc> docs,
        boolean hasNext,
        long totalHits
) {
    // 벡터 검색용 간편 생성자
    public TechBlogSearchResult(List<TechBlogDoc> docs, boolean hasNext) {
        this(docs, hasNext, -1L);
    }
}
```
