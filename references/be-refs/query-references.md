# Query References

`be-add-jdbc-query` 스킬이 참고하는 JdbcRepository / EntityRepository 작성 레퍼런스.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## 공통 요소

### 클래스 3종과 네이밍

| 역할 | 이름 | 위치 | 가시성 |
|---|---|---|---|
| infra port | `{Domain}Repository` | `infrastructure/.../infra/{domain}/repository/` | public interface |
| JDBC adapter | `{Domain}JdbcRepository` | `repository-jdbc/.../jdbc/{domain}/repository/` | package-private class |
| Spring Data JDBC | `{Domain}EntityRepository` | 동일 패키지 | package-private interface |

- 두 클래스 모두 `@Repository` 부여
- JdbcRepository 에 `@RequiredArgsConstructor`, 의존은 `private final XxxEntityRepository entityRepository`
- EntityRepository 는 `CrudRepository<{Domain}Entity, Long>` 상속

### 주요 import

```java
import org.springframework.data.jdbc.repository.query.Query;       // JPA 의 @Query 아님
import org.springframework.data.jdbc.repository.query.Modifying;   // JPA 의 @Modifying 아님
import org.springframework.data.repository.CrudRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
```

### soft-delete 필터링

- derived query 이름 접미사: `...AndIsDeletedFalse`
- `@Query` SQL: `... AND is_deleted = false`

### deleteById 패턴

```java
@Override
public void deleteById(FooIdentity identity) {
    entityRepository.findById(identity.getFooId())
            .ifPresent(entity -> entityRepository.save(entity.softDelete()));
}
```

- 반환 타입 `void`
- 이미 삭제된 row 는 조용히 no-op
- 404 의미는 Usecase 레이어에서 `findWriteById().orElseThrow(NotFoundException)` 선검증으로 확보

### insert / update 구분

- Spring Data JDBC 의 `CrudRepository.save(entity)` 가 entity 의 `@Id` 필드로 자동 구분
- `from(domain)` 에서 새 레코드는 `domain.get{Domain}Id()` 가 null 이면 insert
- 업데이트는 id 를 유지한 채 `from()`

### DTO 파일 배치

JOIN 결과를 담는 DTO 는 `{Domain}WithXxxDto.java` 네이밍, **EntityRepository 와 같은 패키지에 평면 배치**. `dto/` 서브디렉토리 만들지 않음.

---

## Pattern Foo — 단순 CRUD (derived query 위임)

**언제 쓰나**: 단건 조회·저장·존재 확인만 필요, JOIN/집계 없음.

### infra port

```java
public interface FooRepository {
    Optional<Foo> findById(FooIdentity identity);
    Optional<Foo> findByEmail(String email);
    boolean existsByNickname(String nickname);
    Foo save(Foo foo);
}
```

### EntityRepository

```java
@Repository
interface FooEntityRepository extends CrudRepository<FooEntity, Long> {
    Optional<FooEntity> findByIdAndIsDeletedFalse(Long id);
    Optional<FooEntity> findByEmailAndIsDeletedFalse(String email);
    boolean existsByNicknameAndIsDeletedFalse(String nickname);
}
```

### JdbcRepository

```java
@Repository
@RequiredArgsConstructor
class FooJdbcRepository implements FooRepository {

    private final FooEntityRepository entityRepository;

    @Override
    public Optional<Foo> findById(FooIdentity identity) {
        return entityRepository.findByIdAndIsDeletedFalse(identity.getFooId())
                .map(FooEntity::toDomain);
    }

    @Override
    public Optional<Foo> findByEmail(String email) {
        return entityRepository.findByEmailAndIsDeletedFalse(email)
                .map(FooEntity::toDomain);
    }

    @Override
    public boolean existsByNickname(String nickname) {
        return entityRepository.existsByNicknameAndIsDeletedFalse(nickname);
    }

    @Override
    public Foo save(Foo foo) {
        FooEntity entity = FooEntity.from(foo);
        FooEntity saved = entityRepository.save(entity);
        return saved.toDomain();
    }
}
```

---

## Pattern Bar — Stats 집계 UPDATE (@Modifying @Query)

**언제 쓰나**: 통계 테이블의 카운터 증감. 전체 엔티티 save 대신 부분 UPDATE 로 쓰기 경쟁 최소화.

### infra port

```java
public interface FooStatsRepository {
    Optional<FooStats> findByFooId(FooIdentity identity);
    FooStats save(FooStats stats);
    void increaseViewCount(FooIdentity identity, long increment);
    FooStats addRating(FooIdentity identity, int score);
}
```

### EntityRepository

```java
@Repository
interface FooStatsEntityRepository extends CrudRepository<FooStatsEntity, Long> {

    Optional<FooStatsEntity> findByFooId(Long fooId);

    @Modifying
    @Query("UPDATE foo_stats SET view_count = view_count + :increment WHERE foo_id = :fooId")
    void increaseViewCount(@Param("fooId") Long fooId, @Param("increment") long increment);

    @Modifying
    @Query("UPDATE foo_stats SET rating_sum = rating_sum + :score, rating_count = rating_count + 1 " +
           "WHERE foo_id = :fooId")
    void addRating(@Param("fooId") Long fooId, @Param("score") int score);
}
```

### JdbcRepository

```java
@Repository
@RequiredArgsConstructor
class FooStatsJdbcRepository implements FooStatsRepository {

    private final FooStatsEntityRepository entityRepository;

    @Override
    public Optional<FooStats> findByFooId(FooIdentity identity) {
        return entityRepository.findByFooId(identity.getFooId())
                .map(FooStatsEntity::toDomain);
    }

    @Override
    public FooStats save(FooStats stats) {
        FooStatsEntity entity = FooStatsEntity.from(stats);
        return entityRepository.save(entity).toDomain();
    }

    @Override
    public void increaseViewCount(FooIdentity identity, long increment) {
        entityRepository.increaseViewCount(identity.getFooId(), increment);
    }

    @Override
    public FooStats addRating(FooIdentity identity, int score) {
        entityRepository.addRating(identity.getFooId(), score);
        return findByFooId(identity).orElseThrow();
    }
}
```

**특징**:
- `@Modifying @Query` UPDATE 는 반환값 void (또는 `int` 변경 행 수)
- 변경 후 최신 상태가 필요하면 JdbcRepository 에서 추가 조회로 합성
- Stats 테이블은 audit 4필드 미적용 (schema-references.md §"Stats 예외" 참조)

---

## Pattern Baz — JOIN + Projection DTO

**언제 쓰나**: 조회 시 다른 테이블의 필드(작성자명·집계)나 자식 컬렉션(카테고리·태그)을 함께 가져와 Read 모델을 합성.

### infra port

```java
public interface BazRepository {
    Optional<BazRead> findById(BazIdentity identity);
    List<BazRead> findByUserId(Long userId, int page, int size);
    Optional<Baz> findWriteById(BazIdentity identity);
    Baz save(Baz baz);
    void deleteById(BazIdentity identity);
}
```

### DTO

같은 패키지에 평면 배치.

```java
@Getter
@AllArgsConstructor
class BazWithDetailsDto {
    private Long id;
    private Long userId;
    private String title;
    private String body;
    private Instant createdAt;
    private Instant updatedAt;
    // JOIN 필드
    private String authorNickname;
    private long viewCount;
    private long ratingSum;
    private int ratingCount;
}
```

### EntityRepository

```java
@Repository
interface BazEntityRepository extends CrudRepository<BazEntity, Long> {

    Optional<BazEntity> findByIdAndIsDeletedFalse(Long id);

    @Query("SELECT b.id, b.user_id, b.title, b.body, b.created_at, b.updated_at, " +
           "u.nickname AS authorNickname, " +
           "COALESCE(bs.view_count, 0) AS viewCount, " +
           "COALESCE(bs.rating_sum, 0) AS ratingSum, " +
           "COALESCE(bs.rating_count, 0) AS ratingCount " +
           "FROM bazs b " +
           "LEFT JOIN users u ON b.user_id = u.id " +
           "LEFT JOIN baz_stats bs ON b.id = bs.baz_id " +
           "WHERE b.id = :id AND b.is_deleted = false")
    BazWithDetailsDto findByIdWithDetails(@Param("id") Long id);

    @Query("SELECT b.id, b.user_id, b.title, b.body, b.created_at, b.updated_at, " +
           "u.nickname AS authorNickname, " +
           "COALESCE(bs.view_count, 0) AS viewCount, " +
           "COALESCE(bs.rating_sum, 0) AS ratingSum, " +
           "COALESCE(bs.rating_count, 0) AS ratingCount " +
           "FROM bazs b " +
           "LEFT JOIN users u ON b.user_id = u.id " +
           "LEFT JOIN baz_stats bs ON b.id = bs.baz_id " +
           "WHERE b.is_deleted = false AND b.user_id = :userId " +
           "ORDER BY b.created_at DESC " +
           "LIMIT :limit OFFSET :offset")
    List<BazWithDetailsDto> findByUserIdWithDetails(@Param("userId") Long userId,
                                                    @Param("limit") int limit,
                                                    @Param("offset") int offset);

    // 자식 컬렉션 개별 조회 (ID 배치)
    @Query("SELECT category FROM baz_categories WHERE baz_id = :bazId")
    List<String> findCategoryNamesByBazId(@Param("bazId") Long bazId);
}
```

### JdbcRepository

```java
@Repository
@RequiredArgsConstructor
class BazJdbcRepository implements BazRepository {

    private final BazEntityRepository entityRepository;

    @Override
    public Optional<BazRead> findById(BazIdentity identity) {
        BazWithDetailsDto dto = entityRepository.findByIdWithDetails(identity.getBazId());
        if (dto == null) {
            return Optional.empty();
        }
        List<BazCategory> categories = entityRepository.findCategoryNamesByBazId(dto.getId())
                .stream().map(BazCategory::valueOf).toList();
        return Optional.of(toRead(dto, categories));
    }

    @Override
    public List<BazRead> findByUserId(Long userId, int page, int size) {
        List<BazWithDetailsDto> dtos = entityRepository.findByUserIdWithDetails(
                userId, size, page * size);
        if (dtos.isEmpty()) {
            return List.of();
        }
        // N+1 회피: ID 배치로 자식 묶음 조회 → Map 으로 매핑
        List<Long> ids = dtos.stream().map(BazWithDetailsDto::getId).toList();
        Map<Long, List<BazCategory>> categoriesMap = findCategoriesMap(ids);

        return dtos.stream()
                .map(dto -> toRead(dto, categoriesMap.getOrDefault(dto.getId(), List.of())))
                .toList();
    }

    @Override
    public Optional<Baz> findWriteById(BazIdentity identity) {
        return entityRepository.findByIdAndIsDeletedFalse(identity.getBazId())
                .map(BazEntity::toDomain);
    }

    @Override
    public Baz save(Baz baz) {
        return entityRepository.save(BazEntity.from(baz)).toDomain();
    }

    @Override
    public void deleteById(BazIdentity identity) {
        entityRepository.findById(identity.getBazId())
                .ifPresent(entity -> entityRepository.save(entity.softDelete()));
    }

    private BazRead toRead(BazWithDetailsDto dto, List<BazCategory> categories) {
        double rating = dto.getRatingCount() > 0
                ? (double) dto.getRatingSum() / dto.getRatingCount() : 0.0;
        return new BazRead(dto.getId(), dto.getTitle(), dto.getBody(),
                dto.getAuthorNickname(), categories,
                dto.getViewCount(), rating, dto.getRatingCount(),
                dto.getCreatedAt(), dto.getUpdatedAt());
    }

    private Map<Long, List<BazCategory>> findCategoriesMap(List<Long> ids) {
        // 별도 쿼리 또는 루프 호출로 구현. 실무에선 IN 절 + groupingBy 권장.
        return ids.stream().collect(Collectors.toMap(
                id -> id,
                id -> entityRepository.findCategoryNamesByBazId(id).stream()
                        .map(BazCategory::valueOf).toList()
        ));
    }
}
```

**특징**:
- `@Query` 에서 컬럼 별칭(`AS authorNickname`)은 DTO 필드명과 정확히 매칭
- 리스트 조회 시 **N+1 회피** — 자식 컬렉션을 ID 배치로 묶어 한 쿼리, Java 레벨에서 `groupingBy`
- `BazRead.rating` 처럼 DB 에 없는 파생값은 JdbcRepository `toRead` 에서 계산

---

## Pattern Qux — 트리·순서 연산 (@Modifying 커스텀 UPDATE)

**언제 쓰나**: 댓글·메뉴·카테고리 같은 트리 구조에서 sort_number·child_count 재배치.

### infra port

```java
public interface QuxRepository {
    void incrementSortNumbersAbove(Long articleId, int commentOrder, int sortNumber);
    void incrementChildCount(QuxIdentity parentId);
    int getMaxCommentOrder(Long articleId);
}
```

### EntityRepository

```java
@Repository
interface QuxEntityRepository extends CrudRepository<QuxEntity, Long> {

    @Modifying
    @Query("UPDATE quxs SET sort_number = sort_number + 1 " +
           "WHERE article_id = :articleId AND comment_order = :commentOrder " +
           "AND sort_number > :sortNumber AND is_deleted = false")
    void incrementSortNumbersAbove(@Param("articleId") Long articleId,
                                   @Param("commentOrder") int commentOrder,
                                   @Param("sortNumber") int sortNumber);

    @Modifying
    @Query("UPDATE quxs SET child_count = child_count + 1 WHERE id = :id")
    void incrementChildCount(@Param("id") Long id);

    @Query("SELECT COALESCE(MAX(comment_order), 0) FROM quxs " +
           "WHERE article_id = :articleId AND is_deleted = false")
    int findMaxCommentOrder(@Param("articleId") Long articleId);
}
```

### JdbcRepository

```java
@Repository
@RequiredArgsConstructor
class QuxJdbcRepository implements QuxRepository {

    private final QuxEntityRepository entityRepository;

    @Override
    public void incrementSortNumbersAbove(Long articleId, int commentOrder, int sortNumber) {
        entityRepository.incrementSortNumbersAbove(articleId, commentOrder, sortNumber);
    }

    @Override
    public void incrementChildCount(QuxIdentity parentId) {
        entityRepository.incrementChildCount(parentId.getQuxId());
    }

    @Override
    public int getMaxCommentOrder(Long articleId) {
        return entityRepository.findMaxCommentOrder(articleId);
    }
}
```

**특징**:
- 트리 재배치는 여러 row 에 걸친 UPDATE — `@Modifying` 필수
- 순서 연산은 **Usecase 레이어의 트랜잭션** 안에서 보호 (동시성)

---

## 패턴 선택 가이드

| 도메인 특성 | Pattern |
|---|---|
| 단순 CRUD (파생 쿼리로 충분) | **Foo** |
| 통계 카운터 증감 UPDATE | **Bar** |
| 조회 시 JOIN/집계 필요 | **Baz** |
| 트리·순서 재배치 UPDATE | **Qux** |

대부분 도메인은 Foo. 집계가 있으면 Bar 를 추가. 상세 조회가 있으면 Baz. 트리는 Qux.

---

## 💡 팁 — 위 4개 패턴으로 풀리지 않을 때

1. **Baz + 보조 쿼리 조합을 먼저 재검토** — 대부분 `@Query` JOIN 여러 개 + 자바 레벨 `Map` / `Collectors.groupingBy` 합성으로 표현 가능.
2. **자바 레이어에서 여러 쿼리 조합** — JdbcRepository 메서드 하나 안에서 EntityRepository 의 여러 파생 쿼리 / `@Query` 호출을 순서대로 실행하고 결과를 합성. 예: 부모 DTO 리스트 조회 → ID 배치로 자식 조회 → `Map` 으로 매핑 → Read 모델 조립.
3. **유저에게 질의** — 위 방식으로도 표현이 어색하면 중단하고 유저에게 물어본다. 동적 SQL 도입 / 검색 엔진 이관 (be-add-es-*) / 스키마 재설계 중 어느 방향을 택할지 결정이 필요하다.

---

## 파일 구성 요약

```
repository-jdbc/src/main/java/{packagePath}/jdbc/{domain}/repository/
├── {Domain}JdbcRepository.java       ← port 구현체 (package-private class)
├── {Domain}EntityRepository.java     ← Spring Data JDBC (package-private interface)
├── {Domain}WithXxxDto.java           ← JOIN 결과 DTO (Baz 필요 시)
└── {Domain}Entity.java               ← entity (be-add-entity-specs 가 생성)
```

---

## 공통 금지 사항

- `@Repository` 생략 금지 (Spring 스캔 누락)
- `@Query` / `@Modifying` 을 JPA 패키지에서 import 금지 (`org.springframework.data.jdbc.repository.query.*` 만)
- JdbcRepository 에서 Entity 를 도메인 외부로 누출 금지 — 반환 타입은 항상 도메인 모델
- Read 모델 반환이 필요한데 Entity 만으로 부족하면 JOIN 쿼리 + DTO 사용 (@Entity 확장 금지)
