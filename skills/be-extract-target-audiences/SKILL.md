---
name: be-extract-target-audiences
description: 문서의 title + body를 분석하여 target_audiences 키워드를 추출하고 ES에 업데이트한다. "타겟 오디언스 추출", "대상 독자 키워드 추출", "target audiences 추출" 등의 요청에 트리거한다.
---

# be-extract-target-audiences

문서의 title + body를 LLM(Claude)에게 분석시켜 **대상 독자 키워드(target_audiences)**를 추출하고,
ES `_update` API로 해당 문서에 반영한다.

## 용도

검색 시점에 `GUIDE_SYSTEM` 프롬프트가 `target_audiences`를 활용하여 "~를 검토 중이라면" 식의 가이드 문구를 생성한다.
이 스킬은 인덱싱 시점(오프라인)에 키워드를 사전 추출하는 역할이다.

## Phase 0. 프로젝트 컨텍스트 해석

`{esIndexName}`, `{esHost}` 플레이스홀더 사용.
실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석 (`es.indexName`, `es.host`).

## 입력

유저에게 아래를 확인한다:

1. **대상 문서** — 아래 중 하나:
   - `doc_id` 1개 (단건 처리)
   - `doc_id` 목록 (다건 처리)
   - "전체" (ES 인덱스의 모든 문서)
2. **ES 인덱스명** — 컨텍스트의 `{esIndexName}` 사용 (없으면 질의)
3. **ES 호스트** — 컨텍스트의 `{esHost}` 사용 (기본값: `localhost:9200`)

## 추출 프롬프트

Claude에게 아래 프롬프트를 **직접 수행**한다 (외부 API 호출 아님, 클로드 자신이 분석).

### 시스템 프롬프트

```
당신은 기술 문서 분석가입니다.
문서의 제목과 본문을 받아, 이 문서가 도움이 될 수 있는 구체적인 상황을 키워드로 추출합니다.

## 규칙
- **5~6개**의 키워드를 추출
- 각 키워드는 **2~8자 명사구**로 짧게
  - 좋은 예: "사내 챗봇 구축", "검색 비용 절감", "E2E 테스트 자동화", "온프레미스 번역 전환"
  - 나쁜 예: "개발자", "엔지니어", "AI 활용" (너무 일반적)
  - 나쁜 예: "Claude Code 장기 세션 품질 관리를 위한 아키텍처 설계" (너무 장황)
- 키워드는 검색 시점에 "~를 검토 중이라면", "~이 고민이라면" 뒤에 자연스럽게 붙을 수 있어야 함
  - 예: "사내 챗봇 구축" → "사내 챗봇 구축을 검토 중이라면"
  - 예: "GPU 비용 절감" → "GPU 비용 절감이 고민이라면"
- "~담당자", "~팀" 같은 사람/조직 키워드 금지. 상황/과제만 추출
- 비슷한 키워드 반복 금지 (예: "검색 구축"과 "검색 시스템 구축" 동시 추출 금지)
- JSON 배열로만 응답

## 참고: 기존 추출 사례

| title | target_audiences |
|-------|-----------------|
| GraphQL 목 데이터를 LLM으로 자동 생성 | GraphQL 목 데이터, 서버 없는 FE 개발, 스키마 동기화, self-heal 루프, LLM 개발 도구 통합 |
| Spring AI와 Typesense로 사내 Q&A 자동화 | 사내 Q&A 봇, Spring AI @Tool, Hybrid Search, Kendra 대안, Slack 챗봇 구축 |
| Claude Code 멀티 인스턴스 역할 분담 | AI 해커톤 운영, Claude Code 역할 분담, AI 코딩 문화 확산, 에이전트 협업, CLAUDE.md 활용 |
| 데이터 파이프라인 알림 노이즈를 AI로 분류 | 알림 노이즈 분류, 온콜 부담 감소, 장애 대응 자동화, AI 로그 분석, 파이프라인 운영 |
```

### 유저 프롬프트

```
제목: "{title}"
본문:
{body}
```

### 기대 출력

```json
["사내 Q&A 봇 구축", "검색 비용 절감", "Kendra 대안 검토", "Slack 봇 연동", "Hybrid Search 설계"]
```

## 실행 흐름

### 단건 처리 (`doc_id` 1개)

1. ES에서 문서 조회:
   ```bash
   curl -s "http://{host}/{index}/_doc/{doc_id}" | jq '.\_source | {doc_id, title, body}'
   ```
2. title + body를 읽고 Claude 자신이 분석하여 target_audiences 추출
3. 추출 결과를 유저에게 보여주고 확인 요청
4. 확인 후 ES `_update` API로 반영:
   ```bash
   curl -X POST "http://{host}/{index}/_update/{doc_id}" \
     -H 'Content-Type: application/json' \
     -d '{"doc": {"target_audiences": ["키워드1", "키워드2", ...]}}'
   ```
5. 결과 확인

### 다건 처리 (`doc_id` 목록 또는 전체)

1. 전체인 경우 `_search`로 문서 목록 조회:
   ```bash
   curl -s "http://{host}/{index}/_search" \
     -H 'Content-Type: application/json' \
     -d '{"query": {"match_all": {}}, "_source": ["doc_id", "title", "body"], "size": 100}'
   ```
2. 각 문서에 대해 순차적으로:
   - title + body 분석 → target_audiences 추출
   - 추출 결과를 표 형태로 누적
3. 전체 결과를 유저에게 표로 보여주고 확인 요청:
   ```
   | doc_id     | title              | target_audiences                            |
   |------------|--------------------|---------------------------------------------|
   | setup_1    | Spring AI와 ...    | 사내 Q&A 봇 구축, 검색 비용 절감, ...        |
   | setup_2    | 데이터 파이프...   | 알림 노이즈 분류, 온콜 부담 감소, ...        |
   ```
4. 확인 후 `_bulk` API로 일괄 반영:
   ```bash
   curl -X POST "http://{host}/{index}/_bulk" \
     -H 'Content-Type: application/x-ndjson' \
     -d '
   {"update": {"_id": "setup_1"}}
   {"doc": {"target_audiences": ["키워드1", "키워드2"]}}
   {"update": {"_id": "setup_2"}}
   {"doc": {"target_audiences": ["키워드3", "키워드4"]}}
   '
   ```

## 추출 품질 기준

- **구체성**: "AI 활용" ✗ → "LLM 기반 알림 분류" ✓
- **상황 지향**: "QA 엔지니어" ✗ → "E2E 테스트 자동화" ✓
- **가이드 호환**: 추출된 키워드가 "~를 검토 중이라면" 패턴에 자연스럽게 연결되는지 확인
- **중복 방지**: 비슷한 키워드 반복 금지 (예: "검색 구축"과 "검색 시스템 구축" 동시 추출 ✗)
- **개수**: 5~8개. 문서 내용이 적으면 5개까지 줄일 수 있음

## 산출물

- 추출된 target_audiences 키워드 (표 형태)
- ES 업데이트 완료 여부
- 실패 건이 있으면 실패 목록

## 주의사항

- body가 매우 길 경우 앞 3000자만 사용한다 (키워드 추출에 전문이 필요하지 않음)
- body가 null이거나 빈 경우 title만으로 추출한다 (키워드 수를 3~5개로 줄임)
- ES 업데이트 전 반드시 유저 확인을 받는다
- 이 스킬은 ES 문서의 `target_audiences` 필드만 업데이트한다. 다른 필드는 건드리지 않는다
