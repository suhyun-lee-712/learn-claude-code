# s17: Autonomous Agents — 보드를 확인하고, 태스크를 가져가기

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → ... → s15 → s16 → `s17` → [s18](../s18_worktree_isolation/) → s19 → s20

> *"보드를 확인하고, 태스크를 가져가라"* — 유휴 상태일 때는 폴링하고, 일을 찾으면 작업한다.
>
> **Harness Layer**: 자율성(Autonomy) — 스스로 조직화하는 팀원들, 리더의 할당이 필요 없음.

---

## 문제

s16의 팀원들은 서로 통신하고 종료를 핸드셰이크할 수 있다. 하지만 각 팀원은 Lead가 태스크를 할당해 주기를 기다린다. 보드에 아직 가져가지 않은 태스크가 10개 있다면, Lead는 10번을 직접 할당해야 한다. 이건 확장성이 없다. 팀원들은 스스로 태스크 보드를 확인하고, 주인 없는 태스크를 가져가고, 끝나면 다음 일을 찾아야 한다.

---

## 해결책

![Autonomous Agents 개요](images/autonomous-agents-overview.en.svg)

S16의 교육용 MessageBus와 프로토콜 tool을 그대로 이어받는다. 이 장에서는 다음을 추가한다: **idle_poll**(유휴 상태일 때 5초마다 폴링), **scan_unclaimed_tasks**(보드를 스캔하여 가져갈 수 있는 태스크를 찾기), **auto-claim**(보이는 즉시 가져가기, Lead 불필요).

팀원의 라이프사이클이 두 단계에서 세 단계로 확장된다:

| Phase | 동작 | 종료 조건 |
|-------|----------|----------------|
| WORK | inbox → LLM → tool loop | `stop_reason != tool_use` |
| IDLE | 5초마다 inbox + 태스크 보드 폴링 | 60초 타임아웃 |
| SHUTDOWN | summary 전송, 종료 | — |

---

## 동작 방식

### idle_poll: 유휴 폴링

태스크를 완료한 뒤에도 팀원은 종료하지 않는다. IDLE 단계로 진입하여 5초마다 새 일감이 있는지 확인한다:

```python
IDLE_POLL_INTERVAL = 5   # seconds
IDLE_TIMEOUT = 60         # seconds

def idle_poll(agent_name, messages, name, role) -> str:
    """Return 'work', 'shutdown', or 'timeout'."""
    for _ in range(IDLE_TIMEOUT // IDLE_POLL_INTERVAL):
        time.sleep(IDLE_POLL_INTERVAL)

        # ① Check inbox (priority)
        inbox = BUS.read_inbox(agent_name)
        if inbox:
            # shutdown_request handled immediately
            for msg in inbox:
                if msg.get("type") == "shutdown_request":
                    # ... reply shutdown_response
                    return "shutdown"
            # Regular messages: inject into context, return to WORK
            messages.append(...)
            return "work"

        # ② Scan task board
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            task = unclaimed[0]
            result = claim_task(task["id"], agent_name)
            if "Claimed" in result:
                messages.append(...)
                return "work"
    return "timeout"
```

inbox가 우선순위를 가지며(shutdown_request 같은 프로토콜 메시지가 들어있을 수 있음), 태스크 보드가 그 다음이다. IDLE 중에 받은 shutdown_request는 즉시 처리된다 — 다음 WORK 단계를 기다릴 필요가 없다.

### scan_unclaimed_tasks: 태스크 보드 스캔

pending 상태이고, 주인이 없으며, 모든 의존성이 완료된(`can_start`) 태스크를 찾는다:

```python
def scan_unclaimed_tasks() -> list[dict]:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
                and not task.get("owner")
                and can_start(task["id"])):
            unclaimed.append(task)
    return unclaimed
```

세 가지 조건: pending이어야 하고, owner가 없어야 하며, 모든 blockedBy 의존성이 완료되어야 한다. `can_start`는 의존성 태스크의 상태를 확인한다 — 의존성이 있다고 해서 태스크를 시작할 수 없는 것은 아니며, 아직 해결되지 않은 의존성만 시작을 막는다. 교육용 버전은 파일명 순으로 첫 번째를 고른다. CC는 여러 팀원이 같은 태스크를 동시에 가져가는 것을 막기 위해 file lock을 사용한다.

### claim_task: Owner 체크

auto-claim은 claim 결과를 확인하며, 실패를 성공으로 취급하지 않는다:

```python
def claim_task(task_id: str, owner: str = "agent") -> str:
    task = load_task(task_id)
    if task.status != "pending":
        return f"Task {task_id} is {task.status}, cannot claim"
    if task.owner:
        return f"Task {task_id} already owned by {task.owner}"
    if not can_start(task_id):
        return f"Blocked by: {deps}"
    task.owner = owner
    task.status = "in_progress"
    save_task(task)
    return f"Claimed {task.id} ({task.subject})"
```

교육용 버전에는 file lock이 없어 동시 claim 시 여전히 race가 발생할 수 있다. 하지만 `task.owner` 체크는 가장 명백한 "마지막 writer가 이긴다" 문제를 피한다. CC는 `proper-lockfile`로 태스크 파일을 보호하며, `claimTask`가 file lock 안에서 read-modify-write를 수행한다(`utils/tasks.ts:541-612`).

### 팀원 라이프사이클: WORK → IDLE → SHUTDOWN

s16의 팀원은 작업이 끝나면 종료한다. s17은 IDLE 단계를 추가한다 — 팀원은 외부 loop에서 WORK → IDLE을 순환한다:

```python
# Outer loop: WORK → IDLE cycle
while True:
    # WORK phase: inner loop (max 10 LLM rounds)
    for _ in range(10):
        # Check inbox, dispatch protocol, call LLM, execute tools
        ...
        if response.stop_reason != "tool_use":
            break  # WORK phase ends

    # IDLE phase
    idle_result = idle_poll(name, messages, name, role)
    if idle_result == "shutdown":
        break
    if idle_result == "timeout":
        break  # 60s timeout → SHUTDOWN

# SHUTDOWN: send summary to Lead
BUS.send(name, "lead", summary, "result")
```

핵심 설계:
- **외부 while True**: 타임아웃 또는 shutdown 요청이 있을 때까지 WORK와 IDLE이 번갈아 실행된다
- **내부 for 10**: WORK 단계는 최대 10 LLM 라운드로 제한된다(무한 loop 방지)
- **IDLE 타임아웃 60초**: 12회 폴링 × 5초 = 60초. 타임아웃되면 summary를 보내고 종료한다
- **shutdown_request는 두 단계 모두에서 동작**: WORK 단계는 `handle_inbox_message`를 통해 dispatch하고, IDLE 단계의 `idle_poll`은 직접 확인하고 응답한다

### Identity 재주입

autoCompact(s08) 이후에는 팀원의 messages 리스트가 summary로 압축될 수 있다. 새로운 WORK 단계에 진입할 때마다 다음을 확인한다:

```python
if len(messages) <= 3:
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}. "
                   f"Continue your work.</identity>"})
```

messages가 짧다는 것은 압축이 일어났음을 시사한다 — identity를 재주입한다. 실제 CC에서는 context compaction이 system prompt를 보존한다. 교육용 버전의 단순화된 구현에서는 수동 처리가 필요하다.

### consume_lead_inbox: 통합 Inbox Consumer

`check_inbox` tool과 메인 loop는 모두 동일한 `consume_lead_inbox()` 함수를 호출한다: 먼저 프로토콜 응답을 라우팅하여 상태를 업데이트한 뒤, 모든 메시지를 Lead의 대화 히스토리에 주입한다. 팀원의 summary와 result는 단순히 터미널에 출력되는 것이 아니다 — Lead의 LLM이 이를 보고 다음 단계를 조율할 수 있다.

### 종합하기

```
1. Lead: "Build the backend — too many tasks, let teammates self-claim"
2. Lead → create_task("Create database schema")
3. Lead → create_task("Write API routes")
4. Lead → create_task("Write unit tests")
5. Lead → spawn_teammate("alice", "backend", "You are a backend developer")
6. Lead → spawn_teammate("bob", "backend", "You are a backend developer")

7. alice thread starts → WORK: no initial inbox → spins → IDLE
8. bob thread starts → WORK: no initial inbox → spins → IDLE

9. alice IDLE poll 1 → scan_unclaimed → finds "Create database schema"
10. alice → claim_task → "Create database schema" → back to WORK
11. bob IDLE poll 1 → scan_unclaimed → finds "Write API routes"
12. bob → claim_task → "Write API routes" → back to WORK

13. alice WORK: write_file("schema.sql", ...) → complete_task → WORK ends
14. alice IDLE → scan → "Write unit tests" → claim → WORK
15. alice WORK: write_file("test_api.py", ...) → complete_task → WORK ends
16. alice IDLE → 60s no new tasks → SHUTDOWN

17. bob similar flow → done → SHUTDOWN
18. Lead consume_lead_inbox → sees alice and bob's summaries
```

두 팀원이 병렬로 태스크를 가져가서 작업한다. Lead는 태스크를 만들고 팀원을 spawn하기만 하면 된다 — 수동 할당이 필요 없다.

---

## s16에서 달라진 점

| 구성 요소 | 이전 (s16) | 이후 (s17) |
|-----------|-------------|-------------|
| 태스크 할당 | Lead가 수동으로 할당 | 팀원이 auto-claim (can_start가 의존성 확인) |
| 팀원 상태 | WORK 또는 종료 | WORK → IDLE (60초 폴링) → SHUTDOWN |
| claim_task | owner 체크 없음 | 이미 owner가 있는 태스크는 거부 |
| IDLE 단계 shutdown | shutdown_request를 처리하지 않음 | shutdown을 즉시 dispatch하고 종료 |
| Lead inbox | 출력만 하고 context에 없음 | consume_lead_inbox가 히스토리에 주입 |
| 새 함수 | — | idle_poll, scan_unclaimed_tasks, consume_lead_inbox |
| Identity 지속성 | system prompt만 | 압축 이후 자동 재주입 |
| Lead tools | 14개 (s16) | 14개 (변경 없음) |
| 팀원 tools | 5개 | 8개 (+ list_tasks, claim_task, complete_task) |
| 팀원 종료 | 태스크 완료 후 종료 | 60초 유휴 타임아웃 이후에만 종료 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s17_autonomous_agents/code.py
```

이 prompt를 시도해 보자:

`Create 3 tasks on the board, then spawn alice and bob. Watch them auto-claim and work.`

관찰할 점: 팀원이 할당되지 않은 태스크를 auto-claim하는가? blockedBy 의존성이 있는 태스크는 의존성이 완료된 후에만 claim되는가? 유휴 타임아웃이 shutdown을 트리거하는가? IDLE 단계의 shutdown_request가 즉각 응답을 받는가? `.tasks/`의 태스크 상태는 어떻게 변하는가?

---

## 다음 단계

이제 팀원들이 스스로 조직화한다. 하지만 Alice와 Bob은 둘 다 같은 디렉터리에서 작업한다 — Alice가 `config.py`를 편집하고, Bob도 `config.py`를 편집하면서 서로를 덮어쓴다.

s18 Worktree Isolation → 각 태스크가 자체 작업 디렉터리를 가져 충돌이 없다.

<details>
<summary>CC 소스 심층 분석</summary>

> 교육용 노트: 이 장의 idle_poll + auto-claim 메커니즘은 교육용 설계로, 통합 폴링 함수를 사용해 "유휴 상태일 때 일을 찾는다"를 보여준다. CC의 실제 구현은 여러 메커니즘을 결합하지만, 같은 목표를 공유한다 — Lead의 수동 할당 부담을 줄이는 것이다.

### 1. CC의 유휴 메커니즘: 단일 폴링이 아닌 결합 방식

교육용 버전은 단일 `idle_poll()`로 유휴 중의 inbox 확인과 태스크 claim을 모두 처리한다. CC의 실제 구현은 네 가지 메커니즘을 결합한다:

**idle_notification**: 한 라운드의 작업을 완료한 뒤, `sendIdleNotification()`(`inProcessRunner.ts:569-589`)이 Lead에게 유휴 알림을 보낸다. Lead는 팀원이 사용 가능함을 알고 새 태스크를 할당하거나 shutdown을 요청할 수 있다.

**mailbox polling**: `waitForNextPromptOrShutdown()`(`inProcessRunner.ts:689-868`)은 **500ms 폴링 loop**로, 세 가지 소스를 지속적으로 확인한다: 대기 중인 사용자 메시지, mailbox 파일 메시지, 태스크 리스트. shutdown 요청이 우선순위를 가지며(`inProcessRunner.ts:768-804`), 일반 메시지에 의한 starvation을 방지한다.

**task watcher**: `useTaskListWatcher`(`hooks/useTaskListWatcher.ts:34-189`)는 `fs.watch()`를 사용해 `.claude/tasks/` 디렉터리를 1초 debounce로 모니터링하며, 새 태스크가 생성되거나 의존성이 해제될 때 확인을 트리거한다. 의존성 체크(`L197-207`)는 "blockedBy에 완료되지 않은 태스크가 없음"을 검증하며, "blockedBy가 비어 있음"이 아니다.

**active claiming**: 폴링 loop은 `tryClaimNextTask()`(`inProcessRunner.ts:853-860`)도 호출한다 — 대기하는 동안 태스크 리스트에서 능동적으로 태스크를 claim한다. 따라서 "팀원이 태스크를 능동적으로 폴링하지 않는다"는 부정확하다. CC에는 수동 알림과 능동 claim이 모두 있다.

### 2. 태스크 Claim: File Lock + Atomic 연산

`claimTask()`(`utils/tasks.ts:541-612`)는 `proper-lockfile` 태스크 수준 lock을 사용하여, lock 안에서 read-check-modify-write를 수행한다. 체크: owner가 이미 존재함(`L575-576`), 이미 완료됨(`L580-581`), blockedBy에 해결되지 않은 blocker가 있음(`L585-594`). `claimTaskWithBusyCheck()`(`utils/tasks.ts:614-692`)는 태스크 리스트 수준 lock을 사용하여, busy 체크와 claim을 atomic하게 만들어 TOCTOU를 피한다.

`findAvailableTask()`(`inProcessRunner.ts:595-604`)는 `task.blockedBy.every(id => !unresolvedTaskIds.has(id))`로 "모든 blockedBy가 완료됨"을 확인한다. `tryClaimNextTask()`(`inProcessRunner.ts:624-657`)는 claim 후 status를 `in_progress`로 업데이트하여, UI가 변경 사항을 즉시 반영한다.

### 3. 교육용 버전 vs CC 비교

| 차원 | 교육용 (s17) | CC |
|-----------|----------------|-----|
| 유휴 메커니즘 | idle_poll 통합 폴링 (5초) | idle_notification + 500ms mailbox polling + task watcher |
| 태스크 발견 | scan_unclaimed_tasks (폴링) | useTaskListWatcher (파일 watching) + tryClaimNextTask (능동 폴링) |
| 의존성 체크 | can_start (모든 blockedBy 완료) | findAvailableTask (동일 의미) |
| 동시성 안전성 | owner 체크 (file lock 없음) | proper-lockfile 태스크 lock + 태스크 리스트 lock |
| Shutdown 처리 | IDLE은 직접 dispatch, WORK은 handle_inbox_message 경유 | 500ms 폴링 loop이 shutdown_request 우선 처리 |
| 타임아웃 종료 | 새 태스크 없이 60초 | 고정 타임아웃 없음, Lead가 수동 shutdown |
| Identity 지속성 | Messages 길이 감지 | Context compaction이 system prompt 보존 |
| Claim 실패 처리 | 반환값 확인, 실패 시 skip | File lock이 atomicity 보장 |

교육용 버전의 `idle_poll()`은 CC의 네 가지 메커니즘을 하나의 폴링 함수로 통합한다 — 핵심 의미(유휴 상태일 때 일을 찾고, 의존성이 해결되면 claim하며, shutdown을 우선시함)가 일관되므로 합리적인 단순화다.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
