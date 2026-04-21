# Query References

add-jdbc-query 스킬이 참고하는 레퍼런스.

## 기본 예시 (단일 Entity)

infra 인터페이스 2개 메서드를 구현하는 최소 예시.

### infra 인터페이스 (구현 대상)

```java
public interface ExampleRepository {
    Optional<Example> findById(ExampleIdentity identity);
    Example save(Example example);
}
```

### EntityRepository

```java
@Repository
interface ExampleEntityRepository extends CrudRepository<ExampleEntity, Long> {
    // derived query 불필요 — CrudRepository 기본 메서드로 충족
}
```

### JdbcRepository

```java
@Repository
@RequiredArgsConstructor
class ExampleJdbcRepository implements ExampleRepository {
    private final ExampleEntityRepository entityRepository;

    @Override
    public Optional<Example> findById(ExampleIdentity identity) {
        return entityRepository.findById(identity.getExampleId())
                .map(ExampleEntity::toDomain);
    }

    @Override
    public Example save(Example example) {
        ExampleEntity entity = ExampleEntity.from(example);
        ExampleEntity saved = entityRepository.save(entity);
        return saved.toDomain();
    }
}
```

## JOIN 예시 (Read 모델 반환)

Read 모델에 다른 테이블 필드가 포함되는 경우.

### infra 인터페이스 (구현 대상)

```java
public interface ExampleRepository {
    Optional<ExampleRead> findById(ExampleIdentity identity);
    Example save(Example example);
}
```

### WithUserDto

```java
@Getter
@AllArgsConstructor
class ExampleWithUserDto {
    private Long id;
    private Long userId;
    private String title;
    private Boolean isDeleted;
    private Instant createdAt;
    private Instant updatedAt;
    // JOIN 필드
    private String nickname;
}
```

### EntityRepository

```java
@Repository
interface ExampleEntityRepository extends CrudRepository<ExampleEntity, Long> {

    @Query("SELECT e.id, e.user_id, e.title, e.is_deleted, e.created_at, e.updated_at, u.nickname " +
           "FROM examples e " +
           "LEFT JOIN users u ON e.user_id = u.id " +
           "WHERE e.id = :id")
    ExampleWithUserDto findByIdWithUser(@Param("id") Long id);
}
```

### JdbcRepository

```java
@Repository
@RequiredArgsConstructor
class ExampleJdbcRepository implements ExampleRepository {
    private final ExampleEntityRepository entityRepository;

    @Override
    public Optional<ExampleRead> findById(ExampleIdentity identity) {
        ExampleWithUserDto dto = entityRepository.findByIdWithUser(identity.getExampleId());
        return Optional.ofNullable(dto).map(this::toRead);
    }

    @Override
    public Example save(Example example) {
        ExampleEntity entity = ExampleEntity.from(example);
        ExampleEntity saved = entityRepository.save(entity);
        return saved.toDomain();
    }

    private ExampleRead toRead(ExampleWithUserDto dto) {
        return new ExampleRead(
                dto.getId(),
                dto.getUserId(),
                dto.getNickname(),
                dto.getTitle(),
                dto.getIsDeleted(),
                dto.getCreatedAt(),
                dto.getUpdatedAt()
        );
    }
}
```
