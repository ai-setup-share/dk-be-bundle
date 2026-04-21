# Query Type System 참고 코드

## FieldName

```java
public interface FieldName {
    String getFieldName();
}
```

## QueryType

```java
public enum QueryType {
    TERM,
    MATCH,
    RANGE,
    EXISTS,
    NESTED
}
```

## BoolType

```java
public enum BoolType {
    MUST,
    FILTER,
    SHOULD,
    MUST_NOT
}
```

## SortOption

```java
import co.elastic.clients.elasticsearch._types.SortOrder;

public record SortOption(String field, SortOrder order) {
    public static SortOption desc(String field) {
        return new SortOption(field, SortOrder.Desc);
    }
    public static SortOption asc(String field) {
        return new SortOption(field, SortOrder.Asc);
    }
}
```
