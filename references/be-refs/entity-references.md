# Entity References

`be-add-entity-specs` 스킬이 참고하는 Entity 작성 레퍼런스.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## 공통 요소

### Auditing + soft-delete 4필드 직접 선언

Spring Data JDBC 는 JPA 의 `@MappedSuperclass` 를 지원하지 않으므로 abstract BaseEntity 상속 대신 각 Entity 가 직접 필드를 선언하고 `AuditFields` 인터페이스를 구현한다 (be-init §4.6 / C-3, C-4).

모든 Entity 가 보유할 필드 4개:
```java
private Boolean isDeleted;
private Instant deletedAt;

@CreatedDate
private Instant createdAt;

@LastModifiedDate
private Instant updatedAt;
```

`AuditFields` 인터페이스 (be-init 제공):
```java
public interface AuditFields {
    Instant getCreatedAt();
    Instant getUpdatedAt();
}
```

**예외**: Stats 분리 Entity (Pattern Bar) 는 audit/soft-delete 미적용 — 이유는 해당 섹션 참조.

### @Table, @Id, Lombok

- `@Table("{snake_case_테이블명}")` 필수 (Spring Data JDBC 의 quoted identifier 특성상 snake_case 로 명시)
- `@Id private Long id` — 테이블 내부 PK. 도메인 모델의 `{domain}Id` 와 1:1 매핑되지만 Entity 에서는 `id` 명명.
- `@Getter` (필수), `@AllArgsConstructor` 는 Pattern Bar 에서만. Pattern Foo/Baz/Qux 는 **명시 생성자** 를 작성해 BaseEntity-like 슈퍼클래스 필드 대신 직접 초기화.

### toDomain / from / softDelete 메서드 시그니처

모든 Entity 가 다음 3개를 제공 (Pattern Bar 는 softDelete 제외):

```java
public {Domain} toDomain() { ... }               // Entity → Domain
public static {Entity} from({Domain} domain) { ... }  // Domain → Entity (new record: isDeleted=false, deletedAt=null)
public {Entity} softDelete() { ... }             // 새 인스턴스: isDeleted=true, deletedAt=Instant.now(), updatedAt=Instant.now()
```

- `from()` 의 `isDeleted`/`deletedAt` 초기값은 항상 `false` / `null`
- `softDelete()` 는 updatedAt 도 함께 갱신
- createdAt 은 softDelete 에서 **유지**

---

## Pattern Foo — 단순 Entity

**언제 쓰나**: 다른 테이블과의 컬렉션·임베드 관계 없는 flat 테이블 (평점, 북마크, 반응, 유저, 알림 등).

```java
@Table("foos")
@Getter
public class FooEntity implements AuditFields {

    @Id
    private Long id;
    private Long userId;
    private Long targetId;
    private FooSource source;
    private int score;

    private Boolean isDeleted;
    private Instant deletedAt;

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    public FooEntity(Long id, Long userId, Long targetId, FooSource source, int score,
                     Boolean isDeleted, Instant deletedAt,
                     Instant createdAt, Instant updatedAt) {
        this.id = id;
        this.userId = userId;
        this.targetId = targetId;
        this.source = source;
        this.score = score;
        this.isDeleted = isDeleted;
        this.deletedAt = deletedAt;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
    }

    public Foo toDomain() {
        return new Foo(id, userId, targetId, source, score, createdAt, updatedAt);
    }

    public static FooEntity from(Foo domain) {
        return new FooEntity(
                domain.getFooId(), domain.getUserId(), domain.getTargetId(),
                domain.getSource(), domain.getScore(),
                false, null,
                domain.getCreatedAt(), domain.getUpdatedAt());
    }

    public FooEntity softDelete() {
        return new FooEntity(id, userId, targetId, source, score,
                true, Instant.now(),
                createdAt, Instant.now());
    }
}
```

**특징**:
- 구성 파일: `FooEntity.java` 1개
- @MappedCollection/@Embedded 없음
- Pattern Bar(Stats)가 아닌 이상 softDelete() 제공

---

## Pattern Bar — Stats 분리 Entity (audit 미적용)

**언제 쓰나**: 본 도메인과 분리된 통계 테이블. 쓰기 경쟁 격리·조회 최적화 목적.

```java
@Table("foo_stats")
@Getter
@AllArgsConstructor
public class FooStatsEntity {

    @Id
    private Long id;
    private Long fooId;
    private long viewCount;
    private long ratingSum;
    private int ratingCount;
    private int commentCount;

    public FooStats toDomain() {
        return new FooStats(fooId, viewCount, ratingSum, ratingCount, commentCount);
    }

    public static FooStatsEntity from(FooStats domain) {
        return new FooStatsEntity(null, domain.getFooId(),
                domain.getViewCount(), domain.getRatingSum(),
                domain.getRatingCount(), domain.getCommentCount());
    }
}
```

**특징**:
- **AuditFields 미구현**, isDeleted/deletedAt/createdAt/updatedAt **필드 없음**
- `@AllArgsConstructor` Lombok 사용 (명시 생성자 불필요)
- `softDelete()` 없음 — 통계는 원 도메인과 생애주기 동일, soft-delete 개념 불필요
- `id` (PK) + `{domain}Id` (FK) 2개 키 보유. `from()` 에서 `id=null` 로 insert, `{domain}Id` 만 매핑
- 구성 파일: `FooStatsEntity.java` 1개

**DDL 상 차이**: `foo_stats` 테이블은 audit/soft-delete 컬럼을 **포함하지 않는다** (be-init 의 "모든 테이블 4필드" 원칙에서 명시 예외). schema-references.md §"공통 컬럼" 하단에 기술.

---

## Pattern Baz — @MappedCollection 포함

**언제 쓰나**: 1:N 자식 레코드를 aggregate 내부 컬렉션으로 표현. enum list 저장, 복합 VO 컬렉션 저장 등.

### 컬렉션 래퍼 클래스

각 자식 테이블마다 래퍼 클래스 1개. **같은 repository 패키지 내에 평면 배치** (`collection/` 서브디렉토리 생성 금지 — share_ai_setup 실제 구조):

```java
@Table("baz_categories")
@Getter
@AllArgsConstructor
public class BazCategoryCollection {
    private BazCategory category;   // enum 1개만 보유
}
```

복합 VO 컬렉션이면 여러 필드:
```java
@Table("baz_items")
@Getter
@AllArgsConstructor
public class BazItemCollection {
    private Long linkedId;
    private int itemOrder;
}
```

### Baz Entity

```java
@Table("bazs")
@Getter
public class BazEntity implements AuditFields {

    @Id
    private Long id;
    private Long userId;
    private BazType type;
    private String title;
    private String body;
    private BazSource source;

    @MappedCollection(idColumn = "baz_id", keyColumn = "baz_id")
    private Set<BazCategoryCollection> categories;

    @MappedCollection(idColumn = "baz_id", keyColumn = "baz_id")
    private Set<BazTagCollection> tags;

    @MappedCollection(idColumn = "baz_id", keyColumn = "baz_id")
    private Set<BazItemCollection> items;

    private Boolean isDeleted;
    private Instant deletedAt;

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    public BazEntity(Long id, Long userId, BazType type, String title, String body, BazSource source,
                     Set<BazCategoryCollection> categories,
                     Set<BazTagCollection> tags,
                     Set<BazItemCollection> items,
                     Boolean isDeleted, Instant deletedAt,
                     Instant createdAt, Instant updatedAt) {
        this.id = id;
        this.userId = userId;
        this.type = type;
        this.title = title;
        this.body = body;
        this.source = source;
        this.categories = categories;
        this.tags = tags;
        this.items = items;
        this.isDeleted = isDeleted;
        this.deletedAt = deletedAt;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
    }

    public Baz toDomain() {
        List<BazCategory> categoryList = categories != null
                ? categories.stream().map(BazCategoryCollection::getCategory).toList()
                : List.of();
        List<BazTag> tagList = tags != null
                ? tags.stream().map(BazTagCollection::getTag).toList()
                : List.of();
        List<BazItem> itemList = items != null
                ? items.stream()
                    .map(i -> new BazItem(i.getLinkedId(), null, i.getItemOrder()))
                    .toList()
                : List.of();
        return new Baz(id, userId, type, title, body, categoryList, tagList,
                source, itemList, createdAt, updatedAt);
    }

    public static BazEntity from(Baz domain) {
        Set<BazCategoryCollection> catSet = domain.getCategories().stream()
                .map(BazCategoryCollection::new)
                .collect(Collectors.toSet());
        Set<BazTagCollection> tagSet = domain.getTags().stream()
                .map(BazTagCollection::new)
                .collect(Collectors.toSet());
        Set<BazItemCollection> itemSet = domain.getItems().stream()
                .map(i -> new BazItemCollection(i.getLinkedId(), i.getOrder()))
                .collect(Collectors.toSet());
        return new BazEntity(domain.getBazId(), domain.getUserId(), domain.getType(),
                domain.getTitle(), domain.getBody(), domain.getSource(),
                catSet, tagSet, itemSet,
                false, null, domain.getCreatedAt(), domain.getUpdatedAt());
    }

    public BazEntity softDelete() {
        return new BazEntity(id, userId, type, title, body, source,
                categories, tags, items,
                true, Instant.now(), createdAt, Instant.now());
    }
}
```

**특징**:
- `@MappedCollection(idColumn, keyColumn)` — 둘 다 **자식 테이블 FK 컬럼명** 과 동일 (share_ai_setup 실제)
- 컬렉션 필드는 항상 `Set<...>` (순서 무관, 중복 불허)
- `toDomain()` 에서 null 체크 필수 (`categories != null ? ... : List.of()`)
- `from()` 에서 도메인의 `List` → `Set` 변환 시 `Collectors.toSet()` 사용
- 구성 파일: `BazEntity.java`, `BazCategoryCollection.java`, `BazTagCollection.java`, `BazItemCollection.java` 등 **자식 수만큼**

**금지**:
- `collection/` 서브디렉토리 생성 금지 (평면 배치)
- `@Embedded.Nullable` 사용 금지 (share_ai_setup 에 실제 없음, 구현 복잡도 대비 효용 낮음)

---

## Pattern Qux — 상태 전이 메서드 (hide/show 등)

**언제 쓰나**: soft-delete 외에 추가 상태 전환(숨김/표시, 읽음/안읽음, 활성/비활성 등)이 있는 도메인.

```java
@Table("quxs")
@Getter
public class QuxEntity implements AuditFields {

    @Id
    private Long id;
    private Long articleId;
    private QuxArticleType articleType;
    private Long userId;
    private String text;
    private Long parentId;
    private int quxOrder;
    private int level;
    private int sortNumber;
    private int childCount;
    private boolean isHidden;                  // ← 상태 플래그

    private Boolean isDeleted;
    private Instant deletedAt;

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    public QuxEntity(/* all fields */) {
        // field assignments
    }

    public Qux toDomain() { ... }
    public static QuxEntity from(Qux domain) { ... }

    public QuxEntity hide() {
        return new QuxEntity(id, articleId, articleType, userId, text,
                parentId, quxOrder, level, sortNumber, childCount, true,
                isDeleted, deletedAt, createdAt, Instant.now());
    }

    public QuxEntity show() {
        return new QuxEntity(id, articleId, articleType, userId, text,
                parentId, quxOrder, level, sortNumber, childCount, false,
                isDeleted, deletedAt, createdAt, Instant.now());
    }

    public QuxEntity softDelete() { ... }    // 필요 시에만 (Comment 는 미제공)
}
```

**특징**:
- 상태 전이는 **동사 메서드** 로 (`hide()`, `show()`, `activate()`, `archive()` 등)
- `withIsHidden(boolean)` 같은 setter-style 금지
- soft-delete 와 hide/show 는 **독립 개념**. 도메인이 hide/show 만 쓰면 softDelete() 는 미구현 (Comment 예)
- 소프트 삭제 필드는 여전히 보유 (공통 규약). 미사용 시 항상 `false`/`null`
- 구성 파일: `QuxEntity.java` 1개

---

## 패턴 선택 가이드

| 도메인 특성 | Pattern |
|---|---|
| 단일 테이블, 컬렉션/임베드 없음 | **Foo** |
| 통계 분리 테이블 (본 도메인의 집계) | **Bar** — audit 미적용 |
| 1:N 자식(enum 리스트 / 복합 VO 컬렉션) 보유 | **Baz** |
| hide/show 등 추가 상태 전이 필요 | **Qux** |

대부분 도메인은 Foo. Baz 는 복합 aggregate(Setup/Series 류). Bar 는 Stats 전용. Qux 는 Comment 류 몇 개뿐.

---

## 파일 구성 요약

```
repository-jdbc/src/main/java/{packagePath}/jdbc/{domain}/repository/
├── {Domain}Entity.java                ← Pattern Foo/Baz/Qux
├── {Domain}StatsEntity.java           ← Pattern Bar (필요 시)
├── {Name}Collection.java              ← Pattern Baz 의 자식 래퍼 (평면 배치)
└── {OtherChild}Collection.java
```

---

## 공통 금지 사항

- `BaseEntity extends` 금지 — @MappedSuperclass 미지원으로 조용한 null timestamp 버그 유발
- `@Embedded.Nullable` 사용 금지 — share_ai_setup 실제 패턴에 없음, 복합 VO 는 자식 테이블 + `@MappedCollection` 으로 처리
- `collection/` / `embedded/` 서브디렉토리 금지 — 평면 배치
- `@Builder` 금지 — 명시 생성자 또는 `@AllArgsConstructor`
- `@Setter` 금지
- `@Getter` 생략 금지
