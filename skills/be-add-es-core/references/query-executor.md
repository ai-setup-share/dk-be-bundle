# Query Executor 참고 코드

## SearchQueryExecutor

ES 클라이언트 래퍼. 4가지 검색 메서드 제공.

```java
import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch._types.query_dsl.Query;
import co.elastic.clients.elasticsearch.core.GetRequest;
import co.elastic.clients.elasticsearch.core.SearchRequest;
import co.elastic.clients.elasticsearch.core.search.Hit;
import jakarta.json.stream.JsonGenerator;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.io.StringWriter;
import java.util.List;
import java.util.Objects;

@Component
@RequiredArgsConstructor
@Slf4j
public class SearchQueryExecutor {
    private final ElasticsearchClient esClient;

    /** 기본 검색 + 페이징 */
    public <T> SearchResult<T> search(
            String indexName, Query query, int from, int size, Class<T> resultType) {
        try {
            SearchRequest req = new SearchRequest.Builder()
                    .index(indexName)
                    .query(query)
                    .from(from)
                    .size(size)
                    .build();

            logQuery(req);

            var response = esClient.search(req, resultType);
            long totalHits = response.hits().total().value();
            List<T> docs = response.hits().hits().stream()
                    .map(Hit::source).filter(Objects::nonNull).toList();

            return new SearchResult<>(docs, totalHits);
        } catch (Exception e) {
            throw new ElasticsearchQueryException(indexName, e);
        }
    }

    /** 정렬 포함 검색 */
    public <T> SearchResult<T> searchWithSort(
            String indexName, Query query, int from, int size,
            SortOption sortOption, Class<T> resultType) {
        try {
            SearchRequest req = new SearchRequest.Builder()
                    .index(indexName)
                    .query(query)
                    .from(from)
                    .sort(s -> s.field(f -> f
                            .field(sortOption.field())
                            .order(sortOption.order())
                            .missing("_last")))
                    .size(size)
                    .build();

            logQuery(req);

            var response = esClient.search(req, resultType);
            long totalHits = response.hits().total().value();
            List<T> docs = response.hits().hits().stream()
                    .map(Hit::source).filter(Objects::nonNull).toList();

            return new SearchResult<>(docs, totalHits);
        } catch (Exception e) {
            throw new ElasticsearchQueryException(indexName, e);
        }
    }

    /** KNN 벡터 검색 */
    public <T> List<T> searchByVector(
            String indexName, List<Float> queryVector, Query filterQuery,
            int size, Class<T> resultType) {
        try {
            SearchRequest req = new SearchRequest.Builder()
                    .index(indexName)
                    .knn(knn -> knn
                            .queryVector(queryVector)
                            .field("vector")
                            .k(100)
                            .numCandidates(300)
                            .filter(filterQuery))
                    .size(size)
                    .build();

            logQuery(req);

            var response = esClient.search(req, resultType);
            return response.hits().hits().stream()
                    .map(Hit::source).filter(Objects::nonNull).toList();
        } catch (Exception e) {
            throw new ElasticsearchQueryException(indexName, e);
        }
    }

    /** ID로 단건 조회 */
    public <T> T getById(String indexName, String docId, Class<T> resultType) {
        try {
            GetRequest req = new GetRequest.Builder()
                    .index(indexName).id(docId).build();
            var response = esClient.get(req, resultType);
            if (!response.found()) {
                throw new ElasticsearchQueryException(indexName,
                        new IllegalArgumentException("Document not found: " + docId));
            }
            return response.source();
        } catch (Exception e) {
            throw new ElasticsearchQueryException(indexName, e);
        }
    }

    private void logQuery(SearchRequest req) {
        try {
            StringWriter w = new StringWriter();
            JsonGenerator g = esClient._transport().jsonpMapper()
                    .jsonProvider().createGenerator(w);
            esClient._transport().jsonpMapper().serialize(req, g);
            g.close();
            log.info("ES query: {}", w);
        } catch (Exception e) {
            log.warn("쿼리 로깅 실패", e);
        }
    }
}
```

## SearchResult

```java
import java.util.List;

public record SearchResult<T>(List<T> docs, long totalHits) {}
```
