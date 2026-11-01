# 산출물 스키마 및 판정 규칙 (`environment-map.json` / `source-registry.json`)

이 문서는 `discover-context-environments` 스킬이 생성하고 `resolve-investigation-storage` 스킬이 소비하는 두 정규 산출물의 데이터 계약과 인식론적 판정 기준을 정의한다.

## 1. 정규 `context_environment` 엔트리 스키마

모든 탐색된 컨텍스트 후보 환경은 최소한 아래 필드를 빠짐없이 기록해야 한다.

```yaml
context_environment:
  id: <environment-id>                  # 예: env.local.repo-root, env.connector.notion-workspace, env.memory.claude-project
  type: local | remote | skill | connector | memory | runtime
  surface_category: local_workspace | remote_dev | document_knowledge | agent_config | skill | connector | memory | runtime_ops
  provider: <provider-or-runtime>       # 예: local-fs, git-remote-github, claude-code, mcp:notion, claude.ai:Google-Drive
  locator: <path-or-native-reference>   # 상대/절대 경로, 저장소 URI, MCP 리소스 URI, 커넥터 워크스페이스/DB ID 등

  access:
    discovered: true
    readable: true | false | partial | unknown
    authorized_scope: <scope-descriptor | unknown>
    probe_note: <optional-short-evidence-or-denial-reason>

  context:
    kind: project | specification | decision | memory | procedure | execution
    related_projects: []                # 귀속 근거가 확인된 project_id 문자열 배열 (미확인 시 빈 배열)
    attribution_status: confirmed | candidate | unattributed | global_shared
    authority: canonical | primary | derived | advisory | personal_memory | unverified | unknown
    revision: <commit-sha | iso-timestamp | version-tag | unknown>

  persistence:
    source_owned: true | false
    writable: true | false | conditional | unknown
    storage_policy: unresolved          # Phase 0~1 기본값은 unresolved; resolve-investigation-storage에서 확정
```

### 1.1 인식론적 필드 판정 기준 (합침 금지)

| 필드 | 허용 값 | 판정 기준 |
| --- | --- | --- |
| `access.discovered` | `true` | 설정, 도구 목록, 파일 트리 등에서 접근점의 존재가 관측됨. (관측되지 않은 부재 표면은 `unobserved_surfaces`에 별도 요약) |
| `access.readable` | `true` / `false` / `"partial"` / `"unknown"` | 비파괴 읽기 프로브가 성공하면 `true`, 일부 하위 범위만 열리면 `"partial"`, 권한 거부·비활성이면 `false`, 아직 프로브하지 않았으면 `"unknown"` |
| `access.authorized_scope` | 문자열 또는 `"unknown"` | 읽기 허용이 확인된 구체적 경계(예: `"repo:mineclover/Skills-Platform (read)"`, `"notion:team-space-eng"`, `"unknown"`) |
| `context.authority` | `canonical` / `primary` / `derived` / `advisory` / `personal_memory` / `unverified` / `unknown` | `canonical`(SSOT 스펙·계약), `primary`(실제 동작 코드·승인된 ADR), `derived`(생성된 요약·캐시), `advisory`(가이드·절차 스킬), `personal_memory`(에이전트/개인 메모리), `unverified`(출처 불명·미대조), `unknown`(미열람) |
| `persistence.source_owned` | `true` / `false` | 원본 코드 저장소, 운영 중인 위키 원천, 스킬/커넥터의 고유 메모리처럼 **원천 소유자의 정본 자산**이면 `true`, 조사 산출물 전용 스크래치·독립 아티팩트 공간이면 `false` |
| `persistence.writable` | `true` / `false` / `"conditional"` / `"unknown"` | 비파괴 메타데이터·도구 스키마·권한 설정으로 확인한 쓰기 가능성. 사용자 승인이나 특정 경로 제한이 조건이면 `"conditional"` |

---

## 2. `environment-map.json` 구조

현재 세션에서 발견된 모든 컨텍스트 보유 환경의 전체 지도다.

```json
{
  "schema_version": "1.0",
  "generated_at": "2026-10-31T03:20:00+09:00",
  "target_scope_input": "checkout-service 및 관련 결제 스펙",
  "runtime_surface": {
    "host": "claude-code",
    "local_filesystem_available": true,
    "shell_available": true,
    "subagent_dispatch_available": true,
    "mcp_tools_detected": ["mcp__notion", "mcp__github"],
    "unobserved_or_absent_surfaces": [
      {
        "surface": "google-drive-connector",
        "reason": "현재 세션에 연결된 Google Drive MCP 또는 claude.ai 커넥터가 없음"
      }
    ]
  },
  "target_projects": [
    {
      "project_id": "checkout-service",
      "discovery_basis": "사용자 인자 및 로컬 저장소 package.json name 일치"
    }
  ],
  "context_environments": [
    {
      "id": "env.local.checkout-repo",
      "type": "local",
      "surface_category": "local_workspace",
      "provider": "local-fs",
      "locator": ".",
      "access": {
        "discovered": true,
        "readable": true,
        "authorized_scope": "workspace-root (read/write capable, source-owned)",
        "probe_note": "루트 매니페스트, CLAUDE.md, git HEAD 확인 완료"
      },
      "context": {
        "kind": "project",
        "related_projects": ["checkout-service"],
        "attribution_status": "confirmed",
        "authority": "primary",
        "revision": "git:9f3c1a2"
      },
      "persistence": {
        "source_owned": true,
        "writable": "conditional",
        "storage_policy": "unresolved"
      }
    },
    {
      "id": "env.skill.cli-anything-mcp-handoff",
      "type": "skill",
      "surface_category": "skill",
      "provider": "claude-code-skills",
      "locator": ".claude/skills/cli-anything-mcp-handoff/SKILL.md",
      "access": {
        "discovered": true,
        "readable": true,
        "authorized_scope": "package-internal skill files"
      },
      "context": {
        "kind": "procedure",
        "related_projects": [],
        "attribution_status": "global_shared",
        "authority": "advisory",
        "revision": "unknown"
      },
      "persistence": {
        "source_owned": true,
        "writable": false,
        "storage_policy": "unresolved"
      }
    },
    {
      "id": "env.memory.notion-agent-memory",
      "type": "memory",
      "surface_category": "memory",
      "provider": "mcp:notion",
      "locator": "notion://database/agent-memory-db",
      "access": {
        "discovered": true,
        "readable": true,
        "authorized_scope": "shared-agent-memory-db (read-only verified)",
        "probe_note": "DB 스키마 조회 성공, 개별 항목의 checkout-service 귀속 여부는 태그/링크 대조 필요"
      },
      "context": {
        "kind": "memory",
        "related_projects": [],
        "attribution_status": "candidate",
        "authority": "personal_memory",
        "revision": "2026-10-30T18:00:00Z"
      },
      "persistence": {
        "source_owned": true,
        "writable": "unknown",
        "storage_policy": "unresolved"
      }
    }
  ]
}
```

---

## 3. `source-registry.json` 구조

Phase 1 매핑 산출물로서, `environment-map.json`의 환경들 중 실제 컨텍스트 원천들의 **소유권·권위 계층·프로젝트 귀속 근거**와 **조사 결과 저장소 후보(`persistence_candidates`)**를 정리한다.

```json
{
  "schema_version": "1.0",
  "generated_at": "2026-10-31T03:20:00+09:00",
  "sources": [
    {
      "source_id": "src.checkout.codebase",
      "environment_id": "env.local.checkout-repo",
      "locator": ".",
      "owner": "checkout-service repository",
      "context_kind": "project",
      "related_projects": ["checkout-service"],
      "attribution_evidence": "git remote origin 및 루트 CLAUDE.md 프로젝트 선언 일치",
      "authority": "primary",
      "revision": "git:9f3c1a2",
      "read_mode": "direct_read",
      "mutation_allowed": false
    },
    {
      "source_id": "src.memory.notion-candidates",
      "environment_id": "env.memory.notion-agent-memory",
      "locator": "notion://database/agent-memory-db",
      "owner": "Notion Agent Memory DB",
      "context_kind": "memory",
      "related_projects": [],
      "attribution_evidence": "미확정 — Phase 3 memory-context 조사에서 project='checkout-service' 또는 저장소 URL 참조 항목만 선별 필요",
      "authority": "personal_memory",
      "revision": "2026-10-30T18:00:00Z",
      "read_mode": "reference_only",
      "mutation_allowed": false
    }
  ],
  "existing_storage_conventions": [
    {
      "found": false,
      "checked_locations": ["CLAUDE.md", "AGENTS.md", "docs/"],
      "note": "프로젝트 내 기존 조사·인계 전용 승인 디렉터리 규약이 선언되어 있지 않음"
    }
  ],
  "persistence_candidates": [
    {
      "candidate_id": "cand.independent-local-workspace",
      "environment_id": "env.local.checkout-repo",
      "backend_type": "filesystem",
      "proposed_locator": "../investigations/checkout-service-20261031",
      "touches_source_project": false,
      "writable_status": "unknown",
      "existing_convention_match": false,
      "requires_user_approval": true,
      "rationale": "원본 프로젝트 폴더를 변경하지 않는 독립 로컬 저장 후보 (상위 디렉터리 쓰기 권한 및 사용자 승인 확인 필요)"
    },
    {
      "candidate_id": "cand.ephemeral-session",
      "environment_id": null,
      "backend_type": "ephemeral_session",
      "proposed_locator": "session://current-conversation",
      "touches_source_project": false,
      "writable_status": true,
      "existing_convention_match": false,
      "requires_user_approval": false,
      "rationale": "영속 저장소 미승인·미확보 시 안전 폴백 (저장 완료 상태로 표시하지 않음)"
    }
  ]
}
```

---

## 4. 완료 자가 점검 체크리스트

`resolve-investigation-storage` 스킬로 넘기기 전 아래 항목을 확인한다.

- [ ] 로컬 파일시스템뿐 아니라 에이전트 설정, 설치된 스킬, 연결된 커넥터, 메모리, 실행 환경을 모두 점검했는가?
- [ ] 스킬의 절차적 지식(`type: skill`, `kind: procedure`)과 메모리(`type: memory`, `kind: memory`)가 서로 다른 `id`로 분리되어 있는가?
- [ ] 프로브하지 않은 커넥터나 문서를 `readable: true`로 추측 기재하지 않았는가?
- [ ] 프로젝트 귀속 근거가 확인되지 않은 메모리·커넥터 데이터를 `related_projects`에 임의로 넣지 않았는가?
- [ ] 탐색 과정에서 원본 프로젝트나 메모리에 어떠한 파일/레코드 생성·수정도 발생하지 않았는가?
- [ ] `interest_query`가 있었다면 `interest-lens.md`의 일치 판정 합계가 인덱스 행 수와 같고, 렌즈 점수만으로 계층 C 승격을 하지 않았는가?
