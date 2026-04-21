---
name: be-assess-api
description: API 레이어 평가. 요청된 API 스펙이 기존 엔드포인트로 처리 가능한지 평가하고, usecase와 model에 필요한 기능을 도출한다.
---

# assess-api

API 레이어에서 요청된 기능을 처리할 수 있는지 평가한다.

## 입력

- 유저의 API 스펙 요청
- specs/api.md (기존 API 엔드포인트) — 프로젝트 루트 기준
- specs/usecase.md (기존 usecase 인터페이스)
- specs/models.md (기존 도메인 모델)

## 평가 절차

### 1. 기존 API 확인

specs/api.md를 읽고, 요청된 기능과 매칭되는 엔드포인트가 있는지 확인한다.

- 완전 일치 → "충족". 작업 불필요.
- 옵션/파라미터 추가로 처리 가능 → "부분 충족". 변경 계획 수립.
- 없음 → "미충족". 신규 생성 계획 수립.

### 2. usecase 가용성 판단

specs/usecase.md를 읽고, 필요한 기능이 이미 있는지 확인한다.

**기존 usecase로 처리 가능한 경우:**
- 기존 Reader/Writer 또는 별도 usecase 인터페이스의 메서드로 충분
- API에서 조합만 하면 됨
- → usecase 변경 불필요

**새로운 usecase 기능이 필요한 경우:**
- 기존 인터페이스에 없는 메서드가 필요
- 여러 도메인을 조합하는 사가성 로직이 필요
- 새로운 Command가 필요
- → usecase에 기능 요청

### 3. model 가용성 판단

specs/models.md를 읽고, API 응답에 필요한 데이터가 기존 도메인 모델에 있는지 확인한다.

**기존 model로 처리 가능한 경우:**
- {domain}Read의 기존 필드로 Response 구성 가능
- → model 변경 불필요

**model 변경/추가가 필요한 경우:**
- 요청된 API가 다루는 도메인 자체가 없음 → 새 도메인 필요
- 기존 도메인에 필드 추가가 필요 → 기존 도메인 확장
- → model에 기능 요청

### 4. API 계획 수립

다음을 결정한다:

- HTTP Method + URL 패턴
- Request DTO 필드 구성
- Response DTO 필드 구성 (specs/models.md의 {domain}Read 기반)
- 인증 필요 여부
- 사용할 usecase 인터페이스 (기존 or 신규 요청)

## 규칙

- Controller에 비즈니스 로직을 넣지 않는다. 위임만.
- Request → Command 변환은 Controller에서 수행한다.
- Response는 static from({domain}Read, ...) 팩토리 메서드로 생성한다.
- 기존 API 변경 시 as-is/to-be를 제시하고 유저 확인을 받는다.

## 산출물

### 1. 충족 여부

```
충족 / 부분 충족 / 미충족
```

### 2. 하위 레이어에 필요한 기능

**A. usecase:**
```
기존 사용 (변경 불필요):
  - {UsecaseInterface}.{메서드명} 사용

신규 요청:
  - {메서드 시그니처}
  - {구현해야 하는 기능에 대한 설명}
  - {동작 요구사항}
```

**B. model:**
```
기존 사용 (변경 불필요):
  - {domain}Read의 기존 필드로 충분

신규 요청:
  - 대상: {도메인명} (신규/신규 아님)
  - {필요한 데이터 구조 설명}
  - 예상: {VO/필드 + 관계 힌트}
  - {왜 필요한지 기획 맥락}
```

### 3. 현재 레이어 작업 계획

```
- [신규/변경] {HTTP Method} {URL}
- Request: {필드 목록}
- Response: {필드 목록} (from {domain}Read)
- 인증: 필요/불필요
- 사용 usecase: {기존 메서드 or 신규 요청}
```
