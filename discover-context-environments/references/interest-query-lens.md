# 관심사 쿼리 렌즈 (`interest_query`)

`interest_query`는 기존 탐색 지침에 **덧붙이는** 선택 입력이다. 사용자가 준 관심 주제와 조사 목적(렌즈)으로, 이미 만든 인덱스 위에 별도 렌즈 레이어를 얹는다. 주제와 렌즈는 모두 사용자가 정한다. 이 스킬은 특정 주제나 목적을 기본값으로 갖지 않는다. 쿼리가 없으면 이 문서의 절차를 건너뛰고 커버리지 원장에 `interest_query: none`을 기록한다.

## 1. 입력 형식

호출 인자나 대화에서 `interest_query:`로 시작하는 블록을 쿼리로 받는다.

```text
interest_query:
  topics: <관심 주제, 쉼표 구분>
  lenses:
    - id: <snake_case 렌즈 ID>
      question: <이 렌즈가 답할 질문 한 문장>
      scope: item | population
      applies_to: <적용 대상 조건>
      signals: <확인할 사실 목록>
      output: <붙일 필드와 값 규칙>
  note: <선택. 조사 목적 한두 문장>
```

- `topics`만 있고 `lenses`가 없으면 주제 일치 판정(§3)만 한다.
- `topics` 없이 자유 문장만 있으면 문장에서 주제어를 뽑아 `topics`로 정리하고, 정리한 결과를 사용자에게 보여 준다.
- 렌즈를 이름이나 한 문장으로만 주면(예: "지식재산화 관점으로") §4 스키마로 정의 초안을 만들어 사용자에게 확인받은 뒤 적용한다. 확인받지 못하면 원장에 `lens_definition: drafted_unconfirmed`를 남긴다.

## 2. 기존 규칙과의 관계 (우선순위)

렌즈는 **정렬과 표시만** 바꾼다. 다음 규칙은 쿼리가 있어도 그대로다.

- 계층 A 전수 열거: 관심사와 무관한 항목도 목록에서 빼지 않는다.
- 항상 제외 경로, 자격 증명 규칙, 클라우드 동기화 폴더 제외.
- 계층 B 읽기 상한(프로젝트당 문서 최대 10개, 파일당 앞 200줄).
- 계층 C 승격은 사용자 확인 후에만 한다. 렌즈 결과만으로 자동 승격하지 않는다.
- 프로젝트 레이어(무엇인가)와 아키텍처 레이어(어떻게 구성되었나)는 따로 쓴다. 렌즈 결과는 세 번째 레이어로, 두 카탈로그에 섞지 않고 `interest-lens.md`에 쓴다.

## 3. 주제 일치 판정 (`interest_match`)

1. **매칭어 펼치기**: 주제마다 실제 파일에서 찾을 수 있는 식별자로 펼친다. 식별자 종류는 의존성·패키지 이름, 파일 확장자·데이터 형식, 설정·스키마 파일 이름, 프로젝트 안쪽 디렉터리 이름, README에 쓰일 용어다. 예를 들어 주제 "RAG"는 `langchain`, `llama-index`, 벡터 DB 클라이언트, `embedding` 같은 식별자로 펼친다. 펼친 목록은 `interest-lens.md` 맨 위에 그대로 적어, 어떤 단어로 판정했는지 남긴다.
2. **판정**:

| 값 | 기준 |
| --- | --- |
| `strong` | 자기 서술(README 제목·본문 앞부분, 매니페스트 `description`)과 구조 신호(의존성 이름, 데이터·스키마 파일 형식, 진입점) 양쪽에서 근거가 나옴 |
| `weak` | 자기 서술이나 구조 신호 중 한쪽에서만 근거가 나옴 |
| `name_only` | 폴더·저장소 이름에서만 주제어가 보임. 근거로 세지 않고 계층 B 확인 후보로만 표시 |
| `none` | 계층 B까지 읽었는데 근거가 없음 |
| `unknown` | 계층 A만 수행해 판정할 근거를 아직 읽지 않음 |

- 근거마다 실제로 연 파일 경로를 적는다. 열지 않은 파일은 근거로 쓰지 않는다.

## 4. 렌즈 정의 스키마

렌즈는 "이 목적에서 어떤 프로젝트를 먼저 봐야 하는가"를 규칙으로 정한 것이다. 각 렌즈는 아래 필드를 모두 가져야 적용할 수 있다.

| 필드 | 뜻 | 규칙 |
| --- | --- | --- |
| `id` | 렌즈 식별자 | snake_case. 출력 필드 이름의 접두어로 쓴다 |
| `question` | 렌즈가 답할 질문 | 한 문장 |
| `scope` | 계산 단위 | `item`: 항목마다 독립 판정. `population`: 전체 인덱스를 만든 뒤 분포와 비교해 판정 |
| `applies_to` | 적용 대상 | 인덱스 열과 `interest_match`만으로 쓴 조건 (예: `interest_match in [strong, weak] and owner_kind != third_party`) |
| `signals` | 확인할 사실 | 인덱스 열, 아키텍처 레이어, 계층 B에서 읽은 파일로 확인 가능한 사실만. 판단어("좋은", "유망한")는 쓰지 않는다 |
| `output` | 붙일 필드와 값 | 값마다 어떤 신호 조합에서 나오는지 규칙을 적는다. 규칙에 안 맞으면 `unknown` |

- `population` 렌즈는 모집단 조건(예: `owner_kind: own`, `noise_class: normal`, 중복 제외)과 최소 모집단 크기를 함께 적는다. 모집단이 최소 크기보다 작으면 판정하지 않고 `insufficient_population`으로 기록한다.
- 렌즈 결과는 우선순위 제안이다. 프로젝트 목적 판정이나 승격 결정으로 쓰지 않는다.

## 5. 출력 (`interest-lens.md`)

`investigations/<investigation-id>/interest-lens.md`에 쓴다. 모든 판정 문장에 `source: lens`를 붙여, 자기 서술(`source: self`)이나 추정(`source: inferred`)과 구분한다.

```text
## 쿼리
- 원문: <받은 쿼리 그대로>
- topics: <정리한 주제> / 매칭어: <펼친 목록>
- lenses: <렌즈 ID와 정의 확인 상태 (confirmed / drafted_unconfirmed)>

## 대조
- 전체 인덱스 행 수 = strong + weak + name_only + none + unknown

## 주제 일치 (strong → weak 순)
| 경로 | interest_match | 일치 주제 | 근거 경로 | <렌즈별 output 필드> |

## 렌즈별 후보
### <lens id>: <question>
- <경로>: <output 필드 값>, 근거: <신호>  source: lens

## 계층 B 확인이 필요한 항목
- name_only / unknown 항목 목록
```

- 쿼리 원문은 사용자가 준 문장만 적는다. 다른 출처(메모리, 커넥터)에서 관심사를 추정해 채우지 않는다. 메모리에 저장된 관심사를 쓰려면 사용자에게 보여 주고 확인받는다.

## 6. 쿼리 예시

아래는 형식을 보여 주는 예시다. 주제와 렌즈는 조사마다 사용자가 정한다.

### 예시 A. 지식재산화·경험 다각화 관점의 개인 작업 스캔

```text
interest_query:
  topics: 온톨로지, 지식 그래프, 시각화, RAG, 웹 에디터, 이미지·비디오 제작 파이프라인, 워크플로우 제작, 캐릭터 챗, 퀀트
  lenses:
    - id: knowledge_asset
      question: 재사용·공개할 수 있는 자산으로 정리할 작업은 무엇인가?
      scope: item
      applies_to: interest_match in [strong, weak] and owner_kind != third_party
      signals: 자산 유형(매니페스트·디렉터리 배치), 정리 공백(no_readme, no_license, no_remote, no_tests_dir, no_usage_example, unpublished_package)
      output: >-
        knowledge_asset_kind = skill | library | schema_or_ontology | dataset | visualization_tool | pipeline | design_record | documentation;
        knowledge_asset_readiness = ready(공백 0~1) | needs_packaging(2~3) | raw(4+), 공백 목록 첨부
    - id: experience_diversification
      question: 관심 주제와 이어지면서 기존 작업 분포에서 드문 경험 축을 가진 작업은 무엇인가?
      scope: population
      applies_to: owner_kind == own and noise_class == normal and duplicate_of is empty (최소 모집단 5)
      signals: 아키텍처 레이어의 언어·런타임, 역할(ui, api, data, infra, agent, cli, library), 저장·실행 형태(로컬 파일, DB, 그래프 DB, 벡터 DB, 클라우드 배포)
      output: >-
        experience_diversification_axes = 모집단의 20% 미만인 축 값 목록;
        interest_match가 strong/weak이고 축이 1개 이상이면 우선 후보
  note: 구현해 둔 작업 중 지식재산으로 정리할 것과 경험을 넓혀 주는 것을 찾고 싶다.
```

이 예시에서 펼친 매칭어 일부: 온톨로지 → `.owl`, `.ttl`, `rdflib`, SHACL 파일 / 시각화 → `d3`, `cytoscape`, `echarts` / 웹 에디터 → `prosemirror`, `tiptap`, `monaco-editor` / 이미지·비디오 → `ffmpeg`, `remotion`, `diffusers` / 워크플로우 → `n8n`, `temporal`, `langgraph`, `reactflow` / 퀀트 → `backtrader`, `vectorbt`, `ccxt`.

### 예시 B. 인계 전 위험 점검

```text
interest_query:
  topics: 결제, 인증, 배치 작업
  lenses:
    - id: handover_risk
      question: 인계 전에 먼저 확인해야 할 위험한 작업은 무엇인가?
      scope: item
      applies_to: interest_match in [strong, weak]
      signals: 테스트 디렉터리 유무, CI 설정 유무, 최근 커밋 날짜, 커밋 작성자 수, README 운영 절차 유무
      output: >-
        handover_risk_level = high(테스트·CI 모두 없음 또는 작성자 1명) | medium(테스트·CI 중 하나 없음) | low(테스트·CI 모두 있고 작성자 2명 이상)
```
