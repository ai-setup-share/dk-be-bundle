---
name: be-add-entity-specs
description: Entity 클래스와 DDL을 생성/변경한다. 도메인 모델 스펙 또는 별도 지시를 기반으로 작업한다.
---

# add-entity-specs

Entity 클래스와 DDL을 생성/변경한다.

## 입력

### 지시
- 지시받은 내용에 따라 Entity와 DDL을 생성/변경한다.
- model 스펙을 맞추는 요청인 경우, 네이밍을 통해 model 모듈에서 해당 도메인 스펙을 참조한다.

### 참고 문서 (읽어올 것)
- ${CLAUDE_PLUGIN_ROOT}/references/be-refs/entity-references.md — AuditFields, 트리 구조, 예시
- ${CLAUDE_PLUGIN_ROOT}/references/be-refs/schema-references.md — DDL 컨벤션 (공통 컬럼, 타입 규칙, 네이밍)
- specs/schema.md — 기존 DDL (프로젝트 루트 기준)
- specs/repository.md — 기존 Entity 정보 (프로젝트 루트 기준)

## 생성 결과물 구조

단순 도메인:
```
repository-jdbc/src/main/java/{packagePath}/jdbc/{domain}/repository/
└── {Domain}Entity.java            ← AuditFields 구현 + @CreatedDate/@LastModifiedDate 직접 선언 + toDomain/from
```

복합 도메인 (Embedded + Collection):
```
repository-jdbc/src/main/java/{packagePath}/jdbc/{domain}/repository/
├── {Domain}Entity.java
├── embedded/
│   └── {VO}Embeddable.java        ← 복합 VO용 (from/toDomain 포함)
└── collection/
    └── {의미있는이름}.java          ← 1:N 컬렉션용 (@Table 필수)
```

DDL (단일 파일, 도메인별 블록 append — be-init C-5):
```
schema/src/main/resources/local_h2/schema.sql    ← -- === {Domain} === 블록으로 append
```

## 생성 규칙

### 1. Auditing + soft-delete 필드 직접 선언 (AuditFields 구현)

Spring Data JDBC 는 JPA 의 `@MappedSuperclass` 를 지원하지 않으므로 abstract BaseEntity 상속이 불가(필드가 상속/매핑되지 않아 timestamp 가 조용히 null 로 남는다). 대신 각 Entity 가 직접 필드를 선언하고 `AuditFields` 인터페이스를 구현한다 (be-init §4.6 / C-3, C-4).

**모든 Entity 가 가져야 할 필드 4개 (전수 적용)**:
- `Boolean isDeleted` (default false)
- `Instant deletedAt` (nullable)
- `@CreatedDate Instant createdAt`
- `@LastModifiedDate Instant updatedAt`

추가 규칙:
- Entity 는 `{packageRoot}.model.audit.AuditFields` 구현 (createdAt/updatedAt getter 계약 통일).
- @Table("{snake_case_테이블명}") 필수.
- entity-references.md 참고.

### 1-1. soft-delete 사용 방식 (도메인별 선택)

필드 자체는 모든 Entity 가 보유하지만, **사용 방식**은 도메인별로 결정:

- **정석 soft-delete** (Setup/Series 류):
  - `softDelete()` 메서드 추가 → `deleteById` 에서 호출
  - 조회 쿼리에 `IsDeletedFalse` 필터 적용
- **hide/show** (Comment 류):
  - soft-delete 필드는 두되 softDelete 메서드 미제공
  - 별도 플래그(`isHidden`) 로 상태 관리
- **upsert only / 삭제 불가** (Rating 류):
  - 필드만 두고, deleteById 자체를 인프라에 정의하지 않음

### 2. 도메인 → Entity 매핑
1:1 매칭이 되는 도메인이 존재한다면 해당 스펙을 따른다.

- Identity VO → Long (unwrap)
- enum → 그대로 (Spring Data JDBC가 VARCHAR 처리)
- 단순 VO → @Embedded.Nullable 직접 사용
- 복합 VO (변환 로직 필요) → embedded/ 하위에 Embeddable 클래스 생성
- 복합 VO 분해 (LinkedContent 등) → 개별 컬럼으로 분해

### 3. 변환 메서드
- toDomain(): Entity → Domain. VO 재구성 포함.
- from(): static. Domain → Entity. VO 분해 포함.

### 4. 컬렉션 (1:N)
- Entity에 @MappedCollection(idColumn, keyColumn) 어노테이션 필수
- collection/ 하위에 의미 있는 이름으로 클래스 생성
- 컬렉션 클래스에 @Table 필수
- 예: JobTechCategory, InterestedLocation
- 별도 테이블 DDL 함께 생성

### 5. DDL
- schema-references.md 의 컨벤션을 따른다 (타입 규칙, 네이밍)
- 공통 컬럼 4개(`is_deleted`, `deleted_at`, `created_at`, `updated_at`) **모든 테이블에 포함** — 도메인 사용 방식과 무관.
- 단일 `schema.sql` 에 `-- === {Domain} ===` 블록으로 append (be-init C-5).

### 6. 도메인 모델 없이 Entity 생성
- 모니터링, 테스트, 중간 테이블 등 별도 지시로도 생성 가능
- 이 경우 도메인 스펙 대신 지시 내용을 따른다
- Auditing 필드 직접 선언 규칙 (§1), DDL 규칙 (§5) 은 동일하게 적용

## 기존 변경 시

- as-is/to-be를 유저에게 제시하고 확인받는다
- Entity 필드 추가 시 DDL 컬럼, toDomain/from 메서드도 함께 변경

## 산출물

- 생성/변경된 Entity, Embeddable, Collection, DDL 파일 목록
- specs/schema.md, specs/repository.md 업데이트
- 추가/변경된 스펙 정리본 출력
