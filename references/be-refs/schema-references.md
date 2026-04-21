# Schema References

add-entity-specs 스킬이 DDL 생성 시 참고하는 레퍼런스.

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

## 공통 컬럼 (BaseEntity 매핑)

모든 테이블에 아래 컬럼을 포함한다.

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

- `is_deleted`, `deleted_at`은 soft delete 전용. 모델(도메인)에는 노출하지 않는다.
- `deleted_at`은 삭제 시점 기록용. 삭제 전에는 NULL.

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
