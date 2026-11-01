# 저장소 결정 매트릭스 및 백엔드 매핑 규약

이 문서는 `resolve-investigation-storage` 스킬이 다양한 Claude Code 실행 환경(로컬 워크스페이스, 다중 저장소, 커넥터 전용 클라우드 세션, 쓰기 불가 세션)에서 논리적 산출물 구조를 일관되게 매핑하는 규칙을 정의한다.

## 1. Storage Resolution 의사결정 흐름

```text
[입력: environment-map.json + source-registry.json]
  │
  ├─ 1. 프로젝트에 기존 승인된 조사·인계 저장 규칙이 있고 writable == true 인가?
  │     ├─ 예 ──► [Rule 1] 기존 승인 저장 체계 채택 (approved_project_store)
  │     │         (다중 프로젝트면 [Rule 2]에 따라 개별 저장소 vs 통합 레지스트리 분리 확인)
  │     └─ 아니오
  │          │
  ├─ 2. 원본 프로젝트를 건드리지 않는 쓰기 가능 독립 저장 후보가 존재하는가?
  │     ├─ 예 ──► [Rule 3] 독립 저장 후보를 제안하고 사용자 승인 요청
  │     │         ├─ 승인됨 ──► 독립 영속 저장소 채택 (approved_independent_store)
  │     │         └─ 보류/거절 ─► [Rule 5] 세션 내 임시 결과 모드로 전환 (ephemeral_session)
  │     └─ 아니오
  │          │
  └─ 3. 적합한 영속 저장 위치 부재 ──► [Rule 5] 세션 내 임시 결과 모드 확정
                                      (persisted: false 고정, 저장 완료 표시 금지)
```

### `persistence.storage_policy` 확정 값 사전

`discover-context-environments`에서 `"unresolved"`로 넘어온 각 환경의 `storage_policy`는 다음 중 하나로 마감한다.

| 확정 `storage_policy` 값 | 의미 | 쓰기 허용 여부 |
| --- | --- | --- |
| `approved_project_store` | Rule 1에 따라 기존 프로젝트 내 승인된 조사/인계 경로로 채택됨 | 지정된 하위 경로/리소스에만 허용 |
| `approved_independent_store` | Rule 3에 따라 사용자 승인을 받은 독립 저장 영역(별도 폴더, 전용 문서/DB) | 지정된 `investigation-root`에만 허용 |
| `federated_registry_store` | Rule 2 다중 프로젝트 조사에서 통합 레지스트리(`environment-map`, `source-registry`, `consolidated-report`)를 보관하는 상위 저장소 | 통합 산출물 슬롯에만 허용 |
| `reference_only_source` | Rule 4 메모리 원천, 스킬 절차 문서, 원본 소스 코드 등 읽기·참조만 허용되는 원천 자산 | **절대 쓰기 금지** |
| `ephemeral_session_only` | Rule 5에 따라 영속 백엔드를 쓰지 않고 현재 대화 세션 내에만 산출 | 외부 저장소 쓰기 없음 |
| `excluded` | 권한 없음, 프로젝트 무관, 또는 비활성화로 이번 조사 범위에서 제외됨 | 접근·쓰기 모두 금지 |

---

## 2. 다중 프로젝트 조사의 저장소 분리 패턴 (Rule 2)

두 개 이상의 프로젝트(`proj-alpha`, `proj-beta` 등)를 조사할 때는 소유권 충돌과 맥락 오염을 막기 위해 반드시 다음 두 패턴 중 하나를 명시한다.

### 패턴 A — 독립 통합 루트 단일화 (권장 기본값)

원본 저장소들과 분리된 상위 독립 조사 루트 하나에 통합 레지스트리와 프로젝트별 파티션을 함께 둔다.

- 통합 메타·인계: `<federated-root>/{environment-map,source-registry,storage-plan,investigation-plan}.json` 및 `<federated-root>/handover/consolidated-report.md`
- 프로젝트별 산출물:
  - `<federated-root>/investigations/proj-alpha/{project-profile,specification-map,memory-context,unresolved-questions}.md`
  - `<federated-root>/investigations/proj-beta/{project-profile,specification-map,memory-context,unresolved-questions}.md`

### 패턴 B — 프로젝트별 고유 승인 저장소 + 통합 인덱스 연결

각 프로젝트가 이미 자체 승인된 인계 보관소(`proj-alpha/docs/handover/`, `proj-beta` 전용 Wiki 공간 등)를 보유한 경우:

- 개별 프로젝트 상세 조사본(`project-profile.md` ~ `unresolved-questions.md`)은 각 프로젝트의 승인 저장소에 기록한다.
- 전체 조사 통합본(`environment-map.json`, `source-registry.json`, `storage-plan.json`, `investigation-plan.json`, `handover/consolidated-report.md`)은 별도 통합 레지스트리 위치(또는 승인된 대표 허브)에 두고 각 프로젝트 저장소의 `locator`와 `revision`을 양방향 참조로 연결한다.

---

## 5. 수집 저장 경로와 전달 압축 파일 규약 (Staging Path & Delivery Archive)

전달용 압축 파일은 **문서만** 담고, 작업 상태(JSON 맵)와 원천 자산은 담지 않는다.

| 항목 | 경로 규약 | 포함 여부 |
| --- | --- | --- |
| 수집 루트 (`staging_root`) | `$storage_preference` → 세션 scratchpad 순서로 결정 | 해당 없음 |
| 조사 디렉터리 | `<staging_root>/<investigation-id>/` | 해당 없음 |
| 전달 압축 파일 (`delivery_archive`) | `<staging_root>/<investigation-id>.zip` | 문서만 |
| 압축 대상 문서 | `investigations/<id>/{project-profile,specification-map,memory-context,unresolved-questions}.md`, `handover/consolidated-report.md` | 포함 |
| 작업 상태 JSON | `environment-map.json`, `source-registry.json`, `storage-plan.json`, `investigation-plan.json` | 제외 |
| 원천 자산 | 원본 저장소, 메모리 원천, 스킬 `SKILL.md` | 제외 (참조만) |

압축은 위 목록의 파일만 지정해 만들고(디렉터리 통째로 묶지 않는다), 생성 후 압축 목록을 읽어 포함 파일이 문서 경로와 정확히 일치하는지 확인한다. 존재하지 않는 문서는 목록에서 뺀다.

---

## 3. 논리적 산출물 구조의 백엔드별 매핑

논리적 경로(`logical_slot`)는 백엔드가 파일시스템이 아니어도 **동일한 슬롯 이름과 역할**을 유지한다.

| 논리 슬롯 (`logical_slot`) | 파일시스템 백엔드 | 문서·위키 커넥터 (Notion / Drive / Confluence) | 데이터베이스·시트 커넥터 (Notion DB / Sheets) | 세션 임시 모드 (`ephemeral_session`) |
| --- | --- | --- | --- | --- |
| `environment-map.json` | `<root>/environment-map.json` | 루트 페이지 하위 `Environment Map` 코드 블록 또는 첨부 JSON | `slot="environment-map"`, `format="json"` 레코드 | 응답 섹션 `[Slot: environment-map.json]` |
| `source-registry.json` | `<root>/source-registry.json` | 루트 페이지 하위 `Source Registry` 코드 블록/테이블 | `slot="source-registry"`, `format="json"` 레코드 | 응답 섹션 `[Slot: source-registry.json]` |
| `storage-plan.json` | `<root>/storage-plan.json` | 루트 페이지 하위 `Storage Plan` 코드 블록 | `slot="storage-plan"`, `format="json"` 레코드 | 응답 섹션 `[Slot: storage-plan.json]` |
| `investigation-plan.json` | `<root>/investigation-plan.json` | 루트 페이지 하위 `Investigation Plan` 코드 블록 | `slot="investigation-plan"`, `format="json"` 레코드 | 응답 섹션 `[Slot: investigation-plan.json]` |
| `investigations/<id>/project-profile.md` | `<root>/investigations/<id>/project-profile.md` | 하위 페이지 `[<id>] Project Profile` | `investigation_id=<id>`, `slot="project-profile"` | 응답 섹션 `[<id>/project-profile.md]` |
| `investigations/<id>/specification-map.md` | `<root>/investigations/<id>/specification-map.md` | 하위 페이지 `[<id>] Specification Map` | `investigation_id=<id>`, `slot="specification-map"` | 응답 섹션 `[<id>/specification-map.md]` |
| `investigations/<id>/memory-context.md` | `<root>/investigations/<id>/memory-context.md` | 하위 페이지 `[<id>] Memory Context` | `investigation_id=<id>`, `slot="memory-context"` | 응답 섹션 `[<id>/memory-context.md]` |
| `investigations/<id>/unresolved-questions.md` | `<root>/investigations/<id>/unresolved-questions.md` | 하위 페이지 `[<id>] Unresolved Questions` | `investigation_id=<id>`, `slot="unresolved-questions"` | 응답 섹션 `[<id>/unresolved-questions.md]` |
| `handover/consolidated-report.md` | `<root>/handover/consolidated-report.md` | 루트 직속 페이지 `Consolidated Handover Report` | `slot="handover/consolidated-report"` | 응답 섹션 `[handover/consolidated-report.md]` |

---

## 4. 스킬·커넥터 메모리 참조 봉투 규약 (Rule 4 — Memory Reference Envelope)

`memory-context.md`와 `source-registry.json`에서 에이전트·스킬·커넥터 메모리를 기록할 때는 원천 데이터를 통째로 덤프하거나 원천 메모리를 수정하는 대신 아래 **참조 봉투(Reference Envelope)** 항목을 기록한다.

1. **`memory_environment_id`**: `environment-map.json`에 등록된 메모리 환경 `id` (예: `env.memory.notion-agent-memory`)
2. **`native_record_locator`**: 원천 시스템 내 고유 식별자 (예: Notion page/block ID, `MEMORY.md#L12-L28`, 메모리 그래프 노드 키)
3. **`provider_and_owner`**: 메모리를 관리하는 주체(스킬명, 커넥터명, 사용자/프로젝트 범위)
4. **`observed_revision`**: 조회 시점 타임스탬프 또는 버전 해시
5. **`project_attribution_basis`**: 이 기억이 왜 대상 프로젝트와 관련된다고 판단했는지(또는 왜 보류했는지)에 대한 근거와 정본 코드/문서와의 일치 여부(`corroborated_by_primary` | `uncorroborated_memory` | `contradicts_primary`)
6. **`source_preservation`**: 항상 `"untouched"` — 원천 메모리 저장소에 역쓰기하지 않았음을 명시
