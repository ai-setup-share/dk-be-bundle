---
name: be-assess-usecase
description: usecase 레이어 평가. 상위(api)에서 요청된 기능이 기존 usecase로 처리 가능한지 평가하고, infrastructure와 model에 필요한 기능을 도출한다.
---

# assess-usecase

usecase 레이어에서 요청된 기능을 처리할 수 있는지 평가한다.

## 입력

- assess-api의 산출물 (usecase에 대한 요청)
- specs/usecase.md (기존 usecase 인터페이스) — 프로젝트 루트 기준
- specs/infrastructure.md (기존 infrastructure 인터페이스)
- specs/models.md (기존 도메인 모델)

## 평가 절차

### 1. 기존 usecase 확인

specs/usecase.md를 읽고, 요청된 기능과 매칭되는 인터페이스/메서드가 있는지 확인한다.

- 완전 일치 → "충족". 작업 불필요.
- 기존 Reader/Writer에 메서드 추가로 처리 가능 → "부분 충족". 변경 계획 수립.
- 없음 → "미충족". 신규 생성 계획 수립.

### 2. infrastructure 가용성 판단

specs/infrastructure.md를 읽고, usecase 구현에 필요한 데이터 접근이 가능한지 확인한다.

**기존 infrastructure로 처리 가능한 경우:**
- 기존 Repository 인터페이스의 메서드로 데이터 조회/저장 가능
- → infrastructure 변경 불필요

**새로운 infrastructure 기능이 필요한 경우:**
- 기존 Repository에 없는 조회 조건이 필요 (예: 정렬, 필터, 집계)
- 새 도메인의 Repository가 필요
- 원자적 연산 메서드가 필요
- → infrastructure에 기능 요청

### 3. model 가용성 판단

specs/models.md를 읽고, usecase가 다루는 도메인 모델이 충분한지 확인한다.

**기존 model로 처리 가능한 경우:**
- Command 변환에 필요한 도메인 타입이 존재
- 반환할 {domain}Read의 필드가 충분
- → model 변경 불필요

**model 변경/추가가 필요한 경우:**
- Command가 다루는 도메인 자체가 없음 → 새 도메인 필요
- 기존 도메인에 필드/VO 추가 필요
- → model에 기능 요청

### 4. usecase 계획 수립

다음을 결정한다:

- Reader 또는 Writer 중 어디에 속하는지 (조회 vs 명령)
- 메서드 시그니처
- Command DTO 필요 여부 및 필드 구성
- 반환 타입 ({domain}Read or void or 기타)
- 크로스 도메인 의존성 (다른 도메인의 infrastructure 참조 여부)
- @Transactional 필요 여부 (Writer만 해당)

## 규칙

- usecase에 DB 접근 코드를 넣지 않는다. infrastructure 인터페이스만 의존.
- CQRS: 조회는 Reader, 명령은 Writer에 배치한다.
- 같은 도메인의 Reader/Writer는 서로 호출 가능하다.
- 다른 도메인의 usecase를 호출하는 경우 → 유저에게 문의. (패키지가 다른 usecase 간 의존)
- 크로스 도메인 데이터 참조는 해당 도메인의 infrastructure 인터페이스를 통해서만.
- Command DTO는 @Value immutable 객체로 정의한다.
- 기존 usecase 변경 시 as-is/to-be를 제시하고 유저 확인을 받는다.

## 산출물

### 1. 충족 여부

```
충족 / 부분 충족 / 미충족
```

### 2. 하위 레이어에 필요한 기능

**A. infrastructure:**
```
기존 사용 (변경 불필요):
  - {Repository}.{메서드명} 사용

신규 요청:
  - {Repository}.{메서드 시그니처}
  - {구현해야 하는 기능에 대한 설명}
  - {동작 요구사항}
```

**B. model:**
```
기존 사용 (변경 불필요):
  - {domain}의 기존 필드/타입으로 충분

신규 요청:
  - 대상: {도메인명} (신규/신규 아님)
  - {필요한 데이터 구조 설명}
  - 예상: {VO/필드 + 관계 힌트}
  - {왜 필요한지 기능 맥락}
```

### 3. 현재 레이어 작업 계획

```
- [신규/변경] {Reader 또는 Writer}.{메서드명}
- 입력: {Command 또는 파라미터}
- 반환: {domain}Read / void
- 의존: {사용하는 infrastructure 인터페이스 목록}
- 트랜잭션: 필요/불필요
```
