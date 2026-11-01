# 저장·조사 계획 스키마 및 서브 에이전트 디스패치 계약

이 문서는 `resolve-investigation-storage` 스킬이 확정하는 `storage-plan.json`과 `investigation-plan.json`의 표준 스키마, 그리고 후속 **Phase 3 (Sub-Agent Dispatch)** 및 **Phase 4 (Evidence Reconciliation & Handover)**가 준수해야 할 실행 계약을 정의한다.

## 1. `storage-plan.json` 스키마 및 예시

```json
{
  "schema_version": "1.0",
  "plan_id": "splan-checkout-20261031",
  "decided_at": "2026-10-31T03:25:00+09:00",
  "resolution_rule_applied": "rule_3_approved_independent_store",
  "persistence_mode": "persisted_approved",
  "persisted": false,
  "persisted_note": "계획 수립 시점에는 false; 실제 저장소 쓰기 및 읽기 검증 완료 후에만 해당 슬롯을 true로 전환",
  "approval": {
    "status": "approved",
    "approved_by": "user_explicit",
    "scope_note": "원본 프로젝트 폴더(./checkout-service) 변경 없이 독립 경로 ../investigations/checkout-20261031 사용 승인"
  },
  "topology": {
    "multi_project": false,
    "federated_root_backend": "filesystem",
    "federated_root_locator": "../investigations/checkout-20261031",
    "per_project_overrides": {}
  },
  "memory_preservation_policy": {
    "mode": "source_immutable_reference_only",
    "write_back_to_skill_or_connector_memory": false,
    "required_reference_fields": [
      "memory_environment_id",
      "native_record_locator",
      "provider_and_owner",
      "observed_revision",
      "project_attribution_basis"
    ]
  },
  "environment_storage_policies": [
    {
      "environment_id": "env.local.checkout-repo",
      "storage_policy": "reference_only_source",
      "mutation_allowed": false
    },
    {
      "environment_id": "env.skill.cli-anything-mcp-handoff",
      "storage_policy": "reference_only_source",
      "mutation_allowed": false
    },
    {
      "environment_id": "env.memory.notion-agent-memory",
      "storage_policy": "reference_only_source",
      "mutation_allowed": false
    }
  ],
  "slot_bindings": {
    "environment-map.json": {
      "backend_type": "filesystem",
      "target_locator": "../investigations/checkout-20261031/environment-map.json",
      "writer": "orchestrator",
      "overwrite_policy": "atomic_replace"
    },
    "source-registry.json": {
      "backend_type": "filesystem",
      "target_locator": "../investigations/checkout-20261031/source-registry.json",
      "writer": "orchestrator",
      "overwrite_policy": "atomic_replace"
    },
    "storage-plan.json": {
      "backend_type": "filesystem",
      "target_locator": "../investigations/checkout-20261031/storage-plan.json",
      "writer": "orchestrator",
      "overwrite_policy": "atomic_replace"
    },
    "investigation-plan.json": {
      "backend_type": "filesystem",
      "target_locator": "../investigations/checkout-20261031/investigation-plan.json",
      "writer": "orchestrator",
      "overwrite_policy": "atomic_replace"
    },
    "investigations/checkout-service/project-profile.md": {
      "backend_type": "filesystem",
      "target_locator": "../investigations/checkout-20261031/investigations/checkout-service/project-profile.md",
      "writer": "subagent_or_orchestrator",
      "overwrite_policy": "versioned_or_replace"
    },
    "investigations/checkout-service/specification-map.md": {
      "backend_type": "filesystem",
      "target_locator": "../investigations/checkout-20261031/investigations/checkout-service/specification-map.md",
      "writer": "subagent_or_orchestrator",
      "overwrite_policy": "versioned_or_replace"
    },
    "investigations/checkout-service/memory-context.md": {
      "backend_type": "filesystem",
      "target_locator": "../investigations/checkout-20261031/investigations/checkout-service/memory-context.md",
      "writer": "subagent_or_orchestrator",
      "overwrite_policy": "versioned_or_replace"
    },
    "investigations/checkout-service/unresolved-questions.md": {
      "backend_type": "filesystem",
      "target_locator": "../investigations/checkout-20261031/investigations/checkout-service/unresolved-questions.md",
      "writer": "subagent_or_orchestrator",
      "overwrite_policy": "versioned_or_replace"
    },
    "handover/consolidated-report.md": {
      "backend_type": "filesystem",
      "target_locator": "../investigations/checkout-20261031/handover/consolidated-report.md",
      "writer": "orchestrator_phase4_only",
      "overwrite_policy": "reconciled_final"
    }
  }
}
```

> **세션 임시 모드(`ephemeral_session`)일 때의 필수 값**:
> - `"resolution_rule_applied": "rule_5_ephemeral_session_fallback"`
> - `"persistence_mode": "ephemeral_session"`
> - `"persisted": false` (절대 `true`로 변경하지 않음)
> - 모든 `slot_bindings[*].backend_type`은 `"ephemeral_session"`, `target_locator`는 `"session://current-conversation#<logical_slot>"`으로 기록한다.

---

## 2. `investigation-plan.json` 스키마 및 예시

```json
{
  "schema_version": "1.0",
  "plan_id": "iplan-checkout-20261031",
  "storage_plan_ref": "splan-checkout-20261031",
  "ready_for_subagent_dispatch": true,
  "investigations": [
    {
      "investigation_id": "checkout-service",
      "target_project_id": "checkout-service",
      "objective": "결제 서비스의 현재 코드 구조, 정본 스펙, 관련 메모리 맥락, 미결 리스크를 복원하여 인계 보고서 작성",
      "allowed_sources": [
        "src.checkout.codebase",
        "src.memory.notion-candidates"
      ],
      "excluded_or_blocked_environments": [
        {
          "environment_id": "google-drive-connector",
          "reason": "세션 내 미연결 — 외부 설계 문서 존재 여부는 unresolved-questions 트랙에서 확인 과제로 기록"
        }
      ],
      "dispatch_tracks": [
        {
          "track_id": "track.project-profile",
          "logical_output_slot": "investigations/checkout-service/project-profile.md",
          "focus": "프로젝트 목적, 디렉터리·모듈 구조, 빌드·테스트·배포 실행 상태",
          "allowed_source_ids": ["src.checkout.codebase"],
          "write_mode": "return_to_orchestrator_or_assigned_slot_only"
        },
        {
          "track_id": "track.specification-map",
          "logical_output_slot": "investigations/checkout-service/specification-map.md",
          "focus": "TypeSpec/OpenAPI/스키마 정본, ADR·아키텍처 결정, 코드와 스펙 간 드리프트",
          "allowed_source_ids": ["src.checkout.codebase"],
          "write_mode": "return_to_orchestrator_or_assigned_slot_only"
        },
        {
          "track_id": "track.memory-context",
          "logical_output_slot": "investigations/checkout-service/memory-context.md",
          "focus": "스킬·커넥터·에이전트 메모리 중 checkout-service 귀속 근거가 확인된 기억의 식별자·출처·정본 대조 결과",
          "allowed_source_ids": ["src.memory.notion-candidates", "src.checkout.codebase"],
          "write_mode": "reference_only_no_memory_mutation"
        },
        {
          "track_id": "track.unresolved-questions",
          "logical_output_slot": "investigations/checkout-service/unresolved-questions.md",
          "focus": "읽기 불가/미확인 환경, 정본과 메모리의 충돌, 인계자가 확인해야 할 의사결정 공백",
          "allowed_source_ids": ["src.checkout.codebase", "src.memory.notion-candidates"],
          "write_mode": "return_to_orchestrator_or_assigned_slot_only"
        }
      ]
    }
  ],
  "reconciliation_and_handover_gate": {
    "phase": "Phase 4",
    "output_slot": "handover/consolidated-report.md",
    "authority_precedence": [
      "canonical",
      "primary",
      "derived",
      "advisory",
      "personal_memory",
      "unverified"
    ],
    "conflict_rule": "메모리(personal_memory)나 파생 문서(derived)가 정본 스펙(canonical) 또는 실제 코드(primary)와 충돌하면 정본을 현재 상태로 채택하되, 충돌 사실과 메모리 출처 식별자를 consolidated-report.md 및 unresolved-questions.md에 모두 명시한다."
  }
}
```

---

## 3. Phase 3 & Phase 4 후속 실행 계약 (Sub-Agent Dispatch & Handover Contract)

`discover-context-environments`와 `resolve-investigation-storage` 두 상위 절차가 완료된 뒤 후속 심층 조사(서브 에이전트 또는 하위 조사 스킬)를 실행할 때는 아래 계약을 반드시 주입한다.

### 3.1 Phase 3 — Claude Code 서브 에이전트(`Agent`) 디스패치 규칙

1. **입력 경계 고정**:
   각 서브 에이전트 프롬프트에는 전체 대화를 무분별하게 넘기지 않고 해당 `track_id`, `allowed_source_ids`(및 각 원천의 `locator`, `authority`, `revision`), 할당된 `logical_output_slot`만 전달한다.
2. **원천 및 메모리 불변 강제**:
   모든 서브 에이전트는 원본 프로젝트 파일(`source_owned: true`)과 스킬·커넥터 메모리에 대해 **순수 읽기 전용**으로 동작한다.
3. **쓰기 충돌 방지**:
   - 커넥터 기반 문서/DB나 `ephemeral_session` 모드에서는 서브 에이전트가 외부 저장소에 직접 쓰지 않고 구조화된 마크다운 본문과 출처 인용 목록을 오케스트레이터에 반환하게 한 뒤, 오케스트레이터가 `storage-plan.json`에 따라 일괄 기록한다.
   - 파일시스템 백엔드(`persisted_approved`)에서도 각 서브 에이전트는 자신에게 할당된 단일 `target_locator` 외에는 어떤 파일도 수정할 수 없다.

### 3.2 Phase 4 — 증거 대조 및 인계 영속화 (`Evidence Reconciliation & Handover`)

1. **권위 계층 대조**:
   4개 트랙 결과를 합칠 때 `canonical`/`primary`(코드·정본 스펙)와 `personal_memory`(메모리) 또는 `advisory`(절차 스킬)를 동급 사실로 섞지 않는다.
2. **미확인 영역 투명 공개**:
   `access.readable`이 `false`이거나 `"unknown"`으로 남은 환경(예: 미연결 Google Drive, 권한 없는 사내 Wiki)은 "문서 없음"이 아니라 **"이번 세션에서 접근·확인하지 못한 환경"**으로 `consolidated-report.md`에 명시한다.
3. **영속화 상태 최종 판정**:
   - `storage-plan.json`의 허용된 슬롯에 실제 쓰기와 사후 확인이 모두 성공했을 때만 `persisted: true`와 저장 위치(`target_locator`)를 보고한다.
   - `ephemeral_session`이거나 일부 슬롯 쓰기가 실패한 경우에는 각각 `EPHEMERAL (세션 임시 결과)` 또는 `PARTIAL_PERSISTENCE`로 정확히 보고하여 소비자가 보존 상태를 오판하지 않게 한다.
