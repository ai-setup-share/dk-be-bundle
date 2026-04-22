# dk-be-bundle — Spring Boot Hexagonal BE Skill Bundle

20개 스킬로 **API 1개 단위 종단 생성** 파이프라인을 오케스트레이션하는 Claude Code 플러그인.
Spring Boot 3 + Java 21 + Hexagonal + Spring Data JDBC + H2 스택 프로젝트용.

**Core idea:** API 스펙 1개 → top-down 평가(assess) → 역순 구현(add) → 테스트 생성.
각 레이어(api / usecase / infrastructure / model) 를 개별 스킬이 담당하고 `be-planner` 가 오케스트레이션한다.

---

## MANDATORY: 시작 전 확인

1. **프로젝트 스캐폴드부터** — 빈 상태이면 `dk-be-bundle:be-init` 먼저 실행. 기존 프로젝트라도 이 번들의 컨벤션(AuditFields, DomainException/HttpException, local_h2/schema.sql 등) 과 맞는지 확인.
2. **`dk-be-bundle.config.json`** 을 프로젝트 루트에 두면 자동 탐지 우선한다. 없으면 스킬이 질의.
3. **be-planner 는 API 1개 단위** — 여러 API 를 한 번에 주지 않는다.
4. **각 스킬은 참고용 레퍼런스를 읽는다** — `references/be-refs/*.md` 에 foo/bar/baz/qux 4~5 패턴이 정리돼 있다. 절대 규칙은 아니지만 결과물 톤을 맞추기 위한 기본값.

---

## 설치

```bash
# 로컬 테스트 (플러그인 경로 지정 실행)
claude --plugin-dir /path/to/dk-be-bundle

# 개인 영구 설치
claude plugin install /path/to/dk-be-bundle --scope user

# 프로젝트 공유 (팀 .claude/settings.json 에 기록)
claude plugin install /path/to/dk-be-bundle --scope project
```

---

## 스킬 목록 (20개)

모든 스킬은 `dk-be-bundle:` 네임스페이스 로 호출.

| 그룹 | 스킬 | 용도 |
|---|---|---|
| **오케스트레이션** | `be-planner` | API 스펙 1개 → assess 체인 → 구현 계획 → 역순 실행 |
| **스캐폴드** | `be-init` | 빈 멀티모듈 프로젝트 생성 (8 모듈 + gradle wrapper + 공통 클래스 + 테스트 인프라) |
| **평가 (top-down)** | `be-assess-api` | API 레이어 충족 여부 평가 |
| | `be-assess-usecase` | usecase 레이어 평가 |
| | `be-assess-infra` | infrastructure port 평가 |
| | `be-assess-model` | 도메인 모델 평가 |
| **구현 (bottom-up)** | `be-add-model` | 도메인 모델 (`{Domain}` / `{Domain}Read` / `{Domain}Identity`) |
| | `be-add-infrastructure` | Repository port 인터페이스 |
| | `be-add-entity-specs` | Entity 클래스 + DDL append |
| | `be-add-jdbc-query` | JdbcRepository + EntityRepository |
| | `be-add-usecase` | Reader / Writer / Usecase |
| | `be-add-api` | Controller + Request/Response DTO |
| **Elasticsearch** | `be-add-es-core` | ES 공통 인프라 (선행 필수) |
| | `be-add-es-field` | IndexField + QueryBuilderRegistry |
| | `be-add-es-doc` | Document + Mapper + Indexer |
| | `be-add-es-search` | Search 컴포넌트 + SearchResult |
| | `be-extract-target-audiences` | 문서 target_audiences LLM 추출 |
| **테스트** | `be-test-api` | Controller 단위 테스트 (Mockito) |
| | `be-test-service` | Reader/Writer/Usecase 단위 테스트 (Mockito) |
| | `be-test-repository-jdbc` | JdbcRepository 통합 테스트 (H2 mem 슬라이스) |
| | `be-test-service-concurrency` | Writer 동시성/순서 통합 테스트 |

---

## Quick Start

### 새 프로젝트

```bash
# 1. 빈 스캐폴드 생성
/dk-be-bundle:be-init
# projectName, group, packageRoot, projectRoot 를 대화로 받음

# 2. 첫 도메인 종단 생성 (예: Article CRUD)
/dk-be-bundle:be-planner
# API 스펙 1개 제시 → assess 체인 → 구현 계획 확인 → 역순 실행

# 3. 테스트 추가
/dk-be-bundle:be-test-repository-jdbc
/dk-be-bundle:be-test-service
/dk-be-bundle:be-test-api
```

### 기존 프로젝트 (be-init 이미 실행된 상태)

```bash
# API 추가
/dk-be-bundle:be-planner

# 개별 레이어만 수정
/dk-be-bundle:be-add-usecase
/dk-be-bundle:be-add-api
```

---

## be-planner — 오케스트레이터

3-phase 흐름:

**Phase 1: Assess (top-down)** — `assess-api` → `assess-usecase` → `assess-infra` → `assess-model`
각 레이어에서 요구 API 가 현재 코드로 충족되는지 판정. 충족된 레이어에서 체인 중단.

**Phase 2: Plan** — assess 결과를 모아 구현 계획 수립. 유저에게 제시하고 확정.

**Phase 3: Add (bottom-up)** — `add-model` → `add-infrastructure` → `add-entity-specs` → `add-jdbc-query` → `add-usecase` → `add-api`
미충족 레이어만 실행. 각 단계에서 specs doc 에 산출물 기록.

**산출물**: `specs/plans/{api-name}.md` (plan doc). 각 phase 의 결과를 누적 기록.

---

## be-init — 스캐폴드

**What it does:**

1. 8 모듈 (`api-application`, `api`, `service`, `repository-jdbc`, `infrastructure`, `model`, `schema`, `exception`) + Gradle wrapper 8.5 생성
2. `Application.java` + `@EnableJdbcAuditing`
3. `AuditFields` 인터페이스 (audit 계약)
4. `DomainException` (abstract) + `HttpException` (interface) + 구체 4종 (`EntityNotFoundException`/`ConflictException`/`ForbiddenException`/`InvalidInputException`)
5. `GlobalExceptionHandler` + `ErrorResponse` (api-application/config/)
6. `IntegrationTestBase` — repository-jdbc 슬라이스 / service 통합 각각
7. `TestApplication.java` (+ `@EnableJdbcAuditing`) — slice 테스트 부트스트랩
8. `application.yml` / `application-local.yml` (H2 file) / `application-test.yml` (H2 mem)
9. `schema.sql` (주석만, `local_h2/` 하위)
10. `IntegrationConfig.java` — service 통합 테스트용 SpringBootTest 구성

생성 직후 검증 명령을 실행하고, 실패 시 원인 분류표 기반으로 최대 3회 자가 수정.

**Usage:**

```
/dk-be-bundle:be-init
```

입력 받는 4개:
- `projectName` — 예: `my-service-be`
- `group` — Gradle group, 예: `com.example.board`
- `packageRoot` — Java 패키지 루트 (group 과 달라도 됨)
- `projectRoot` — 생성 위치 절대 경로

---

## References — Foo / Bar / Baz / Qux 패턴

각 add-* 스킬은 `references/be-refs/*.md` 를 읽는다. 모든 레퍼런스는 같은 구조:

- **공통 요소** — 네이밍, 어노테이션, 파일 배치
- **Pattern Foo / Bar / Baz / Qux (/ Quux)** — 4~5 패턴을 foo/bar 추상 이름으로 제시
- **패턴 선택 가이드** — 도메인 특성 → 어떤 패턴
- **팁** — 위 패턴으로 안 풀리면 유저에게 질의
- **공통 금지 사항**

| 레이어 | 레퍼런스 | 패턴 |
|---|---|---|
| model | `be-refs/model-references.md` | Foo(단순) / Bar(R+W 분리) / Baz(JOIN+집계) / Qux(상태전이·트리) |
| entity | `be-refs/entity-references.md` | Foo / Bar(Stats 분리) / Baz(@MappedCollection) / Qux(hide/show) |
| schema | `be-refs/schema-references.md` | 공통 컬럼 + 타입 매핑 + Stats 예외 |
| jdbc-query | `be-refs/query-references.md` | Foo(CRUD) / Bar(Stats UPDATE) / Baz(JOIN DTO) / Qux(@Modifying 트리) |
| service | `be-refs/service-references.md` | Foo(Reader만) / Bar(R+W) / Baz(Usecase) / Qux(Usecase 단독) / Quux(ViewMemory) |
| infrastructure | `be-refs/infrastructure-references.md` | Foo(단순) / Bar(R+W 분리) / Baz(Stats 분리) / Qux(트리) |
| api | `be-refs/api-references.md` | Foo(RESTful CRUD) / Bar(Action) / Baz(공개 조회) / Qux(마이페이지) |

**참고용 자료** 로서의 성격 (절대 규칙 아님). 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트를 한 번 더 탐색 후 필요한 변형을 적용한다. 여러 레퍼런스로 해결 가능하면 가장 간단한 방식을 택한다.

---

## 프로젝트 설정 (`dk-be-bundle.config.json`)

스킬이 자동 탐지 (build.gradle.kts 의 `group`, `modules/` 스캔) 하지만 명시 설정이 있으면 우선 사용.

프로젝트 루트에 배치:

```json
{
  "packageRoot": "com.example.board",
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
해석 규칙: [`references/plugin-config-resolver.md`](references/plugin-config-resolver.md)

---

## be-init Downstream 계약 (C-1 ~ C-9)

be-init 가 고정하는 규약. 다른 스킬은 이 계약 위에서 동작한다.

| # | 내용 |
|---|---|
| C-1 | 모듈 `include` 는 be-init 가 고정. downstream 은 `settings.gradle.kts` 를 수정하지 않는다. |
| C-2 | 패키지 루트는 `{packageRoot}`. downstream 은 하위 세그먼트만 추가한다. |
| C-3 | audit 필드명 `createdAt` / `updatedAt`, 타입 `Instant`. 각 엔티티가 `@CreatedDate` / `@LastModifiedDate` 로 **직접 선언**. |
| C-4 | 엔티티는 `AuditFields` 인터페이스를 구현 (옵션이지만 권장). |
| C-5 | 스키마 위치: `modules/schema/src/main/resources/local_h2/schema.sql` 단일. 도메인별 DDL 은 `-- === {Domain} ===` 블록으로 append. |
| C-6 | 도메인 예외는 `DomainException` extend + `HttpException` implement. `httpStatusCode()` / `httpErrorMessage()` 노출. 404 계열은 `EntityNotFoundException` 의 하위 클래스. |
| C-7 | `GlobalExceptionHandler` 는 be-init 소유. 추가 핸들러는 **별도 `@RestControllerAdvice`** 로 분리. |
| C-8 | 테스트 베이스는 be-init 제공 `IntegrationTestBase` extend. 추가 Bean 은 `@ComponentScan(basePackages = {...})` 또는 `@Import` 로 로드. |
| C-9 | 프로파일 `local`(H2 file) / `test`(H2 mem). 추가 프로파일은 별도 yml. |

---

## 호환 스택

| 항목 | 버전 / 값 |
|---|---|
| Gradle | 8.5 (Kotlin DSL) |
| Spring Boot | 3.2.1 |
| Java | 21 |
| io.spring.dependency-management | 1.1.4 |
| Lombok | 1.18.30 |
| JUnit BOM | 5.14.0 |
| Mockito BOM | 5.20.0 |
| AssertJ | 3.26.3 |
| Springdoc OpenAPI | 2.3.0 |
| DB | H2 (local file + test mem, `MODE=MySQL;CASE_INSENSITIVE_IDENTIFIERS=TRUE;DATABASE_TO_LOWER=TRUE`) |
| Data | Spring Data JDBC (JPA 아님) |
| ES (선택) | elasticsearch-java 8.x |

구조가 다른 프로젝트는 config 로 경로만 맞추면 대부분 동작. 다만 도메인 관례(Reader/Writer 분리, AuditFields 매핑, port-adapter 의존 방향)는 변경하지 않는다.

---

## 파일 레이아웃 (생성 후)

```
{projectRoot}/
├── settings.gradle.kts
├── build.gradle.kts              # java toolchain 21 + -parameters + BOM imports
├── gradle/wrapper/               # 8.5
├── dk-be-bundle.config.json      # (선택)
└── modules/
    ├── applications/api-application/
    │   └── src/main/java/{packagePath}/
    │       ├── Application.java              # @SpringBootApplication @EnableJdbcAuditing
    │       └── application/config/
    │           ├── GlobalExceptionHandler.java
    │           └── ErrorResponse.java
    ├── api/                                   # Controller + dto
    ├── service/                               # Reader/Writer/Usecase + dto/impl/usecase
    ├── repository-jdbc/                       # Entity + JdbcRepository + EntityRepository
    ├── infrastructure/                        # Repository port interfaces
    ├── model/                                 # 도메인 모델 + AuditFields interface
    ├── schema/
    │   └── src/main/resources/local_h2/schema.sql
    └── exception/                             # DomainException + HttpException + 4종
```

---

## 참고 문서

- [`references/be-refs/`](references/be-refs/) — 도메인/엔티티/쿼리/스키마/서비스/인프라/API 패턴 레퍼런스
- [`references/be-test-refs/`](references/be-test-refs/) — 테스트 패턴 레퍼런스
- [`references/plugin-config-resolver.md`](references/plugin-config-resolver.md) — 프로젝트 컨텍스트 자동 해석 규칙

---

## 라이선스

MIT
