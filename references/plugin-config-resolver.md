# dk-be-bundle Project Context Resolver

이 플러그인의 스킬은 실행 시 **프로젝트 컨텍스트**를 먼저 해석한다. 해석된 값은 각 스킬 본문의 `{플레이스홀더}`에 대응된다.

## 해석 우선순위 (hybrid)

1. **config 파일** — 프로젝트 루트의 `dk-be-bundle.config.json`이 있으면 그 값을 최우선으로 사용
2. **자동 탐지** — config 없으면 아래 규칙으로 추출
3. **사용자 질의** — 탐지 실패 또는 모호하면 유저에게 명시적으로 묻고, 결과를 config 파일로 저장 제안

## 변수 목록

| 변수 | 의미 | 기본값 / 탐지 방법 |
|---|---|---|
| `{packageRoot}` | Java 패키지 루트 | 루트 `build.gradle.kts`의 `group = "..."` |
| `{packagePath}` | `{packageRoot}`의 `.` → `/` 변환 | 자동 파생 |
| `{modulesPath}` | 모듈 디렉토리 루트 | `modules` (프로젝트에 `modules/` 폴더 있으면) |
| `{modules.api}` | api 모듈 경로 | `{modulesPath}/api` |
| `{modules.service}` | service 모듈 경로 | `{modulesPath}/service` |
| `{modules.repository-jdbc}` | JDBC 모듈 경로 | `{modulesPath}/repository-jdbc` |
| `{modules.schema}` | schema 모듈 경로 | `{modulesPath}/schema` |
| `{modules.model}` | model 모듈 경로 | `{modulesPath}/model` |
| `{modules.infrastructure}` | infrastructure 모듈 경로 | `{modulesPath}/infrastructure` |
| `{esModule}` | ES 모듈 이름 | config 명시 또는 질의 (예: `ai-search`) |
| `{esPackageSuffix}` | ES 패키지 서브 | `{esModule}`에서 `-` 제거 (예: `aisearch`) |
| `{esPackage}` | ES 전체 패키지 | `{packageRoot}.{esPackageSuffix}` |
| `{esIndexName}` | ES 인덱스명 | config 명시 또는 질의 |
| `{esHost}` | ES 호스트 | `localhost:9200` |

### 파생 변수 (자동 변환)

| 변수 | 변환 규칙 | 예시 |
|---|---|---|
| `{packagePath}` | `{packageRoot}`의 `.` → `/` | `share.ai.setup` → `share/ai/setup` |
| `{esPackage-as-path}` | `{esPackage}`의 `.` → `/` | `share.ai.setup.aisearch` → `share/ai/setup/aisearch` |
| `{modules.api.gradlePath}` | `{modules.api}`의 `/` → `:` (leading colon 포함) | `modules/api` → `modules:api` |
| `{modules.service.gradlePath}` | 동일 | `modules/service` → `modules:service` |

사용 위치: 파일 시스템 경로에는 `{packagePath}` / `{esPackage-as-path}`, Gradle 태스크 경로에는 `{modules.xxx.gradlePath}`, FQCN에는 `{packageRoot}` / `{esPackage}`.

## 자동 탐지 절차

```bash
# 1. packageRoot 추출
grep -E '^group\s*=' {project-root}/build.gradle.kts | head -1
# → group = "share.ai.setup" → packageRoot = share.ai.setup

# 2. modulesPath 확인
ls {project-root}/modules 2>/dev/null && modulesPath=modules || modulesPath=.

# 3. 각 모듈 존재 확인
ls {project-root}/{modulesPath}/api {modulesPath}/service ...
```

## config 파일 스키마

`{project-root}/dk-be-bundle.config.json`:

```json
{
  "packageRoot": "share.ai.setup",
  "modulesPath": "modules",
  "modules": {
    "api": "modules/api",
    "service": "modules/service",
    "repository-jdbc": "modules/repository-jdbc",
    "schema": "modules/schema",
    "model": "modules/model",
    "infrastructure": "modules/infrastructure"
  },
  "es": {
    "module": "ai-search",
    "packageSuffix": "aisearch",
    "indexName": "setup",
    "host": "localhost:9200"
  }
}
```

모든 필드 선택(optional). 빠진 값은 자동 탐지 또는 질의로 보완.

## 스킬 실행 시 체크리스트

스킬 본문에서 `{플레이스홀더}`를 만나면:

1. config 파일 읽었는가? (없으면 읽기 시도)
2. 해당 변수가 config에 있는가? → 사용
3. 없으면 자동 탐지 시도 — 추출 규칙 적용
4. 실패하면 유저에게 질의: "이 프로젝트의 `{변수명}`은 무엇인가요?" + 기본값 제시
5. 답 받으면 메모리에 캐시하고, `dk-be-bundle.config.json`에 저장할지 제안
