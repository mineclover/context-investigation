# 환경 유형 분류 및 Claude Code 탐색 프로브 가이드

이 문서는 `discover-context-environments` 스킬의 Step 1~4에서 8대 컨텍스트 보유 환경을 표준 스키마로 분류하고, Claude Code 실행 환경에서 안전하게 비파괴 프로브를 수행하는 구체 절차를 정의한다.

## 1. 8대 환경 범주와 표준 스키마 매핑

사용자·보고서 관점의 8대 환경 범주(`surface_category`)는 정규 레코드의 6종 `type`(`local | remote | skill | connector | memory | runtime`) 및 6종 `context.kind`(`project | specification | decision | memory | procedure | execution`)와 다음과 같이 대응한다.

| 환경 범주 (`surface_category`) | 정규 `type` | 주요 `context.kind` | 탐색 대상 예시 | 주의할 경계 |
| --- | --- | --- | --- | --- |
| `local_workspace` (로컬 작업 환경) | `local` | `project`, `specification`, `decision`, `execution` | 워크스페이스 디렉터리, 소스 코드, `package.json`/`Cargo.toml`, `docs/`, 로컬 Git 로그·작업 트리 | 원본 프로젝트 폴더는 기본 `source_owned: true`이며 임의 쓰기 대상이 아님 |
| `remote_dev` (원격 개발 환경) | `remote` | `project`, `decision`, `execution` | GitHub/GitLab 원격 저장소, 이슈·PR·리뷰 이력, 원격 워크스페이스·브랜치 | 로컬 클론과 원격 업스트림의 `revision`(커밋/브랜치) 차이를 분리 기록 |
| `document_knowledge` (문서·지식 환경) | `connector` / `remote` / `local` | `specification`, `decision`, `project` | Notion 워크스페이스, Google Drive, 사내 Wiki/Confluence, OpenWiki 지식 그래프 | 접근 경로가 MCP/플러그인이면 `connector`, 로컬 마운트 볼륨이면 `local`, 웹/API 원격이면 `remote` |
| `agent_config` (에이전트 환경) | `local` / `runtime` | `procedure`, `decision`, `execution` | `CLAUDE.md`, `AGENTS.md`, `ALIGNS.md`, `.claude/rules/`, `.claude/settings*.json`, `.mcp.json`, 세션 로그 | 작업 규칙(`procedure`/`decision`)과 에이전트 장기 기억(`memory`)을 혼동하지 않음 |
| `skill` (스킬 환경) | `skill` | `procedure` | `.claude/skills/`, `~/.claude/skills/`, `.agents/skills/`, 활성 플러그인 스킬의 `SKILL.md` 및 `references/` | 스킬이 제공하는 **절차적 지식**만 이 범주에 등록함 |
| `connector` (커넥터 환경) | `connector` | 대상에 따라 판정 | MCP 서버(`mcp__*`), `claude.ai <Name>` 계정 커넥터, 플러그인이 노출하는 외부 도구·리소스 | 연결 여부(`discovered`)와 실제 프로젝트 관련성(`related_projects`)을 절대 동일시하지 않음 |
| `memory` (메모리 환경) | `memory` | `memory` | Claude 자동/프로젝트 메모리(`MEMORY.md`), 스킬 관리 메모리 파일/DB, Notion/MCP 메모리 서버, 장기 기억 | 출처가 스킬이든 커넥터든 **맥락·기억 저장소**는 반드시 `type: memory`로 별도 등록 |
| `runtime_ops` (실행·운영 환경) | `runtime` | `execution` | 실행 스크립트 구성, Docker/배포 매니페스트, CI 워크플로 상태, 테스트·검증 리포트, 런타임 로그 | 비밀값(환경 변수 값, 시크릿)은 읽거나 출력하지 않고 구성 키와 상태만 수집 |

---

## 2. 절차적 지식(`skill`)과 프로젝트 메모리(`memory`) 분리 규칙

하나의 스킬이나 플러그인이 **절차 문서**와 **상태·기억 데이터**를 함께 갖는 경우가 많다. 조사 시 두 요소를 하나의 엔트리로 합치면 "무엇을 어떻게 하라는 일반 규칙"과 "이 프로젝트에서 과거에 무엇을 결정·관측했는가"의 권위가 뒤섞인다.

| 구분 항목 | 스킬 절차적 지식 (`type: skill`) | 스킬·커넥터·에이전트 메모리 (`type: memory`) |
| --- | --- | --- |
| 보유 정보 | 재사용 가능한 작업 절차, 스키마, 판단 기준, 템플릿 | 특정 프로젝트·사용자의 과거 결정, 선호, 상태 스냅샷, 학습 기록 |
| `context.kind` | `procedure` | `memory` |
| `context.related_projects` | 일반적으로 `[]` (범용) 또는 프로젝트 전용 로컬 스킬인 경우 해당 프로젝트 | 귀속 근거가 확인된 프로젝트만 기재 (미확인 시 `[]`) |
| `context.authority` | 절차에 대한 `advisory` 또는 프로젝트 표준일 때 `primary` | 원천 사실이 아닌 기억이므로 기본적으로 `personal_memory` 또는 `derived` (원천 대조 필요) |
| 후속 조사 방식 | 작업 수행 방법과 도메인 규칙 해석에 활용 | 원천 보존(`source_owned: true`), 식별자·출처·판단 근거를 참조 형태로 기록 |

**예시**: 프로젝트에 `align` 스킬이 설치되어 있고 루트에 `ALIGNS.md`가 존재하며 Notion Memory 커넥터가 연결된 경우:
- `env.skill.align` (`type: skill`, `kind: procedure`, `locator: ".claude/skills/align/SKILL.md"`)
- `env.local.aligns-doc` (`type: local`, `kind: decision`, `locator: "ALIGNS.md"`, `authority: "primary"`)
- `env.memory.notion-memory` (`type: memory`, `kind: memory`, `provider: "notion-mcp"`, `related_projects: []` — 항목별 관련성 근거 확인 전까지 미귀속)

---

## 3. Claude Code 세션별 비파괴 탐색 프로브

### 3.1 로컬 작업 환경 & 에이전트 구성 (로컬 파일 도구가 있을 때)

1. **작업 공간 루트 및 Git 토폴로지**:
   - 루트 디렉터리 목록, `README.md`, `CLAUDE.md`, `AGENTS.md`, `ALIGNS.md` 존재 확인 (`Glob` / `Read`).
   - `Bash`가 허용된 경우 읽기 전용 명령(`git rev-parse --show-toplevel`, `git status --short`, `git remote -v`, `git rev-parse HEAD`)으로 `locator`와 `revision`을 취득한다.
2. **Claude Code 설정 및 거버넌스 파일**:
   - 프로젝트 범위: `.mcp.json`, `.claude/settings.json`, `.claude/settings.local.json`, `.claude/CLAUDE.md`, `.claude/rules/*`, `.claude/skills/*`.
   - 상위/사용자 범위(허가된 경우에만): `~/.claude/skills/`, `~/.claude/settings.json`의 활성/차단 상태(`deniedMcpServers`, `disableClaudeAiConnectors`, `disabledMcpjsonServers`, `enabledPlugins`, `skillOverrides`).
   - **주의**: `~/.claude.json`에는 OAuth·인증 정보가 포함될 수 있으므로 원문 전체를 덤프하거나 읽지 않는다. 세션에 이미 노출된 MCP 도구 목록을 우선 신뢰한다.
3. **기존 조사·인계 저장 규약 탐지**:
   - `CLAUDE.md`, `AGENTS.md`, `docs/`, `handover/`, `.investigation/`, `ADR` 등에서 기존에 승인된 조사/인계 산출물 경로 규칙이 있는지 확인하여 `source-registry.json`의 `persistence_candidates` 근거로 남긴다.

### 3.2 커넥터 · MCP · 문서·지식 환경

1. **도구 표면 기반 발견 (`discovered`)**:
   - 현재 세션에서 호출 가능한 `mcp__<server>__*` 도구 이름과 `ListMcpResourcesTool` 결과를 통해 연결된 서버/커넥터 목록을 추출한다.
   - `.mcp.json`에 선언되어 있으나 도구 목록에 없다면 미승인(`pending_approval`) 또는 비활성(`disabledMcpjsonServers`) 상태이므로 `discovered: true`, `readable: false`로 구분 기록한다.
2. **최소 권한·관련성 프로브 (`readable` & `authorized_scope`)**:
   - 읽기 전용 메타데이터 도구(예: 현재 사용자/워크스페이스 조회, 폴더/데이터베이스 목록 1페이지 조회, `$target_scope` 키워드 검색 상위 3~5건)만 호출한다.
   - 생성·수정·삭제·전송·댓글 작성 도구는 절대 호출하지 않는다.
   - 검색 결과에서 대상 프로젝트와의 명시적 연결(프로젝트 코드, 저장소 링크, 동일 제품 스코프)이 확인된 리소스 경로/ID만 `source-registry.json`의 관련 원천으로 승격한다.

### 3.3 메모리 환경

1. **Claude 프로젝트/자동 메모리**:
   - 허가된 범위에서 프로젝트 메모리 파일(예: `MEMORY.md`, `.claude/` 하위 메모리 노트) 존재 여부와 최종 수정 시점을 확인한다.
2. **스킬·커넥터 관리 메모리**:
   - 지식 그래프 MCP, 벡터/RAG 메모리 도구, 노트/위키 기반 메모리 컬렉션, 스킬 로컬 캐시/원장(예: `.skills-platform/`, `.serena/`, 옵시디언 볼트 등)을 탐색한다.
   - 각 메모리 환경은 `persistence.source_owned: true`로 표시하여 후속 단계에서 원천 변조가 일어나지 않게 보호한다.

---

## 4. 특수 실행 환경에서의 폴백 (Zero-Assumption Resilience)

| 실행 시나리오 | 기대 탐색 동작 |
| --- | --- |
| **로컬 PC 접근 불가 (Cloud / Web / 원격 샌드박스)** | 로컬 파일시스템 프로브를 건너뛰거나 빈 컨테이너로 기록하고, 연결된 GitHub/원격 저장소 커넥터, 문서 커넥터(Notion 등), 대화 첨부 자료, 에이전트 메모리를 중심으로 `environment-map.json`을 완성한다. |
| **외부 커넥터 전혀 없음 (순수 로컬/에어갭 세션)** | 커넥터 표면을 비어 있음으로 기록하고 로컬 코드베이스, Git 이력, `CLAUDE.md`, 로컬 스킬(`SKILL.md`), 로컬 검증 로그만으로 환경 지도를 구성한다. |
| **다중 프로젝트 / 멀티레포 조사** | 각 환경 엔트리의 `context.related_projects`를 배열(`["proj-a", "proj-b"]`)로 관리하고, 공유 환경(공용 Wiki, 조직 메모리)과 단일 프로젝트 전용 환경(개별 Git 저장소)을 명확히 구분한다. |
