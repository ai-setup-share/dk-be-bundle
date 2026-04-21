---
name: be-add-es-core
description: ES 검색 공통 인프라를 점검하고, 없는 컴포넌트만 생성한다. be-add-es-field, be-add-es-search, be-add-es-doc 실행 전에 선행되어야 한다. "ES 공통 인프라 확인", "ES 코어 셋업" 등의 요청에 트리거한다.
---

# be-add-es-core

ES 검색 공통 인프라가 존재하는지 점검하고, 없는 컴포넌트만 생성한다.

## Phase 0. 프로젝트 컨텍스트 해석

`{packageRoot}`, `{esModule}`, `{esPackage}` 플레이스홀더 사용.
실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.
- `{esModule}`: ES 전용 모듈명 (config 또는 질의로 확정)
- `{esPackage}` = `{packageRoot}.{esPackageSuffix}` (예: `{packageRoot}.aisearch`)

## 실행 흐름

1. 아래 체크리스트의 각 파일이 존재하는지 확인
2. 이미 존재하면 **스킵**
3. 없으면 references의 원본 코드를 참고하여 **생성**
4. 결과 리포트 출력 (생성됨 / 이미 존재)

## 체크리스트

대상 모듈: `{esModule}`
기본 패키지: `{esPackage}`

### 쿼리 타입 시스템

| 파일 | 패키지 위치 | 설명 |
|---|---|---|
| `FieldName.java` | `es.queryBuilder` | 필드명 인터페이스 |
| `QueryType.java` | `es.queryBuilder` | 쿼리 타입 enum (TERM, MATCH, RANGE, EXISTS, NESTED) |
| `BoolType.java` | `es.queryBuilder` | Bool 절 타입 enum (MUST, FILTER, SHOULD, MUST_NOT) |
| `SortOption.java` | `es.queryBuilder` | 정렬 옵션 record |

### 검색 요소

| 파일 | 패키지 위치 | 설명 |
|---|---|---|
| `SearchElement.java` | `es.queryBuilder` | 검색 조건 단위 (필드+값/범위, 타입 자동 변환) |
| `SearchCommand.java` | `es.queryBuilder` | 검색 명령 record (conditions + from/to 페이징) |

### 쿼리 빌더

| 파일 | 패키지 위치 | 설명 |
|---|---|---|
| `FieldQueryBuilder.java` | `es.queryBuilder` | TERM/MATCH 빌더 인터페이스 |
| `MatchQueryBuilder.java` | `es.queryBuilder.queryBuilder` | MATCH 구현체 |
| `TermQueryBuilder.java` | `es.queryBuilder.queryBuilder` | TERM 구현체 |
| `RangeQueryBuilder.java` | `es.queryBuilder.queryBuilder` | RANGE 구현체 |

### 쿼리 조합

| 파일 | 패키지 위치 | 설명 |
|---|---|---|
| `QueryWithBoolType.java` | `es.queryBuilder.queryComposer` | Query + BoolType 페어 record |
| `BoolQueryComposer.java` | `es.queryBuilder.queryComposer` | Bool 쿼리 조합기 |
| `GenericSearchQueryBuilder.java` | `es.queryBuilder` | 레지스트리 기반 쿼리 빌드 오케스트레이터 |

### 쿼리 실행

| 파일 | 패키지 위치 | 설명 |
|---|---|---|
| `SearchQueryExecutor.java` | `es.queryBuilder.queryExecutor` | ES 클라이언트 래퍼 (search, searchWithSort, searchByVector, getById) |
| `SearchResult.java` | `es.queryBuilder.queryExecutor` | 결과 wrapper record (docs + totalHits) |

### 유틸리티

| 파일 | 패키지 위치 | 설명 |
|---|---|---|
| `PaginationUtils.java` | `es.utils` | 페이지네이션 (+1 hasNext 패턴) |
| `QueryHelper.java` | `es.utils` | should 절 병합 유틸 |

### 인덱서 (be-add-es-doc과 공유)

| 파일 | 패키지 위치 | 설명 |
|---|---|---|
| `DocBase.java` | `document` | 문서 공통 인터페이스 (getDocId) |
| `DocIndexer.java` | `indexer` | 인덱서 인터페이스 (indexOne) |
| `AbstractDocIndexer.java` | `indexer` | 인덱서 추상 클래스 |
| `IndexResponseType.java` | `indexer` | 인덱스 응답 enum |
| `InstantConverter.java` | `util` | Instant → epoch millis 변환 |

### 설정

| 파일 | 패키지 위치 | 설명 |
|---|---|---|
| `ElasticsearchConfig.java` | `config` | ES 클라이언트 Bean 설정 (RestClient + ObjectMapper) |

### 예외

| 파일 | 패키지 위치 | 설명 |
|---|---|---|
| `DocumentIndexingException.java` | `exception` | 인덱싱 실패 예외 |
| `ElasticsearchQueryException.java` | `exception` | 쿼리 실패 예외 |
| `SearchFieldNotFoundException.java` | `exception` | 필드 미등록 예외 |

## 패키지 구조

```
{esModule}/src/main/java/{esPackage-as-path}/
├── config/
│   └── ElasticsearchConfig.java
├── document/
│   └── DocBase.java
├── indexer/
│   ├── DocIndexer.java
│   ├── AbstractDocIndexer.java
│   └── IndexResponseType.java
├── es/
│   ├── queryBuilder/
│   │   ├── FieldName.java
│   │   ├── QueryType.java
│   │   ├── BoolType.java
│   │   ├── SortOption.java
│   │   ├── SearchElement.java
│   │   ├── SearchCommand.java
│   │   ├── FieldQueryBuilder.java
│   │   ├── GenericSearchQueryBuilder.java
│   │   ├── queryBuilder/
│   │   │   ├── MatchQueryBuilder.java
│   │   │   ├── TermQueryBuilder.java
│   │   │   └── RangeQueryBuilder.java
│   │   ├── queryComposer/
│   │   │   ├── QueryWithBoolType.java
│   │   │   └── BoolQueryComposer.java
│   │   └── queryExecutor/
│   │       ├── SearchQueryExecutor.java
│   │       └── SearchResult.java
│   └── utils/
│       ├── PaginationUtils.java
│       └── QueryHelper.java
├── exception/
│   ├── DocumentIndexingException.java
│   ├── ElasticsearchQueryException.java
│   └── SearchFieldNotFoundException.java
└── util/
    └── InstantConverter.java
```

## 참고 코드

모든 원본 코드는 `references/` 에 있다:
- `references/query-type-system.md` — FieldName, QueryType, BoolType, SortOption
- `references/search-elements.md` — SearchElement, SearchCommand
- `references/query-builders.md` — FieldQueryBuilder, Match/Term/RangeQueryBuilder
- `references/query-composition.md` — QueryWithBoolType, BoolQueryComposer, GenericSearchQueryBuilder
- `references/query-executor.md` — SearchQueryExecutor, SearchResult
- `references/utils.md` — PaginationUtils, QueryHelper
- `references/indexer-infra.md` — DocBase, DocIndexer, AbstractDocIndexer, IndexResponseType
- `references/es-config.md` — ElasticsearchConfig

생성 시 패키지명만 `{esPackage}.*` 으로 변환하고, 코드 구조는 원본을 그대로 따른다.

## 의존성

`{esModule}/build.gradle.kts` 에 elasticsearch-java 의존성이 필요:
```kotlin
implementation("co.elastic.clients:elasticsearch-java:8.12.2")
```

## 산출물

- 존재 여부 체크 결과 테이블
- 새로 생성된 파일 목록
