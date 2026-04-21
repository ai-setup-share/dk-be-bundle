# TechBlogDoc 참고 예시 (breakin-be)

실제 운영 중인 ES Document 예시. 새 도메인 Doc 작성 시 이 패턴을 따른다.

## Document

```java
@Value
@JsonInclude(JsonInclude.Include.NON_NULL)
public class TechBlogDoc implements DocBase {

    // ES Type: keyword (used as document ID)
    @JsonProperty("doc_id")
    String docId;

    // ES Type: long
    @JsonProperty("tech_blog_id")
    Long techBlogId;

    // ES Type: text + keyword (multi-field)
    @JsonProperty("title")
    String title;

    // ES Type: text
    @JsonProperty("summary")
    String summary;

    // ES Type: keyword (array)
    @JsonProperty("tech_categories")
    List<String> techCategories;

    // Popularity fields (flattened)
    // ES Type: long
    @JsonProperty("popularity_view_count")
    Long popularityViewCount;

    // ES Type: boolean
    @JsonProperty("deleted")
    Boolean deleted;

    // ES Type: date (epoch_millis)
    @JsonProperty("created_at")
    Long createdAt;

    // ES Type: date (epoch_millis)
    @JsonProperty("updated_at")
    Long updatedAt;

    // ES Type: dense_vector
    @JsonProperty("vector")
    List<Float> vector;

    // ES Type: text (벡터화 대상 합성 텍스트)
    @JsonProperty("vector_text")
    String vectorText;

    // ES Type: text (BM25 검색용 합성 텍스트)
    @JsonProperty("search_word")
    String searchWord;

    public static TechBlogDoc of(/* 필드들 (vector 제외) */) {
        String vectorText = buildVectorText(title, summary);
        String searchWord = buildSearchWord(title, company, techCategories, oneLiner, summary);
        return new TechBlogDoc(..., null, vectorText, searchWord);
    }

    private static String buildVectorText(String title, String summary) {
        StringBuilder sb = new StringBuilder();
        if (title != null && !title.isBlank()) sb.append(title).append("\n");
        if (summary != null && !summary.isBlank()) sb.append(summary);
        return sb.toString().trim();
    }

    private static String buildSearchWord(
            String title, String company, List<String> categories,
            String oneLiner, String summary) {
        StringBuilder sb = new StringBuilder();
        if (title != null && !title.isBlank()) sb.append(title).append("\n");
        if (company != null && !company.isBlank()) sb.append(company).append("\n");
        if (categories != null && !categories.isEmpty()) sb.append(String.join(" ", categories)).append("\n");
        if (oneLiner != null && !oneLiner.isBlank()) sb.append(oneLiner).append("\n");
        if (summary != null && !summary.isBlank()) sb.append(summary);
        return sb.toString().trim();
    }
}
```

## Mapper

```java
@Component
@RequiredArgsConstructor
public class TechBlogDocMapper {

    private final OpenAiVectorClient vectorClient;

    public TechBlogDoc toDoc(TechBlog techBlog) {
        var categories = techBlog.getTechCategories().stream()
                .map(Enum::name)
                .collect(Collectors.toList());

        TechBlogDoc doc = TechBlogDoc.of(
                generateDocId(techBlog),
                techBlog.getTechBlogId(),
                techBlog.getTitle(),
                techBlog.getSummary(),
                categories,
                // ... popularity, meta fields
                toEpochMillis(techBlog.getCreatedAt()),
                toEpochMillis(techBlog.getUpdatedAt())
        );

        // vectorText로 임베딩 생성
        float[] embedding = vectorClient.embed(doc.getVectorText());
        // vector 주입한 새 인스턴스 반환
        return new TechBlogDoc(
                doc.getDocId(), doc.getTechBlogId(), doc.getTitle(), /* ... */
                toFloatList(embedding), doc.getVectorText(), doc.getSearchWord()
        );
    }

    private String generateDocId(TechBlog techBlog) {
        return "techblog_" + techBlog.getTechBlogId();
    }

    private List<Float> toFloatList(float[] arr) {
        List<Float> list = new ArrayList<>(arr.length);
        for (float f : arr) list.add(f);
        return list;
    }
}
```

## Indexer

```java
@Component
public class TechBlogIndexer extends AbstractDocIndexer {

    private final String indexName;

    public TechBlogIndexer(
            ElasticsearchClient esClient,
            @Value("${elasticsearch.index.techblog}") String indexName
    ) {
        super(esClient);
        this.indexName = indexName;
    }

    @Override
    protected String getIndex() { return indexName; }

    @Override
    protected String getDocId(DocBase doc) { return doc.getDocId(); }

    @Override
    protected boolean validateDoc(DocBase doc) { return doc.getDocId() != null; }
}
```
