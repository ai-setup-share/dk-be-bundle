# Query Composition 참고 코드

## QueryWithBoolType

Query와 BoolType을 묶는 페어.

```java
import co.elastic.clients.elasticsearch._types.query_dsl.Query;

public record QueryWithBoolType(Query query, BoolType type) {}
```

## BoolQueryComposer

QueryWithBoolType 리스트를 하나의 Bool 쿼리로 조합.
SHOULD 절이 있으면 minimumShouldMatch=1 설정.

```java
import co.elastic.clients.elasticsearch._types.query_dsl.Query;
import java.util.List;

public class BoolQueryComposer {

    public static Query compose(List<QueryWithBoolType> queries) {
        return Query.of(
                q -> q.bool(
                        b -> {
                            for (QueryWithBoolType qbt : queries) {
                                switch (qbt.type()) {
                                    case MUST -> b.must(qbt.query());
                                    case SHOULD -> b.should(qbt.query());
                                    case FILTER -> b.filter(qbt.query());
                                    case MUST_NOT -> b.mustNot(qbt.query());
                                }
                            }
                            long shouldCount = queries.stream()
                                    .filter(bq -> bq.type() == BoolType.SHOULD).count();
                            if (shouldCount > 0) {
                                b.minimumShouldMatch(String.valueOf(1));
                            }
                            return b;
                        }));
    }
}
```

## GenericSearchQueryBuilder

레지스트리 기반 쿼리 빌드 오케스트레이터.
SearchElement 리스트를 순회하며 레지스트리에서 빌더를 찾아 개별 쿼리를 만들고, BoolQueryComposer로 조합.

```java
import co.elastic.clients.elasticsearch._types.query_dsl.Query;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.function.Function;

public final class GenericSearchQueryBuilder {

    private GenericSearchQueryBuilder() {}

    public static <F extends FieldName> Query build(
            SearchCommand<F> command,
            Function<? super F, Optional<FieldQueryBuilder>> qbRegistry,
            Function<? super F, Optional<RangeQueryBuilder>> rangeRegistry
    ) {
        return build(command.conditions(), qbRegistry, rangeRegistry);
    }

    public static <F extends FieldName> Query build(
            List<SearchElement<F>> requests,
            Function<? super F, Optional<FieldQueryBuilder>> qbRegistry,
            Function<? super F, Optional<RangeQueryBuilder>> rangeRegistry
    ) {
        var queriesWithBool = new ArrayList<QueryWithBoolType>();

        for (var req : requests) {
            var field = req.getField();
            var value = req.getValue();
            var gte   = req.getGte();
            var lte   = req.getLte();

            var qOpt  = qbRegistry.apply(field);
            var rOpt  = rangeRegistry.apply(field);

            if (qOpt.isEmpty() && rOpt.isEmpty()) {
                throw new SearchFieldNotFoundException(field.getFieldName());
            }

            if (qOpt.isPresent()) {
                if (value == null) continue;
                var qb   = qOpt.get();
                var q    = qb.build(field.getFieldName(), value);
                var type = qb.boolType();
                queriesWithBool.add(new QueryWithBoolType(q, type));
            } else {
                if (gte == null && lte == null) continue;
                var rb   = rOpt.get();
                var q    = rb.build(field.getFieldName(), gte, lte);
                var type = rb.boolType();
                queriesWithBool.add(new QueryWithBoolType(q, type));
            }
        }

        return BoolQueryComposer.compose(queriesWithBool);
    }
}
```
