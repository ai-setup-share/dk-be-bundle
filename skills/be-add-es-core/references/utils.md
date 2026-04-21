# Utils 참고 코드

## PaginationUtils

+1 패턴으로 hasNext 판단. count 쿼리 없이 페이지네이션.

```java
import java.util.List;

public class PaginationUtils {

    public static <T> PaginatedResult<T> paginate(List<T> searchResults, int requestedSize) {
        boolean hasNext = searchResults.size() > requestedSize;
        List<T> resultData = hasNext
                ? searchResults.subList(0, requestedSize) : searchResults;
        return new PaginatedResult<>(resultData, hasNext);
    }

    public static PaginationInfo calculatePaginationInfo(
            Integer from, Integer to, int defaultPageSize) {
        int actualFrom = from != null ? from : 0;
        int actualTo = to != null ? to : actualFrom + defaultPageSize;
        int requestedSize = actualTo - actualFrom;
        int searchSize = requestedSize + 1; // hasNext용 +1
        return new PaginationInfo(actualFrom, actualTo, requestedSize, searchSize);
    }

    public record PaginationInfo(int from, int to, int requestedSize, int searchSize) {}
    public record PaginatedResult<T>(List<T> data, boolean hasNext) {}
}
```

## QueryHelper

기존 Bool 쿼리에 should 절을 병합하는 유틸. boost 쿼리 등을 추가할 때 사용.

```java
import co.elastic.clients.elasticsearch._types.query_dsl.Query;
import java.util.ArrayList;
import java.util.List;

public class QueryHelper {

    /** should 절 병합 (minimumShouldMatch=0, 선택사항으로 점수만 증가) */
    public static Query mergeShoulds(Query base, List<Query> shoulds) {
        if (shoulds == null || shoulds.isEmpty()) return base;

        if (base.isBool()) {
            var b = base.bool();
            var merged = new ArrayList<Query>();
            if (b.should() != null) merged.addAll(b.should());
            merged.addAll(shoulds);

            return Query.of(x -> x.bool(bb -> bb
                    .must(b.must())
                    .filter(b.filter())
                    .mustNot(b.mustNot())
                    .should(merged)
                    .minimumShouldMatch("0")));
        } else {
            return Query.of(x -> x.bool(bb -> bb
                    .must(base)
                    .should(shoulds)
                    .minimumShouldMatch("0")));
        }
    }

    /** minimumShouldMatch 직접 지정 버전 */
    public static Query mergeShoulds(Query base, List<Query> shoulds, Integer minimumShouldMatch) {
        if (shoulds == null || shoulds.isEmpty()) return base;

        var minShould = minimumShouldMatch == null || minimumShouldMatch < 0
                ? "0" : String.valueOf(minimumShouldMatch);

        if (base.isBool()) {
            var b = base.bool();
            var merged = new ArrayList<Query>();
            if (b.should() != null) merged.addAll(b.should());
            merged.addAll(shoulds);

            return Query.of(x -> x.bool(bb -> bb
                    .must(b.must())
                    .filter(b.filter())
                    .mustNot(b.mustNot())
                    .should(merged)
                    .minimumShouldMatch(minShould)));
        } else {
            return Query.of(x -> x.bool(bb -> bb
                    .must(base)
                    .should(shoulds)
                    .minimumShouldMatch(minShould)));
        }
    }
}
```
