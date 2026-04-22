# Schema References

add-entity-specs 스킬이 DDL 생성 시 참고하는 레퍼런스.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

## 파일 구조

DDL은 H2(로컬/테스트)와 MySQL(프로덕션) 두 벌을 관리한다.

```
schema/src/main/resources/
├── schema.sql              ← H2 (로컬/테스트)
└── mysql/
    └── schema.sql          ← MySQL (프로덕션)
```

테이블을 추가/변경할 때 **양쪽 모두 반영**해야 한다.

## H2 vs MySQL 차이점

| | H2 | MySQL |
|---|---|---|
| Timestamp | `TIMESTAMP WITH TIME ZONE NOT NULL` | `TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP` |
| Updated | 애플리케이션에서 설정 | `ON UPDATE CURRENT_TIMESTAMP` |
| deleted_at | `TIMESTAMP WITH TIME ZONE` | `TIMESTAMP NULL` |

## 공통 컬럼

모든 테이블에 아래 4개 컬럼을 포함한다. **사용 방식**(softDelete 메서드 추가 / `IsDeletedFalse` 쿼리 사용)만 도메인별로 결정.

**H2:**
```sql
is_deleted BOOLEAN                  NOT NULL DEFAULT FALSE,
deleted_at TIMESTAMP WITH TIME ZONE,
created_at TIMESTAMP WITH TIME ZONE NOT NULL,
updated_at TIMESTAMP WITH TIME ZONE NOT NULL
```

**MySQL:**
```sql
is_deleted BOOLEAN   NOT NULL DEFAULT FALSE,
deleted_at TIMESTAMP NULL,
created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
```

- `created_at`, `updated_at`: Entity 의 `@CreatedDate` / `@LastModifiedDate` 가 채운다.
- `is_deleted`, `deleted_at`: 모든 테이블이 보유. 도메인별 사용 방식:
  - **정석 soft-delete** (Setup/Series 류): `deleteById` 에서 `softDelete()` 호출, 조회 시 `IsDeletedFalse` 필터.
  - **hide/show** (Comment 류): soft-delete 필드는 두되 별도 플래그(`isHidden`)로 상태 관리.
  - **upsert only** (Rating 류): 삭제 자체를 지원하지 않음.
- `deleted_at`은 삭제 시점 기록용. 삭제 전에는 NULL.

### 예외: Stats 분리 테이블 (Pattern Bar)

분리된 통계 테이블(`*_stats`)은 audit/soft-delete 4 컬럼을 **포함하지 않는다**:

- `id` (PK) + `{domain}_id` (FK) + 집계 컬럼들만 보유.
- 이유: 통계는 본 도메인과 생애주기 동일(본 도메인 삭제 시 cascade/stats 고아 처리 별도), 독자 audit 의미 없음.
- 쓰기 경쟁이 잦아 최소 컬럼 유지가 락 경합 최소화에 유리.

예:
```sql
CREATE TABLE setup_stats (
    id             BIGINT AUTO_INCREMENT PRIMARY KEY,
    setup_id       BIGINT NOT NULL,
    view_count     BIGINT NOT NULL DEFAULT 0,
    rating_sum     BIGINT NOT NULL DEFAULT 0,
    rating_count   INT    NOT NULL DEFAULT 0,
    comment_count  INT    NOT NULL DEFAULT 0
);
```

### 자식 컬렉션 테이블 (@MappedCollection 대상)

Pattern Baz 의 자식 테이블 — 별도 컨벤션:
- PK 는 부모 FK 와 자식 값의 복합 or 독자 PK
- audit/soft-delete 컬럼 **미포함** (부모와 생애주기 동일)
- 예: `setup_categories (setup_id, category)` — PK (setup_id, category).

## 타입 매핑

| Java | H2 | MySQL |
|------|-----|-------|
| String | VARCHAR(255) 또는 TEXT | VARCHAR(255) 또는 TEXT |
| Long | BIGINT | BIGINT |
| Boolean | BOOLEAN | BOOLEAN |
| Instant | TIMESTAMP WITH TIME ZONE | TIMESTAMP |
| enum | VARCHAR(255) | VARCHAR(255) |

## 네이밍 규칙

- 테이블명: **소문자 + 복수형** (예: `users`, `community_posts`)
- 컬럼명: **snake_case** (예: `user_id`, `created_at`)

## 제약조건 규칙

- **FK 제약조건 미사용** — 참조 무결성은 애플리케이션 레벨에서 관리
- optional 필드 → NULL 허용
- required 필드 → NOT NULL
- 카운터 필드 → NOT NULL DEFAULT 0

## DDL 예시

**H2 (schema.sql):**
```sql
CREATE TABLE IF NOT EXISTS examples (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(255)             NOT NULL,
    is_deleted BOOLEAN                  NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL
);
```

**MySQL (mysql/schema.sql):**
```sql
CREATE TABLE IF NOT EXISTS examples (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    is_deleted BOOLEAN      NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMP    NULL,
    created_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```
