---
name: be-add-infrastructure
description: infrastructure 모듈에 Repository 인터페이스를 생성/변경한다. assess-infra의 산출물을 기반으로 작업한다.
---

# add-infrastructure

infrastructure 모듈에 Repository 인터페이스를 생성하거나 변경한다.

## 입력

- assess-infra 산출물 (작업 계획)
- specs/infrastructure.md (기존 Repository 인터페이스) — 프로젝트 루트 기준 (없으면 소스에서 추론)
- specs/models.md (도메인 모델 — 반환 타입 참조) — 프로젝트 루트 기준

## 참고 문서 (읽어올 것)

- `${CLAUDE_PLUGIN_ROOT}/references/be-refs/infrastructure-references.md` — 4 패턴 (Foo/Bar/Baz/Qux) + 공통 요소 + 선택 가이드

## 패턴 선택 (먼저 결정)

infrastructure-references.md §"패턴 선택 가이드" 로 도메인 특성에 맞는 패턴을 먼저 고른다:

| 도메인 특성 | Pattern |
|---|---|
| 단순 CRUD, JOIN 불필요 | **Foo** |
| Read 모델에 JOIN/집계 필요, 리스트·카운트 | **Bar** (주력) |
| 통계 카운터 증감 | **Baz** (별도 파일: `{Domain}StatsRepository`) |
| 트리·순서 연산 | **Qux** (Bar 에 병합) |

조합: Setup = Bar + Baz(파일 2개), Comment = Bar + Qux(한 파일), Rating = Foo.

## 생성 결과물 구조

```
infrastructure/src/main/java/{packagePath}/infra/{domain}/repository/
├── {Domain}Repository.java          ← Pattern Foo / Bar / Qux
└── {Domain}StatsRepository.java     ← Pattern Baz (필요 시)
```

## 생성 규칙

### 1. 인터페이스만 정의
- 구현 없음. 메서드 시그니처만.

### 2. 반환 타입 원칙
- 읽기 메서드 → {Domain}Read 반환
- 쓰기 메서드 → {Domain} 반환
- 삭제/원자적 연산 → void

### 3. 파라미터 원칙
- ID 조회 → {Domain}Identity 사용 (Long 직접 사용 지양)
- 벌크 조회 → List<{Domain}Identity>

### 4. 원자적 연산은 별도 메서드
- increaseViewCount, increaseLikeCount 등
- 도메인 객체 전체를 save하지 않고 특정 컬럼만 업데이트하는 경우

### 5. 의존
- model과 exception만 의존
- 다른 모듈 의존 금지

### 6. 한국어 주석
- 인터페이스는 다른 레이어의 설명서 역할을 한다
- 각 메서드에 한국어 주석을 작성한다
- 목적: 이 메서드가 무엇을 하는지
- 특이사항: 시간복잡도가 높은 경우, 부수효과가 있는 경우 등 명시

```java
public interface {Domain}Repository {

    /** 게시글 단건 조회. 유저 닉네임 포함. */
    Optional<{Domain}Read> findById({Domain}Identity identity);

    /**
     * 키워드로 태그 검색. prefix 매칭.
     * 특이사항: LIKE 검색으로 데이터 많을 시 느려질 수 있음.
     */
    List<String> searchTags(String keyword);
}
```

## 기존 인터페이스 변경 시

- as-is/to-be를 유저에게 제시하고 확인받는다
- 메서드 추가는 하위 호환, 시그니처 변경은 유저 확인 필수
- 기존 메서드와 기능적으로 유사한 요청이 있는 경우, 병합 여부를 유저에게 문의

## 산출물

- 생성/변경된 인터페이스 파일 목록
- specs/infrastructure.md 업데이트
- 추가/변경된 스펙 정리본 출력
