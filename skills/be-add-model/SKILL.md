---
name: be-add-model
description: model 모듈에 도메인 모델을 생성/변경한다. assess-model의 산출물을 기반으로 작업한다.
---

# add-model

model 모듈에 도메인 모델을 생성하거나 변경한다.

## 입력

- assess-model 산출물 (작업 계획)
- specs/models.md (기존 도메인 모델) — 프로젝트 루트 기준

## 생성 결과물 구조

신규 도메인 생성 시 아래 트리가 만들어져야 한다:

```
model/src/main/java/dev/{project}/model/{domain}/
├── {Domain}.java              ← Write 모델 (@Value, AuditProps 상속)
├── {Domain}Read.java          ← Read 모델 (@Value, AuditProps 상속)
├── {Domain}Identity.java      ← Identity VO (@Value)
├── {Domain}Category.java      ← enum (필요 시)
└── {VOName}.java              ← 하위 VO (필요 시)
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

### 5. AuditProps 상속
- Write/Read 모델 모두 AuditProps를 implements하여 상속
- AuditProps가 요구하는 createdAt, updatedAt: Instant 필드를 반드시 포함
- AuditProps를 상속함으로써 audit 관련 공통 처리가 자동 적용됨

### 6. String vs Enum 검증
- 값이 유한하고 고정적 → enum (company, category, status, role)
- 자유 입력 → String (title, content, url)
- 애매하면 → 유저에게 문의

### 7. Read 모델은 항상 생성
- Write 모델 필드 전부 + 조회 시 필요한 추가 필드 (nickname 등)
- JOIN 필드가 없더라도 Read 모델을 별도로 둔다

### 8. toRead 변환 메서드
- Write 모델에 toRead() 메서드를 제공한다
- Write → Read 변환 시 추가 필드(nickname 등)를 파라미터로 받는다
- 도메인 변경 시 toRead 스펙도 함께 변경한다

### 9. soft delete 필드 제외
- soft delete 필드(isDeleted, deletedAt)는 모델에 포함하지 않는다
- Entity 레이어에서만 관리한다

## 기존 도메인 변경 시

- as-is/to-be를 유저에게 제시하고 확인받는다
- 필드 추가 시 기존 팩토리 메서드 시그니처 변경 여부 확인
- Write 변경 시 Read + toRead도 함께 변경

## 산출물

- 생성/변경된 모델 파일 목록
- specs/models.md 업데이트
- 추가/변경된 스펙 정리본 출력
