---
name: be-planner
description: API 스펙을 받아 평가 체인을 실행하고, 구현 계획을 수립하여 역순으로 실행한다.
---

# be-planner

API 스펙을 받아, 각 레이어를 평가하고, 확정된 계획을 역순으로 구현한다.
각 assess/add 스킬을 호출하는 오케스트레이터 역할.

## 공통 규칙

- **API 1개 단위**로 평가 및 구현한다.
- 하위 단계에서 예외가 발생하거나, 크리티컬한 스펙 불일치가 있는 경우, **유저에게 상황을 브리핑 후 진행 또는 중단을 결정**한다.
- 각 스킬의 산출물은 **계획 문서(plan md)**에 기록하여 다음 스킬의 입력으로 사용한다.

## 호출 대상 스킬 ID

아래 본문에서 `assess-xxx`, `add-xxx`로 줄여 지칭하는 스킬들의 실제 호출 ID:

| 줄임 이름 | 실제 스킬 ID |
|---------|------------|
| assess-api | `dk-be-bundle:be-assess-api` |
| assess-usecase | `dk-be-bundle:be-assess-usecase` |
| assess-infra | `dk-be-bundle:be-assess-infra` |
| assess-model | `dk-be-bundle:be-assess-model` |
| add-model | `dk-be-bundle:be-add-model` |
| add-infrastructure | `dk-be-bundle:be-add-infrastructure` |
| add-entity-specs | `dk-be-bundle:be-add-entity-specs` |
| add-jdbc-query | `dk-be-bundle:be-add-jdbc-query` |
| add-usecase | `dk-be-bundle:be-add-usecase` |
| add-api | `dk-be-bundle:be-add-api` |

Skill tool 호출 시 위 ID를 사용한다.

## 입력

- 유저로부터 API 스펙 1개를 수신한다.

## 계획 문서

모든 평가 산출물과 구현 계획을 하나의 파일에 누적 기록한다.

**경로**: `specs/plans/{api-name}.md`
- `{api-name}`은 API 엔드포인트에서 파생 (예: `get-community-post-tags.md`)

**구조**:
```markdown
# Plan: {HTTP Method} {URL}

## 요청 스펙
{유저가 제시한 API 스펙}

## Phase 1: 평가

### assess-api
{산출물 기록}

### assess-usecase
{산출물 기록}

### assess-infra
{산출물 기록}

### assess-model
{산출물 기록}

## Phase 2: 구현 계획
{평가 결과 요약 + 구현 순서}

## Phase 3: 구현 진행
{각 add 스킬 완료 시 결과 기록}
```

**운용 규칙**:
- 각 assess 스킬 완료 시 해당 섹션에 산출물을 **즉시 기록**한다.
- Phase 2에서 유저가 계획을 수정하면, 해당 섹션을 **직접 수정**한다.
- Phase 3에서 각 add 스킬 완료 시 결과를 **추가 기록**한다.
- 다음 스킬은 이 문서를 읽고 이전 산출물을 참조한다.

## Phase 1: 평가 (top-down)

각 assess 스킬을 호출하고, 산출물을 계획 문서에 기록한다.

```
assess-api        → 산출물을 계획 문서에 기록
  └→ assess-usecase  → 산출물을 계획 문서에 기록
       └→ assess-infra    → 산출물을 계획 문서에 기록
            └→ assess-model    → 산출물을 계획 문서에 기록
```

### 평가 흐름

1. **assess-api** 호출: API 레이어 판단
   - 산출물을 계획 문서 `### assess-api` 섹션에 기록
   - 충족 → Phase 2로 (api만 추가)
   - 미충족 → 다음 단계로

2. **assess-usecase** 호출: 계획 문서의 assess-api 산출물을 입력으로 사용
   - 산출물을 계획 문서 `### assess-usecase` 섹션에 기록
   - 충족 → Phase 2로
   - 미충족 → 다음 단계로

3. **assess-infra** 호출: 계획 문서의 assess-usecase 산출물을 입력으로 사용
   - 산출물을 계획 문서 `### assess-infra` 섹션에 기록
   - 충족 → Phase 2로
   - 미충족 → 다음 단계로

4. **assess-model** 호출: 계획 문서의 assess-infra 산출물을 입력으로 사용
   - 산출물을 계획 문서 `### assess-model` 섹션에 기록
   - Phase 2로

### 중단 규칙

- 해당 레이어에서 "충족"이면 하위 평가를 하지 않는다.
- 데이터 스펙 변경(model/entity)이 필요한 경우, 유저에게 즉시 문의한다.

## Phase 2: 계획 제시

평가 결과를 모아 계획 문서의 `## Phase 2: 구현 계획` 섹션에 기록하고, 유저에게 제시한다.

```
[API 스펙]: GET /api/community-posts/tags?keyword=

[평가 결과]:
- api: 미충족 — 새 엔드포인트 필요
- usecase: 미충족 — CommunityPostReader.searchTags 필요
- infra: 미충족 — CommunityPostRepository.searchTags 필요
- model: 미충족 — Tag 데이터 구조 필요

[구현 계획] (역순):
1. add-model: Tag VO 추가, CommunityPost에 tags 필드
2. add-infrastructure: searchTags 메서드 추가
3. add-entity-specs: Entity 변경 + DDL
4. add-jdbc-query: EntityRepository + JdbcRepository
5. add-usecase: CommunityPostReader.searchTags 추가
6. add-api: GET /api/community-posts/tags 엔드포인트
```

유저가 수정을 요청하면 계획 문서의 해당 부분을 수정한다.
유저 확정을 받은 후 Phase 3으로 진행한다.

## Phase 3: 구현 (bottom-up)

확정된 계획 문서를 읽고, 아래에서 위로 각 add 스킬을 실행한다.
평가에서 "미충족"으로 판정된 레이어만 실행한다.

```
add-model
  → add-infrastructure
    → add-entity-specs
      → add-jdbc-query
        → add-usecase
          → add-api
```

### 실행 규칙

- 각 add 스킬은 계획 문서를 읽고, 자기 레이어의 평가 산출물과 구현 계획을 참조한다.
- 각 add 스킬 완료 시 계획 문서 `## Phase 3: 구현 진행` 섹션에 결과를 기록한다.
- 순서대로 실행한다 (병렬 불가).
- 각 스킬 완료 후 해당 specs/ 문서가 업데이트된다.

## 케이스별 실행 범위

```
케이스 A: 새 도메인
  평가: api → usecase → infra → model (전부 미충족)
  구현: add-model → add-infrastructure → add-entity-specs
       → add-jdbc-query → add-usecase → add-api

케이스 B: 기존 도메인에 API 추가
  평가: api(미충족) → usecase(충족) → 여기서 끝
  구현: add-usecase 변경 → add-api 추가

케이스 C: 테이블 구조 변경
  평가: api → usecase → infra(미충족) → model(미충족)
  구현: add-model → add-infrastructure → add-entity-specs
       → add-jdbc-query
```
