# AbstractDocIndexer 참고 코드 (breakin-be)

ES 클라이언트를 이용한 문서 인덱싱 추상 클래스.

```java
@RequiredArgsConstructor
@Slf4j
@Component
public abstract class AbstractDocIndexer implements DocIndexer {

    protected final ElasticsearchClient esClient;

    protected abstract String getIndex();
    protected abstract String getDocId(DocBase doc);
    protected abstract boolean validateDoc(DocBase doc);

    public IndexResponseType indexOne(DocBase doc) {
        if (!validateDoc(doc))
            throw new IllegalStateException("document not validated status");

        try {
            var indexResponse = sendRequest(doc);
            return IndexResponseType.from(indexResponse.result());
        } catch (Exception e) {
            log.error(e.getLocalizedMessage());
            log.error("", e);
            throw new DocumentIndexingException(doc.getDocId(), e);
        }
    }

    private IndexResponse sendRequest(DocBase doc) throws IOException {
        return esClient.index(i -> i
                .index(getIndex())
                .id(getDocId(doc))
                .document(doc)
        );
    }
}
```

## IndexResponseType

```java
public enum IndexResponseType {
    Created("created"),
    Updated("updated"),
    Deleted("deleted"),
    NotFound("not_found"),
    NoOp("noop"),
    Unknown("unknown");

    private final String value;

    IndexResponseType(String value) { this.value = value; }

    public String getValue() { return value; }

    public static IndexResponseType from(Result result) {
        if (result == null) return Unknown;
        String jsonValue = result.jsonValue();
        for (IndexResponseType type : values()) {
            if (type.value.equals(jsonValue)) return type;
        }
        return Unknown;
    }
}
```
