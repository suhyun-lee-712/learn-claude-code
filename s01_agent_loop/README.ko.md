# s01: 에이전트 루프 — 루프 하나면 충분하다

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

`s01` → [s02](../s02_tool_use/) → s03 → s04 → ... → s20
> *"One loop & Bash is all you need"* — 도구 하나 + 루프 하나 = 에이전트 하나.
>
> **하니스 계층(Harness Layer)**: 루프 — 모델과 현실 세계를 잇는 첫 번째 다리.

---

## 문제

모델에게 이렇게 요청합니다: "내 디렉터리의 파일 목록을 보여주고 XXX.py를 실행해줘."

모델은 bash 명령어를 출력할 수 있지만, 일단 출력이 끝나면 거기서 멈춥니다 — 스스로 명령어를 실행하지도 않고, 그 결과를 바탕으로 추론을 이어가지도 않습니다.

직접 명령어를 실행하고, 출력 결과를 다시 채팅에 붙여넣어 모델이 계속하게 만들 수도 있습니다. 다음 명령어가 나오면, 또 실행하고, 또 붙여넣습니다.

매 왕복마다 당신이 중간 계층이 되는 셈입니다. 이 과정을 자동화하는 것이 이 장에서 다루는 내용입니다.

---

## 해결책

![Agent Loop](images/agent-loop.en.svg)

`while True` 루프: 모델이 도구를 호출하면 계속하고, 호출하지 않으면 멈춥니다. 전체 과정은 두 가지 신호에 달려 있습니다:

| 신호 | 의미 | 루프 동작 |
|--------|---------|-------------|
| `stop_reason == "tool_use"` | 모델이 손을 듭니다: "도구가 필요해" | 실행 → 결과를 다시 전달 → 계속 |
| `stop_reason != "tool_use"` | 모델이 말합니다: "끝났어" | 루프 종료 |

---

## 작동 방식

이 과정을 코드로 옮겨봅시다. 단계별로:

**1단계**: 사용자의 질문을 첫 번째 메시지로 시작합니다.

```python
messages = [{"role": "user", "content": query}]
```

**2단계**: 메시지와 도구 정의를 LLM에 보냅니다.

```python
response = client.messages.create(
    model=MODEL, system=SYSTEM, messages=messages,
    tools=TOOLS, max_tokens=8000,
)
```

**3단계**: 모델의 응답을 추가하고 도구를 호출했는지 확인합니다. 도구 호출이 없으면 → 완료.

```python
messages.append({"role": "assistant", "content": response.content})
if response.stop_reason != "tool_use":
    return
```

**4단계**: 모델이 요청한 도구를 실행하고 결과를 수집합니다.

```python
results = []
for block in response.content:
    if block.type == "tool_use":
        output = run_bash(block.input["command"])
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
```

**5단계**: 도구 결과를 새 메시지로 추가하고 2단계로 돌아갑니다.

```python
messages.append({"role": "user", "content": results})
```

하나의 완전한 함수로 조립하면:

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

30줄 미만 — 이것이 실행 가능한 최소 에이전트 하니스 커널입니다. 지능 그 자체는 아니지만, 모델이 계속 행동할 수 있게 해주는 가장 작은 런타임 프레임워크입니다. 모델이 결정하고(도구를 호출할지, 어떤 도구를 쓸지), 하니스가 실행합니다(호출되면 실행하고 결과를 다시 전달). 다음 18개 장은 모두 이 루프 위에 메커니즘을 더해 나갈 뿐입니다. 루프 자체는 결코 바뀌지 않습니다.

---

## 직접 해보기

> **교육용 데모 안내**: 이 코드는 모델이 생성한 셸 명령어를 실행합니다. 프로젝트 파일에 영향을 주지 않도록 임시 테스트 디렉터리에서 실행하세요. s03에서 실제 권한 시스템을 다룹니다.

**준비** (최초 실행 시):

```sh
pip install -r requirements.txt
cp .env.example .env
# .env를 편집하여 ANTHROPIC_API_KEY와 MODEL_ID를 입력하세요
```

**실행**:

```sh
python s01_agent_loop/code.py
```

다음 프롬프트들을 시도해보세요:

1. `Create a file called hello.py that prints "Hello, World!"`
2. `List all Python files in this directory`
3. `What is the current git branch?`

주목할 점: 모델이 언제 도구를 호출하고(루프가 계속됨), 언제 호출하지 않는가(루프가 종료됨)?

---

## 다음 단계

지금 모델에게는 bash밖에 없습니다 — 파일을 읽으려면 `cat`, 파일을 쓰려면 `echo ... >`, 파일을 찾으려면 `find`가 필요합니다. 보기 흉하고 오류가 발생하기 쉽습니다.

→ s02 도구 사용(Tool Use): 제대로 된 도구 5개를 주면 무슨 일이 일어날까요? 모델이 여러 도구를 한 번에 호출할까요? 병렬 도구 실행이 서로 충돌하지는 않을까요?

<details>
<summary>CC 소스 코드 깊이 들여다보기</summary>

> 아래 내용은 CC 소스 코드 `src/query.ts`(1729줄)에 대한 리뷰를 바탕으로 합니다. 핵심 차이는 두 가지입니다: CC는 루프를 계속할지 결정할 때 `stop_reason` 필드에 의존하지 않고 — 대신 콘텐츠에 `tool_use` 블록이 포함되어 있는지를 확인합니다(스트리밍 응답에서는 `stop_reason`이 신뢰할 수 없기 때문입니다); CC는 프로덕션 등급의 보호를 위해 더 많은 종료 경로와 복구 전략을 갖추고 있습니다.

**교육용 버전의 30줄짜리 `while True`가 바로 CC의 1729줄의 핵심입니다.** 아래의 모든 것은 그 핵심 위에 겹겹이 쌓인 보호 메커니즘입니다.

<details>
<summary>1. 루프 구조의 차이</summary>

교육용 버전은 `response.stop_reason`을 확인합니다. CC는 이를 루프 지속의 유일한 신호로 사용하지 않습니다 — 스트리밍 응답에서는 `tool_use` 블록이 이미 존재하는데도 `stop_reason`이 아직 갱신되지 않았을 수 있습니다. CC는 `needsFollowUp` 플래그를 사용합니다: 스트리밍 메시지 수신 중(`query.ts:830-834`), `tool_use` 블록이 감지될 때마다 `true`로 설정됩니다. `QueryEngine.ts`는 다른 로직을 위해 `message_delta`에서 실제 `stop_reason`을 포착하지만, 쿼리 루프 자체는 `needsFollowUp`에 의존합니다.

```typescript
// query.ts:554-558
// stop_reason === 'tool_use'는 신뢰할 수 없다.
// 스트리밍 중 tool_use 블록이 도착할 때마다 설정된다.
let needsFollowUp = false
```

</details>

<details>
<summary>2. 상태 객체 — 10개 필드 (교육용 버전은 messages만 사용)</summary>

| # | 필드 | 용도 | 장 |
|---|-------|---------|---------|
| 1 | `messages` | 현재 반복의 메시지 배열 | s01 |
| 2 | `toolUseContext` | 도구, 신호, 권한 컨텍스트 | s02 |
| 3 | `autoCompactTracking` | 컴팩션 상태 추적 | s08 |
| 4 | `maxOutputTokensRecoveryCount` | 토큰 복구 시도 횟수 (최대 3) | s11 |
| 5 | `hasAttemptedReactiveCompact` | 이번 라운드에 반응형 컴팩션을 시도했는지 여부 | s08 |
| 6 | `maxOutputTokensOverride` | 8K→64K 업그레이드 오버라이드 | s11 |
| 7 | `pendingToolUseSummary` | 백그라운드 Haiku가 생성한 도구 사용 요약 | s08 |
| 8 | `stopHookActive` | stop hook이 차단 오류를 발생시켰는지 여부 | s04 |
| 9 | `turnCount` | 턴 수 (maxTurns 확인용) | s01 |
| 10 | `transition` | 마지막 continue 사유 | s11 |

> 참고: `taskBudgetRemaining`(`query.ts:291`)은 State에 있는 것이 아니라 루프 지역 변수입니다. 소스 주석에 명시적으로 "Loop-local (not on State)"라고 적혀 있습니다.

</details>

<details>
<summary>3. 다중 종료 및 계속 경로</summary>

교육용 버전에는 종료 경로가 단 1개뿐입니다(모델이 도구를 호출하지 않음 → 완료). 프로덕션 버전에는 다중 종료 및 계속 경로가 있어, 차단 한도, 너무 긴 프롬프트, 모델 오류, 중단(abort), hook 중단, 최대 턴 수, 토큰 예산 계속, 반응형 컴팩트 재시도 등을 다룹니다. 각 시나리오에는 그에 대응하는 복구 또는 종료 전략이 있습니다.

</details>

<details>
<summary>4. 스트리밍 도구 실행과 QueryEngine</summary>

CC의 `StreamingToolExecutor`(`query.ts:561`)는 모델이 아직 생성 중일 때도 도구가 병렬 실행을 시작할 수 있게 합니다(동시성에 안전한 도구는 병렬로 실행되고, 그 외에는 배타적으로 실행됨). `QueryEngine.ts`는 비용 초과, 구조화된 출력 검증 실패 등에 대한 추가 보호를 더합니다. 교육용 버전은 이런 것들을 구현하지 않습니다 — 목표는 최고의 성능이 아니라 개념적 명료함입니다.

</details>

**한 문장으로**: query.ts 1729줄의 핵심은 30줄짜리 `while True`입니다. 모든 복잡한 필드와 종료 경로는 보호 메커니즘입니다. 핵심 루프를 먼저 이해하면, 그 뒤에 이어지는 모든 것이 자연스럽게 펼쳐집니다.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1, ko@v1 -->
