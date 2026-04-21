# Search Elements 참고 코드

## SearchElement

제네릭 검색 조건 단위. 다양한 타입(String, boolean, Enum, Number, LocalDateTime, Instant, 범위)을 String으로 통일 변환.

```java
import lombok.Getter;
import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneId;

@Getter
public class SearchElement<F extends FieldName> {

    private final F field;
    private final String value;  // TERM/MATCH 값
    private final String gte;    // RANGE 하한
    private final String lte;    // RANGE 상한

    private static final ZoneId ZONE_ID = ZoneId.of("Asia/Seoul");

    /** 풀 커스텀 생성자 */
    public SearchElement(F field, String value, String gte, String lte) {
        this.field = field;
        this.value = value;
        this.gte = gte;
        this.lte = lte;
    }

    /** TERM / MATCH */
    public SearchElement(F field, String value) {
        this(field, value, null, null);
    }
    public SearchElement(F field, boolean value) {
        this(field, String.valueOf(value), null, null);
    }
    public SearchElement(F field, Enum<?> value) {
        this(field, value == null ? null : value.name(), null, null);
    }
    public SearchElement(F field, Number value) {
        this(field, value == null ? null : String.valueOf(value), null, null);
    }

    /** RANGE (시간) — LocalDateTime → epochMillis(KST) */
    public SearchElement(F field, LocalDateTime gte, LocalDateTime lte) {
        this.field = field;
        this.value = null;
        this.gte = toEpochMillisString(gte);
        this.lte = toEpochMillisString(lte);
    }
    /** RANGE (시간) — Instant → epochMillis */
    public SearchElement(F field, Instant gte, Instant lte) {
        this(field, null,
                gte == null ? null : String.valueOf(gte.toEpochMilli()),
                lte == null ? null : String.valueOf(lte.toEpochMilli()));
    }

    /** RANGE (숫자) */
    public SearchElement(F field, Long gte, Long lte) {
        this(field, null,
                gte == null ? null : String.valueOf(gte),
                lte == null ? null : String.valueOf(lte));
    }
    public SearchElement(F field, Integer gte, Integer lte) {
        this(field, null,
                gte == null ? null : String.valueOf(gte),
                lte == null ? null : String.valueOf(lte));
    }
    public SearchElement(F field, Double gte, Double lte) {
        this(field, null,
                gte == null ? null : String.valueOf(gte),
                lte == null ? null : String.valueOf(lte));
    }

    private static String toEpochMillisString(LocalDateTime dt) {
        if (dt == null) return null;
        long epochMillis = dt.atZone(ZONE_ID).toInstant().toEpochMilli();
        return String.valueOf(epochMillis);
    }
}
```

## SearchCommand

검색 명령 record. conditions + 페이징(from/to).

```java
import java.util.List;

public record SearchCommand<F extends FieldName>(
        List<SearchElement<F>> conditions,
        int from,
        int to    // exclusive
) {
    public SearchCommand {
        conditions = (conditions == null) ? List.of() : List.copyOf(conditions);
        from = Math.max(0, from);
        if (to < from) throw new IllegalArgumentException("to must be >= from");
    }

    public int size() { return to - from; }

    public static <F extends FieldName> SearchCommand<F> of(
            List<SearchElement<F>> conditions, int from, int to) {
        return new SearchCommand<>(conditions, from, to);
    }
}
```
