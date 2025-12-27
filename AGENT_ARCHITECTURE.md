# IMO25 Agent Architecture Documentation

이 문서는 `code/agent.py`에 구현된 AI 에이전트의 작동 원리, 소통 방식, 그리고 진화(Self-Correction) 메커니즘을 상세히 설명합니다.

## 1. 핵심 아키텍처: Solver-Verifier Loop

이 시스템은 복잡한 다중 에이전트 통신 프로토콜을 사용하는 대신, **단일 LLM 모델이 두 가지 역할(Solver, Verifier)을 번갈아 수행**하는 **"생성-검증 루프(Generation-Verification Loop)"**를 기반으로 합니다.

### 작동 흐름 요약

1. **Solver (해결사)**: 문제를 풀고 엄밀한 증명을 생성합니다.
2. **Verifier (검증관)**: 생성된 풀이를 비판적으로 검토하고 오류를 지적합니다.
3. **Correction (교정)**: 지적된 오류를 바탕으로 풀이를 수정합니다.
4. **Stability Check (안정성 검증)**: 수정된 풀이가 5회 연속으로 검증을 통과할 때까지 2~4 과정을 반복합니다.

---

## 2. 소통 및 기억 메커니즘 (Communication & Memory)

API 기반 LLM은 기본적으로 상태(State)가 없습니다. 따라서 에이전트는 **대화 내역(Context)을 명시적으로 조립**하여 API에 전송함으로써 "기억"을 유지하고 소통합니다.

### 프롬프트 페이로드 구성 (`build_request_payload`)

에이전트가 진화할 때 전송하는 프롬프트(`p1`)는 다음과 같이 **3단계 레이어**로 구성됩니다.

| 순서 | 역할 (Role) | 내용 (Content) | 비고 |
| :-- | :--- | :--- | :--- |
| **1** | `user` | **문제 정의** (`problem_statement`) <br> **초기 지시** (`step1_prompt`) | - 시스템 프롬프트(Google Gemini) 또는 첫 메시지로 주입 <br> - 수학적 엄밀성, TeX 사용 등 기본 규칙 정의 |
| **2** | `model` | **이전 풀이** (`solution`) | - **강제 주입(Injection)** <br> - LLM에게 "네가 과거에 이렇게 답했다"라고 인식시킴 |
| **3** | `user` | **검증 피드백** (`verify`) <br> **수정 지시** (`correction_prompt`) | - **현재의 피드백** <br> - "방금 그 풀이는 틀렸다. ~때문이다. 다시 풀어라" |

이 구조를 통해 LLM은 **[문제 -> 내 풀이 -> 비평 -> 수정 지시]**의 흐름을 하나의 긴 대화로 인식하고, 비평이 반영된 **"수정된 풀이"**를 생성하게 됩니다.

---

## 3. 진화 및 자기 교정 프로세스 (Evolution Process)

`agent()` 함수의 `for` 루프(최대 30회) 내에서 에이전트의 진화가 이루어집니다.

### 3.1. 초기 탐색 (`init_explorations`)

- **입력**: 문제 텍스트.
- **동작**:
    1. 첫 번째 풀이를 생성합니다.
    2. 즉시 `self_improvement_prompt`("검토하고 개선하라")를 보내어 1차적으로 다듬습니다.
    3. `verify_solution`을 호출하여 초기 검증을 수행합니다.

### 3.2. 검증 실패 시 교정 루프 (`Verification does not pass`)

검증 결과(`good_verify`)가 "no"일 경우(오류 발견), 다음 절차를 수행합니다:

1. **상태 초기화**: `correct_count` = 0.
2. **컨텍스트 조립**:
    - `p1` 객체(대화 내역)에 `solution`(방금 틀린 답)을 `model` 역할로 추가.
    - `verify`(비평 내용)와 `correction_prompt`를 `user` 역할로 추가.
3. **재요청**: 조립된 `p1`을 API로 전송.
4. **진화**: API는 비평을 반영하여 **새로운 Solution**을 반환.

### 3.3. 안정성 필터 (The 5-Verify Rule)

풀이가 "맞는지" 확신하기 위해, 에이전트는 **5번 연속 정답 판정**을 요구합니다.

- **조건**: `correct_count >= 5`
- **이유**: LLM은 확률적이어서, 틀린 풀이도 우연히 "맞다"고 할 수 있습니다. 이를 방지하기 위해 같은 풀이를 반복하여 검증관(Verifier)에게 보여줍니다.
- **동작**:
  - 검증 통과("yes") -> `correct_count` + 1.
  - 검증 실패("no") -> `correct_count` 리셋, 다시 **3.2 교정 루프**로 진입.

---

## 4. 주요 함수 및 변수 설명

### 주요 함수 (`code/agent.py`)

| 함수명 | 설명 | 핵심 로직 |
| :--- | :--- | :--- |
| `agent(...)` | 메인 루프 | - 30회 반복(Iteration) <br> - 교정(Evolution)과 검증(Verification) 흐름 제어 |
| `init_explorations(...)` | 초기화 | - 최초 풀이 생성 <br> - Self-Improvement 1회 수행 |
| `verify_solution(...)` | 검증 수행 | - `verification_system_prompt` 사용 (IMO 채점관 페르소나) <br> - "Critical Error" / "Justification Gap" 판별 |
| `build_request_payload(...)` | 통신 준비 | - `user`/`model` 턴을 구성하여 JSON 페이로드 생성 |
| `send_api_request(...)` | API 호출 | - Google Gemini API 통신 (HTTP POST) |

### 주요 프롬프트 변수

- **`step1_prompt`**: "수학적 엄밀성(Rigor)이 최우선", "완벽하지 않으면 솔직해져라" 등의 핵심 지침.
- **`verification_system_prompt`**: "너는 IMO 채점관이다(Verifier NOT Solver)", "논리적 오류를 찾아라" 지시.
- **`correction_prompt`**: "Bug Report에 동의하면 고치고, 아니면 반박해라" (Self-Correction 유도).

---

## 5. 결론: 어떻게 진화하는가?

이 에이전트의 "진화"는 유전 알고리즘이나 모델 파라미터 업데이트가 아닙니다. **"비평(Criticism)을 기억(Context)에 포함시켜 다시 생각하게(Reasoning) 만드는 과정"**입니다.

1. **Self-Correction**: 자신의 답을 스스로(또는 다른 페르소나의 자신이) 비판합니다.
2. **In-Context Learning**: 비판 내용을 프롬프트에 포함시킴으로써, 즉각적으로 행동을 교정합니다.
3. **Iterative Refinement**: 이 과정을 `correct_count`가 5가 될 때까지 반복하여, 점진적으로 완벽한 풀이에 수렴합니다.
