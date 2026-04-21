---
name: be-add-jdbc-query
description: infra 인터페이스의 구현체(JdbcRepository)와 쿼리(EntityRepository)를 생성/변경한다.
---

# add-jdbc-query

infra 인터페이스의 구현체와 쿼리를 생성/변경한다.

## 입력

### 지시
- 기본적으로 모든 infra 인터페이스를 구현한다.
- 개별 인터페이스 구현 요청이 오는 경우에도 아래의 공통 규칙을 따르시오.

### 참고 문서 (읽어올 것)
- ${CLAUDE_PLUGIN_ROOT}/references/be-refs/query-references.md — 구현 스타일 레퍼런스 (이 스타일을 따를 것)
- ${CLAUDE_PLUGIN_ROOT}/references/be-refs/entity-references.md — Entity 변환 패턴 (toDomain/from)
- specs/repository.md — 기존 쿼리 정보 (프로젝트 루트 기준)
- specs/infrastructure.md — 구현 대상 인터페이스 (프로젝트 루트 기준)
- specs/models.md — 도메인 모델 (반환 타입, VO 구조 참조) (프로젝트 루트 기준)

## 생성 결과물 구조

```
repository-jdbc/src/main/java/dev/{project}/jdbc/{domain}/repository/
├── {Domain}EntityRepository.java  ← Spring Data JDBC (CrudRepository)
├── {Domain}JdbcRepository.java    ← infra 구현체 (Adapter)
└── {Domain}WithUserDto.java       ← JOIN 결과 DTO (필요 시)
```

## 기본 원칙

1. EntityRepository에서 **derived query** 또는 **@Query**를 이용해 쿼리를 작성한다.
2. JdbcRepository에서 이를 **wrapping하거나 간단한 조작**을 통해 외부로 제공하는 것이 목표이다.
3. 가능하면 **최소 조건, 최대한 간단하게** 각 단계의 쿼리를 작성 & 구현하시오.
4. 직접 쿼리(@Query)를 작성하기보다 **derived query를 우선** 사용하시오.

## 생성 규칙

### 1. EntityRepository

```java
@Repository
interface {Domain}EntityRepository extends CrudRepository<{Domain}Entity, Long> {
}
```

**A. Derived Query (기본)**

Spring Data 메서드 네이밍 규칙으로 자동 생성되는 쿼리.
단일 테이블 조회의 기본 수단.

**조회 쿼리에는 반드시 `IsDeletedFalse` 조건을 포함한다.** soft delete된 데이터가 조회되지 않도록 한다.

```java
List<{Domain}Entity> findByUserIdAndIsDeletedFalse(Long userId);
Optional<{Domain}Entity> findByIdAndIsDeletedFalse(Long id);
long countByUserIdAndIsDeletedFalse(Long userId);
```

**B. 원자적 UPDATE (@Modifying @Query)**

카운터 증감 등 특정 컬럼만 변경하는 경우.

```java
@Modifying
@Query("UPDATE {table} SET {column} = {column} + :increment WHERE id = :id")
void increase{Column}(@Param("id") Long id, @Param("increment") long increment);
```

**C. Bulk 조회**

```java
List<{Domain}Entity> findByIdIn(List<Long> ids);
```

### 2. JdbcRepository

```java
@Repository
@RequiredArgsConstructor
class {Domain}JdbcRepository implements {Domain}Repository {
    private final {Domain}EntityRepository entityRepository;
}
```

infra 인터페이스를 implements하고, EntityRepository에 위임한다.
비즈니스 로직을 넣지 않는다. **변환과 위임만.**

**A. Write 메서드 (Domain 반환)**

```java
public {Domain} save({Domain} domain) {
    {Domain}Entity entity = {Domain}Entity.from(domain);
    {Domain}Entity saved = entityRepository.save(entity);
    return saved.toDomain();
}
```

**B. Read 메서드 — 단일 Entity (Domain 반환)**

soft delete 필터가 적용된 EntityRepository 메서드를 사용한다.

```java
public Optional<{Domain}> findById({Domain}Identity identity) {
    return entityRepository.findByIdAndIsDeletedFalse(identity.get{Domain}Id())
            .map({Domain}Entity::toDomain);
}
```

**C. 삭제 메서드 (Soft Delete)**

deleteById는 실제 DELETE가 아닌 soft delete로 구현한다.
Entity의 softDelete() 메서드로 isDeleted/deletedAt을 설정하고 save한다.

```java
public void deleteById({Domain}Identity identity) {
    entityRepository.findById(identity.get{Domain}Id())
            .ifPresent(entity -> {
                {Domain}Entity deleted = entity.softDelete();
                entityRepository.save(deleted);
            });
}
```

**D. 원자적 연산**

Identity VO → Long unwrap 후 EntityRepository에 직접 위임.

```java
public void increase{Column}({Domain}Identity identity, long increment) {
    entityRepository.increase{Column}(identity.get{Domain}Id(), increment);
}
```

### 3. JOIN 쿼리 (여러 도메인에 걸친 조회)

Read 모델에 다른 테이블 필드(nickname 등)가 포함된 경우.

**유저에게 아래 계획을 허락받을 것:**
1. 응답 DTO (`{Domain}WithUserDto`)를 만든다
2. EntityRepository에 @Query LEFT JOIN 메서드를 추가한다
3. JdbcRepository에서 DTO → Read 변환 메서드를 추가한다

**3-1. WithUserDto**

```java
@Getter
@AllArgsConstructor
class {Domain}WithUserDto {
    // Entity 컬럼 전부 (VO는 개별 필드로 분해)
    private Long id;
    private Long userId;
    // ...

    // JOIN 필드
    private String nickname;
}
```

**3-2. @Query 규칙**

- SELECT 컬럼을 **명시**한다 (SELECT * 금지)
- **LEFT JOIN** 사용 (nullable 허용)
- WHERE만 다른 쿼리는 **동일 DTO 재사용**

```java
// 단건 조회
@Query("SELECT t.id, t.user_id, ..., u.nickname " +
       "FROM {table} t " +
       "LEFT JOIN users u ON t.user_id = u.id " +
       "WHERE t.id = :id AND t.is_deleted = false")
{Domain}WithUserDto findByIdWithUser(@Param("id") Long id);

// 목록 조회 (고정 WHERE + 페이징)
@Query("SELECT t.id, t.user_id, ..., u.nickname " +
       "FROM {table} t " +
       "LEFT JOIN users u ON t.user_id = u.id " +
       "WHERE t.user_id = :userId AND t.is_deleted = false " +
       "ORDER BY t.created_at DESC " +
       "LIMIT :limit OFFSET :offset")
List<{Domain}WithUserDto> findByUserIdWithUser(@Param("userId") Long userId,
                                                @Param("limit") int limit,
                                                @Param("offset") int offset);
```

**3-3. toRead 변환 (JdbcRepository 내부)**

```java
private {Domain}Read toRead({Domain}WithUserDto dto) {
    // VO 재구성: DTO의 개별 컬럼을 VO로 조립
    // JOIN 필드 포함
}
```

**Read 반환 메서드 구현:**

```java
public Optional<{Domain}Read> findById({Domain}Identity identity) {
    {Domain}WithUserDto dto = entityRepository.findByIdWithUser(identity.get{Domain}Id());
    return Optional.ofNullable(dto).map(this::toRead);
}
```

### 4. 동적 SQL이 필요한 경우

WHERE 절의 조건 자체가 입력에 따라 있거나 없거나 하는 경우 (검색어 필터, 태그 필터, 정렬 분기 등),
@Query로는 처리할 수 없다.

**이 경우 자체 판단으로 JdbcTemplate을 도입하지 말고, 유저에게 문의한다.**

문의 시 포함할 것:
- 어떤 메서드가 동적 SQL을 필요로 하는지
- 어떤 조건이 동적인지 (optional WHERE, 정렬 분기 등)

> 참고: WHERE 조건이 고정이고 파라미터만 바뀌는 경우는 동적 SQL이 **아니다.**
> `WHERE user_id = :userId`, `LIMIT :limit OFFSET :offset` 등은 @Query로 처리한다.

---

## 기존 변경 시

- as-is/to-be를 유저에게 제시하고 확인받는다
- infra에 메서드 추가되면 JdbcRepository에도 구현 추가
- Entity 필드 변경되면 toRead 변환 메서드도 함께 수정
- 기존 쿼리와 유사한 요청이 오면 병합 여부를 유저에게 문의

## 산출물

- 생성/변경된 EntityRepository, JdbcRepository, WithUserDto 파일 목록
- specs/repository.md 업데이트
- 추가/변경된 스펙 정리본 출력
