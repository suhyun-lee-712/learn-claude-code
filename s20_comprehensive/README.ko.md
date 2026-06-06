# s20: 종합 Agent — 모든 메커니즘, 하나의 loop

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → ... → s18 → s19 → `s20`

> *"여러 메커니즘, 하나의 loop"* — tool, 권한, 메모리, task, team, 플러그인이 모두 동일한 `while True`에 매달려 있습니다.
>
> **Harness 계층**: 종합 — 앞선 19개의 메커니즘을 하나의 실행 가능한 시스템으로 되돌려 놓습니다.

---

## 문제

처음 19개 장은 메커니즘을 하나씩 추가합니다. 학습에는 이 방식이 옳지만, 실제 agent는 메커니즘 하나만 켠 채로 돌아가지 않습니다.

장시간 동작하는 코딩 agent에는 다음이 한꺼번에 필요합니다:

- tool 디스패치와 권한 경계
- hook 확장 지점
- todo 계획과 task 그래프
- skill, 메모리, 그리고 런타임 system prompt 조립
- compaction과 에러 복구
- 백그라운드 task와 cron 스케줄링
- team, 프로토콜, 자율적 claim
- worktree 격리
- MCP 외부 tool 통합

어려운 부분은 기능을 쌓아 올리는 것이 아닙니다. 어려운 부분은 각 메커니즘이 loop 주위 어디에 속하는지 파악하는 것입니다. S20은 종착점에 해당하는 장입니다: 모든 구성 요소가 하나의 harness 안으로 다시 배치됩니다.

---

## 해결책

![시스템 아키텍처](images/system-architecture.en.svg)

S20은 새로운 메커니즘을 만들어 내지 않습니다. 앞선 장들의 학습용 구성 요소를 하나의 완전한 harness로 병합합니다:

```text
user input
  → UserPromptSubmit hooks
  → cron/background notification injection
  → context compact
  → memory + skills + MCP state assemble the system prompt
  → LLM
  → has tool_use block?
      no  → Stop hooks → return
      yes → PreToolUse hooks + permission
          → TOOL_HANDLERS / MCP handlers / background dispatch
          → PostToolUse hooks
          → tool_result / task_notification back to messages
          → next round
```

loop는 여전히 같은 구조입니다: 모델을 호출하고, 응답에 `tool_use` block이 포함되어 있는지 확인하고, tool을 실행하고, 결과를 다시 `messages`에 덧붙입니다. CC 소스는 `stop_reason == "tool_use"`를 직접 신뢰하지 않습니다. 실제로 tool_use block이 존재하는지가 계속 진행하라는 신호입니다. 달라진 점은 loop를 둘러싼 harness가 이제 완전해졌다는 것입니다.

---

## 각 구성 요소가 자리 잡는 위치

| 위치 | 구성 요소 | 역할 |
|----------|-----------|------|
| user input 주위 | `UserPromptSubmit` hooks | user input을 로깅, 주입, 감사 |
| LLM 이전 | cron queue | 예약된 prompt를 `messages`에 주입 |
| LLM 이전 | background notifications | 완료된 백그라운드 작업을 `<task_notification>`으로 주입 |
| LLM 이전 | compaction 파이프라인 | 큰 출력에 예산 적용, 히스토리 정리, 오래된 tool 결과 compact, 필요 시 요약 |
| LLM 이전 | memory / skills / MCP state | 모델이 현재 능력과 장기 context를 볼 수 있도록 system prompt 조립 |
| LLM 호출 | error recovery | 429/529 재시도, `max_tokens` 상향, prompt-too-long 시 compact |
| tool 실행 이전 | `PreToolUse` hooks + permission | 위험한 명령, 범위를 벗어난 쓰기, 파괴적인 MCP tool 차단 |
| tool 디스패치 | `assemble_tool_pool` | 내장 tool과 동적 MCP tool 조립 |
| tool 실행 중 | background dispatch | 느린 bash 작업을 데몬 스레드로 옮기고 placeholder 결과 반환 |
| tool 실행 이후 | `PostToolUse` hooks | 큰 출력 경고, 로깅, 후처리 |
| loop로 복귀 | tool_result | `tool_use`마다 `tool_result` 하나, 그 다음 모델 라운드 |
| 이번 라운드에 tool_use 없음 / stop 시 | `Stop` hooks | 통계, 정리, 감사 |

---

## code.py에 담긴 것

### Tool과 디스패치

내장 tool pool에는 27개의 tool이 들어 있습니다:

```text
bash, read_file, write_file, edit_file, glob
todo_write, task, load_skill, compact
create_task, list_tasks, get_task, claim_task, complete_task
schedule_cron, list_crons, cancel_cron
spawn_teammate, send_message, check_inbox
request_shutdown, request_plan, review_plan
create_worktree, remove_worktree, keep_worktree
connect_mcp
```

`assemble_tool_pool()`는 매 라운드 다음을 조립합니다:

```text
BUILTIN_TOOLS + connected MCP tools
BUILTIN_HANDLERS + mcp__server__tool handlers
```

`connect_mcp("docs")` 이후 다음 라운드에서는 `mcp__docs__search` 같은 tool이 노출됩니다.

### 권한과 Hook

권한은 tool 실행 라인에 하드코딩되어 있지 않습니다. 그것은 `PreToolUse` hook입니다:

```python
blocked = trigger_hooks("PreToolUse", block)
if blocked:
    results.append(tool_result(block.id, blocked))
    continue
```

즉, 권한, 로깅, 감사 로직이 모두 동일한 hook 지점에 붙습니다. 실행 후에는 `PostToolUse` hook이 실행됩니다.

### 계획과 Task

S20은 두 개의 계획 계층을 유지합니다:

- `todo_write`: 현재 세션을 위한 가벼운 계획, 메모리에 보관
- task 그래프: 세션을 가로지르고, 의존성을 인식하며, claim 가능한 task 파일들로 `.tasks/task_*.json` 아래에 위치

전자는 단일 agent가 길을 잃지 않게 합니다. 후자는 team 협업을 지원합니다.

### Subagent와 Team

S20에는 두 종류의 위임이 있습니다:

- `task`: 일회성 subagent. 격리된 `messages[]`를 사용하고, 중간 context를 버리며, 최종 요약만 반환합니다.
- `spawn_teammate`: 지속적인 teammate 스레드. `MessageBus`를 통해 통신하고, 유휴 상태일 때 task 보드를 폴링하며, 작업을 자율적으로 claim할 수 있습니다.

일회성 subagent는 context 격리 문제를 해결합니다. 지속적인 teammate는 장시간 병렬 협업 문제를 해결합니다.

### 메모리, Skill, Prompt

`assemble_system_prompt(context)`는 매 라운드 다음으로부터 조립합니다:

- 정체성과 tool 가이드
- workspace
- skill 카탈로그
- `.memory/MEMORY.md`
- 연결된 MCP 서버

skill은 카탈로그만 system prompt에 넣습니다. 전체 내용은 `load_skill(name)`을 통해 필요할 때 로드됩니다.

### Compaction과 복구

LLM 호출 이전에 S20은 compaction 파이프라인을 실행합니다:

```text
tool_result_budget → snip_compact → micro_compact → compact_history
```

모델 호출은 복구 로직으로 감싸여 있습니다:

- 429: 지수 백오프 재시도
- 529: 지수 백오프, 반복 실패 시 선택적으로 fallback 모델로 전환
- `max_tokens`: max tokens를 올린 뒤 continuation 요청
- prompt too long: 반응형 compact 후 재시도

### Background와 Cron

느린 bash 작업은 메인 loop를 막지 않습니다:

```text
should_run_background → start_background_task → placeholder tool_result
background done → task_notification → next round injects messages
```

cron 스케줄러는 데몬 스레드로 실행되며 초당 한 번 확인합니다. CLI는 `cron_queue`를 감시하고, 작업이 발화하면 `[Scheduled] ...`를 주입하고 자동으로 agent 턴 하나를 실행합니다.

### Worktree와 MCP

worktree 격리는 디렉터리를 소유합니다:

- `create_worktree(name, task_id)`는 격리된 브랜치와 디렉터리를 생성합니다
- task의 `worktree` 필드는 task를 해당 디렉터리에 바인딩합니다
- teammate가 worktree를 가진 task를 claim하면, 그 teammate의 bash/read/write tool은 해당 디렉터리에서 실행됩니다

MCP는 외부 능력을 소유합니다:

- `connect_mcp(name)`은 mock 서버에 연결합니다
- `assemble_tool_pool()`은 MCP tool을 tool pool에 조립합니다
- tool 이름은 `mcp__server__tool`을 사용합니다

---

## s19로부터의 변경 사항

| 구성 요소 | s19 | s20 |
|-----------|-----|-----|
| tool pool | 내장 + MCP | 내장 + MCP, s01-s18 tool 복원 |
| permission | 학습 본문에서 생략 | `PreToolUse` hook 안에서 실행 |
| hooks | 생략 | UserPromptSubmit / PreToolUse / PostToolUse / Stop |
| todo | 생략 | `todo_write` + reminder |
| skill | 생략 | system prompt의 카탈로그 + `load_skill` |
| compact | 생략 | LLM 이전 compaction + `compact` tool + 반응형 compact |
| error recovery | 단순 try/except | 재시도 / max_tokens / prompt too long |
| background | 생략 | 느린 작업 스레드 + task notification |
| cron | 생략 | 데몬 스케줄러 + 영속 작업 |
| multi-agent | 유지 | 유지; teammate는 격리된 디렉터리에서 기본 tool 사용 |
| worktree | 유지 | 유지 |
| MCP | 신규 | 최종 tool pool의 일부로 유지 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s20_comprehensive/code.py
```

해볼 것:

1. `Create a todo list for inspecting this repo, then list Python files`
2. `Connect to the docs MCP server and search for agent loop`
3. `Create two tasks, create worktrees for them, then spawn alice and bob. Ask them to submit plans before claiming tasks.`
4. `remind me of the meeting in 3 minutes.`
5. `Run npm install in the background and continue reading README.md`

관찰할 것:

- 각 tool 호출이 hook/permission을 통과하는지
- `connect_mcp` 이후 다음 라운드에 MCP tool이 나타나는지
- 느린 작업이 background placeholder를 반환하는지
- 시간이 되었을 때 cron이 자동으로 알려주는지
- teammate가 계획을 제출하고 승인 전에 멈추는지
- 계획 승인 후 teammate가 task를 claim할 수 있는지
- teammate가 바인딩된 worktree 디렉터리로 전환하는지

---

## 끝은 곧 시작

s01부터 s20까지, 코드는 점점 더 강력해지지만 핵심은 변하지 않습니다:

```python
while True:
    response = LLM(messages, tools)
    if not has_tool_use(response.content):
        return
    results = execute_tools(response.content)
    messages.append(tool_results)
```

Claude Code의 복잡성은 "또 다른 agent 두뇌"가 아닙니다. 그것은 성숙한 harness의 복잡성입니다. 모델은 결정을 내리고 행동을 선택하며, harness는 환경, tool, 권한, 메모리, team, 외부 능력을 조직합니다.

이것이 이 강의의 종착점입니다: 여러 메커니즘, 하나의 loop.
