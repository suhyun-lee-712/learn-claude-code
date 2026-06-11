# s16: Team Protocols — 팀원에게는 합의가 필요하다

> **핵심 한 줄**: 텍스트 메시지만으로는 부족하다. request_id로 요청과 응답을 연결하고, 상태를 추적해야 신뢰할 수 있는 협업이 가능하다.

---

## 왜 필요한가?

s15의 팀원들은 소통은 할 수 있지만 구조가 없다. 그냥 텍스트 메시지를 주고받는 것뿐이다. 두 가지 시나리오가 이 빈틈을 드러낸다.

**시나리오 1: Shutdown**

```
Lead: "Alice, 종료해."
Alice: (스레드 강제 종료)
```

Alice가 파일을 쓰는 중이었다면? 디스크에 반쯤 쓰인 파일이 남는다. 핸드셰이크가 필요하다.

```
Lead:  shutdown_request  →  Alice
Alice: 파일 마무리 후    →  shutdown_response (approve: True)
Lead:  "확인됨. 종료."
```

**시나리오 2: Plan Approval**

```
Bob: "auth 모듈 리팩터링할게" → (바로 실행 → 위험)
```

위험도 높은 작업은 Lead가 승인하기 전까지 실행하면 안 된다.

```
Bob:  plan_approval_request (계획 제출)  →  Lead
Lead: 검토 후 plan_approval_response (approve: True)  →  Bob
Bob:  (그제서야 실행)
```

두 시나리오는 같은 구조를 공유한다.

```
한쪽이 요청 → 다른 쪽이 응답 → 같은 ID로 연결됨
```

이것이 s16의 핵심인 **request-response protocol**이다.

---

## 왜 텍스트 메시지만으로는 부족한가?

"그냥 shutdown해 라는 텍스트 메시지로 충분하지 않나?" 라고 생각할 수 있다. 세 가지 문제가 있다.

**① 완료 확인 불가**

```
Lead → "shutdown해" (inbox에 씀)
Lead: 다음 작업 진행...
Alice: (아직 파일 쓰는 중)
```

Lead는 Alice가 실제로 종료했는지 알 방법이 없다.

**② 같은 상대에게 요청이 여러 개일 때**

`from` 필드로 발신자는 알 수 있다. 하지만 어떤 요청에 대한 응답인지는 알 수 없다.

```
Lead → alice: plan_approval_request  # 1번 계획
Lead → alice: plan_approval_request  # 2번 계획 (재제출)

alice → lead: plan_approval_response { approve: True }
```

어떤 계획이 승인된 건지 알 수가 없다.

> - `from` 필드 → **누가** 보냈는지
> - `request_id` → **어떤 요청에 대한** 응답인지

**③ 타이밍 제어 불가**

Lead가 "shutdown해"를 보낸 뒤 즉시 다음 작업을 진행하면, Alice가 마무리하기 전에 Lead가 그 파일을 읽으려 할 수도 있다.

---

## 핵심 구성 요소

s16에서 새로 추가된 것들:

1. **ProtocolState** — 요청 상태 추적 (pending → approved/rejected)
2. **dispatch_message** — type에 따라 메시지를 적절한 핸들러로 라우팅
3. **match_response** — request_id로 응답을 요청과 매칭 + type 검증
4. **consume_lead_inbox** — 통합된 Lead inbox 소비자
5. **Idle Loop** — 10라운드 대신 메시지를 기다리는 대기 상태

---

## 1. ProtocolState: 요청 상태 추적

request_id로 요청을 추적하려면 어딘가에 저장해야 한다. 그것이 `ProtocolState`다.

```python
@dataclass
class ProtocolState:
    request_id: str      # "req_004281"
    type: str            # "shutdown" | "plan_approval"
    sender: str          # 요청 보낸 쪽
    target: str          # 요청 받는 쪽
    status: str          # "pending" | "approved" | "rejected"
    payload: str         # 계획 내용 또는 종료 이유
    created_at: float    # 타임스탬프

pending_requests: dict[str, ProtocolState] = {}
```

```
Lead가 요청 보낼 때:
  pending_requests["req_004281"] = ProtocolState(status="pending", ...)

Lead가 응답 받을 때:
  pending_requests["req_004281"].status = "approved"
```

딕셔너리의 key가 request_id다. 응답이 오면 같은 key로 찾아서 상태를 업데이트한다.

### ProtocolState는 왜 inbox 메시지에 들어가지 않나?

inbox 메시지와 ProtocolState는 역할이 다르기 때문이다.

```
Lead가 요청을 보낼 때:
  1. BUS.send(...)         → alice의 inbox에 메시지 씀  (alice 것)
  2. pending_requests[id]  → Lead 자신의 장부에 기록    (Lead 것)

Alice가 응답을 보낼 때:
  3. alice는 읽고 삭제 (consume)
  4. alice의 inbox는 이미 비워짐
```

inbox 메시지는 전달용이고, 읽히는 순간 사라진다. Lead가 자신이 보낸 요청을 계속 추적하려면 자기 쪽에 별도로 저장해야 한다.

> - **inbox 메시지** = 상대방에게 보내는 편지 (전달 후 소비됨)
> - **pending_requests** = 내가 보낸 편지를 추적하는 내 장부 (나한테 남아있음)

---

## 2. 4단계 Protocol 흐름

shutdown을 예로 전체 흐름을 보면:

```
① Lead가 요청 생성
   req_id = "req_004281"
   pending_requests[req_id] = ProtocolState(type="shutdown", status="pending")
   BUS.send("lead", "alice", "shutdown_request", metadata={"request_id": req_id})

② Alice inbox에서 수신 → type 보고 핸들러로 라우팅 (dispatch_message)
   msg_type = "shutdown_request"
   → handle_shutdown_request() 호출

③ Alice가 응답
   BUS.send("alice", "lead", "shutdown_response",
            metadata={"request_id": req_id, "approve": True})

④ Lead가 응답 수신 → request_id로 매칭 (match_response)
   pending_requests["req_004281"].status = "approved"
```

`request_id`가 ①~④ 전체를 관통하는 연결 키다.

---

## 3. dispatch_message: type에 따른 라우팅

s15에서는 inbox 메시지를 전부 텍스트로 취급했다. s16에서는 type에 따라 다르게 처리해야 한다.

```python
def handle_inbox_message(name, msg, messages):
    msg_type = msg.get("type", "message")
    req_id = msg.get("metadata", {}).get("request_id", "")

    if msg_type == "shutdown_request":
        BUS.send(name, "lead", "Shutting down.", "shutdown_response",
                 {"request_id": req_id, "approve": True})
        return True   # 루프 종료

    if msg_type == "plan_approval_response":
        approve = msg["metadata"].get("approve", False)
        messages.append({"role": "user",
            "content": "[Plan approved]" if approve else "[Plan rejected]"})
    return False      # 계속 진행
```

새 protocol을 추가하려면 `if` 분기 하나만 추가하면 된다. 확장이 쉬운 구조다.

---

## 4. match_response: 응답-요청 매칭 + type 검증

```python
def match_response(response_type, request_id, approve):
    state = pending_requests.get(request_id)
    if not state:
        return  # 모르는 요청
    if state.type == "shutdown" and response_type != "shutdown_response":
        return  # type 불일치
    if state.status != "pending":
        return  # 이미 처리된 요청 (중복 응답 방지)
    state.status = "approved" if approve else "rejected"
```

request_id로 찾는 것뿐 아니라 **type도 검증**한다. `shutdown_response`가 실수로 `plan_approval` 요청을 승인하는 일을 막는다.

---

## 5. consume_lead_inbox: 통합된 inbox 소비자

s15에서는 inbox를 읽는 곳이 두 군데였다:
- `check_inbox` tool — Lead LLM이 직접 호출할 때
- 메인 loop 끝 — 매 턴마다 자동으로 확인

s16에서 이 구조가 문제가 된다. `shutdown_response`가 Lead inbox에 들어왔다고 치자.

```
경우 A: 메인 loop가 먼저 읽음
  → 메시지를 history에 주입 (LLM은 봄)
  → 근데 match_response를 안 불렀으니 pending_requests 상태는 여전히 "pending"
  → 핸드셰이크가 완료됐는데 Lead는 모름

경우 B: check_inbox tool이 먼저 읽음
  → match_response 호출 → pending_requests 상태 "approved"로 업데이트
  → 파일은 소비(consume)됨
  → 메인 loop가 나중에 읽으려 하면 이미 없음 → history 주입 안 됨
```

메시지를 읽으면 삭제(consume)되니까, 두 곳이 따로 읽으면 **읽기**와 **상태 업데이트** 중 하나가 반드시 빠진다.

해결책이 `consume_lead_inbox`다. `check_inbox` tool도, 메인 loop도 이 함수 하나만 호출한다.

```python
def consume_lead_inbox(route_protocol=True) -> list[dict]:
    msgs = BUS.read_inbox("lead")       # 한 번만 읽음
    if route_protocol:
        for msg in msgs:
            meta = msg.get("metadata", {})
            req_id = meta.get("request_id", "")
            msg_type = msg.get("type", "")
            if req_id and msg_type.endswith("_response"):
                match_response(msg_type, req_id, meta.get("approve", False))
    return msgs                         # history 주입은 호출한 쪽에서
```

읽기 + 상태 업데이트가 항상 같이 일어나니까, 어느 쪽이 먼저 호출하든 protocol 처리가 빠지는 일이 없다.

---

## 6. Idle Loop: s15 타이밍 문제의 구조적 해결

s15에서 alice가 round 1을 먼저 돌아서 Lead의 메시지를 놓쳤던 문제를 기억하는가. s16에서 이것이 해결된다.

**s15**: 최대 10라운드 돌고 종료
**s16**: LLM이 tool 없이 응답하면 → idle 대기

```
LLM이 non-tool_use 응답 반환
  → inbox를 1초마다 폴링
  → shutdown_request 오면 → 응답 후 종료
  → 새 메시지 오면 → history에 주입 → LLM 재개
```

alice가 일을 마쳐도 종료하지 않고 기다린다. Lead가 메시지를 언제 보내든 놓치지 않는다. s15의 race condition이 구조적으로 사라진다.

> 교육용 버전은 idle 상태 진입 시 `idle_notification` 전송을 생략한다. 실제 CC는 idle이 되면 Lead에게 알려서, Lead가 팀원이 새 작업을 받을 수 있는 상태임을 알 수 있게 한다.

---

## 전체 흐름 (Shutdown 예시)

```
1. Lead: "Have Alice create a file, then shut her down"
2. Lead → spawn_teammate("alice", "backend", "Create config.py")
3. alice thread 시작 → write_file("config.py", "...") → 완료 → idle 대기

4. Lead → request_shutdown("alice")
   → BUS.send("shutdown_request", {request_id: "req_000142"})
   → pending_requests["req_000142"] = ProtocolState(status="pending")

5. alice idle 폴링 중 → shutdown_request 수신 → handle_shutdown_request
   → BUS.send("shutdown_response", {request_id: "req_000142", approve: True})

6. Lead consume_lead_inbox → match_response("req_000142", approve=True)
   → pending_requests["req_000142"].status = "approved"
   → inbox 메시지가 history에 주입 → LLM이 shutdown 완료를 인지
```

모든 단계가 `request_id`로 추적된다.

---

## 실제 CC의 Protocol 타입

교육용 코드는 shutdown과 plan_approval 두 가지만 구현했지만, 실제 CC에는 request-response 패턴을 쓰는 protocol이 더 있다.

| Protocol | 메시지 타입 | 방향 |
|---|---|---|
| Shutdown | `shutdown_request` / `shutdown_approved` / `shutdown_rejected` | Lead → Teammate |
| Plan approval | `plan_approval_request` / `plan_approval_response` | Teammate → Lead |
| Permission | `permission_request` / `permission_response` | Teammate → Lead |
| Sandbox permission | `sandbox_permission_*` | 양방향 |

Permission은 s15에서 배운 **Permission Bubbling**이다. 팀원이 위험한 작업을 만났을 때 사용자 승인을 요청하는 흐름인데, 내부적으로 동일한 request_id 기반 handshake를 사용한다.

이 모든 protocol이 같은 FSM(pending → approved | rejected)을 공유한다. 교육용의 단일 ProtocolState 구조가 실제 CC에도 그대로 적용되는 이유다.

---

## s15에서 달라진 점

| 구성 요소 | s15 | s16 |
|---|---|---|
| 협업 방식 | 느슨한 텍스트 메시지 | 구조화된 request-response protocol |
| 요청 추적 | 없음 | ProtocolState + pending_requests dict |
| 메시지 라우팅 | 모두 텍스트로 취급 | dispatch_message가 type에 따라 라우팅 |
| Shutdown | 자연 종료 또는 강제 종료 | request_id 핸드셰이크 |
| Plan approval | 없음 | 메시지 흐름 예시 (execution gating 없음) |
| 팀원 라이프사이클 | 최대 10라운드 후 종료 | Idle loop (메시지를 기다림) |
| Lead inbox | check_inbox와 메인 loop가 각각 따로 읽음 | 통합된 consume_lead_inbox |
| 팀원 tool | 4개 (s15) | + submit_plan (5개) |

---

## CC 소스 깊이 들여다보기

<details>
<summary>펼치기</summary>

### Shutdown protocol — 3자 통신

교육용:
```
Lead → alice: shutdown_request
alice → lead: shutdown_response { approve: True }
```

실제 CC:
```
Lead → alice: shutdown_request
alice → lead: shutdown_approved  (또는 shutdown_rejected)
system → 모두: teammate_terminated
```

세 번째 메시지가 핵심이다. `teammate_terminated`는 **CC 런타임(하네스)**이 보내는 공식 종료 알림이다. Lead도 Alice도 아닌, 그 둘을 실행시키는 프레임워크 코드 자체가 행동하는 것이다.

비유하면:
- **Lead** = 팀장 ("Alice, 오늘부로 프로젝트 종료해")
- **Alice** = 팀원 ("알겠어요, 마무리했어요")
- **CC 런타임** = 인사팀 ("퇴사 처리 완료, 사내 시스템 접근 권한 회수, 전체 공지 발송")

`teammate_terminated`가 오면:
- tmux/iTerm2 pane 정리
- 작업 할당 해제
- team config에서 멤버 제거

이 역할을 담당하는 코드가 `useInboxPoller.ts:677-800`이다.

교육용은 응답 type을 `shutdown_response` 하나로 통합했지만, 실제 CC는 `shutdown_approved` / `shutdown_rejected` 두 개로 분리돼 있다. 거절 시 이유도 담을 수 있다.

### Plan Approval — 실제로는 더 복잡

교육용에서는 Lead가 `review_plan` tool을 수동으로 써서 승인하는 구조다.

실제 CC는 다르다:
- 팀원이 plan mode에서 빠져나올 때 **자동으로** plan_approval_request가 생성됨 (`ExitPlanModeV2Tool.ts`)
- 승인 시 `permissionMode`도 함께 설정 가능 → "승인하되, plan mode로 실행해"
- 거절 시 `feedback` 문자열 포함 가능 → "이 부분 수정해서 다시 제출해"

단순한 approve/reject가 아니라 협상이 가능한 구조다.

### 메시지 포맷 — Zod 검증 + 필드명 불일치

교육용은 그냥 Python dict다:
```python
{"type": "shutdown_request", "metadata": {"request_id": "req_001"}}
```

실제 CC는 TypeScript + Zod schema로 검증한다. 잘못된 포맷이 들어오면 런타임에서 차단된다.

그런데 필드명이 일관되지 않다:
```
permission 메시지   → request_id  (snake_case)
shutdown / plan     → requestId   (camelCase)
```

실제 프로덕션 코드에도 이런 역사적 불일치가 존재한다.

### Execution Gating — 교육용에 없는 것

교육용 버전은 plan_approval_request를 보내고 응답을 받지만, 실제로 실행을 막지는 않는다. Bob이 승인 안 받고 바로 `bash`를 써도 막을 방법이 없다.

실제 CC는 다르다:
```
bob이 bash 호출 시도
→ permission gate 확인
→ 승인된 plan 범위 내? → 허용
→ 아니면? → 차단 + Lead에게 permission_request
```

승인되지 않은 작업은 물리적으로 실행이 안 된다. 교육용은 메시지 흐름만 보여주고 gating은 구현하지 않은 것이다.

### 요약

| | 교육용 | 실제 CC |
|---|---|---|
| Protocol 종류 | shutdown, plan_approval (2개) | shutdown, plan_approval, permission, sandbox_permission (4개+) |
| Shutdown | 2자 (request/response) | 3자 (+ CC 런타임의 terminated) |
| Plan approval | 수동 approve | 자동 생성 + 협상 가능 |
| 메시지 포맷 | dict | Zod 검증 JSON |
| Execution gating | 없음 | 물리적으로 차단 |

</details>

---

## 직접 해보기

```sh
cd learn-claude-code
python s16_team_protocols/code.py
```

시도해볼 prompt:

1. `Spawn alice as a backend dev. Ask her to create a file. Then request her shutdown.`
2. `Spawn bob with a refactoring task. Have him submit a plan first. Then review and approve it.`

관찰 포인트:
- shutdown 핸드셰이크가 완전한가? (request → confirm → shutdown)
- `pending_requests` 상태가 올바르게 전이되는가? (pending → approved)
- `request_id`가 요청과 응답 사이에서 일관되게 유지되는가?
- idle 상태의 팀원이 shutdown_request를 받을 수 있는가?

---

## 실행 로그 (Debug 모드)

```sh
python s16_team_protocols/code.py --debug
```

<details>
<summary>전체 로그 펼치기</summary>

```
s16: team protocols
[debug mode ON — set DEBUG=0 or remove --debug to disable]

s16 >> Spawn alice as a backend dev. Ask her to create a file. Then request her shutdown.
[DBG:lead] sending 1 message(s) to LLM
[DBG:lead] stop_reason=tool_use input_tokens=1584 output_tokens=125
> spawn_teammate
[DBG:teammate:alice] system prompt: You are 'alice', a backend dev. ...
[DBG:teammate:alice] initial prompt: You are Alice, a backend developer. Wait for instructions from the Lead.
[DBG:teammate:alice] ── round 1 ──
  [teammate] alice spawned as backend dev
[DBG:bus:inbox] alice received 1 message(s):
[DBG:bus:inbox]   from=lead type=message | Hi Alice! Please create a file called `schema.sql` ...
[DBG:lead] stop_reason=tool_use input_tokens=1732 output_tokens=109
> send_message
[DBG:lead] tool=send_message input={"to": "alice", "content": "Hi Alice! Please create a file called `hello.txt` ..."}
  [bus] lead → alice: (message) Hi Alice! Please create a file called `hello.txt`
[DBG:lead] stop_reason=tool_use input_tokens=1857 output_tokens=64
> request_shutdown
[DBG:protocol:new] shutdown request created: req=req_473139 target=alice status=pending
[DBG:teammate:alice] stop_reason=tool_use msgs_in_context=2
[DBG:teammate:alice] tool=write_file input={"path": ".../schema.sql", ...}
  [bus] lead → alice: (shutdown_request) Please shut down gracefully.
  [protocol] shutdown_request → alice (req_473139)
[DBG:lead] tool_result: Shutdown request sent to alice (req: req_473139)
[DBG:teammate:alice] tool_result: Wrote 781 bytes to .../schema.sql
[DBG:teammate:alice] ── round 2 ──
[DBG:bus:inbox] alice received 2 message(s):
[DBG:bus:inbox]   from=lead type=message | Hi Alice! Please create a file called `hello.txt` ...
[DBG:bus:inbox]   from=lead type=shutdown_request req=req_473139 | Please shut down gracefully.
[DBG:teammate:alice] dispatch: type=shutdown_request req=req_473139
[DBG:teammate:alice] shutdown_request received → sending shutdown_response approve=True
  [bus] alice → lead: (shutdown_response) Shutting down gracefully.
  [protocol] alice approved shutdown (req_473139)
  [bus] alice → lead: (result) I'll get started on that right away! ...
  [teammate] alice finished
[DBG:lead] stop_reason=end_turn input_tokens=1946 output_tokens=106

All three steps are done! ...

[DBG:bus:inbox] lead received 2 message(s):
[DBG:bus:inbox]   from=alice type=shutdown_response req=req_473139 | Shutting down gracefully.
[DBG:bus:inbox]   from=alice type=result | I'll get started on that right away! ...
[DBG:consume] routing protocol response: type=shutdown_response req=req_473139 approve=True
[DBG:protocol:match] response_type=shutdown_response request_id=req_473139 approve=True
  [protocol] shutdown ✓ (req_473139: approved)
[DBG:protocol:match] state transition: pending → approved (type=shutdown sender=lead target=alice)
[DBG:consume] non-protocol message: type=result from=alice
[DBG:lead:inject] injecting 2 inbox message(s) into history:
[DBG:lead:inject]   from=alice type=shutdown_response | Shutting down gracefully.
[DBG:lead:inject]   from=alice type=result | I'll get started on that right away! ...

[Inbox: 2 messages injected]

s16 >> Spawn bob with a refactoring task. Have him submit a plan first. Then review and approve it.
[DBG:lead] sending 10 message(s) to LLM
> spawn_teammate
[DBG:teammate:bob] ── round 1 ──
  [teammate] bob spawned as backend dev
[DBG:bus:inbox] bob → (empty)
[DBG:teammate:bob] stop_reason=tool_use msgs_in_context=1
[DBG:teammate:bob] tool=bash input={"command": "cat /inbox/bob 2>/dev/null || echo \"No messages yet.\""}
[DBG:teammate:bob] ── round 2 ──
[DBG:bus:inbox] bob → (empty)
> request_plan
  [bus] lead → bob: (message) Please submit a plan for: Refactor the existing codebase ...
[DBG:teammate:bob] stop_reason=end_turn msgs_in_context=3
[DBG:teammate:bob] final text: No messages in my inbox yet. I'm standing by and ready to receive instructions.
[DBG:teammate:bob] → idle loop (waiting for inbox)
> check_inbox (×여러 번, 모두 empty)

[DBG:bus:inbox] bob received 1 message(s):
[DBG:bus:inbox]   from=lead type=message | Please submit a plan for: Refactor the existing codebase ...
[DBG:teammate:bob] idle: received 1 message(s) → resuming
[DBG:teammate:bob] ── round 3 ──
... (bob이 코드베이스 분석: bash로 파일 목록, wc -l, grep def 등)
[DBG:teammate:bob] ── round 6 ──
[DBG:teammate:bob] tool=submit_plan input={"plan": "# Refactoring Plan ..."}
[DBG:protocol:new] plan_approval request created: req=req_282644 sender=bob status=pending
  [bus] bob → lead: (plan_approval_request) # Refactoring Plan ...
[DBG:teammate:bob] → idle loop (waiting for inbox)
[DBG:bus:inbox] bob → (empty)  ← (반복)

> check_inbox
[DBG:consume] non-protocol message: type=plan_approval_request from=bob
> review_plan
[DBG:lead] tool=review_plan input={"request_id": "req_282644", "approve": true, "feedback": "Great plan, Bob! ..."}
  [bus] lead → bob: (plan_approval_response) Great plan, Bob! ...
  [protocol] plan ✓ (req_282644)

[DBG:bus:inbox] bob received 1 message(s):
[DBG:bus:inbox]   from=lead type=plan_approval_response req=req_282644 | Great plan, Bob! ...
[DBG:teammate:bob] idle: received 1 message(s) → resuming
[DBG:teammate:bob] dispatch: type=plan_approval_response req=req_282644
[DBG:teammate:bob] plan_approval_response received: approve=True feedback=Great plan, Bob! ...

[DBG:lead] stop_reason=end_turn
All done! Bob is now cleared to start the refactoring work!
```

</details>

### 이 로그에서 주목할 점

**① 잔여 inbox 메시지로 인한 예상치 못한 동작 (alice round 1)**

```
[DBG:teammate:alice] ── round 1 ──
[DBG:bus:inbox] alice received 1 message(s):
[DBG:bus:inbox]   from=lead type=message | Hi Alice! Please create a file called `schema.sql` ...
```

Lead가 이번 세션에서 보낸 건 `hello.txt`인데, alice는 round 1에서 `schema.sql`을 만들었다. 이전 실행에서 `.mailboxes/alice.jsonl`에 남아있던 메시지가 이번 세션에서 처리된 것이다. 코드를 반복 실행할 때 `.mailboxes/` 디렉토리를 정리하지 않으면 잔여 메시지가 다음 실행에 영향을 줄 수 있다.

**② alice의 graceful shutdown 성공 — s15의 race condition 해결 확인**

```
[DBG:teammate:alice] ── round 2 ──
[DBG:bus:inbox] alice received 2 message(s):
[DBG:bus:inbox]   from=lead type=message        | Hi Alice! Please create a file called `hello.txt` ...
[DBG:bus:inbox]   from=lead type=shutdown_request req=req_473139 | Please shut down gracefully.
[DBG:teammate:alice] dispatch: type=shutdown_request req=req_473139
[DBG:teammate:alice] shutdown_request received → sending shutdown_response approve=True
```

s15에서는 alice가 round 1에서 Lead의 메시지를 놓쳤지만, s16의 idle loop 덕분에 alice는 shutdown_request를 round 2에서 정확히 수신하고 graceful shutdown을 완료했다. 단, shutdown을 먼저 dispatch하면서 `hello.txt` 요청은 처리되지 못했다 — shutdown이 우선순위가 더 높기 때문이다.

**③ Lead가 summary를 출력한 뒤에 protocol 상태가 전이됨**

```
[DBG:lead] stop_reason=end_turn
All three steps are done! ...      ← Lead LLM이 먼저 출력

(이후에)
[DBG:protocol:match] state transition: pending → approved  ← 그다음 상태 전이
```

Lead의 LLM이 "완료됐다"고 출력한 시점에는 `pending_requests`가 아직 `pending`이다. protocol 상태가 `approved`로 전이되는 건 메인 루프의 `consume_lead_inbox`가 실행되는 그 다음 단계다. LLM의 판단과 실제 protocol 상태 전이는 서로 다른 타이밍에 일어난다.

**④ bob의 idle loop → 재개 흐름**

```
[DBG:teammate:bob] final text: No messages in my inbox yet. I'm standing by...
[DBG:teammate:bob] → idle loop (waiting for inbox)
[DBG:bus:inbox] bob → (empty)  ← 1초마다 폴링
...
[DBG:bus:inbox] bob received 1 message(s): from=lead type=message
[DBG:teammate:bob] idle: received 1 message(s) → resuming
[DBG:teammate:bob] ── round 3 ──
```

bob이 round 2에서 idle loop에 진입하고, Lead의 `request_plan` 메시지를 받자 즉시 재개했다. s15에서 alice가 메시지를 놓쳤던 것과 달리, idle loop가 타이밍 문제를 구조적으로 해결하는 모습이 명확히 보인다.

**⑤ plan_approval_request는 non-protocol로 분류됨**

```
[DBG:consume] non-protocol message: type=plan_approval_request from=bob
```

`consume_lead_inbox`는 `_response`로 끝나는 메시지만 `match_response`로 라우팅한다. `plan_approval_request`는 request(요청)이므로 non-protocol로 분류되어 LLM에게 그대로 전달되고, Lead가 직접 `review_plan` tool로 처리한다.

