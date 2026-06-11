# s17: Autonomous Agents — 보드를 확인하고, 태스크를 가져가기

> **핵심 한 줄**: 팀원이 스스로 태스크 보드를 폴링하고 일감을 auto-claim한다. Lead는 태스크를 만들고 팀원을 spawn하면 끝 — 수동 할당이 필요 없다.

---

## 왜 필요한가?

s16의 팀원들은 shutdown 핸드셰이크도 하고 plan approval도 한다. 하지만 한 가지 문제가 남아있다.

```
태스크 10개가 있으면?
  → Lead가 alice에게 "task_1 해", bob에게 "task_2 해" ... 10번
```

**Lead가 태스크를 수동으로 할당해야 한다.** 팀원이 많아질수록, 태스크가 쌓일수록 Lead의 부담이 선형으로 증가한다.

s17의 해결책:

> 팀원이 유휴 상태일 때 보드를 직접 보고, 할 일이 있으면 가져간다.

Lead는 태스크만 만들고 팀원을 spawn하면 끝. 중간에 개입할 필요가 없다.

---

## 팀원 라이프사이클: 2단계 → 3단계

s16까지는 WORK 아니면 종료였다. s17은 **IDLE 단계**를 추가한다.

```
s16: WORK → SHUTDOWN

s17: WORK → IDLE → WORK → IDLE → ... → SHUTDOWN
              ↑
         (60초 타임아웃이면 SHUTDOWN)
         (shutdown_request 오면 SHUTDOWN)
         (새 일감 찾으면 WORK 재진입)
```

| 단계 | 동작 | 종료 조건 |
|------|------|-----------|
| **WORK** | inbox → LLM → tool loop | `stop_reason != tool_use` |
| **IDLE** | 5초마다 inbox + 태스크 보드 폴링 | 새 일감 → WORK, shutdown → 종료, 60초 → SHUTDOWN |
| **SHUTDOWN** | summary 전송 → 종료 | — |

---

## 핵심 구성 요소

s17에서 새로 추가된 것들:

1. **idle_poll** — IDLE 단계의 폴링 루프 (inbox 우선 → 태스크 보드 → 타임아웃)
2. **scan_unclaimed_tasks** — pending + no owner + can_start 조건의 태스크 탐색
3. **claim_task** — owner 체크 포함한 태스크 claim
4. **외부 while True** — WORK↔IDLE 반복 구조
5. **Identity 재주입** — context compaction 후 identity 복원

---

## 1. idle_poll: IDLE 단계의 폴링

WORK가 끝나면 팀원은 이 함수를 호출한다. 60초 동안 5초마다 두 가지를 확인한다.

```python
IDLE_POLL_INTERVAL = 5   # 초
IDLE_TIMEOUT = 60         # 초 (= 12번 폴링)

def idle_poll(agent_name, messages, name, role) -> str:
    for _ in range(12):
        time.sleep(5)

        # 우선순위 ①: inbox 먼저 (shutdown 같은 프로토콜 메시지가 있을 수 있음)
        inbox = BUS.read_inbox(agent_name)
        if inbox:
            for msg in inbox:
                if msg.get("type") == "shutdown_request":
                    BUS.send(name, "lead", "Shutting down.", "shutdown_response", ...)
                    return "shutdown"
            # 일반 메시지 → context에 주입 → WORK 재개
            messages.append(...)
            return "work"

        # 우선순위 ②: 태스크 보드 스캔
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            result = claim_task(unclaimed[0]["id"], agent_name)
            if "Claimed" in result:   # claim 성공 확인
                messages.append(...)  # <auto-claimed> 메시지 주입
                return "work"

    return "timeout"  # 60초 동안 아무것도 없음
```

반환값 세 가지:
- `"work"` → 새 일감 발견, WORK 단계로 재진입
- `"shutdown"` → shutdown_request 수신, 종료
- `"timeout"` → 60초 동안 아무것도 없음, SHUTDOWN

inbox가 태스크 보드보다 **우선순위가 높은 이유**: inbox에 shutdown_request 같은 프로토콜 메시지가 들어있을 수 있기 때문이다.

### 실제 CC의 유휴 메커니즘

교육용 `idle_poll()`은 하나의 함수로 inbox 확인과 태스크 claim을 모두 처리한다. 실제 CC는 네 가지 메커니즘을 조합한다:

**idle_notification**: 한 라운드 작업 완료 후 `sendIdleNotification()`(`inProcessRunner.ts:569-589`)이 Lead에게 유휴 알림을 보낸다. Lead는 팀원이 새 태스크를 받을 수 있는 상태임을 알 수 있다.

**mailbox polling**: `waitForNextPromptOrShutdown()`(`inProcessRunner.ts:689-868`)은 **500ms 폴링 루프**로 세 가지를 지속 확인한다: 대기 중인 사용자 메시지, mailbox 파일 메시지, 태스크 리스트. shutdown 요청이 우선순위를 가진다.

**task watcher**: `useTaskListWatcher`(`hooks/useTaskListWatcher.ts:34-189`)는 `fs.watch()`로 `.claude/tasks/` 디렉터리를 1초 debounce로 모니터링한다. 새 태스크가 생기거나 의존성이 해제될 때 확인을 트리거한다.

**active claiming**: 폴링 루프가 `tryClaimNextTask()`(`inProcessRunner.ts:853-860`)도 호출해서 능동적으로 태스크를 claim한다. "팀원이 알림을 기다리기만 한다"는 부정확하다 — 능동적인 폴링과 수동적인 알림이 모두 있다.

교육용 `idle_poll()`은 이 네 가지 메커니즘을 하나로 통합한 단순화다.

---

## 2. scan_unclaimed_tasks: 태스크 보드 스캔

```python
def scan_unclaimed_tasks() -> list[dict]:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"          # 아직 시작 안 됨
                and not task.get("owner")             # 아무도 안 가져감
                and can_start(task["id"])):           # 의존성 모두 완료됨
            unclaimed.append(task)
    return unclaimed
```

세 조건을 모두 만족해야 claim 가능하다:

1. `status == "pending"` — 이미 진행 중이거나 완료된 태스크는 제외
2. `not owner` — 다른 팀원이 이미 가져간 태스크는 제외
3. `can_start` — **`blockedBy` 목록의 태스크가 전부 `completed`인 경우에만 가능**

> "의존성이 없음"이 아니라 "아직 완료되지 않은 의존성이 없음"이 조건이다.
> blockedBy에 태스크 ID들이 있어도, 그것들이 전부 done 상태면 시작 가능하다.

---

## 3. claim_task: Owner 체크

여러 팀원이 동시에 같은 태스크를 claim하려 할 수 있다.

```python
def claim_task(task_id, owner) -> str:
    task = load_task(task_id)
    if task.status != "pending":
        return f"Task {task_id} is {task.status}, cannot claim"
    if task.owner:                       # ← 이미 누군가 가져갔으면 거부
        return f"Task {task_id} already owned by {task.owner}"
    if not can_start(task_id):
        return f"Cannot start — blocked by: {deps}"
    task.owner = owner
    task.status = "in_progress"
    save_task(task)
    return f"Claimed {task.id} ({task.subject})"
```

반환값에 `"Claimed"`가 포함되는지 확인한다 — `idle_poll`이 claim 성공 여부를 이걸로 판단한다. 실패했는데 성공으로 처리하지 않는다.

교육용 버전은 file lock이 없어서 완벽한 동시성 보호는 아니다. 실제 CC는 `proper-lockfile`을 사용해 `read-check-modify-write`를 atomic하게 처리한다.

---

## 4. 외부 while True 구조

팀원의 전체 루프 구조:

```python
while True:                          # ← 외부 루프: WORK↔IDLE 반복
    # Identity 재주입 (s17)
    if len(messages) <= 3:
        messages.insert(0, {"role": "user",
            "content": f"<identity>You are '{name}', role: {role}. Continue.</identity>"})

    # WORK 단계 (최대 10 LLM 라운드)
    for _ in range(10):
        # inbox 확인 → LLM 호출 → tool 실행
        if response.stop_reason != "tool_use":
            break

    # IDLE 단계
    result = idle_poll(name, messages, name, role)
    if result in ("shutdown", "timeout"):
        break                        # 외부 루프 탈출 → SHUTDOWN

# SHUTDOWN
BUS.send(name, "lead", summary, "result")
```

설계 포인트:
- **외부 `while True`** — WORK와 IDLE이 번갈아 반복됨
- **내부 `for 10`** — WORK는 최대 10라운드 (무한루프 방지)
- `"work"` 반환 시 → 외부 루프가 다시 내부 for loop을 실행
- `"shutdown"` / `"timeout"` 반환 시 → 외부 루프 탈출 → SHUTDOWN

---

## 5. Identity 재주입

s08의 autoCompact로 context가 압축되면 팀원의 messages 배열이 요약으로 대체될 수 있다. 그러면 팀원이 자신의 이름과 역할을 잊을 수 있다.

```python
if len(messages) <= 3:    # messages가 너무 짧다 = 압축이 일어남
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}. "
                   f"Continue your work.</identity>"})
```

WORK 재진입 시마다 messages 길이를 체크해서, 압축됐으면 identity를 첫 메시지로 재삽입한다.

실제 CC는 system prompt로 identity를 전달한다. context compaction은 messages만 압축하고 **system prompt는 항상 보존**하므로, 수동 재주입이 필요 없다.

| | 교육용 s17 | 실제 CC |
|--|-----------|---------|
| Identity 저장 위치 | messages[0] | system prompt (별도 파라미터) |
| compaction 후 상태 | 사라질 수 있음 | 항상 보존됨 |
| 대응 방식 | `len(messages) <= 3` 체크 후 재주입 | 없음 (자동 보존) |

---

## 전체 흐름 예시

```
1. Lead: "DB schema, API routes, unit tests 태스크 만들고 alice, bob spawn"

2. Lead → create_task("DB schema")                → task_001
3. Lead → create_task("API routes")               → task_002
4. Lead → create_task("Unit tests",
           blockedBy=["task_002"])                 → task_003 (API routes 의존)
5. Lead → spawn alice, bob

6. alice: WORK (할 일 없음) → IDLE 진입
7. bob:   WORK (할 일 없음) → IDLE 진입

8. alice IDLE poll ① → scan → task_001 (DB schema) claim → WORK 재진입
9. bob   IDLE poll ① → scan → task_002 (API routes) claim → WORK 재진입

10. alice: schema.sql 작성 → complete_task(task_001) → IDLE 재진입
11. bob:   routes.py 작성  → complete_task(task_002) → IDLE 재진입

12. alice IDLE poll ② → scan
          → task_003 (Unit tests): task_002 완료됨 → can_start ✓ → claim → WORK
13. bob   IDLE poll ② → scan → 남은 unclaimed 없음 → 폴링 계속 → 60초 → SHUTDOWN

14. alice: tests.py 작성 → complete_task(task_003) → IDLE → 60초 → SHUTDOWN
15. Lead consume_lead_inbox → alice, bob의 summary 수신
```

Lead는 1~5번만 하면 끝. 중간 태스크 할당 없음.

---

## s16에서 달라진 점

| 구성 요소 | s16 | s17 |
|---|---|---|
| 태스크 할당 | Lead가 수동 | 팀원이 auto-claim (`can_start` 의존성 확인) |
| 팀원 라이프사이클 | WORK → SHUTDOWN | WORK → IDLE (60초 폴링) → SHUTDOWN |
| claim_task | owner 체크 없음 | 이미 owner 있으면 거부 |
| IDLE 단계 shutdown | 처리 안 함 | 즉시 dispatch하고 종료 |
| 팀원 tools | 5개 | 8개 (+list_tasks, claim_task, complete_task) |
| 팀원 종료 시점 | 태스크 완료 후 | 60초 유휴 타임아웃 후 |
| Identity 지속성 | system prompt만 | 압축 감지 후 자동 재주입 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s17_autonomous_agents/code.py
```

시도해볼 prompt:

`Create 3 tasks on the board, then spawn alice and bob. Watch them auto-claim and work.`

의존성 있는 태스크도 시도해보자:

`Create task A. Create task B that is blocked by task A. Spawn alice. Watch her claim A, complete it, then auto-claim B.`

관찰 포인트:
- 팀원이 할당되지 않은 태스크를 auto-claim하는가?
- `blockedBy` 의존성이 있는 태스크가 의존성 완료 후에만 claim되는가?
- 유휴 타임아웃 60초 후 shutdown이 트리거되는가?
- IDLE 단계의 `shutdown_request`가 즉각 응답을 받는가?
- `.tasks/` 디렉터리의 태스크 JSON 파일 상태가 어떻게 변하는가?
