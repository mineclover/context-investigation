---
name: discover-context-environments
description: >-
  프로젝트나 작업 조사에 앞서 현재 Claude Code 세션이 접근할 수 있는 컨텍스트 보유 환경(로컬·원격·문서·에이전트·스킬·커넥터·메모리·런타임)을
  비파괴적으로 탐색하고, 발견·접근 권한·실제 읽기 가능 여부·권위·소유권을 분리 판정하여 environment-map.json과
  source-registry.json을 산출한다. 심층 프로젝트 조사나 인계 작업의 최상위 사전 단계(Phase 0~1)로 사용한다.
when_to_use: >-
  새 프로젝트·멀티 저장소·레거시 시스템을 조사하거나 인계 자료를 만들기 전 어디에 맥락이 있는지 파악해야 할 때,
  로컬 파일뿐 아니라 MCP 서버·claude.ai 커넥터·설치된 스킬·에이전트 메모리에 흩어진 컨텍스트 접근점을 식별해야 할 때,
  또는 로컬 PC나 특정 클라우드 드라이브가 없는 환경에서 가용 컨텍스트 지도를 먼저 확정해야 할 때 호출한다.
argument-hint: "[조사 대상 프로젝트명, 저장소/식별자, 또는 탐색 범위 힌트] [interest_query: 관심 주제와 렌즈]"
arguments:
  - target_scope
effort: high
license: Apache-2.0
compatibility: >-
  Claude Code CLI, IDE, Desktop, Cloud/Web 세션 지원. 로컬 파일시스템이나 특정 커넥터(Google Drive, Notion 등)의
  존재를 가정하지 않으며 현재 세션에 노출된 도구와 허가된 인터페이스만 탐색한다.
metadata:
  package: context-investigation
  phase: "Phase 0-1 (Context Environment Discovery & Source Mapping)"
  produces: "environment-map.json, source-registry.json"
  companion_skill: resolve-investigation-storage
---

# 컨텍스트 보유 환경 탐색 (`discover-context-environments`)

심층 프로젝트 조사(Phase 3)나 산출물 저장 위치 결정(Phase 2)에 앞서, **현재 실행 중인 에이전트가 어떤 환경에 놓여 있으며 허가된 인터페이스를 통해 어디까지 작업 맥락을 읽을 수 있는지**를 비파괴적으로 식별·검증한다.

## 1. 핵심 정의와 분류 기준

> **Context-Bearing Environment (컨텍스트 보유 환경)**
>
> 현재 실행 주체가 허가된 인터페이스를 통해 접근할 수 있으며, 특정 프로젝트·작업·시스템의 목적, 상태, 구조, 결정, 의도 또는 이력을 복원하는 데 활용할 수 있는 정보를 보유한 환경.

컨텍스트는 로컬 파일이나 정적 문서 형태로만 존재하지 않는다. 물리적 작업 공간뿐 아니라 스킬, 플러그인, 커넥터, 에이전트 메모리를 통한 논리적 접근점 전체를 탐색 대상으로 삼는다.

| 환경 범주 (`surface_category`) | 표준 `type` | 대표 조사 대상 | 기본 `context.kind` |
| --- | --- | --- | --- |
| 로컬 작업 환경 (`local_workspace`) | `local` | 프로젝트 디렉터리, 소스 코드, 설정, 로컬 작업·커밋 기록 | `project` / `specification` |
| 원격 개발 환경 (`remote_dev`) | `remote` | GitHub/GitLab 원격 저장소, PR·이슈, 원격 개발 워크스페이스 | `project` / `decision` |
| 문서·지식 환경 (`document_knowledge`) | `connector` / `remote` / `local` | Wiki, Notion, Google Drive, 사내 문서, 지식 그래프 | `specification` / `decision` |
| 에이전트 환경 (`agent_config`) | `local` / `runtime` | `CLAUDE.md`, `AGENTS.md`, `.claude/` 구성, 규칙 및 세션 기록 | `procedure` / `decision` |
| 스킬 환경 (`skill`) | `skill` | 설치된 스킬(`SKILL.md`), 스킬의 `references/` 자료, 관련 설정 | `procedure` |
| 커넥터 환경 (`connector`) | `connector` | 연결된 플러그인, MCP 서버, `claude.ai` 커넥터 노출 데이터·기능 | 대상별 판정 |
| 메모리 환경 (`memory`) | `memory` | 스킬 관리 메모리, 커넥터 메모리, 프로젝트 메모리, 장기 기억 | `memory` |
| 실행·운영 환경 (`runtime_ops`) | `runtime` | 실행·배포 구성, CI/CD, 작업 상태, 로그 및 검증 기록 | `execution` |

### 반드시 지켜야 할 두 가지 분리 원칙

1. **절차적 지식과 프로젝트 메모리의 분리**:
   스킬 자체가 제공하는 절차적 지식(`type: skill`, `context.kind: procedure`)과 스킬·커넥터·에이전트가 관리하는 프로젝트/사용자 메모리(`type: memory`, `context.kind: memory`)는 서로 다른 컨텍스트다. 같은 플러그인이나 스킬 디렉터리에서 기원하더라도 **별도의 `context_environment` 엔트리로 분리 등록**한다.
2. **발견 · 접근 권한 · 읽기 검증 · 프로젝트 관련성의 분리**:
   커넥터나 도구가 연결되어 있다는 사실(`access.discovered: true`)만으로 그 안의 모든 데이터가 조사 대상이 되지는 않는다. 실제 접근 허가 범위(`authorized_scope`), 비파괴 읽기 성공 여부(`readable`), 자료의 권위(`authority`), 대상 프로젝트 귀속 근거(`related_projects`)를 각각 별도 필드로 판정한다.

환경별 상세 프로브 방법과 Claude Code 설정 점검 위치는 [환경 유형 분류 및 Claude Code 탐색 프로브](references/environment-taxonomy-and-claude-probes.md)를 읽는다.

## 2. 실행 절차 (Phase 0 → Phase 1)

사용자가 `$ARGUMENTS`(또는 `$target_scope`)로 대상 프로젝트명, 저장소, 도메인, 탐색 범위를 지정했다면 이를 기준 스코프로 삼는다. 인자가 없으면 현재 작업 경로, 에이전트 지침(`CLAUDE.md`, `AGENTS.md`), 활성 세션 문맥에서 후보 프로젝트 식별자를 먼저 추출한다. 인자나 대화에 `interest_query:` 블록이 있으면 Step 2.2의 관심사 렌즈를 함께 적용한다.

### Step 1. 실행 표면 및 도구 가용성 감사 (Runtime Surface Audit)

특정 환경(로컬 PC 디스크, Google Drive, Git CLI 등)이 반드시 존재한다고 가정하지 않는다. 현재 Claude Code 세션에서 실제 호출 가능한 인터페이스를 먼저 확인한다.

- **파일·셸 표면**: `Read`, `Glob`, `Grep`, `Bash` 가용 여부 및 현재 작업 디렉터리 마운트 상태.
- **커넥터·MCP 표면**: 세션에 로드된 `mcp__<server>__<tool>` 도구군, `ListMcpResourcesTool` / `ReadMcpResourceTool`, `claude.ai` 계정 커넥터 도구.
- **에이전트·스킬·플러그인 표면**: 활성 스킬 목록, `.claude/` 및 `.agents/` 설정 파일(접근 가능 시), 서브 에이전트(`Agent`) 디스패치 지원 여부.

접근할 수 없는 표면(예: 클라우드 전용 세션의 로컬 디스크 부재, 비활성화된 커넥터)은 오류가 아니라 환경 특성으로 기록하고 가용한 표면만으로 다음 단계를 진행한다.

### Step 2. Phase 0 — 컨텍스트 보유 환경 후보 탐색 (Context Environment Discovery)

Step 1에서 확인한 인터페이스를 따라 8대 환경 범주의 접근점을 수집한다.

1. 로컬 작업 공간이 있으면 루트 매니페스트, `CLAUDE.md` / `AGENTS.md`, `.mcp.json`, `.claude/settings.json`, `.claude/settings.local.json`(비활성 MCP·커넥터 토글 확인), `.claude/skills/` 및 `.agents/skills/`를 확인한다.
2. 설치된 스킬의 절차 문서(`type: skill`)와 별도로, Claude 프로젝트 메모리(`MEMORY.md` 등 허가된 경로), 스킬 전용 상태·메모리 저장소, MCP/커넥터 기반 메모리 도구를 `type: memory` 후보로 식별한다.
3. 원격 저장소(`git remote`, GitHub 커넥터 등), 문서·지식 커넥터(Notion, Drive, Wiki 등), 실행·운영 기록(배포 설정, 검증 로그 등) 접근점을 후보 목록에 추가한다.
4. 발견된 모든 후보에 안정적인 `id`, `type`, `provider`, `locator`를 부여하고 `access.discovered: true`로 기록한다.

### Step 2.0 루트 스캔 우선 및 읽기 범위 선택 (Root Scan First & Scope Selection)

범위 모드(Step 2.1)를 쓰는 경우, 내용을 읽기 전에 **반드시 아래 순서로 루트 스캔을 먼저** 수행하고 사용자에게 읽을 범위를 확인받는다.

1. **저장소 루트 스캔**: `.git` 디렉터리를 찾아 git 저장소 루트 목록을 만든다. 마커 파일이 전혀 없는 저장소 루트도 포함한다. 캐시·도구·의존성 경로는 가지치기하되, 가지친 개수는 기록한다.
2. **프로젝트 루트 스캔**: 프로젝트 마커(Step 2.1 계층 A)로 프로젝트 루트 목록을 만든다.
3. **대조**: 두 목록을 맞춘다.
   - 모든 저장소 루트는 프로젝트 목록에 항목으로 있거나, 그 하위 프로젝트 루트들의 상위로 명시되어야 한다. 저장소가 조용히 빠지면 누락으로 보고한다.
   - **저장소 단위 중복 정리**: 같은 저장소의 하위 루트와 같은 원격 URL을 가진 사본은 `repo_group`으로 묶는다. 원격이 없으면 저장소 경로로 묶는다. 묶음의 첫 항목을 대표로 두고 나머지는 `duplicate_of`에 대표 경로를 적는다. 중복 항목을 독립 프로젝트로 세지 않는다.
   - 합계는 중복 제거 전후 수를 함께 적는다. 루트 수를 다른 범주 수와 더해 중복 집계하지 않는다.
4. **범위 선택 질문**: 아래 결정은 추측하지 말고 `AskUserQuestion`으로 사용자에게 묻는다. 기본값에는 `(Recommended)`를 붙인다.
   - 읽을 범위: 목록만(계층 A) / 저장소 단위로 골라 계층 B 판정 / 지정한 상위 폴더만 / (`interest_query`가 있을 때) 관심사 후보(`name_only` 포함)를 먼저 계층 B 판정
   - 제외 경로 포함 여부: `ref/`(외부 참조 클론), `worktree/`, 도구 설정 디렉터리(`.claude`, `.codex` 등) 안의 프로젝트를 포함할지
   - 중복 루트 처리: 하나로 묶기(권장) / 루트별로 따로 보기
   - 사용자가 묻지 말라고 하거나 질문 도구가 없는 세션이면 권장 기본값으로 진행하고, 커버리지 원장에 `scope_decision: default_unconfirmed`를 기록한다.
5. 선택 결과를 커버리지 원장에 기록한 뒤에만 계층 B·C를 진행한다.

이 단계에서 확인하는 것은 목록과 범위뿐이다. 프로젝트 내용과 설정 값은 읽지 않는다.

### Step 2.1 탐색 범위 모드 (Scope Mode: 저장소 전체 · 컴퓨터 전체)

사용자가 저장소 전체 또는 컴퓨터 전체 파악을 목표로 지정하면 아래 계층으로 나눠 수행한다. 계층마다 읽기 범위를 달리해 넓게 훑되, 모든 파일의 내용을 무차별로 읽지 않는다.

| 계층 | 대상 | 방식 | 내용 읽기 |
| --- | --- | --- | --- |
| A. 인벤토리 (항상) | 지정 루트 전체 (저장소 전체 또는 홈 디렉터리) | 디렉터리를 깊이 상한까지 열거하고 프로젝트 마커(`.git`, `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `CLAUDE.md`, `AGENTS.md`, `SKILL.md`, `.mcp.json`)를 탐지한다. 크기와 최종 수정 시각을 기록한다. | 없음 (메타데이터만) |
| B. 관련성 판정 | 계층 A에서 발견된 프로젝트 루트 | 루트 매니페스트, README 앞부분, `CLAUDE.md`, 명세·문서 디렉터리 목록을 읽어 프로젝트 목적과 상태를 판정한다. | 프로젝트당 상한 (문서 최대 10개, 파일당 앞 200줄) |
| C. 심층 조사 | 사용자가 지정했거나 계층 B에서 조사 대상으로 승격된 항목 | 판정된 범위의 파일만 읽는다. | 승격된 범위 안에서만 |

**전수 열거 규칙 (계층 A 필수 조건)**

- 계층 A는 **후보를 하나도 빠뜨리지 않고** 목록화한다. 날짜, 활동 여부, 크기 같은 기준으로 목록에서 미리 거르지 않는다. 필터는 목록을 만든 뒤 판정 단계에서 사유와 함께 적용한다.
- 프로젝트 마커는 `.git`만이 아니다. 저장소가 없는 폴더도 매니페스트나 마커가 있으면 후보가 된다(예: `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `CLAUDE.md`, `SKILL.md`). `.git`이 있는 폴더와 마커만 있는 폴더를 따로 센다.
- 후보 목록은 **전체 목록 파일**로 남긴다(`investigations/<investigation-id>/project-inventory.md`). 경로, 마커 종류, 저장소 여부, 원격 URL, 최종 커밋 날짜(또는 최종 수정 시각)를 한 줄씩 적는다. 요약만 남기지 않는다.
- 목록의 각 항목에는 판정 결과(`read_deep`, `read_summary`, `inventory_only`, `excluded_<사유>`)를 하나씩 붙인다. 판정되지 않은 항목이 남으면 완료로 보고하지 않는다.
- 발견 수는 **읽음 + 메타데이터만 + 제외 + 미조사**의 합과 일치해야 한다. 합이 맞지 않으면 누락으로 보고한다.

**인덱스 열 (필수)**: 목록 파일은 아래 열을 갖는다. 열 값은 규칙으로만 채우고, 판단이 어려우면 `unknown`으로 둔다.

| 열 | 값 | 판정 규칙 |
| --- | --- | --- |
| `owner_kind` | `own` / `third_party` / `unknown` | 원격 URL의 소유자가 사용자 계정이면 `own`, 다른 소유자면 `third_party`, 원격이 없으면 `unknown` |
| `noise_class` | `run_artifact` / `codex_work` / `version_copy` / `sample_template` / `normal` | 경로 규칙으로 분류한다. 실행 산출물(`runs/`, `bundle/`, `outputs/`, `work/`, 평가 기록), 작업 폴더, 버전·최종본 복사, 샘플·템플릿·예제 순서로 먼저 맞는 것을 쓴다 |
| `description` | 한 줄 | `README.md`의 첫 제목, 없으면 `package.json`의 `description`. 둘 다 없으면 빈 값 |
| `repo_group` / `duplicate_of` | 그룹 키 / 대표 경로 | 저장소 단위 중복 정리 규칙 참조 |

- 기본 목록은 `noise_class: normal`인 항목을 먼저 보여 주고, 나머지는 접어서 개수만 표시한다. 접힌 항목도 목록 파일에는 남긴다.
- 인덱스는 "무엇이 있는가"를 보여 주는 용도다. `description`은 목적 판정이 아니며, 목적 근거는 계층 B에서 따로 확인한다.

**프로젝트와 아키텍처는 별도 레이어로 기록한다 (필수)**

- **프로젝트 레이어 (무엇인가)**: 목적, 소유 주체(원격 URL 또는 소유자 식별자), 상태(활성·보관·종료 신호), 최근 활동 시점, 근거 파일 경로. 근거는 README 제목·문장, 매니페스트 `description`, 명세 문서 같은 **자기 서술**에서만 가져온다. 이름으로 추측하지 않는다.
- **아키텍처 레이어 (어떻게 구성되었나)**: 사용 언어와 런타임, 워크스페이스 정의(`package.json`의 `workspaces`, `pnpm-workspace.yaml`, `Cargo.toml`의 `[workspace]`, `turbo.json`, `docker-compose*`), 하위 패키지 구성 단위와 역할 분포, 진입점. 구조 신호는 **매니페스트와 디렉터리 배치**에서만 가져온다.
- 두 레이어를 한 문단이나 한 열에 섞지 않는다. 프로젝트 목적을 아키텍처 설명으로 대체하거나, 구조 분류를 목적 근거로 쓰지 않는다.
- 두 레이어는 같은 항목 ID로 연결하되 별도 파일(`project-catalog.md`, `architecture-catalog.md`)에 쓴다.

범위 원칙:

- **계층 A는 전체에 적용하고, 계층 B·C는 승격된 대상에만 적용한다.** 발견했다는 사실만으로 모든 저장소의 내용을 읽지 않는다.
- **항상 제외**: 의존성·캐시·빌드 산출물(`node_modules`, `.venv`, `target`, `dist`, `build`, `.cache`, `Library`, `.Trash`), 시스템 경로(`/System`, `/Library`, `/usr`, `/Applications`), 다른 사용자 홈, 대용량 바이너리와 미디어.
- **클라우드 동기화 폴더 (기본 제외, 원장 기록 필수)**: `~/Library/CloudStorage/` 하위(Google Drive, OneDrive)는 `Library` 제외 규칙에 묶이므로 기본적으로 스캔하지 않는다. 사용자가 제외를 유지하기로 하면 커버리지 원장에 `excluded_cloud_sync: ~/Library/CloudStorage`와 사용자 결정 사실을 남긴다. 누락이 아니라 결정된 제외로 보고한다.
- **자격 증명 파일은 내용뿐 아니라 목록에도 넣지 않는다.** `.env*`, `*.pem`, `*.key`, `.ssh/`, `.gnupg/`, `.aws/`, `.boto`, `~/.claude.json*`, 셸 히스토리(`.bash_history`, `.zsh_history`)는 존재 여부만 `excluded_sensitive`로 기록한다.
- **커버리지 원장을 함께 만든다.** 스캔한 루트, 건너뛴 경로와 사유, 계층별로 읽은 파일 수를 기록한다. "전체 파악"이라는 결론은 원장에 기록된 범위 안에서만 주장한다.
- **인용 규칙**: 산출물에 근거로 적는 파일은 실제로 열어 읽은 것만이다. 열지 않은 파일을 근거로 쓰면 `authority: unverified`로 표시한다.

### Step 2.2 관심사 쿼리 렌즈 (Interest Query Lens, 선택)

사용자가 `interest_query`로 관심 주제와 조사 목적 렌즈를 주면, 위 규칙으로 만든 인덱스 위에 렌즈 레이어를 추가한다. 주제와 렌즈는 사용자가 정하며, 이 스킬에는 기본 주제나 기본 렌즈가 없다. 쿼리가 없으면 건너뛰고 커버리지 원장에 `interest_query: none`을 기록한다.

- **덧붙이기만 한다**: 렌즈는 정렬과 표시만 바꾼다. 전수 열거, 제외 규칙, 계층별 읽기 상한, 사용자 확인 후 승격 규칙은 그대로 적용한다.
- **주제 일치**: 주제를 의존성 이름·파일 형식 같은 식별자로 펼친 뒤, 각 항목에 `interest_match`(`strong` / `weak` / `name_only` / `none` / `unknown`)와 근거 경로를 붙인다. 이름에만 주제어가 있으면 근거로 세지 않는다.
- **렌즈 정의**: 렌즈마다 질문, 계산 단위(`item` / `population`), 적용 대상, 확인할 사실, 출력 규칙을 정한다. 이름만 주어지면 정의 초안을 만들어 사용자 확인 후 적용한다.
- **별도 레이어**: 결과는 `interest-lens.md`에 `source: lens`로 쓴다. 프로젝트·아키텍처 카탈로그에 섞지 않는다.

입력 형식, 렌즈 정의 스키마, 출력 형식, 쿼리 예시는 [관심사 쿼리 렌즈](references/interest-query-lens.md)를 읽는다.

### Step 3. 비파괴 접근 범위 및 읽기 가능 여부 판정 (Access & Scope Probing)

각 후보 환경에 대해 **최소한의 읽기 전용 프로브(Read-Only Probe)**만 수행하여 `access.readable`과 `access.authorized_scope`를 판정한다.

- **최소 프로브 원칙**: 커넥터 전체를 크롤링하거나 무차별 검색하지 않는다. 연결 상태 확인, 최상위 리소스 목록 조회, 또는 `$target_scope`와 관련된 메타데이터/문서 1~3건 조회 수준으로 제한한다.
- **판정 분리**:
  - 읽기 성공 시: `readable: true` (일부 경로/페이지만 허용되면 `"partial"`), 확인된 워크스페이스·폴더·계정 범위를 `authorized_scope`에 명시.
  - 권한 거부·인증 만료·비활성 차단 시: `readable: false`, 차단 사유(예: `denied_by_settings`, `auth_required`, `permission_denied`)를 함께 기록.
  - 프로브를 실행하지 않았거나 비용·안전상 보류한 경우: `readable: "unknown"`, `authorized_scope: "unknown"`으로 남긴다. 확인하지 않은 항목을 `true`나 데이터 부재로 바꾸지 않는다.

### Step 4. Phase 1 — 원천 소유권·권위·관련성 및 저장 후보 매핑 (Source Mapping)

읽기 가능하거나 존재가 확인된 환경을 대상으로 컨텍스트의 성격과 소유권·쓰기 후보 성격을 매핑한다.

1. **프로젝트 귀속 판정 (`context.related_projects`)**:
   식별자, 저장소 URL, 경로, 명시적 프로젝트 태그 등 **구체적 귀속 근거**가 있을 때만 프로젝트 ID를 넣는다. 예를 들어 Notion Memory가 연결되어 있더라도 개별 항목의 프로젝트 소속 근거를 아직 확인하지 못했다면 `related_projects: []`로 두고 미확정 사유를 기록한다.
2. **권위 및 개정 기록 (`context.authority`, `context.revision`)**:
   정본 스펙·실제 코드(`canonical` / `primary`), 파생 요약(`derived`), 참고 메모(`advisory`), 개인/에이전트 기억(`personal_memory`), 미검증(`unverified`), 미상(`unknown`)을 구분하고, 커밋 SHA·문서 버전·최종 수정 시각 등 확인 가능한 개정을 `revision`에 남긴다.
3. **소유권 및 쓰기 가능성 기초 판정 (`persistence`)**:
   - 해당 환경이 원천 시스템/사용자 소유인지(`source_owned: true | false`) 기록한다.
   - 쓰기 가능 여부(`writable: true | false | "conditional" | "unknown"`)는 도구 스키마(쓰기 도구 존재 여부), 파일시스템/커넥터 권한 메타데이터, 프로젝트의 기존 문서 규약만으로 **비파괴 판정**한다. 테스트용 임시 파일이나 빈 페이지를 생성해 쓰기를 시험하지 않는다.
   - 최종 저장 정책은 후속 단계에서 결정하므로 기본값은 `storage_policy: "unresolved"`로 두되, 프로젝트 내에 이미 승인된 조사·인계 보관 위치가 발견되면 `persistence_candidates`에 근거와 함께 등록한다.

### Step 5. 산출물 조립 및 `resolve-investigation-storage` 인계

[산출물 스키마 및 판정 규칙](references/output-schemas.md)의 필수 필드를 충족하는지 자체 점검한 뒤 두 논리 산출물을 구성한다.

1. **`environment-map.json`**: 세션 실행 표면 요약 + 모든 `context_environment` 엔트리 + 미탐색/차단 표면 목록.
2. **`source-registry.json`**: 조사 대상 프로젝트별 권위 원천 매핑 + 미귀속 메모리/문서 보류 목록 + 저장소 후보(`persistence_candidates`).

> **영속화 전 주의**: 저장 위치 계약(`storage-plan.json`)이 확정되기 전이므로 원본 프로젝트 폴더에 `environment-map.json`이나 `source-registry.json` 파일을 임의로 생성하지 않는다. 이미 승인된 조사 루트 경로가 명시적으로 위임된 경우가 아니라면 세션 내 구조화 결과로 유지하고, 즉시 `resolve-investigation-storage` 스킬(`/resolve-investigation-storage`)로 넘겨 저장 위치와 조사 계획을 확정한다.

## 3. 비협상 경계 (Non-Negotiable Boundaries)

- **비파괴 읽기 전용**: Phase 0~1 탐색 중 원본 프로젝트 파일, Git 상태, 스킬/플러그인 설정, 커넥터 문서, 에이전트 메모리를 수정·생성·삭제하지 않는다.
- **Git 명령 허용 목록**: 로컬 상태를 읽는 명령만 쓴다(`status`, `log`, `branch`, `remote`, `rev-parse`, `rev-list`, `for-each-ref`, `stash list`, `worktree list`, `cat-file -e`). 원격 상태는 `git ls-remote`로만 조회한다. `fetch`, `pull`, `push`, `checkout`, `switch`, `stash`(push·pop), `gc`, `reset`, `clean`, `add`, `commit`은 탐색 중 실행하지 않는다. 원격 추적 refs를 갱신하는 명령도 금지한다.
- **원격 조회 결과는 확인 시점 스냅샷이다**: `ls-remote` 결과에 로컬에 없는 커밋이 있으면 "원격이 앞서 있음(개수 미확인)"으로 기록하고 개수를 추정하지 않는다.
- **비밀 정보 비노출**: `.env`, 인증 토큰, `~/.claude.json` 내 자격 증명 값, API 키 본문을 읽거나 산출물에 기록하지 않는다. 설정 확인 시에는 키 이름(`env_keys`)이나 활성화 상태만 기록한다.
- **범위 모드에서도 예외 없음**: 저장소 전체나 컴퓨터 전체를 대상으로 하더라도 제외 목록과 자격 증명 규칙을 우회하지 않는다. 넓은 범위에서 하는 일은 인벤토리와 판정까지이며, 심층 읽기는 승격된 대상에만 적용한다.
- **인식론적 정직성**: 미확인(`unknown`), 권한 없음(`readable: false`), 지원 안 됨, 실제 부재를 하나의 빈 값으로 합치지 않는다.
- **조기 심층 조사 금지**: 환경 지도와 저장 계획이 확정되기 전에 서브 에이전트를 병렬 디스패치하거나 최종 인계 보고서를 작성하지 않는다.
