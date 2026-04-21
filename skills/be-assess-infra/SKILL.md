---
name: be-assess-infra
description: infrastructure 레이어 평가. 상위(usecase)에서 요청된 기능이 기존 Repository 인터페이스로 처리 가능한지 평가하고, model에 필요한 기능을 도출한다.
---

# assess-infra

infrastructure 레이어에서 요청된 기능을 처리할 수 있는지 평가한다.

## 입력

- assess-usecase의 산출물 (infrastructure에 대한 요청)
- specs/infrastructure.md (기존 Repository 인터페이스) — 프로젝트 루트 기준
- specs/models.md (기존 도메인 모델)

## 평가 절차

### 1. 기존 infrastructure 확인

specs/infrastructure.md를 읽고, 요청된 기능과 매칭되는 Repository 메서드가 있는지 확인한다.

- 완전 일치 → "충족". 작업 불필요.
- 기존 Repository에 메서드 추가로 처리 가능 → "부분 충족". 변경 계획 수립.
- 해당 도메인의 Repository 자체가 없음 → "미충족". 신규 생성 계획 수립.

### 2. model 가용성 판단

specs/models.md를 읽고, Repository 메서드의 입출력에 필요한 도메인 타입이 있는지 확인한다.

**기존 model로 처리 가능한 경우:**
- 반환할 {domain}Read / {domain}의 타입이 존재
- 파라미터에 필요한 Identity VO가 존재
- → model 변경 불필요

**model 변경/추가가 필요한 경우:**
- 새 도메인의 Repository → 해당 도메인 모델 자체가 필요
- 기존 도메인에 새 필드 → Read/Write 모델 모두 확장 필요
- 새 Identity VO 필요
- → model에 기능 요청

### 3. infrastructure 계획 수립

다음을 결정한다:

- 기존 Repository에 추가 or 신규 Repository 생성
- 메서드 시그니처 (파라미터 + 반환 타입)
- 읽기 메서드 → {domain}Read 반환
- 쓰기 메서드 → {domain} 반환
- 원자적 연산 여부 (increaseXxxCount 등)

## 규칙

- 인터페이스만 정의한다. 구현은 repository-jdbc의 영역.
- 읽기 → {domain}Read 반환, 쓰기 → {domain} 반환 원칙을 따른다.
- model과 exception만 의존한다.
- 기존 인터페이스 변경 시 as-is/to-be를 제시하고 유저 확인을 받는다.

## 산출물

### 1. 충족 여부

```
충족 / 부분 충족 / 미충족
```

### 2. 하위 레이어에 필요한 기능

**A. model:**
```
기존 사용 (변경 불필요):
  - {domain}, {domain}Read, {Identity} 기존 타입으로 충분

신규 요청:
  - 대상: {도메인명} (신규/신규 아님)
  - {필요한 데이터 구조 설명}
  - 예상: {VO/필드 + 관계 힌트}
  - {왜 필요한지 구현 맥락}
```

### 3. 현재 레이어 작업 계획

```
- [신규/변경] {Repository}.{메서드명}
- 파라미터: {타입 목록}
- 반환: {domain}Read / {domain} / void
- 원자적 연산: 여부
```
