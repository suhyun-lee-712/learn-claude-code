# s16: Team Protocols — 팀원에게는 합의가 필요하다

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → ... → s14 → s15 → `s16` → [s17](../s17_autonomous_agents/) → s18 → s19 → s20
> *"팀원에게는 합의가 필요하다"* — request-response 패턴이 모든 협상을 이끈다.
>
> **Harness Layer**: Protocols — Agent 간의 구조화된 핸드셰이크.

---

## 문제

s15의 팀원들은 일은 할 수 있지만 협업이 느슨하다. Lead가 메시지를 보내면 팀원이 답하는 식으로, 구조화된 protocol이 없다. 두 가지 시나리오가 이 빈틈을 드러낸다.

**Shutdown**: Lead가 Alice를 종료시키려 한다. 스레드를 곧바로 죽여 버리면 디스크에 절반만 쓰인 파일이 남는다. 핸드셰이크가 필요하다. Lead가 요청을 보내면 Alice가 마무리를 한 뒤 확인 응답을 준다.

**Plan approval**: Bob이 auth 모듈을 리팩터링하려 하는데, 이는 위험도가 높은 작업이다. Lead가 먼저 Bob의 plan을 검토하고 승인한 뒤에 Bob이 진행하도록 해야 한다.

두 시나리오는 같은 구조를 공유한다. 한쪽이 요청을 보내고, 다른 쪽이 응답하며, 둘은 같은 ID로 연결된다. 상태 머신이 pending → approved / rejected를 추적한다.

---

## 해결책

![팀 Protocol 개요](images/team-protocols-overview.en.svg)

교육용 코드는 앞 챕터들에서 이어져 온 agent 역량의 흐름을 계속 이어가면서, S15의 팀 커뮤니케이션 위에 구조화된 protocol을 더한다. protocol 메커니즘 자체에 집중하기 위해 완전한 error recovery, memory, skill 시스템은 생략했다. 추가된 것: **ProtocolState**(요청 상태 추적), **dispatch_message**(들어온 메시지를 type에 따라 handler로 라우팅), **match_response**(request_id로 응답을 요청과 연결하며 type 검증 수행).

두 개의 protocol, 하나의 메커니즘:

| Protocol | 방향 | 목적 |
|----------|-----------|---------|
| shutdown_request / response | Lead → Teammate | Graceful shutdown 핸드셰이크 |
| plan_approval_request / response | Teammate → Lead | Plan 승인 protocol 예시 |

> 교육용 버전은 plan 승인을 위한 request-response 메시지 흐름을 보여주지만, execution gating(승인되지 않았을 때 bash/write_file을 가로채는 것)은 구현하지 않는다. 실제 CC에는 팀원을 위한 permission gating 메커니즘이 있다.

---

## 동작 방식

### ProtocolState: 요청 상태

각 protocol 요청은 누가 누구에게 보냈는지, 현재 상태, 그리고 payload를 추적하는 상태 레코드를 생성한다:

```python
@dataclass
class ProtocolState:
    request_id: str      # Unique ID, e.g. "req_004281"
    type: str            # "shutdown" | "plan_approval"
    sender: str          # Sender
    target: str          # Recipient
    status: str          # pending | approved | rejected
    payload: str         # Plan text or shutdown reason
    created_at: float    # Timestamp

pending_requests: dict[str, ProtocolState] = {}
```

요청을 보낼 때 레코드가 생성되고, 응답을 받을 때 `request_id`로 찾아 그 상태를 업데이트한다.

### 4단계 Protocol 흐름

shutdown을 예로 든 전체 체인:

```
1. Lead sends request
   req_id = new_request_id()           # "req_004281"
   pending_requests[req_id] = ProtocolState(type="shutdown", status="pending", ...)
   BUS.send("lead", "alice", "shutdown_request", metadata={"request_id": req_id})

2. Teammate receives → dispatch
   inbox = BUS.read_inbox("alice")
   msg_type = msg["type"]              # "shutdown_request"
   → routed to handle_shutdown_request()

3. Teammate replies
   BUS.send("alice", "lead", "shutdown_response",
            metadata={"request_id": req_id, "approve": True})

4. Lead receives response → match
   match_response("shutdown_response", req_id, approve=True)
   pending_requests[req_id].status = "approved"
```

`request_id`는 체인 전체를 관통하는 연결 키다. 요청이 이를 가지고 나가고, 응답이 이를 가지고 돌아온다.

### dispatch_message: Type에 따른 라우팅

팀원의 inbox에는 일반 메시지와 protocol 메시지가 모두 들어온다. `handle_inbox_message`는 메시지 type에 따라 dispatch한다:

```python
def handle_inbox_message(name, msg, messages):
    msg_type = msg.get("type", "message")
    req_id = msg.get("metadata", {}).get("request_id", "")

    if msg_type == "shutdown_request":
        BUS.send(name, "lead", "Shutting down.", "shutdown_response",
                 {"request_id": req_id, "approve": True})
        return True   # Stop the loop

    if msg_type == "plan_approval_response":
        approve = msg["metadata"].get("approve", False)
        messages.append({"role": "user",
            "content": "[Plan approved]" if approve else "[Plan rejected]"})
    return False       # Continue
```

새로운 protocol type을 추가한다는 것은 새로운 `if` 분기를 추가하는 것을 의미한다.

### match_response: Type 검증

`match_response`는 단지 `request_id`로 상태를 찾는 데 그치지 않고, 응답 type이 요청 type과 일치하는지도 검증한다:

```python
def match_response(response_type, request_id, approve):
    state = pending_requests.get(request_id)
    if not state:
        return
    if state.type == "shutdown" and response_type != "shutdown_response":
        return  # type mismatch, skip
    if state.type == "plan_approval" and response_type != "plan_approval_response":
        return
    if state.status != "pending":
        return  # already resolved, skip duplicate
    state.status = "approved" if approve else "rejected"
```

shutdown_response가 plan_approval 요청을 실수로 승인하는 일은 일어날 수 없다.

### 통합 Inbox Consumer: consume_lead_inbox

`check_inbox` tool과 메인 loop 모두 같은 `consume_lead_inbox()` 함수를 호출하여, 남은 내용을 반환하기 전에 protocol 메시지를 라우팅한다. 이는 메시지가 protocol 상태 업데이트 없이 소비되는 것을 방지한다:

```python
def consume_lead_inbox(route_protocol=True) -> list[dict]:
    msgs = BUS.read_inbox("lead")
    if route_protocol:
        for msg in msgs:
            meta = msg.get("metadata", {})
            req_id = meta.get("request_id", "")
            msg_type = msg.get("type", "")
            if req_id and msg_type.endswith("_response"):
                match_response(msg_type, req_id, meta.get("approve", False))
    return msgs
```

메인 loop는 또한 inbox 메시지를 `history`에 주입하여 LLM이 이를 보고 반응할 수 있게 한다.

### 팀원 Idle Loop: 종료 대신 대기

s15의 팀원들은 10라운드 후에 종료된다. s16의 팀원들은 LLM이 non-tool_use 응답을 반환한 뒤 idle 대기 상태로 들어간다. inbox를 폴링하다가, shutdown_request에 응답하고 종료하거나, 새 메시지가 오면 계속 작업한다.

```
LLM returns non-tool_use
  → idle: poll inbox every second
  → receives shutdown_request → reply shutdown_response → exit
  → receives new message → inject into messages → continue LLM turn
```

교육용 버전은 Lead로의 idle_notification을 생략한다. 실제 CC는 idle 상태가 되면 `idle_notification`을 보내, Lead가 팀원이 새 작업을 받을 수 있는 상태임을 알 수 있게 한다.

### 종합

```
1. Lead: "Have Alice create a file, then shut her down"
2. Lead → spawn_teammate("alice", "backend", "Create config.py")
3. alice thread starts → write_file("config.py", "...") → done → idle
4. Lead → request_shutdown("alice")
   → BUS.send("shutdown_request", {request_id: "req_000142"})
5. alice idle poll receives → handle_shutdown_request
   → BUS.send("shutdown_response", {request_id: "req_000142", approve: True})
6. Lead consume_lead_inbox → match_response("req_000142", approve=True)
   → pending_requests["req_000142"].status = "approved"
   → inbox message injected into history, LLM sees shutdown result
```

Shutdown 핸드셰이크 완료: request → confirm → shutdown. 모든 단계가 `request_id`로 추적된다.

---

## s15에서 달라진 점

| 구성 요소 | 이전 (s15) | 이후 (s16) |
|-----------|-------------|-------------|
| 협업 | 느슨한 텍스트 메시지 | 구조화된 request-response protocol |
| 요청 추적 | 없음 | ProtocolState + pending_requests dict |
| 메시지 라우팅 | 모두 텍스트로 취급 | dispatch_message가 type에 따라 라우팅 |
| Shutdown | 자연 종료 또는 스레드 강제 종료 | request_id 핸드셰이크 메커니즘 |
| Plan approval | 없음 | 메시지 흐름 예시 (execution gating 없음) |
| 새 메시지 type | message, result | + shutdown_request/response, plan_approval_request/response |
| 팀원 라이프사이클 | 최대 10라운드 | Idle loop (inbox 메시지를 대기) |
| Lead inbox | check_inbox와 메인 loop가 각각 따로 읽음 | 통합된 consume_lead_inbox |
| Lead tool | 14개 (s15) | 14개 (core tool set에 request_shutdown, request_plan, review_plan 추가) |
| 팀원 tool | 4개 (s15) | + submit_plan (5개) |

---

## 직접 해보기

```sh
cd learn-claude-code
python s16_team_protocols/code.py
```

다음 prompt들을 시도해 보자:

1. `Spawn alice as a backend dev. Ask her to create a file. Then request her shutdown.`
2. `Spawn bob with a refactoring task. Have him submit a plan first. Then review and approve it.`

관찰할 점: shutdown 핸드셰이크가 완전한가(request → confirm → shutdown)? `pending_requests` 상태가 올바르게 전이되는가? `request_id`가 요청과 응답 사이에서 일관되게 유지되는가? idle 상태의 팀원이 shutdown_request를 받을 수 있는가?

---

## 다음 단계

s15-s16에서는 Lead가 각 팀원에게 일일이 작업을 할당해야 한다. "Alice는 이걸 하고, Bob은 저걸 한다." 보드에 할당되지 않은 작업이 10개 있으면, Lead가 그것들을 하나하나 직접 할당해야 한다.

만약 팀원들이 보드를 직접 확인하고 스스로 작업을 가져갈 수 있다면 어떨까? Lead는 작업을 만들기만 하면 되고, 팀원들이 스스로 발견하고, 가져가고, 완료한다.

s17 Autonomous Agents → 리더의 할당이 필요 없는, 스스로 조직화하는 팀원들.

<details>
<summary>CC 소스 깊이 들여다보기</summary>

CC의 팀 protocol 구현(`teammateMailbox.ts`, 1184줄)은 교육용 버전과 동일한 핵심 구조를 공유한다. request_id + approve/reject request-response 패턴이다. 차이점:

**Shutdown protocol**: CC의 shutdown은 3자 간 통신이다(`teammateMailbox.ts:720-763`, `SendMessageTool.ts:268-430`). Lead가 `shutdown_request`를 보내면 팀원이 `shutdown_approved`(또는 이유를 담은 `shutdown_rejected`)로 응답하고, 시스템이 `teammate_terminated`를 보내 모든 당사자에게 알린다. 확인이 끝나면 시스템은 pane(tmux/iTerm2)을 정리하고, 작업 할당을 해제하며, team config에서 멤버를 제거한다(`useInboxPoller.ts:677-800`). 교육용 버전은 `shutdown_response`라는 통합된 이름을 사용하지만, 실제 소스는 이를 `shutdown_approved`와 `shutdown_rejected`라는 두 개의 별도 메시지 type으로 나눈다.

**Plan approval**: 실제 소스에서 plan 승인 요청은 plan-mode가 필요한 팀원이 plan mode를 빠져나갈 때 `ExitPlanModeV2Tool.ts:263-312`에 의해 생성된다. `useInboxPoller.ts:599-661`은 현재 승인을 자동으로 기록하고 그 요청을 Lead에게 context(일반 메시지)로 전달한다. `SendMessageTool.ts:434-518`은 명시적인 approve/reject 응답 기능을 유지한다 — 승인은 동시에 `permissionMode`를 설정할 수 있고(예: "승인됐지만 plan mode로 실행"), 응답에는 팀원이 수정 후 재제출하도록 `feedback` 문자열을 포함할 수 있다. 단순한 "Lead가 review_plan tool을 수동으로 사용하는" 흐름이 아니다.

**메시지 포맷**: CC의 protocol 메시지는 구조화된 JSON이고(Zod schema 검증 포함), 교육용 버전은 단순한 type + metadata dict를 사용한다. 필드 이름도 일관되지 않다. permission은 `request_id`를 쓰고(`teammateMailbox.ts:453-462`), shutdown과 plan approval은 `requestId`를 쓴다(`teammateMailbox.ts:684-763`).

**Execution gating**: CC의 팀원들은 완전한 permission gating을 갖는다. 승인되지 않은 위험도 높은 작업은 가로채지며, 선택 사항이 아니다. 교육용 버전은 메시지 흐름만 보여줄 뿐 execution 가로채기는 하지 않는다.

**범용성**: 교육용 버전의 단일 FSM(pending → approved | rejected)이 두 protocol에 매핑된다. 이 단순화는 올바르다. CC의 protocol 메시지들은 모두 같은 request id 연결 메커니즘을 공유한다.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
