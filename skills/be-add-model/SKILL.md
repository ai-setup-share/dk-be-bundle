---
name: be-add-model
description: model 모듈에 도메인 모델을 생성/변경한다. assess-model의 산출물을 기반으로 작업한다.
---

# add-model

model 모듈에 도메인 모델을 생성하거나 변경한다.

## 입력

- assess-model 산출물 (작업 계획)
- specs/models.md (기존 도메인 모델) — 프로젝트 루트 기준 (없으면 소스에서 직접 추론)

## 참고 문서 (읽어올 것)

- `${CLAUDE_PLUGIN_ROOT}/references/be-refs/model-references.md` — 4가지 패턴(Foo/Bar/Baz/Qux) 전체 예시 + 공통 요소 + 선택 가이드

## 패턴 선택 (먼저 결정)

model-references.md §"패턴 선택 가이드" 를 적용해 도메인 특성에 맞는 패턴을 먼저 고른다:

| 도메인 특성 | Pattern |
|---|---|
| Upsert 만 필요, 조회 시 추가 정보 없음 | **Foo** — Write-only, Read 생성 안 함 |
| 기본 CRUD, Read 는 Write 의 투영 | **Bar** — Write+Read, 단순 `toRead()` |
| 조회 시 JOIN/집계 필요, 하위 VO 존재 | **Baz** — Write+Read+Stats+하위 VO |
| 상태 전이·트리·숨김 로직 있음 | **Qux** — 상태 전이 메서드 |

## 생성 결과물 구조

선택한 패턴에 따라 아래 중 필요한 파일만 생성:

```
model/src/main/java/{packagePath}/model/{domain}/
├── {Domain}.java              ← Write 모델 (필수, implements AuditFields)
├── {Domain}Read.java          ← Read 모델 (Bar/Baz/Qux 만)
├── {Domain}Identity.java      ← Identity VO (필수)
├── {Domain}Stats.java         ← 분리 통계 (Baz 만)
├── {Name}VO.java              ← 하위 VO (Baz 등 필요 시)
└── {Domain}Category.java      ← enum (필요 시)
    {Domain}Type.java
    ...
```

## 생성 규칙

### 1. 불변 객체
- @Value (Lombok) 사용. 모든 필드 final, setter 없음.

### 2. Identity 분리
- 도메인마다 {Domain}Identity 별도 생성
- 필드명: Long {domain}Id

### 3. 상태 변경은 새 인스턴스 반환
- 예: with 패턴으로 새 인스턴스 반환

### 4. 하위 VO 분리
- 유사한 필드의 모임(3개 이상)은 별도 VO로 추출
- 도메인 패키지 하위에 위치
- 예: Popularity, JobDescription

### 5. AuditFields 구현
- Write/Read 모델 모두 `{packageRoot}.model.audit.AuditFields` 인터페이스를 implements 한다 (be-init §4.6 / C-4)
- `createdAt: Instant`, `updatedAt: Instant` 필드를 반드시 포함 — AuditFields 의 getter 계약을 충족
- AuditFields 는 be-init 이 제공하며, 값은 Entity 레이어의 `@CreatedDate`/`@LastModifiedDate` 가 채운 후 toDomain 으로 전달됨

### 6. String vs Enum 검증
- 값이 유한하고 고정적 → enum (company, category, status, role)
- 자유 입력 → String (title, content, url)
- 애매하면 → 유저에게 문의

### 7. Read 모델 — 패턴에 따라
- **Pattern Foo**: Read 생성 금지 (Write-only 도메인)
- **Pattern Bar/Baz/Qux**: Read 별도 생성
- JOIN/집계 필드 없는 Bar 도 Read 를 별도로 둬 투영(민감 필드 제거, 타입 안전성)의 기회를 유지

### 8. toRead 변환 메서드
- Bar/Baz/Qux 의 Write 모델에 `toRead(...)` 제공
- Bar: 인자 없음 (self-contained projection)
- Baz: 의존 데이터를 파라미터로 명시 (authorNickname, stats, links)
- 도메인 변경 시 `toRead` 시그니처도 함께 변경

### 9. soft delete 필드 — 모델 레이어 제외
- 도메인이 soft-delete 를 **사용하더라도** `isDeleted`/`deletedAt` 은 모델에 포함하지 않는다.
- Entity/DB 레이어에서만 관리 (be-init 컨벤션: 모든 테이블 보유, 사용만 도메인별).
- 모델에서 노출되는 "삭제" 개념은 조회 결과에 포함되지 않는 것으로 충분.

## 기존 도메인 변경 시

- as-is/to-be를 유저에게 제시하고 확인받는다
- 필드 추가 시 기존 팩토리 메서드 시그니처 변경 여부 확인
- Write 변경 시 Read + toRead도 함께 변경

## 산출물

- 생성/변경된 모델 파일 목록
- specs/models.md 업데이트
- 추가/변경된 스펙 정리본 출력
