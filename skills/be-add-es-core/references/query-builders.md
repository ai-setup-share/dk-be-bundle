# Query Builders 참고 코드

## FieldQueryBuilder (인터페이스)

TERM/MATCH 쿼리 빌더 인터페이스.

```java
import co.elastic.clients.elasticsearch._types.query_dsl.Query;

public interface FieldQueryBuilder {
    Query build(String fieldName, String value);
    BoolType boolType();
}
```

## MatchQueryBuilder

전문 검색 (text 필드). 토큰화 + 관련도 스코어링.

```java
import co.elastic.clients.elasticsearch._types.query_dsl.Query;

public class MatchQueryBuilder implements FieldQueryBuilder {
    private final BoolType boolType;

    public MatchQueryBuilder(BoolType boolType) {
        this.boolType = boolType;
    }

    @Override
    public Query build(String fieldName, String value) {
        return Query.of(q -> q.match(m -> m.field(fieldName).query(value)));
    }

    @Override
    public BoolType boolType() {
        return boolType;
    }
}
```

## TermQueryBuilder

정확 일치 (keyword 필드). 토큰화 없음.

```java
import co.elastic.clients.elasticsearch._types.query_dsl.Query;

public class TermQueryBuilder implements FieldQueryBuilder {
    private final BoolType boolType;

    public TermQueryBuilder(BoolType boolType) {
        this.boolType = boolType;
    }

    @Override
    public Query build(String fieldName, String value) {
        return Query.of(q -> q.term(m -> m.field(fieldName).value(value)));
    }

    @Override
    public BoolType boolType() {
        return boolType;
    }
}
```

## RangeQueryBuilder

범위 쿼리 (숫자, 날짜). gte/lte 조합.

```java
import co.elastic.clients.elasticsearch._types.query_dsl.Query;
import co.elastic.clients.elasticsearch._types.query_dsl.RangeQuery;
import co.elastic.clients.json.JsonData;

public class RangeQueryBuilder {
    private final BoolType boolType;

    public RangeQueryBuilder(BoolType boolType) {
        this.boolType = boolType;
    }

    public Query build(String fieldName, String gte, String lte) {
        var range = RangeQuery.of(r -> {
            RangeQuery.Builder builder = new RangeQuery.Builder().field(fieldName);
            if (gte != null) builder.gte(JsonData.of(gte));
            if (lte != null) builder.lte(JsonData.of(lte));
            return builder;
        });
        return Query.of(q -> q.range(range));
    }

    public BoolType boolType() {
        return boolType;
    }
}
```
