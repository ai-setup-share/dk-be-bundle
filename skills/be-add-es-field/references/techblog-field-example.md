# TechBlog IndexField + Registry 참고 예시

## TechBlogIndexField

```java
public enum TechBlogIndexField implements DocFieldName, FieldName {
    DOC_ID("doc_id", DocQueryType.TERM),
    TECH_BLOG_ID("tech_blog_id", DocQueryType.TERM),
    URL("url", DocQueryType.TERM),

    COMPANY("company", DocQueryType.TERM),
    TITLE("title", DocQueryType.MATCH),

    ONE_LINER("one_liner", DocQueryType.MATCH),
    SUMMARY("summary", DocQueryType.MATCH),
    KOREAN_SUMMARY("korean_summary", DocQueryType.MATCH),
    MARKDOWN_BODY("markdown_body", DocQueryType.MATCH),

    THUMBNAIL_URL("thumbnail_url", DocQueryType.TERM),
    TECH_CATEGORIES("tech_categories", DocQueryType.TERM),
    ORIGINAL_URL("original_url", DocQueryType.TERM),

    POPULARITY_VIEW_COUNT("popularity_view_count", DocQueryType.RANGE),
    POPULARITY_COMMENT_COUNT("popularity_comment_count", DocQueryType.RANGE),
    POPULARITY_LIKE_COUNT("popularity_like_count", DocQueryType.RANGE),

    DELETED("deleted", DocQueryType.TERM),
    CREATED_AT("created_at", DocQueryType.RANGE),
    UPDATED_AT("updated_at", DocQueryType.RANGE),

    SEARCH_WORD("search_word", DocQueryType.MATCH);

    private final String fieldName;
    private final DocQueryType queryType;

    TechBlogIndexField(String fieldName, DocQueryType queryType) {
        this.fieldName = fieldName;
        this.queryType = queryType;
    }

    @Override
    public String getFieldName() { return fieldName; }
    public DocQueryType getQueryType() { return queryType; }
}
```

## TechBlogIndexQueryBuilderRegistry

```java
public class TechBlogIndexQueryBuilderRegistry {
    private static final Map<TechBlogIndexField, FieldQueryBuilder> builderMap =
            new EnumMap<>(TechBlogIndexField.class);

    static {
        register(TechBlogIndexField.DOC_ID, new TermQueryBuilder(BoolType.FILTER));
        register(TechBlogIndexField.TECH_BLOG_ID, new TermQueryBuilder(BoolType.FILTER));
        register(TechBlogIndexField.URL, new TermQueryBuilder(BoolType.FILTER));

        register(TechBlogIndexField.COMPANY, new TermQueryBuilder(BoolType.FILTER));
        register(TechBlogIndexField.TITLE, new MatchQueryBuilder(BoolType.SHOULD));

        register(TechBlogIndexField.ONE_LINER, new MatchQueryBuilder(BoolType.SHOULD));
        register(TechBlogIndexField.SUMMARY, new MatchQueryBuilder(BoolType.SHOULD));
        register(TechBlogIndexField.KOREAN_SUMMARY, new MatchQueryBuilder(BoolType.SHOULD));
        register(TechBlogIndexField.SEARCH_WORD, new MatchQueryBuilder(BoolType.SHOULD));
        register(TechBlogIndexField.MARKDOWN_BODY, new MatchQueryBuilder(BoolType.SHOULD));

        register(TechBlogIndexField.THUMBNAIL_URL, new TermQueryBuilder(BoolType.FILTER));
        register(TechBlogIndexField.TECH_CATEGORIES, new TermQueryBuilder(BoolType.FILTER));
        register(TechBlogIndexField.ORIGINAL_URL, new TermQueryBuilder(BoolType.FILTER));

        register(TechBlogIndexField.DELETED, new TermQueryBuilder(BoolType.MUST));
    }

    private static void register(TechBlogIndexField field, FieldQueryBuilder builder) {
        builderMap.put(field, builder);
    }

    public static Optional<FieldQueryBuilder> getBuilder(TechBlogIndexField field) {
        return Optional.ofNullable(builderMap.get(field));
    }

    public static final Function<TechBlogIndexField, Optional<FieldQueryBuilder>> LOOKUP =
            TechBlogIndexQueryBuilderRegistry::getBuilder;
}
```

## TechBlogIndexRangeQueryBuilderRegistry

```java
public class TechBlogIndexRangeQueryBuilderRegistry {
    private static final Map<TechBlogIndexField, RangeQueryBuilder> builderMap =
            new EnumMap<>(TechBlogIndexField.class);

    static {
        register(TechBlogIndexField.POPULARITY_VIEW_COUNT, new RangeQueryBuilder(BoolType.FILTER));
        register(TechBlogIndexField.POPULARITY_COMMENT_COUNT, new RangeQueryBuilder(BoolType.FILTER));
        register(TechBlogIndexField.POPULARITY_LIKE_COUNT, new RangeQueryBuilder(BoolType.FILTER));

        register(TechBlogIndexField.CREATED_AT, new RangeQueryBuilder(BoolType.FILTER));
        register(TechBlogIndexField.UPDATED_AT, new RangeQueryBuilder(BoolType.FILTER));
    }

    private static void register(TechBlogIndexField field, RangeQueryBuilder builder) {
        builderMap.put(field, builder);
    }

    public static Optional<RangeQueryBuilder> getBuilder(TechBlogIndexField field) {
        return Optional.ofNullable(builderMap.get(field));
    }

    public static final Function<TechBlogIndexField, Optional<RangeQueryBuilder>> LOOKUP =
            TechBlogIndexRangeQueryBuilderRegistry::getBuilder;
}
```
