# dk-be-bundle TODO

플러그인 발전 계획. 각 항목의 디테일은 대화하면서 확정.

---

## Phase A. 테스트 환경 구축

독립 테스트베드 만들고 플러그인 종단 실행으로 검증.

- [x] A0. `be-init` 스킬 설계 (autoresearch:reason 3-0 만장일치로 확정) → `skills/be-init/SKILL.md`
- [ ] A1. 테스트베드 위치 — TBD (함께 논의)
- [ ] A2. 최소 멀티모듈 스캐폴드: **be-init 실행**으로 대체 (gradle init / 복사 불필요)
- [ ] A3. 플러그인 로드 상태로 기동
- [ ] A4. 샘플 도메인 `Article` (identity/title/body/createdAt/updatedAt/deletedAt/isDeleted) CRUD를 be-planner로 종단 생성 → 이슈 기록

## Phase B. References 강화

share_ai_setup 실제 코드 분석 → refs 디테일 추가.

### 공통 규칙 (모든 references 파일 필수 삽입)

각 레퍼런스 파일 맨 위 (H1 + 서브타이틀 직후) 에 **반드시** 아래 blockquote 를 포함한다:

```markdown
> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.
```

### 작업 리스트

- [x] B1. model — `be-refs/model-references.md` 신규 (Foo/Bar/Baz/Qux 4 패턴)
- [x] B2. entity — `be-refs/entity-references.md` 재구성 (Foo/Bar/Baz/Qux, @MappedCollection + Stats 분리 + 평면 배치)
- [x] B3. jdbc-query — `be-refs/query-references.md` 재작성 (Foo/Bar/Baz/Qux + 유저 질의 팁)
- [x] B4. service — `be-refs/service-references.md` 신규 (Foo/Bar/Baz/Qux/Quux 5 패턴)
- [x] B5. infrastructure — `be-refs/infrastructure-references.md` 신규 (Foo/Bar/Baz/Qux 4 패턴, Read/Write 분리 + Stats 분리)
- [x] B6. api — `be-refs/api-references.md` 신규 (Foo/Bar/Baz/Qux 4 패턴, 공통 Req/Resp DTO + 인증 + 로깅)

## Phase C. 에이전틱 루프 전환

각 스킬을 단일 프롬프트 → 자기검증 루프로 변환 

- [ ] C1. add 스킬 6개 루프화 — metric TBD
- [ ] C2. test 스킬 4개 루프화 — metric TBD
- [ ] C3. assess 스킬 4개 루프화 — metric TBD
- [ ] C4. be-planner 전체 오케스트레이션 루프화 — metric TBD

---

## 의존성

```
A ──┬─→ B (병렬 가능)
    └─→ C (A 완료 후 — 검증에 테스트베드 필요)
```

## 결정 대기 사항

1. A2 스캐폴드 방식
2. Phase 진행 순서 (A→B→C 순차 vs B 병렬)
3. C 범위 (전체 vs add 먼저)
4. 각 C 단계의 검증 metric 정의
