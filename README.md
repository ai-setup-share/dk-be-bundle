# dk-be-bundle

Spring Boot 3 + Java 21 + Hexagonal Architecture 프로젝트를 위한 Claude Code 백엔드 스킬 번들.

API 1개 단위로 top-down 평가(assess) → bottom-up 구현(add) 파이프라인을 오케스트레이션한다. Elasticsearch 인덱싱 스킬과 테스트 스캐폴딩 스킬도 포함.

## 설치

```bash
# 로컬 테스트
claude --plugin-dir /path/to/dk-be-bundle

# 개인 영구 설치
claude plugin install /path/to/dk-be-bundle --scope user

# 프로젝트 공유 (팀 .claude/settings.json에 기록)
claude plugin install /path/to/dk-be-bundle --scope project
```

## 스킬 목록 (20개)

### 오케스트레이터
- `dk-be-bundle:be-planner` — API 스펙 → 평가 체인 → 구현 계획 → 역순 실행

### 평가 (top-down)
- `dk-be-bundle:be-assess-api` — API 레이어
- `dk-be-bundle:be-assess-usecase` — usecase 레이어
- `dk-be-bundle:be-assess-infra` — infrastructure 레이어
- `dk-be-bundle:be-assess-model` — model 레이어

### 구현 (bottom-up)
- `dk-be-bundle:be-add-model` — 도메인 모델
- `dk-be-bundle:be-add-infrastructure` — Repository 인터페이스
- `dk-be-bundle:be-add-entity-specs` — Entity + DDL
- `dk-be-bundle:be-add-jdbc-query` — JdbcRepository + @Query
- `dk-be-bundle:be-add-usecase` — Reader/Writer/Usecase
- `dk-be-bundle:be-add-api` — Controller + DTO

### Elasticsearch
- `dk-be-bundle:be-add-es-core` — 공통 인프라 (선행)
- `dk-be-bundle:be-add-es-field` — IndexField + QueryBuilderRegistry
- `dk-be-bundle:be-add-es-doc` — Document + Mapper + Indexer
- `dk-be-bundle:be-add-es-search` — Search 컴포넌트 + SearchResult
- `dk-be-bundle:be-extract-target-audiences` — 문서 target_audiences LLM 추출

### 테스트
- `dk-be-bundle:be-test-api` — Controller 단위 테스트 (Mockito)
- `dk-be-bundle:be-test-service` — Usecase/Reader/Writer 단위 테스트
- `dk-be-bundle:be-test-repository-jdbc` — JdbcRepository 통합 테스트 (H2)
- `dk-be-bundle:be-test-service-concurrency` — Writer 동시성/순서 통합 테스트

## 프로젝트 설정 (`dk-be-bundle.config.json`)

스킬이 프로젝트 구조(패키지, 모듈, ES 설정)를 자동 탐지하지만, 명시적 설정이 있으면 우선 사용한다.

프로젝트 루트에 `dk-be-bundle.config.json`:

```json
{
  "packageRoot": "share.ai.setup",
  "modulesPath": "modules",
  "modules": {
    "api": "modules/api",
    "service": "modules/service",
    "repository-jdbc": "modules/repository-jdbc",
    "schema": "modules/schema",
    "model": "modules/model",
    "infrastructure": "modules/infrastructure"
  },
  "es": {
    "module": "ai-search",
    "packageSuffix": "aisearch",
    "indexName": "setup",
    "host": "localhost:9200"
  }
}
```

모든 필드 선택. 빠진 값은 자동 탐지 → 실패 시 유저에게 질의.

자세한 해석 규칙은 [references/plugin-config-resolver.md](references/plugin-config-resolver.md) 참조.

## 전제 조건 (프로젝트 구조)

이 번들은 아래 구조를 전제로 한다:

- **아키텍처**: Hexagonal (api/usecase/infrastructure/model 분리)
- **빌드**: Gradle Kotlin DSL (`build.gradle.kts`)
- **DB**: JDBC (Spring Data JDBC), Entity = `BaseEntity` 상속, `@Query` 우선
- **테스트 DB**: H2 mem + `MODE=MySQL`
- **ES** (선택): elasticsearch-java 8.x

구조가 다른 프로젝트는 config로 경로만 맞추면 대부분 동작하지만, 도메인 관례(Reader/Writer, BaseEntity 등)는 변경하지 않는다.

## 참고 문서

- `references/be-refs/` — 도메인/엔티티/쿼리/스키마 참조
- `references/be-test-refs/` — 테스트 패턴 레퍼런스
- `references/plugin-config-resolver.md` — 프로젝트 컨텍스트 해석

## 라이선스

MIT
