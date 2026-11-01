---
name: resolve-investigation-storage
description: >-
  탐색된 컨텍스트 환경 지도(environment-map.json, source-registry.json)와 자료 소유권·기존 관리 규칙을 바탕으로
  조사 결과의 저장 위치·권한·보존 정책(storage-plan.json)과 조사 범위·서브 에이전트 분배 계획(investigation-plan.json)을
  확정한다. 서브 에이전트 병렬 조사(Phase 3)와 인계 영속화(Phase 4) 직전의 필수 게이트(Phase 1~2)로 사용한다.
when_to_use: >-
  컨텍스트 환경 탐색 후 조사 산출물을 로컬 폴더·별도 레지스트리·커넥터 DB/문서·세션 임시 결과 중 어디에 어떤 구조로
  보존할지 결정해야 할 때, 다중 프로젝트 조사에서 개별 저장소와 통합 레지스트리를 분리해야 할 때, 또는 스킬·커넥터 메모리의
  원천을 훼손하지 않고 서브 에이전트 조사 계획을 수립해야 할 때 호출한다.
argument-hint: "[조사 식별자, 대상 프로젝트 범위, 또는 선호 저장 위치/모드]"
arguments:
  - target_scope
  - storage_preference
effort: high
license: Apache-2.0
compatibility: >-
  Claude Code CLI, IDE, Desktop, Cloud/Web 세션 지원. 특정 저장 서비스(로컬 디스크, Google Drive, Notion 등)를
  고정하지 않으며 가용 환경에 맞춰 파일시스템·커넥터 문서/DB·세션 임시 모드로 논리 구조를 매핑한다.
metadata:
  package: context-investigation
  phase: "Phase 1-2 (Storage Resolution & Investigation Persistence Plan)"
  consumes: "environment-map.json, source-registry.json"
  produces: "storage-plan.json, investigation-plan.json"
  companion_skill: discover-context-environments
---

# 조사 결과 저장 위치 결정 및 영속화 계획 (`resolve-investigation-storage`)

컨텍스트가 읽히는 장소(원천 코드, 위키, 스킬·커넥터 메모리)와 **조사 결과가 저장될 장소**는 서로 다를 수 있으며, 굳이 같게 만들 필요가 없다. 이 스킬은 특정 클라우드나 로컬 경로를 고정하지 않고 환경별 **Storage Resolution** 절차에 따라 안전한 저장 계약(`storage-plan.json`)과 조사 분배 계획(`investigation-plan.json`)을 확정한다.

## 1. 입력 전제 조건 (Pre-Gate Check)

1. 현재 세션 문맥 또는 사용자 입력(`$ARGUMENTS` / `$target_scope`)에 검증된 `environment-map.json` 및 `source-registry.json`(또는 이에 준하는 컨텍스트 환경 탐색 결과)이 존재하는지 먼저 확인한다.
2. 환경 탐색 없이 곧바로 호출된 경우, 추측으로 저장 위치를 결정하지 말고 먼저 `discover-context-environments` 스킬(`/discover-context-environments`) 절차를 실행해 가용 환경과 소유권·읽기/쓰기 후보를 확보한다.
3. 사용자가 `$storage_preference`로 특정 저장 위치나 백엔드(예: 별도 아티팩트 폴더, Notion 데이터베이스, 세션 내 출력만)를 지정했다면 아래 5대 규칙과 실제 쓰기 권한을 기준으로 수용 가능 여부를 검증한다.

## 2. Storage Resolution 5대 결정 규칙

저장 위치는 항상 다음 우선순위와 보존 원칙을 따른다.

1. **기존 승인 저장 체계 최우선 사용**:
   조사 대상 프로젝트나 워크스페이스에 이미 조사·아키텍처·인계 산출물을 보관하는 규칙과 승인된 저장 위치(예: `CLAUDE.md`나 거버넌스 문서에 명시된 인계 경로, 승인된 조사 전용 Wiki/Notion/Drive 공간)가 있고 쓰기 권한이 확인되면 이를 우선 채택한다.
2. **다중 프로젝트 조사 시 저장소 이원화**:
   여러 프로젝트를 함께 조사할 때는 **개별 프로젝트의 산출물 저장소(`per_project_stores`)**와 **조사 전체를 아우르는 통합 레지스트리(`federated_investigation_root`)**를 명시적으로 구별한다. 한 프로젝트 폴더 안에 타 프로젝트의 조사 결과를 섞어 쓰지 않는다.
3. **기존 체계 부재 시 독립 저장 후보 제안 (원본 임의 변경 금지)**:
   기존 저장 규칙이 없으면 쓰기 가능한 **독립 저장 영역**(원본 저장소 외부의 전용 작업 경로, 승인된 문서/DB 커넥터 공간 등)을 후보로 제안하고 사용자 확인을 받는다. **절대 임의로 원본 프로젝트 폴더를 변경하거나 미승인 디렉터리를 생성하지 않는다.**
4. **스킬·커넥터 메모리의 원천 보존 및 참조 기록**:
   스킬·커넥터·에이전트 메모리는 기본적으로 원천(`source_owned: true`)을 그대로 보존한다. 조사 산출물(`memory-context.md`, `source-registry.json`)에는 메모리 원천을 복제·덮어쓰지 않고 **식별자(`locator`/`record_id`), 출처(`provider`), 조회 시점/개정(`revision`), 프로젝트 관련성 판단 근거**만 참조(Reference) 형태로 기록한다.
5. **영속 저장소 미확보 시 세션 내 임시 결과 취급 (허위 완료 금지)**:
   쓰기 가능하고 승인된 영속 저장 위치를 확인하지 못한 경우에는 `persistence_mode: "ephemeral_session"`으로 결정하여 현재 세션 내 임시 결과로만 다룬다. 이 경우 **`persisted: false`로 봉인하며, 절대 "저장 완료" 상태로 표시하지 않는다.**

## 2.1 수집 저장 경로와 전달 압축 파일 (Staging Path & Delivery Archive)

이 스킬의 최종 목적은 수집한 정보를 전달 가능한 **문서 묶음**으로 만드는 것이다. 따라서 수집 결과가 어디에 모이는지와 최종 압축 파일의 위치를 항상 함께 안내한다.

1. **수집 저장 경로 (`staging_root`)**: 조사 산출물은 `<staging_root>/<investigation-id>/` 아래에 모은다. `<staging_root>`는 다음 순서로 정한다.
   - `$storage_preference`로 사용자가 지정한 경로가 있으면 그 경로를 쓴다.
   - 지정이 없으면 세션 scratchpad 디렉터리를 쓴다. 세션 범위의 임시 경로라 별도 승인 없이 쓸 수 있다.
   - 원본 프로젝트 폴더는 `staging_root`가 될 수 없다. 원본 저장소 안에 조사 파일을 만들려면 사용자 승인을 먼저 받는다.
2. **전달 압축 파일 (`delivery_archive`)**: 최종 산출물은 `<staging_root>/<investigation-id>.zip` 하나다.
   - 포함 대상은 **문서(`.md`)만**이다: `investigations/<investigation-id>/`의 네 문서와 `coverage-ledger.md`, 그리고 `handover/consolidated-report.md`. 관심사 쿼리를 적용했으면 `interest-lens.md`도 넣는다.
   - JSON 맵(`environment-map.json`, `source-registry.json`, `storage-plan.json`, `investigation-plan.json`)은 작업 상태이므로 압축에서 제외한다.
   - 원천 메모리 파일, 원본 저장소 파일, 스킬 `SKILL.md`는 압축 대상에 넣지 않는다 (Rule 4).
   - 아직 작성되지 않은 문서는 빈 파일로 채우지 않고 목록에서 뺀다. 누락 문서는 `unresolved-questions.md`와 `storage-plan.json`에 기록한다.
3. **압축 후 검증**: 생성 뒤 압축 목록을 확인하고 포함 파일이 문서만 있는지, JSON이나 원본 경로가 섞이지 않았는지 검사한다. 검사에 실패하면 `persisted: false`로 보고한다.
4. **보고 형식**: 최종 응답에는 `staging_root` 절대 경로, `delivery_archive` 절대 경로, 압축에 든 문서 목록, 누락 문서 목록을 함께 적는다.

## 2.2 커버리지 원장 (Coverage Ledger)

탐색 범위가 저장소 전체나 컴퓨터 전체처럼 넓으면, 결과 문서에 **무엇을 봤고 무엇을 안 봤는지**를 반드시 남긴다. 원장은 `investigations/<investigation-id>/coverage-ledger.md`에 둔다.

- 스캔 루트와 계층별 범위 (인벤토리 A, 관련성 판정 B, 심층 조사 C)
- 건너뛴 경로와 사유 (제외 목록, 권한 없음, 상한 초과)
- 자격 증명 파일의 존재 여부 (내용과 경로 세부는 기록하지 않음)
- 읽은 파일 목록과 각 파일의 읽은 범위 (전체 또는 앞 N줄)
- 상한에 걸려 미조사로 남긴 항목

- **대조 표 (필수)**: 발견 수, 판정별 수(`read_deep`, `read_summary`, `inventory_only`, `excluded_<사유>`), 미판정 수를 적고 합이 발견 수와 같은지 확인한다. 같지 않으면 "전체 파악 완료"로 보고하지 않는다.
- **전체 프로젝트 목록**: `project-inventory.md`의 전 항목을 원장에 연결한다. 요약 클러스터만 남기지 않는다.
- **날짜·활동 기준 필터는 목록 밖에서 적용하지 않는다.** 필터로 빠진 항목은 `excluded_<사유>`로 목록에 남긴다.

원장과 목록은 전달 압축 파일에 포함한다(§2.1). 보고서의 "전체 파악" 결론은 원장에 적힌 범위로 한정하고, 원장에 없는 영역을 조사했다고 쓰지 않는다.

## 3. 논리적 산출물 표준 구조 (Logical Output Standard)

아래 구조는 조사 전체가 공유하는 **논리적 산출물 계약**이다. 반드시 로컬 폴더 트리를 물리적으로 생성한다는 뜻이 아니며, 파일시스템·문서 관리 시스템·커넥터 기반 데이터베이스·세션 응답 블록에 동일한 논리 슬롯으로 매핑한다.

```text
investigation-root/
├── environment-map.json
├── source-registry.json
├── storage-plan.json
├── investigation-plan.json
├── investigations/
│   └── <investigation-id>/
│       ├── project-profile.md
│       ├── specification-map.md
│       ├── memory-context.md
│       └── unresolved-questions.md
└── handover/
    └── consolidated-report.md
```

백엔드 유형별(로컬/원격 파일시스템, Notion/Wiki/Drive 문서 커넥터, 구조화 DB 커넥터, 세션 임시 모드) 상세 슬롯 매핑과 다중 프로젝트 토폴로지는 [저장소 결정 매트릭스 및 백엔드 매핑 규약](references/storage-resolution-and-mapping.md)을 읽는다.

## 4. 실행 절차 (Phase 1 마감 → Phase 2 확정)

### Step 1. 저장 후보 및 소유권·권한 대조

`environment-map.json`의 `persistence` 블록과 `source-registry.json`의 `existing_storage_conventions`, `persistence_candidates`를 점검한다.

- 기존 승인 저장 위치가 있는가? 실제 `writable: true`(또는 조건 충족된 `"conditional"`)인가?
- 조사 대상이 단일 프로젝트인가, 다중 프로젝트인가?
- 메모리 환경(`type: memory`)과 절차 스킬 환경(`type: skill`)이 쓰기 대상 후보에서 제외되고 읽기·참조 전용(`reference_only_source`)으로 보호되어 있는가?

### Step 2. 저장 백엔드 결정 및 필요 시 사용자 승인 확보

- **Rule 1 충족 시**: 기존 승인 저장소를 채택하고 근거 문서를 기록한다.
- **Rule 3 해당 시 (기존 저장 체계 없음)**: 원본 프로젝트 폴더를 건드리지 않는 쓰기 가능 독립 후보(1순위 추천 포함)와 `ephemeral_session` 폴백 옵션을 명확히 제시하여 사용자 선택(`AskUserQuestion` 또는 대화 확인)을 받는다.
- **승인 불가·쓰기 도구 부재·클라우드 읽기 전용 세션 시**: 지체 없이 **Rule 5 (`ephemeral_session`)**를 채택하고 미영속 사유를 명시한다.

### Step 3. `storage-plan.json` 확정 및 `storage_policy` 갱신

선택된 백엔드에 논리적 산출물 표준 구조의 모든 슬롯을 바인딩하여 `storage-plan.json`을 작성한다.

- 각 후보 환경의 `persistence.storage_policy`를 `unresolved`에서 확정 정책(`approved_project_store`, `approved_independent_store`, `federated_registry_store`, `reference_only_source`, `ephemeral_session_only`, `excluded`)으로 갱신한다.
- 메모리 참조 규칙(`source_immutable_reference_only`), 덮어쓰기/버전 보존 규칙, 민감 정보 마스킹 규칙을 명시한다.

### Step 4. Phase 2 — 조사 범위 및 서브 에이전트 분배 계획 (`investigation-plan.json`) 수립

확정된 환경 지도와 저장 계약을 기준으로 후속 **Phase 3 (Sub-Agent Dispatch)** 및 **Phase 4 (Evidence Reconciliation & Handover)**의 실행 계약을 작성한다.

1. 각 조사 단위(`<investigation-id>`)별로 확정된 읽기 허용 원천(`allowed_source_ids`)과 미확인/차단 환경(`deferred_or_blocked_env_ids`)을 지정한다.
2. 병렬 서브 에이전트 4대 조사 트랙의 책임을 분리한다:
   - `project-profile.md`: 프로젝트 목적, 아키텍처 구조, 현재 상태, 실행·운영 구성
   - `specification-map.md`: 정본 스펙, ADR/결정 이력, 권위 계층 및 구현 정합성
   - `memory-context.md`: 스킬·커넥터·에이전트 메모리의 식별자·출처·관련성 참조 복원 (원천 변조 금지)
   - `unresolved-questions.md`: 미열람(`readable: unknown` / `false`) 환경, 권위 충돌, 추가 권한·확인이 필요한 미결 질문
3. Phase 4(`handover/consolidated-report.md`)에서 출처 권위 대조 후 **허용된 저장소에만** 영속화하는 인계 게이트 조건을 정의한다.

정확한 JSON 스키마와 Claude Code `Agent` 서브 에이전트 디스패치 패킷 규격은 [저장·조사 계획 스키마 및 서브 에이전트 디스패치 계약](references/plan-schemas-and-dispatch-contract.md)을 따른다.

### Step 5. 사전 계약 영속화(허용 시) 및 디스패치 준비 판정

- `persistence_mode`가 승인된 영속 저장소(`persisted_approved`)인 경우에만, 약속된 `investigation-root` 위치에 최상위 4종 기준 문서(`environment-map.json`, `source-registry.json`, `storage-plan.json`, `investigation-plan.json`)를 기록하고 실제 기록 성공 여부를 확인한다.
- `persistence_mode`가 `ephemeral_session`인 경우 파일/커넥터 쓰기를 수행하지 않고 세션 내 확정 계획으로 출력하며, 상태 배지를 `EPHEMERAL (미저장 — 세션 내 임시 결과)`로 표시한다.
- 수집 결과가 `staging_root`에 모였으면 §2.1 규칙대로 **문서만** 묶은 `delivery_archive`(`<investigation-id>.zip`)를 만들고 압축 목록을 검증한다. 이 단계에서 문서 본문이 아직 없으면 있는 문서만 묶고 누락을 기록한다.
- 마지막으로 **Phase 3 서브 에이전트 디스패치 가능 여부(`ready_for_subagent_dispatch: true | false`)**와 각 트랙의 입출력 경계를 요약 보고한다.

## 5. 비협상 경계 (Non-Negotiable Boundaries)

- **원본 프로젝트 무결성 보호**: 기존 승인 규칙이나 명시적 사용자 승인 없이 원본 프로젝트 디렉터리 안에 조사 폴더·파일을 생성하거나 기존 문서를 수정하지 않는다.
- **메모리 원천 불변성**: 스킬 메모리나 커넥터 메모리에 조사 중간 결과나 추론을 역기록하지 않는다. 메모리는 참조 식별자와 관련 판단 근거로만 인용한다.
- **미검증 쓰기 금지**: `writable: unknown`이거나 권한 범위가 불분명한 커넥터를 영속 저장소로 확정하지 않는다.
- **영속 상태 정직성**: 세션 임시 출력, 쓰기 실패, 부분 저장을 전체 영속 저장 완료(`persisted: true`)로 표시하지 않는다.
