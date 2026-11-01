# Context Investigation (`context-investigation`)

프로젝트·시스템 심층 조사 및 인계에 앞서 **컨텍스트 보유 환경 탐색(Context Discovery)**과 **조사 산출물 저장 위치·영속화 계획 결정(Context Persistence)**을 최상위 단계로 분리해 수행하는 독립 설치형 에이전트 스킬 패키지입니다.

단독 Git 저장소(또는 `Skills-Platform`의 Git 서브모듈)로 호스팅하여 **`npx skills add`**, **Claude Code 플러그인(`/plugin`)**, **Skills Platform 프리셋(`npx-preset`)**, **Antigravity 플러그인(`agy`)** 네 가지 표준 경로로 즉시 설치할 수 있도록 패키징되어 있습니다.

---

## 1. 패키지 구조 및 매니페스트

```text
context-investigation/
├── .claude-plugin/
│   ├── plugin.json               # Claude Code 독립 플러그인 매니페스트
│   └── marketplace.json          # 단일 저장소용 Claude Code 마켓플레이스 매니페스트
├── plugin.json                   # Google Antigravity (agy) 플러그인 매니페스트
├── package.json                  # npm / Open Agent Skills 패키지 메타데이터
├── .gitattributes                # LF 줄바꿈 결정론적 해시 고정
├── .gitignore                    # 독립 Git 저장소 / 서브모듈용 ignore 규칙
├── LICENSE                       # Apache-2.0
├── README.md
├── discover-context-environments/
│   ├── SKILL.md
│   └── references/
│       ├── environment-taxonomy-and-claude-probes.md
│       ├── interest-query-lens.md
│       └── output-schemas.md
├── resolve-investigation-storage/
    ├── SKILL.md
    └── references/
        ├── storage-resolution-and-mapping.md
        └── plan-schemas-and-dispatch-contract.md
└── analyze-undocumented-projects/
    └── SKILL.md
```

| 순서 | 스킬 | 담당 단계 | 핵심 산출물 |
| --- | --- | --- | --- |
| 1 | [`discover-context-environments`](./discover-context-environments/SKILL.md) | **Phase 0 ~ Phase 1** (Context Environment Discovery & Source Mapping) | `environment-map.json`, `source-registry.json` |
| 2 | [`resolve-investigation-storage`](./resolve-investigation-storage/SKILL.md) | **Phase 1 ~ Phase 2** (Storage Resolution & Investigation Persistence Plan) | `storage-plan.json`, `investigation-plan.json` |
| 3 | [`analyze-undocumented-projects`](./analyze-undocumented-projects/SKILL.md) | **보조 분석** (자기 서술 없는 프로젝트 목적 추정) | `project-catalog.md`의 자기 서술 없음 항목별 추정 근거와 신뢰도 |

매니페스트 파일: [`package.json`](./package.json) · [`.claude-plugin/plugin.json`](./.claude-plugin/plugin.json) · [`.claude-plugin/marketplace.json`](./.claude-plugin/marketplace.json) · [`plugin.json`](./plugin.json) · [`LICENSE`](./LICENSE)

---

## 2. 공개 경로 기준 설치 가이드

공개 GitHub 경로는 배포 형태에 따라 **독립 공개 저장소(`mineclover/context-investigation`)** 또는 **`Skills-Platform` 공개 모노레포 서브패스** 두 가지를 모두 지원합니다.

| 구분 | 공개 GitHub 경로 |
| --- | --- |
| 독립 저장소 (서브모듈 원격) | `https://github.com/mineclover/context-investigation` (`mineclover/context-investigation`) |
| 모노레포 서브패스 | `https://github.com/mineclover/Skills-Platform/tree/main/skills-packages/context-investigation` |

### A. `npx skills add` 표준 설치 (권장)

대상 프로젝트 루트 디렉터리에서 실행합니다. `.agents/skills/` 원본 배치, `.claude/skills/` 심볼릭 링크 연결, `skills-lock.json` 해시 기록이 한 번에 수행됩니다.

```bash
# 1) 독립 공개 저장소에서 설치 (Claude Code 프로젝트 로컬)
npx skills add mineclover/context-investigation --list
npx skills add mineclover/context-investigation \
  --skill discover-context-environments \
  --skill resolve-investigation-storage \
  --agent claude-code --yes

# 2) 전역(Global, ~/.claude/skills) 또는 멀티 에이전트 동시 설치
npx skills add mineclover/context-investigation \
  --skill discover-context-environments \
  --skill resolve-investigation-storage \
  --agent claude-code --agent codex --agent antigravity -g --yes

# 3) Skills-Platform 모노레포 공개 서브패스에서 직접 설치
npx skills add https://github.com/mineclover/Skills-Platform/tree/main/skills-packages/context-investigation \
  --skill discover-context-environments \
  --skill resolve-investigation-storage \
  --agent claude-code --yes

# 설치 확인 및 업데이트
npx skills list
npx skills check
npx skills update
```

### B. Claude Code 네이티브 플러그인 설치 (`/plugin`)

Claude Code 프롬프트에서 바로 실행합니다.

```text
# 1) 독립 공개 저장소에서 직접 설치
/plugin marketplace add mineclover/context-investigation
/plugin install context-investigation@context-investigation

# 2) Skills-Platform 통합 마켓플레이스에서 설치
/plugin marketplace add mineclover/Skills-Platform
/plugin install context-investigation@skills-platform
```

### C. Skills Platform `npx-preset` 일괄 설치

`Skills-Platform` CLI(`skills-catalog`)가 연결된 환경에서는 어느 작업 디렉터리에서든 프리셋 ID `context-investigation`으로 즉시 설치할 수 있습니다.

```bash
skills-catalog npx-preset plan context-investigation --table
skills-catalog npx-preset apply context-investigation --clean-zero-config --execute --table
```

---

## 3. GitHub 공개 저장소 생성 및 서브모듈 연결 절차

GitHub CLI(`gh`)가 로그인된 터미널에서 아래 명령을 실행하면 `mineclover/context-investigation` 공개 저장소 생성·푸시와 `Skills-Platform` 서브모듈 등록·커밋까지 한 번에 완료됩니다.

```bash
cd <Skills-Platform 저장소 루트>

# 1) 임시 경로로 분리 후 로컬 Git 초기화 및 첫 커밋
TMP_REPO="$(mktemp -d)/context-investigation"
mv skills-packages/context-investigation "$TMP_REPO"

git -C "$TMP_REPO" init -b main
git -C "$TMP_REPO" add .
git -C "$TMP_REPO" commit -m "feat: initial context-investigation skills package"

# 2) GitHub 공개 저장소(mineclover/context-investigation) 생성 및 main 브랜치 푸시
gh repo create mineclover/context-investigation \
  --public \
  --description "Context-bearing environment discovery & ownership-aware investigation storage resolution skills for Claude Code, Codex, and Antigravity" \
  --source "$TMP_REPO" \
  --remote origin \
  --push

# 3) Skills-Platform에 Git 서브모듈로 재등록 및 부모 저장소 공개 푸시
git submodule add -b main https://github.com/mineclover/context-investigation.git skills-packages/context-investigation
npm run check:claude-marketplace
npm run lint:links
node scripts/sync-submodules.mjs --verify

git add .gitmodules skills-packages/context-investigation skills-packages/provenance.json .claude-plugin/marketplace.json presets/context-investigation.json presets/index.json
git commit -m "feat(skills): add context-investigation submodule, marketplace entry, and preset"
git push origin main
```

---

## 4. 5단계 조사 파이프라인에서의 위치와 호출 순서

```text
Phase 0 — Context Environment Discovery      ┐
  파일시스템·도구·스킬·커넥터·메모리 접근점 탐색 │  /discover-context-environments
Phase 1 — Source & Storage Mapping           ├────────────────────────────────
  보관 위치·소유권·권위·접근 범위·저장 후보 확인 │  /resolve-investigation-storage
Phase 2 — Investigation Scope & Persistence  ┘
  조사 대상 확정 및 저장 위치·형식·보존 규칙 결정
────────────────────────────────────────────── [검증된 환경 지도 + 저장 계약 게이트]
Phase 3 — Sub-Agent Dispatch
  확정된 환경 지도와 저장 계약 기준으로 프로젝트·스펙·메모리·미결 질문 병렬 조사
Phase 4 — Evidence Reconciliation & Handover
  출처·권위 대조 및 허용된 저장소(또는 세션 임시 모드)에 인계 자료 산출
```

---

## 5. 스킬 사용 가이드

세 스킬은 **읽기 전용 상황 파악**용입니다. 프로젝트 파일, Git 상태, 설정, 메모리를 수정하거나 `fetch`, `pull`, `checkout` 같은 명령을 실행하지 않습니다. 원격 확인은 `git ls-remote`만 사용합니다.

### 5.1 실행 순서

| 순서 | 스킬 | 언제 쓰는가 | 결과 |
| --- | --- | --- | --- |
| 1 | `discover-context-environments` | 조사를 시작할 때. 어디에 무엇이 있는지 먼저 알고 싶을 때 | 프로젝트 인덱스, 환경 지도, 원천 레지스트리 |
| 2 | `analyze-undocumented-projects` | 인덱스에서 설명이 비어 있는 프로젝트의 목적을 추정할 때 | 추정 목적, 근거 경로, 신뢰도 |
| 3 | `resolve-investigation-storage` | 조사 결과를 어디에 저장할지 정하고 전달 문서를 묶을 때 | 저장 계획, 문서만 묶은 압축 파일 |

### 5.2 `discover-context-environments`

- **Step 2.0 순서**: 저장소 루트 스캔 → 프로젝트 루트 스캔 → 두 목록 대조 → 읽을 범위를 질문합니다. 범위를 정하기 전에는 내용을 읽지 않습니다.
- **기본 제외**: `ref/`(외부 참조 클론), `worktree/`, 도구 설정 디렉터리(`.claude`, `.codex` 등), `~/Library/CloudStorage/`(Google Drive, OneDrive).
- **인덱스 열**: `owner_kind`(own / third_party / unknown), `noise_class`(run_artifact, version_copy, sample_template 등), `description`, `repo_group`, `duplicate_of`.
- **중복 정리**: 같은 저장소의 하위 루트나 같은 원격 URL의 사본은 한 그룹으로 묶고, 나머지는 `duplicate_of`로 표시합니다.
- **옵시디언 볼트**: `.obsidian` 폴더를 기준으로 홈 전체에서 찾고, 노트 수와 최근 수정일을 기록합니다.

호출 예시:

```text
/discover-context-environments 홈 디렉터리 전체를 범위 모드로 조사해줘.
```

### 5.3 `analyze-undocumented-projects`

- 대상은 인덱스에서 `description`이 비어 있는 프로젝트입니다. 흩어진 폴더를 프로젝트로 볼지 판정할 때도 씁니다.
- 신뢰도 기준: 근거 3개 이상이면 `high`, 2개면 `medium`, 1개면 `low`, 0개이거나 근거끼리 충돌하면 `unknown`입니다.
- 모든 추정 항목에는 `source: inferred`를 붙여, 자기 서술에서 읽은 사실과 구분합니다.

호출 예시:

```text
/analyze-undocumented-projects 인덱스에서 description이 비어 있고 noise_class가 normal인 프로젝트의 목적을 추정해줘.
```

### 5.4 `resolve-investigation-storage`

- 원본 프로젝트 폴더에는 쓰지 않습니다. 조사 결과는 `staging_root`(기본은 세션 scratchpad)에 모읍니다.
- 최종 산출물은 `<investigation-id>.zip` 하나이며, 문서(`.md`)만 담습니다. JSON 맵, 메모리 원본, `SKILL.md`는 넣지 않습니다.
- 기존 승인 저장소나 쓰기 권한이 없으면 `ephemeral_session`으로 두고, 저장 완료로 표시하지 않습니다.

호출 예시:

```text
/resolve-investigation-storage storage_preference: <scratchpad 경로>. 문서만 묶은 압축 파일을 만들어줘.
```

### 5.5 관심사 쿼리 (`interest_query`)

기존 지침에 덧붙이는 선택 입력입니다. 관심 주제와 조사 목적(렌즈)을 주면 인덱스 위에 렌즈 레이어를 얹어 `interest-lens.md`를 따로 만듭니다. 주제와 렌즈는 조사마다 사용자가 정하며, 스킬에는 기본 주제나 기본 렌즈가 없습니다. 전수 열거, 제외 규칙, 읽기 상한, 승격 확인 규칙은 바뀌지 않습니다.

| 입력 | 내용 |
| --- | --- |
| `topics` | 관심 주제. 의존성 이름·파일 형식 같은 식별자로 펼쳐 각 항목의 `interest_match`(`strong` / `weak` / `name_only` / `none` / `unknown`)를 판정합니다 |
| `lenses` | 조사 목적. 렌즈마다 질문, 계산 단위(`item` / `population`), 적용 대상, 확인할 사실, 출력 규칙을 정합니다. 이름만 주면 정의 초안을 만들어 확인받은 뒤 적용합니다 |
| `note` | 선택. 조사 목적 한두 문장 |

호출 예시 (렌즈를 이름으로만 주는 경우):

```text
/discover-context-environments 홈 디렉터리 전체를 범위 모드로 조사해줘.
interest_query:
  topics: <관심 주제, 쉼표 구분>
  lenses: <렌즈 이름 또는 한 문장 목적>
```

- 렌즈를 완전히 정의한 예시는 [관심사 쿼리 렌즈 §6](./discover-context-environments/references/interest-query-lens.md#6-쿼리-예시)에 있습니다. 예시 A는 개인 작업에서 지식재산화·경험 다각화 후보를 찾고, 예시 B는 인계 전 위험 작업을 찾습니다.
- 폴더 이름에만 주제어가 있는 `name_only`는 근거로 세지 않습니다.
- 렌즈 결과는 우선순위 제안입니다. 렌즈 결과만으로 계층 C 심층 조사로 넘어가지 않습니다.

### 5.6 조사 결과 확인 체크리스트

- 저장소 루트 수 = 주요 저장소 수 + 제외 수 (누락 없음)
- 프로젝트 루트 수 = 인덱스 행 수
- 중복 그룹 수 + 중복 행 수 = 전체 행 수
- 제외한 경로마다 사유와 사용자 결정이 커버리지 원장에 남아 있음
- 관심사 쿼리를 썼다면 `strong` + `weak` + `name_only` + `none` + `unknown` = 인덱스 행 수
