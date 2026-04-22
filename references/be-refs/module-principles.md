# BE 모듈 원칙과 목적 (RDBMS 기준)

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

## 본질

**모듈을 왜 나누는가?**
→ 제작/수정에서 케어의 범위를 좁히기 위해서.

---

## 목적

### 사람적 이유: 인지의 범위를 제한
- 이 모듈에서는 이것만 신경 쓰면 된다
- 도메인마다 동일한 패턴이므로, 한 도메인만 이해하면 나머지는 공짜

### 기계적 이유

#### 1. 레퍼런스의 범위를 좁힌다
기능을 제작하기 위해 탐색해야 하는 코드베이스의 범위를 모듈로 제한.
"A 모듈은 B, C만 참조" → 스캔해야 하는 코드를 줄인다.

#### 2. 컨텍스트에 레퍼런스되는 양을 줄인다
구현체 대신 인터페이스만 읽으면 된다.
모듈 경계 = LLM이 읽어야 할 컨텍스트의 경계.

#### 3. 각 모듈의 동작과 검증의 범위를 의도적으로 narrow하게 만든다
모듈이 수행하는 것이 좁고 명확하면, 스킬로 자동화할 수 있다.
패턴이 고정되어 있으므로, 도메인명과 필드 스펙만 주면 전 계층을 생성 가능.

---

## 모듈 분리의 근거

### 기본 Layered 분리: api / usecase / infra

각 계층이 케어하는 것이 다르고, 변경 이유가 다르다.

```
api:      "HTTP 요청을 어떻게 받고 응답하나" 만 케어
usecase:  "비즈니스 규칙이 뭔가" 만 케어
infra:    "데이터를 어떻게 저장/조회하나" 만 케어
```

### infra의 인터페이스 분리 (infrastructure 모듈)

infra는 레퍼런스하는 쪽이 여러 개이므로 인터페이스를 별도 모듈로 분리한다.

```
usecase (API용 비즈니스 로직) → infrastructure 인터페이스
batch task (크롤링, 동기화)   → infrastructure 인터페이스
outbox (이벤트 처리)          → infrastructure 인터페이스
```

소비자가 많을수록 인터페이스 분리의 이득이 커진다.
구현체(repository-jdbc)의 내부 변경이 소비자에게 전파되지 않는다.

---

## 모듈 구조 (RDBMS 기본 흐름)

```
        model, exception (공유 기반)
              ↑
        infrastructure (인터페이스)
         ↑          ↑
   usecase     repository-jdbc
      ↑
     api               schema
```

---

## 각 모듈의 원칙

### model
- **역할**: 도메인의 본질을 표현
- **규칙**:
  - @Value (Immutable Value Object)
  - 상태 변경은 새 인스턴스 반환
  - 팩토리 메서드로 생성 (생성자 직접 호출 지양)
  - 외부 의존성 없음 (순수 Java)
- **포함하는 것**: 도메인 모델, Identity VO, enum, 공유 VO(Popularity 등)
- **포함하지 않는 것**: DB 매핑, 프레임워크 어노테이션

### exception
- **역할**: 도메인별 예외 정의
- **규칙**:
  - 공통 상위 클래스 상속 (BadRequestException 등)
  - 도메인별 패키지 분리
- **포함하는 것**: 도메인 특화 예외
- **포함하지 않는 것**: 비즈니스 로직, 에러 핸들링

### infrastructure
- **역할**: 데이터 접근 계층의 인터페이스 (Port)
- **규칙**:
  - 인터페이스만 존재, 구현 없음
  - 읽기 메서드 → Read 모델 반환
  - 쓰기 메서드 → Write 모델 반환
  - 원자적 연산은 별도 메서드 (increaseViewCount 등)
  - model과 exception만 의존
- **포함하는 것**: Repository 인터페이스
- **포함하지 않는 것**: 구현체, DB 특화 코드, 쿼리

### repository-jdbc
- **역할**: infrastructure 인터페이스의 JDBC 구현 (Adapter)
- **규칙**:
  - infrastructure 인터페이스를 implements
  - Entity ↔ Domain 변환은 이 모듈 안에서
  - VO 분해/재구성 (LinkedContent → jobId, commentId 컬럼)
  - @Embedded로 VO flatten (Popularity)
  - Spring Data JDBC 사용 (CrudRepository)
  - 원자적 UPDATE는 @Query로 직접 작성
- **포함하는 것**: Entity, EntityRepository, JdbcRepository, JOIN용 DTO
- **포함하지 않는 것**: 비즈니스 로직

### usecase
- **역할**: 비즈니스 로직 (애플리케이션 유스케이스)
- **규칙**:
  - CQRS: Reader(조회) / Writer(명령) 분리
  - infrastructure의 Repository 인터페이스만 의존 (구현체 모름)
  - @Transactional은 Writer에만
  - 크로스 도메인 참조는 다른 도메인의 infrastructure 인터페이스를 통해
- **포함하는 것**: Reader/Writer, Command DTO
- **포함하지 않는 것**: Controller 로직, DB 접근 코드

### api
- **역할**: REST API 엔드포인트
- **규칙**:
  - usecase의 Reader/Writer만 의존
  - Request → Command 변환은 Controller에서
  - Response는 static from() 팩토리 메서드
  - @AuthenticationPrincipal로 인증 정보 수신
  - 비즈니스 로직 없음, 위임만
- **포함하는 것**: Controller, Request DTO, Response DTO
- **포함하지 않는 것**: 비즈니스 로직, DB 접근

### schema
- **역할**: DB 테이블 정의
- **규칙**:
  - DDL만 포함 (CREATE TABLE)
  - enum은 VARCHAR로 저장
  - VO는 flatten (Popularity → view_count, like_count, ...)
  - soft delete (is_deleted)
  - audit 필드 (created_at, updated_at)
- **포함하는 것**: SQL DDL
- **포함하지 않는 것**: DML, 마이그레이션 스크립트

---

## 의존성 규칙

```
허용되는 의존 방향:
  api → usecase, model, exception
  usecase → infrastructure, model, exception
  repository-jdbc → infrastructure, model, exception
  infrastructure → model, exception

금지:
  usecase → repository-jdbc (구현체 직접 의존)
  api → infrastructure (계층 건너뛰기)
  model → 다른 모듈 (순수 도메인)
```

---

## 스펙 문서 (계층별)

플래너가 견적을 내고, 각 스킬이 작업할 때 참조하는 문서.
계층별로 분리하여 각 스킬이 자기 계층 문서만 읽으면 된다.

```
specs/
├── models.md           ← add-model 스킬이 참조
├── infrastructure.md   ← add-infrastructure 스킬이 참조
├── schema.md           ← add-schema 스킬이 참조
├── repository.md       ← add-repository 스킬이 참조
├── usecase.md          ← add-usecase 스킬이 참조
└── api.md              ← add-api 스킬이 참조
```

- 플래너: 전체 specs/를 훑어서 기존 상태 파악 → 견적
- 스킬: 자기 계층 문서만 읽고 작업 → 케어 범위 제한
- 작업 완료 후: 해당 계층 문서 업데이트

---

## 스킬 구조

### 스킬 종류: assess (평가) + add (구현)

각 레이어에 두 종류의 스킬이 있다.

```
평가 스킬 (위에서 아래로):
  assess-api            → API 레이어 평가
  assess-usecase        → usecase 레이어 평가
  assess-infra          → infrastructure 레이어 평가
  assess-model          → model 레이어 평가

구현 스킬 (아래에서 위로):
  add-model             → model 모듈의 작업 규칙
  add-exception         → exception 모듈의 작업 규칙
  add-infrastructure    → infrastructure 모듈의 작업 규칙
  add-entity-specs      → Entity 클래스 + DDL 생성 규칙
  add-jdbc-query        → EntityRepository 쿼리 + JdbcRepository 구현 규칙
  add-usecase           → usecase 모듈의 작업 규칙
  add-api               → api 모듈의 작업 규칙
```

### assess 스킬 공통 원칙

**행동 규칙:**
1. 가능하면 현재 레이어에서 처리
2. 데이터 스펙(domain, entity) 변경이 필요하면 → 유저에게 문의 + 제안 동봉
3. 명시된 모듈에 대해서만 쿼리
4. 모듈 간 통신은 정해진 스펙으로만
   - infra → return: domain model
   - usecase → return: {domain}Read, input: command

**공통 산출물:**
1. 현재 레이어에서 작업 충족 여부
2. 필요한 하위 기능 (모듈별)
3. 현재 레이어의 작업 계획과 스펙

산출물이 다음 레이어의 입력이 된다.

### 진행 방향

```
평가: 위에서 아래로 (top-down)
  api → usecase → infra → model
  "이 요청을 처리할 수 있는가? 아래에 뭐가 필요한가?"

제작: 아래에서 위로 (bottom-up)
  model → infra → repo → usecase → api
  "확정된 설계를 기반으로 구현"
```

### 플래너 = 경량 오케스트레이터

플래너는 판단을 직접 하지 않는다. assess 스킬의 결과를 모아서 조율만 한다.

1. 유저 요청 + API 스펙 수신
2. assess 스킬을 위에서 아래로 체이닝
3. 결과를 모아 유저에게 전체 계획 제시
4. 유저 확정 후 add 스킬을 아래에서 위로 실행

### 케이스별 실행 범위

assess 결과에 따라 필요한 스킬만 실행한다.

```
케이스 A: 새 도메인
  평가: assess-api → assess-usecase → assess-infra → assess-model (전부 "없음")
  제작: add-model → add-infrastructure → add-entity-specs → add-jdbc-query → add-usecase → add-api

케이스 B: 기존 도메인에 API 추가
  평가: assess-api("없음") → assess-usecase("충분") → 여기서 끝
  제작: add-usecase 변경 → add-api 추가

케이스 C: 테이블 구조 변경
  평가: assess-api → assess-usecase → assess-infra("변경필요") → assess-model("변경필요")
  제작: add-model → add-infrastructure → add-entity-specs → add-jdbc-query (인터페이스 안 바뀌면 나머지 그대로)
```

---

## 도메인 추가 시 생성 순서 (전체 실행의 경우)

```
1. model/           → 도메인 모델, Identity, enum
2. exception/       → 도메인 예외 (필요 시)
3. infrastructure/  → Repository 인터페이스
4. schema/          → DDL
5. repository-jdbc/ → Entity, EntityRepository, JdbcRepository
6. usecase/         → Reader/Writer + Command DTO
7. api/             → Controller + Request/Response DTO
```

---

## 미결 사항

- auth, elasticsearch 모듈의 헥사고날 적용 여부 → 스킬 제작 중 판단
- usecase 모듈의 인터페이스 분리 여부 → batch에서 usecase를 쓰게 될 경우 필요할 수 있음. 선택의 문제로 보류.

## TODO

- assess 스킬이 하위 레이어에 기능을 요청할 때, 해당 레이어의 스타일/컨벤션도 함께 지정해야 함. 예: assess-api가 usecase에 신규 메서드를 요청할 때, Reader/Writer 중 어디에 넣을지, Command DTO 필요 여부, 반환 타입 컨벤션 등을 명시해야 받는 쪽(assess-usecase 또는 add-usecase)이 일관성 있게 작업 가능.
- model/entity 스펙 변경 시 유저 확인 타이밍: 각 assess 단계에서 즉시 vs plan 완료 후 일괄. 실행해보면서 결정.
