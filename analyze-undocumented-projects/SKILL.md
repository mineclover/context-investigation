---
name: analyze-undocumented-projects
description: >-
  자기 서술(README 제목, package.json description 등)이 없는 프로젝트의 목적과 상태를 구조·매니페스트·
  커밋 이력의 증거로 추정하고, 추정 근거와 신뢰도를 함께 기록한다. 프로젝트 카탈로그에서
  "자기 서술 없음"으로 분류된 항목을 대상으로 사용한다.
when_to_use: >-
  project-catalog.md에 "자기 서술 없음" 항목이 있고 그 목적을 알아야 할 때, 또는 저장소 없이 흩어진
  폴더를 프로젝트로 판정해야 할 때 호출한다.
argument-hint: "[프로젝트 경로 또는 project-catalog.md의 자기 서술 없음 목록]"
arguments:
  - target_paths
effort: high
license: Apache-2.0
compatibility: >-
  파일 읽기와 셸(읽기 전용 git 명령)을 가정한다. 프로젝트를 설치·빌드·실행하지 않는다.
metadata:
  package: context-investigation
  phase: "Phase 0-1 보조 (Undocumented Project Inference)"
  consumes: "project-catalog.md (자기 서술 없음 항목)"
  produces: "undocumented-project-profiles.md"
  companion_skill: discover-context-environments
---

# 자기 서술 없는 프로젝트 분석 (`analyze-undocumented-projects`)

프로젝트 카탈로그에서 목적을 스스로 밝히지 않는 항목은 이 스킬로 분석한다. 목적을 **추정**하는 것이므로, 모든 주장에 근거 경로와 신뢰도를 붙인다. 추정 결과를 자기 서술처럼 쓰지 않는다.

## 1. 원칙

- **읽기 전용**: 의존성 설치, 빌드, 테스트 실행, 코드 실행을 하지 않는다. git 명령도 `log`, `status`, `remote`처럼 읽기 전용만 쓴다.
- **추정 표시**: 결과의 모든 목적 문장에 `source: inferred`를 붙인다. 근거 없이 목적을 쓰지 않는다.
- **신뢰도 구분**: `high`(서로 독립된 증거 3개 이상이 같은 결론), `medium`(증거 2개), `low`(증거 1개), `unknown`(증거 없음 또는 충돌). 증거가 충돌하면 `unknown`으로 두고 충돌 내용을 적는다.
- **자격 증명 비노출**: `.env*`, 키 파일, 인증 설정은 존재 여부만 기록하고 내용은 열지 않는다.
- **대상 밖 확장 금지**: 하위 패키지는 `subpackages.tsv`에서 역할을 가져오고, 필요하면 한 단계만 본다.

## 2. 증거 순서

증거는 다음 순서로 모은다. 앞 단계에서 결론이 나도 뒤 단계로 교차 확인한다.

1. **폴더 구조**: 최상위 디렉터리 이름과 배치 (`src/`, `app/`, `api/`, `docs/`, `skills/`, `tests/`, `scripts/` 등).
2. **매니페스트**: `package.json`의 `scripts`, `bin`, `main`, `dependencies` 이름. `pyproject.toml`의 `[project]`, `Cargo.toml`의 `[package]`, `go.mod`의 모듈 경로.
   - 의존성 이름은 프레임워크와 역할을 드러낸다(예: 웹 프레임워크, DB 클라이언트, MCP SDK, 에이전트 SDK).
3. **진입점**: `main.*`, `index.*`, `app.*`, `server.*`, `cmd/`, `__main__.py`, `bin/`의 첫 몇 줄.
4. **설정·배포 신호**: `Dockerfile`, `docker-compose*`, `wrangler.toml`, `vercel.json`, CI 설정 파일 이름.
5. **커밋 이력**: 최근 20개 커밋 제목(`git log -20 --format=%s`). 커밋 제목은 목적의 간접 증거다. 제목 하나만으로 목적을 결정하지 않는다.
6. **부가 문서**: `docs/`, `CHANGELOG`, 라이선스 파일의 저작권 문장(주체 확인용). 여기서 얻은 문장은 인용 한 문장 이내로만 쓴다.

## 3. 출력 형식 (`undocumented-project-profiles.md`)

프로젝트마다 다음 항목을 한 블록으로 적는다.

```text
### <경로>
- 목적 (추정): <한 문장>  source: inferred  confidence: high | medium | low | unknown
- 근거:
  - <증거 종류>: <경로 또는 값>
  - <증거 종류>: <경로 또는 값>
- 충돌·공백: <없으면 "없음">
- 상태 신호: <최근 커밋 날짜, 원격 여부>
- 아키텍처 참조: architecture-catalog.md의 해당 행 (구성은 여기서 다시 쓰지 않는다)
```

맨 위에 대조 표를 둔다: 대상 수 = `high` + `medium` + `low` + `unknown`.

`interest_query`가 함께 주어졌으면 블록에 `- 관심사 일치: <interest_match>, <일치 주제>  source: lens` 한 줄을 더한다. 판정 기준은 `discover-context-environments`의 [관심사 쿼리 렌즈](../discover-context-environments/references/interest-query-lens.md)를 따른다. 추정 목적만 근거로 한 일치는 `weak`을 넘지 않고, 목적 신뢰도가 `unknown`이면 일치도 `unknown`으로 둔다.

## 4. 금지

- 추정 목적을 README나 package.json에서 온 것처럼 쓰지 않는다.
- 근거 없이 "~로 보인다"만 쓰고 끝내지 않는다. 증거가 없으면 `unknown`이다.
- 프로젝트 파일을 수정하거나 생성하지 않는다. 결과는 조사 산출물 폴더에만 쓴다.
- 자격 증명 값을 옮겨 적지 않는다.
