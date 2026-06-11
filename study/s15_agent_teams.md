# s15: Agent Teams — Agent 하나로는 부족하다, 팀을 꾸려라

> **핵심 한 줄**: 서로 소통하며 병렬로 일하는 teammate를 만들어, context window 한계와 속도 문제를 동시에 해결한다.

---

## 왜 필요한가?

"백엔드 전체를 리팩터링해줘" 같은 작업을 Agent 하나에게 시키면 두 가지 문제가 생긴다.

1. **Context window 한계**: 인증 코드, DB 레이어, API 라우트, 테스트가 전부 context에 들어가야 하는데, 작업하다 보면 앞에서 읽은 내용이 밀려나간다.
2. **순차 처리**: 한 Agent가 모든 걸 순서대로 처리하면 시간이 너무 오래 걸린다.

그렇다면 s06의 sub-agent를 쓰면 되지 않을까?

---

## Sub-agent vs Teammate

Sub-agent는 "심부름꾼", Teammate는 "진짜 팀원"이다.

| | s06 Sub-agent | s15 Teammate |
|---|---|---|
| 수명 | **일회성** — 일 끝나면 사라짐 | **멀티 턴** — 계속 살아있음 |
| 통신 | 결론만 반환 | 언제든 메시지 주고받기 가능 |
| Context | 완전히 격리 | 메시지를 통해 공유 |
| 적합한 상황 | 독립적인 단순 작업 | 서로 결과를 참조해야 하는 작업 |

### 언제 무엇을 써야 하나?

**Sub-agent가 적합한 경우** — "시켜놓고 결과만 받으면 되는" 작업
- 파일 10개를 각각 요약 → 10개를 병렬로 던지고 결과 취합
- 여러 URL에서 데이터 수집 → 각 URL마다 sub-agent 하나씩

**Agent Team이 적합한 경우** — "소통이 필요한" 작업
- alice(인증 담당)가 코드를 바꾸면 bob(API 담당)에게 알려야 할 때
- A가 만들고 → B가 검토하고 → A가 수정하는 사이클이 있을 때
- 장애 대응처럼 여러 영역을 **동시에** 빠르게 파야 할 때

> **핵심 판단 기준**: 작업 규모보다 **"agent 간에 정보가 흘러야 하는가"**가 더 중요하다.
> 소통이 필요하면 Team, 결과만 돌려주면 되면 Sub-agent.

### Agent Team의 실질적인 장점

```
혼자:  로그 분석(3분) → DB 확인(2분) → API 확인(2분) = 7분
Team:  alice: 로그 분석(3분) ─┐
       bob:   DB 확인(2분)   ──→ 3분에 끝
       charlie: API 확인(2분) ─┘
```

1. **병렬 처리로 시간 단축**: 독립적인 영역을 동시에 처리
2. **Context 분리**: 각 teammate가 자기 전담 영역에만 집중하므로 context가 가볍게 유지됨

> **주의**: Team은 스레드 관리, 메시지 버스, 동기화 같은 복잡성이 추가된다.  
> 일단 Agent 하나로 시작하고, 느려지거나 context 에러가 생길 때 Team으로 전환하는 게 현실적이다.

---

## 세 가지 핵심 요소

s15에서 새로 추가된 건 딱 세 가지다.

1. **MessageBus** — 파일 기반 메시지 전달 시스템
2. **spawn_teammate_thread** — teammate를 background thread로 생성
3. **Inbox Injection** — Lead가 teammate 메시지를 받아 대화 history에 주입

---

## 1. MessageBus: 파일 기반 편지함

각 Agent는 자기 이름의 `.jsonl` 파일(inbox)을 가진다.

```
.mailboxes/
  alice.jsonl   ← alice의 편지함
  bob.jsonl     ← bob의 편지함
  lead.jsonl    ← lead의 편지함
```

```python
class MessageBus:
    def send(self, from_agent, to_agent, content, msg_type="message"):
        msg = {"from": from_agent, "to": to_agent, "content": content, ...}
        inbox = MAILBOX_DIR / f"{to_agent}.jsonl"
        with open(inbox, "a") as f:
            f.write(json.dumps(msg) + "\n")   # 편지 넣기 (append)

    def read_inbox(self, agent):
        inbox = MAILBOX_DIR / f"{agent}.jsonl"
        msgs = [json.loads(line) for line in inbox.read_text().splitlines()]
        inbox.unlink()   # 읽으면 삭제 (소비)
        return msgs
```

- **쓰기** = 받는 사람 파일에 append (편지 넣기)
- **읽기** = read + **unlink(파일 삭제)** → 한 번 읽으면 사라짐 (consume 패턴)

### 왜 읽으면 삭제하나?

"한 번 읽은 메시지를 두 번 처리하지 않기 위해서"다. 이를 **consume(소비)** 패턴이라고 한다.

### 왜 메모리 큐가 아닌 파일을 쓰나?

| | 메모리 큐 | 파일 inbox |
|---|---|---|
| 프로세스 재시작 시 | 메시지 유실 | 메시지 유지 (영속성) |
| 디버깅 | 외부에서 볼 수 없음 | 파일 열어서 바로 확인 가능 |
| 프로세스 간 통신 | 같은 프로세스 내에서만 | 다른 프로세스, 다른 머신에서도 가능 |

파일 방식 = 속도는 조금 느리지만, **눈에 보이고, 죽어도 안 사라지고, 어디서든 접근 가능**하다.

### 교육용 버전 vs 실제 CC의 차이

교육용 코드는 `MessageBus` 클래스를 통해 메시지를 주고받는다. 실제 CC는 더 직접적이다.  
**각 Agent가 다른 Agent의 inbox 파일에 직접 쓴다** — 중앙 MessageBus 클래스가 없다.

| | 교육용 (s15) | 실제 CC |
|---|---|---|
| inbox 경로 | `.mailboxes/{name}.jsonl` | `~/.claude/teams/{teamName}/inboxes/{agentName}.json` |
| 파일 포맷 | JSONL (한 줄씩 append) | JSON 배열 (읽기 → append → 다시 쓰기) |
| 동시 쓰기 | 락 없음 (race condition 가능) | `proper-lockfile` (최대 10회 재시도) |

### 통신은 항상 inbox를 통해서

모든 통신은 inbox가 유일한 채널이다.

```
Lead     → Teammate:  teammate inbox에 append
Teammate → Lead:      lead inbox에 append
Teammate → Teammate:  상대방 inbox에 append
```

메시지 타입에 상관없이 **항상 받는 사람의 inbox 파일에 append**하는 방식으로 통일돼 있다.  
Agent들이 서로의 존재를 직접 알 필요가 없다는 점이 핵심 — 이를 **느슨한 결합(loose coupling)** 이라고 한다.

### 실제 CC의 메시지 타입 15가지

교육용 코드는 `message`, `result` 두 가지만 쓰지만, 실제 CC에는 15가지 구조화된 메시지 타입이 있다.

| Type | 방향 | 용도 |
|---|---|---|
| `plain text` | 양방향 | teammate 간 일반 통신 |
| `idle_notification` | Teammate → Lead | teammate가 한 턴을 끝내고 idle 상태가 됨 |
| `permission_request` | Teammate → Lead | teammate가 작업 승인을 필요로 함 |
| `permission_response` | Lead → Teammate | Lead의 승인 결과 |
| `plan_approval_request` | Teammate → Lead | teammate가 검토를 위해 plan 제출 |
| `plan_approval_response` | Lead → Teammate | Lead의 plan 검토 |
| `shutdown_request` | Lead → Teammate | graceful shutdown 요청 |
| `shutdown_approved` | Teammate → Lead | shutdown 확인 |
| `shutdown_rejected` | Teammate → Lead | shutdown 거절 (사유 포함) |
| `task_assignment` | Lead → Teammate | task 할당 |
| `team_permission_update` | Lead → Teammate | 권한 변경 broadcast |
| `mode_set_request` | Lead → Teammate | teammate의 권한 mode 변경 |
| `sandbox_permission_*` | 양방향 | 네트워크 권한 요청/응답 |
| `teammate_terminated` | System | teammate 제거 알림 |

텍스트 메시지는 모델에 전달하기 위해 `<teammate-message>` XML 태그로 감싸진다.

---

## 2. spawn_teammate_thread: 팀원 생성

Lead가 `spawn_teammate` tool을 호출하면, teammate가 **별도의 background thread**에서 실행된다.

```python
def spawn_teammate_thread(name, role, prompt):
    system = f"You are '{name}', a {role}. Use tools to complete tasks."

    def run():
        messages = [{"role": "user", "content": prompt}]
        sub_tools = [bash, read_file, write_file, send_message]  # 4개만

        for _ in range(10):                       # 최대 10라운드 (교육용)
            inbox = BUS.read_inbox(name)          # ① 편지 확인
            if inbox:
                messages.append(...)              # ② history에 주입
            response = client.messages.create(...)# ③ LLM 호출
            # ... 도구 실행 ...

        BUS.send(name, "lead", summary, "result") # 완료 후 Lead에 보고

    threading.Thread(target=run, daemon=True).start()
```

### System prompt는 누가 정하나?

- **템플릿 구조**는 코드에 고정: `"You are '{name}', a {role}. Use tools to complete tasks."`
- **내용(name, role)** 은 Lead가 spawn할 때 결정

Lead가 `spawn_teammate("alice", "security reviewer", "Check auth module")` 을 호출하는 순간 system prompt가 동적으로 만들어진다.

> **실제 CC에서는**: 자유로운 role 문자열 대신 미리 검증된 **agentType**을 선택하는 방식이다.
> (`general-purpose`, `Explore`, `Plan` 등)

### 왜 teammate의 도구가 4개뿐인가?

교육용에서 의도적으로 줄인 것이다. 팀 간 통신 메커니즘에 집중하기 위해 bash, read, write, send_message만 남겼다.  
실제 CC의 teammate는 `TaskCreate`, `TaskUpdate` 같은 task 관련 도구도 모두 갖는다.

### 실제 CC의 Teammate 생명주기

실제 CC에서 teammate는 `spawnTeammate()`로 생성된다 (`spawnMultiAgent.ts`):

```
1. Spawn:  tmux pane 생성(또는 in-process), 색상 할당, 팀 config 작성
2. Work:   useInboxPoller가 1초마다 inbox 확인 → 메시지 도착 시 새 턴으로 제출
3. Idle:   Stop hook 발동 → Lead에게 idle_notification 전송
4. Shutdown: Lead가 shutdown_request 전송 → teammate가 shutdown_approved 응답 → Lead가 정리
```

교육용 코드의 teammate는 10라운드 후 자동 종료되지만, 실제 CC는 **idle loop**를 사용한다.  
`idle_notification`을 보내고 inbox 메시지를 기다리다가, 메시지가 오면 재개하고, `shutdown_request`가 올 때만 종료한다.

### 실제 CC의 팀 Config

팀 정보는 `~/.claude/teams/{teamName}/config.json`에 저장된다:

```json
{
  "name": "my-team",
  "leadAgentId": "lead@my-team",
  "members": [{
    "agentId": "researcher@my-team",
    "name": "researcher",
    "agentType": "general-purpose",
    "color": "blue",
    "isActive": true
  }]
}
```

> **중요한 제약**: Teammate는 중첩될 수 없다.  
> teammate가 다른 teammate를 spawn하는 것은 명시적으로 금지되어 있다 (`AgentTool.tsx:273`).

---

## 3. Inbox Injection: 팀원 결과를 Lead에게

Lead의 main loop가 한 번 돌 때마다 inbox를 확인해서, 온 메시지를 **history에 주입**한다.

```python
# 메인 loop 끝에서
inbox = BUS.read_inbox("lead")
if inbox:
    inbox_text = "\n".join(f"From {m['from']}: {m['content']}" for m in inbox)
    history.append({"role": "user", "content": f"[Inbox]\n{inbox_text}"})
```

그냥 화면에 출력만 하는 게 아니라 **history에 넣어서 LLM이 읽게** 한다.  
그래야 Lead가 "alice가 뭘 했는지"를 알고 다음 행동을 결정할 수 있다.

> 실제 CC에서 Lead의 `useInboxPoller`는 1초마다 확인하며, 사용자 입력을 기다리지 않고 메시지를 **새 턴으로 즉시 제출**한다.

---

## 전체 흐름

```
사용자: "Create schema.sql for alice"

1. Lead → spawn_teammate("alice", "backend dev", "Create users table")

2. alice thread 시작
   → LLM 호출 → write_file("schema.sql", ...) 실행
   → 완료

3. alice → BUS.send("alice", "lead", "Done! Created schema.sql", "result")

4. Lead main loop
   → BUS.read_inbox("lead") → history에 주입
   → Lead LLM: "alice가 완료했다고 함. schema.sql 확인..."
```

두 teammate가 병렬로 작업하는 경우:

```
1. Lead → spawn_teammate("alice", "backend dev", "Create database schema")
2. Lead → spawn_teammate("bob",   "frontend dev", "Write API client")
3. alice thread: write_file("schema.sql", ...)  ─┐ 병렬 실행
4. bob thread:   write_file("client.ts", ...)   ─┘
5. alice 완료 → BUS.send("alice", "lead", "Schema done")
6. bob 완료  → BUS.send("bob",   "lead", "Client written")
7. Lead inbox → 두 결과 모두 history에 주입 → LLM이 보고 다음 단계 조율
```

---

## Permission Bubbling (실제 CC)

교육용 코드는 생략하지만, 실제 CC에서는 teammate가 위험한 작업을 만나면 사용자에게 승인을 요청하는 흐름이 있다.

```
1. Teammate → Lead inbox:  permission_request 전송
2. Lead의 useInboxPoller (1초마다):  감지 → ToolUseConfirmQueue로 라우팅
3. Lead UI:  teammate 이름과 색상이 표시된 승인 대화창 표시
4. 사용자 승인
5. Lead → Teammate inbox:  permission_response 전송
6. Teammate의 useSwarmPermissionPoller (500ms마다):  응답 수신 → 작업 재개
```

Teammate polling이 Lead(1초)보다 빠른(500ms) 이유: permission 응답을 받으면 **즉시 작업을 재개**해야 하기 때문이다.

---

## s14에서 달라진 점 요약

| 구성 요소 | s14 이전 | s15 이후 |
|---|---|---|
| Agent 개수 | 1개 | Lead 1 + N개 teammate thread |
| 통신 | 없음 | MessageBus (.mailboxes/*.jsonl) |
| Lead 도구 수 | 11개 | +3개 (spawn_teammate, send_message, check_inbox) |
| Teammate 도구 | 없음 | bash, read, write, send_message (4개) |

---

## 직접 해보기

```sh
cd learn-claude-code
python s15_agent_teams/code.py
```

시도해볼 prompt:

1. `Spawn alice as a backend developer. Ask her to create a file called schema.sql with a users table.`
2. `Check your inbox for alice's result.`
3. `Spawn bob as a tester. Ask him to check if schema.sql exists and list its contents.`

관찰 포인트:
- `.mailboxes/` 디렉터리에 JSONL 파일이 어떻게 생기는가?
- teammate가 끝난 후 Lead inbox가 history에 주입되는가?
- 두 teammate가 병렬로 작업하는가?

---

## 실행 로그 (Debug 모드)

```sh
python s15_agent_teams/code.py --debug
```

<details>
<summary>전체 로그 펼치기</summary>

```
  [cron] scheduler thread started
s15: agent teams
[debug mode ON — set DEBUG=0 or remove --debug to disable]
Enter a question, press Enter to send. Type q to quit.

s15 >> Spawn alice as a backend developer. Ask her to create a file called schema.sql with a users table.
[DBG:lead] system prompt:
You are a coding agent. Act, don't explain.

Available tools: bash, read_file, write_file, get_task, create_task, list_tasks, claim_task, complete_task, schedule_cron, list_crons, cancel_cron, spawn_teammate, send_message, check_inbox.

Working directory: /Users/suhyun.lee/workspace/learn-claude-code/sandbox

Relevant memories:
- [user-preference-tabs](user-preference-tabs.md) — User prefers tabs for indentation over spaces
[DBG:lead] sending 1 message(s) to LLM
[DBG:lead] stop_reason=tool_use input_tokens=1631 output_tokens=244
> spawn_teammate
[DBG:lead] tool=spawn_teammate input={"name": "alice", "role": "backend developer", "prompt": "You are Alice, a backend developer. ..."}
[DBG:teammate:alice] system prompt: You are 'alice', a backend developer. Use tools to complete tasks. Send results via send_message to 'lead'.
[DBG:teammate:alice] initial prompt: You are Alice, a backend developer. ...
[DBG:teammate:alice] ── round 1 ──
  [teammate] alice spawned as backend developer
Teammate 'alice' spawned as backend developer
[DBG:lead] tool_result: Teammate 'alice' spawned as backend developer
> send_message
[DBG:lead] tool=send_message input={"to": "alice", "content": "Hi Alice! Please create a file called `schema.sql` ..."}
[DBG:bus:inbox] alice → (empty)
  [bus] lead → alice: Hi Alice! Please create a file called `schema.sql`
[DBG:bus:send] type=message | full content: Hi Alice! Please create a file called `schema.sql` in the working directory with a `users` table. Include common fields like id, name, email, created_at, etc.
Sent to alice
[DBG:lead] tool_result: Sent to alice
[DBG:lead] sending 3 message(s) to LLM
[DBG:teammate:alice] stop_reason=end_turn msgs_in_context=1
[DBG:teammate:alice] final text: Understood! I'm Alice, the backend developer. I'm ready and waiting for instructions from the Lead.
  [bus] alice → lead: Understood! I'm Alice, the backend developer. I'm
[DBG:bus:send] type=result | full content: Understood! I'm Alice, the backend developer. I'm ready and waiting for instructions from the Lead.
  [teammate] alice finished
[DBG:lead] stop_reason=tool_use input_tokens=1953 output_tokens=47
> check_inbox
[DBG:bus:inbox] lead received 1 message(s):
[DBG:bus:inbox]   from=alice type=result | Understood! I'm Alice, the backend developer. ...
[DBG:lead] stop_reason=tool_use input_tokens=2062 output_tokens=54
> check_inbox
[DBG:bus:inbox] lead → (empty)
(inbox empty)
[DBG:lead] stop_reason=tool_use input_tokens=2130 output_tokens=36
> check_inbox
[DBG:bus:inbox] lead → (empty)
(inbox empty)
[DBG:lead] stop_reason=tool_use input_tokens=2180 output_tokens=133
> bash
[DBG:lead] tool=bash input={"command": "ls .../schema.sql 2>/dev/null && cat .../schema.sql"}
(no output)
[DBG:lead] stop_reason=tool_use input_tokens=2327 output_tokens=199
> write_file
[DBG:lead] tool=write_file input={"path": ".../schema.sql", "content": "-- Users table\nCREATE TABLE users (\n\tid SERIAL PRIMARY KEY, ..."}
Wrote 264 bytes to .../schema.sql
[DBG:lead] stop_reason=end_turn input_tokens=2568 output_tokens=215

Done! Here's a summary of what was set up:
- ✅ Alice was spawned as a backend developer and messaged with the task.
- ✅ schema.sql was created at sandbox/schema.sql with a users table.

s15 >> Check your inbox for alice's result.
[DBG:lead] sending 15 message(s) to LLM
> check_inbox
[DBG:bus:inbox] lead → (empty)
(inbox empty)
Inbox is still empty — Alice hasn't sent back a result yet.

s15 >> Spawn bob as a tester. Ask him to check if schema.sql exists and list its contents.
[DBG:lead] sending 19 message(s) to LLM
> spawn_teammate
[DBG:lead] tool=spawn_teammate input={"name": "bob", "role": "tester", ...}
[DBG:teammate:bob] system prompt: You are 'bob', a tester. Use tools to complete tasks. Send results via send_message to 'lead'.
[DBG:teammate:bob] ── round 1 ──
  [teammate] bob spawned as tester
> send_message
[DBG:lead] tool=send_message input={"to": "bob", "content": "Hi Bob! Please check if schema.sql exists ..."}
[DBG:bus:inbox] bob → (empty)
  [bus] lead → bob: Hi Bob! Please check if the file `schema.sql` exis
Sent to bob
[DBG:lead] sending 21 message(s) to LLM
[DBG:teammate:bob] stop_reason=tool_use msgs_in_context=1
[DBG:teammate:bob] tool=send_message input={"to": "lead", "content": "Hi, this is Bob (tester). I'm online and ready ..."}
  [bus] bob → lead: Hi, this is Bob (tester). I'm online and ready to
[DBG:teammate:bob] ── round 2 ──
[DBG:bus:inbox] bob received 1 message(s):
[DBG:bus:inbox]   from=lead type=message | Hi Bob! Please check if the file `schema.sql` exists ...
[DBG:lead] stop_reason=tool_use input_tokens=3290 output_tokens=36
> check_inbox
[DBG:bus:inbox] lead received 1 message(s):
[DBG:bus:inbox]   from=bob type=message | Hi, this is Bob (tester). I'm online and ready ...
[DBG:lead] sending 25 message(s) to LLM
[DBG:teammate:bob] stop_reason=tool_use msgs_in_context=4
[DBG:teammate:bob] tool=bash input={"command": "ls .../schema.sql 2>&1"}
[DBG:teammate:bob] tool_result: .../schema.sql
[DBG:teammate:bob] tool=read_file input={"path": ".../schema.sql"}
[DBG:teammate:bob] tool_result:
-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
[DBG:teammate:bob] ── round 3 ──
[DBG:bus:inbox] bob → (empty)
[DBG:lead] stop_reason=tool_use input_tokens=3374 output_tokens=49
> check_inbox
[DBG:bus:inbox] lead → (empty)
(inbox empty)
[DBG:lead] stop_reason=tool_use input_tokens=3437 output_tokens=126
> bash
[DBG:lead] tool=bash input={"command": "ls .../schema.sql && cat .../schema.sql"}
.../schema.sql
--- FILE CONTENTS ---
-- Users table
CREATE TABLE users ( ... );
[DBG:lead] sending 29 message(s) to LLM
[DBG:teammate:bob] stop_reason=tool_use msgs_in_context=6
[DBG:teammate:bob] tool=send_message input={"to": "lead", "content": "Hi Lead, here are my findings:\n\n✅ File EXISTS\n\nFull Contents: CREATE TABLE users (...)"}
  [bus] bob → lead: Hi Lead, here are my findings: ...
[DBG:teammate:bob] ── round 4 ──
[DBG:bus:inbox] bob → (empty)
[DBG:teammate:bob] stop_reason=end_turn msgs_in_context=8
[DBG:teammate:bob] final text: I've reported back to the Lead with the full findings. Standing by for further instructions!
  [bus] bob → lead: I've reported back to the Lead with the full findi
  [teammate] bob finished
[DBG:lead] stop_reason=end_turn input_tokens=3683 output_tokens=108

Here's the verification result:
- ✅ schema.sql exists at sandbox/schema.sql
- ✅ Contents look good — the users table is properly defined.

[DBG:bus:inbox] lead received 2 message(s):
[DBG:bus:inbox]   from=bob type=message | Hi Lead, here are my findings: ...
[DBG:bus:inbox]   from=bob type=result  | I've reported back to the Lead with the full findings. ...
[DBG:lead:inject] injecting 2 inbox message(s) into history:
[DBG:lead:inject]   from=bob type=message | Hi Lead, here are my findings: ...
[DBG:lead:inject]   from=bob type=result  | I've reported back to the Lead with the full findings. ...

[Inbox: 2 messages injected]
```

</details>

### 이 로그에서 주목할 점

**① alice가 메시지를 못 받는 타이밍 문제**

```
[DBG:teammate:alice] ── round 1 ──       ← alice가 먼저 시작
[DBG:bus:inbox] alice → (empty)          ← inbox 비어있음 (Lead가 아직 send_message 전)
[DBG:teammate:alice] stop_reason=end_turn ← 할 일 없으니 종료
  [teammate] alice finished

(나중에) [bus] lead → alice: Hi Alice! ...  ← 편지를 보냈지만 alice는 이미 없음
```

spawn과 send_message 사이에 타이밍 차이가 있다. alice가 round 1을 먼저 돌고 종료된 뒤 메시지가 도착했다.

근본 원인은 10라운드 제한이 아니라 **idle loop의 부재**다. 교육용 코드는 스레드가 시작되자마자 즉시 round 1을 실행한다. 실제 CC라면 alice는 시작 후 `idle_notification`을 보내고 메시지가 올 때까지 기다린다 — Lead의 send_message가 언제 오든 놓치지 않는다. 10라운드를 100라운드로 늘려도 이 race condition 자체는 그대로다.

**② Lead가 직접 보완**

alice가 응답하지 않자 Lead가 alice의 inbox를 몇 번 확인(`check_inbox`)한 뒤, 직접 `write_file`로 schema.sql을 만들었다. Agent가 팀원에게만 의존하지 않고 스스로 판단해 보완하는 모습이다.

**③ bob과의 정상적인 비동기 통신**

```
[DBG:teammate:bob] ── round 1 ──
→ bob이 먼저 ready 메시지를 Lead에게 전송

[DBG:teammate:bob] ── round 2 ──
[DBG:bus:inbox] bob received 1 message(s): from=lead  ← Lead의 지시 도착
→ bob이 bash + read_file로 파일 확인 후 결과 전송
```

bob은 round 1에서 자기 소개 메시지를 보내고, round 2에서 Lead의 지시를 inbox로 받아 실제 작업을 수행했다. Lead와 bob이 비동기로 동시에 실행되면서 메시지를 주고받는 흐름이 명확히 보인다.

**왜 alice는 실패하고 bob은 성공했나?** 실행 순서 때문이다.

```
spawn alice   → alice thread 시작 (즉시 round 1)
spawn bob     → bob thread 시작 (즉시 round 1)
send_message("alice", ...)   ← alice는 이미 round 1 끝내고 inbox 지나침
send_message("bob", ...)     ← bob은 round 1 마치고 inbox 대기 중 → 수신 성공
```

alice는 spawn → send_message 간격이 짧아서 Lead의 메시지가 늦게 도착했다. bob은 alice에게 먼저 메시지를 보내는 시간만큼 간격이 벌어져, bob의 round 1이 끝날 타이밍에 맞게 메시지가 도착했다. 둘 다 동일한 race condition을 가지고 있고, bob은 우연히 타이밍이 맞아떨어진 것이다.

**④ 마지막 inbox injection**

```
[DBG:lead:inject] injecting 2 inbox message(s) into history:
  from=bob type=message | Hi Lead, here are my findings: ...
  from=bob type=result  | I've reported back to the Lead ...

[Inbox: 2 messages injected]
```

bob의 결과 2개(중간 보고 + 최종 결과)가 Lead의 history에 주입됐다. 다음 사용자 입력 시 Lead LLM이 이 내용을 보고 이어서 조율할 수 있다.
