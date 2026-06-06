# s15: Agent Teams — Agent 하나로는 부족하다, 팀을 꾸려라

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → ... → s13 → s14 → `s15` → [s16](../s16_team_protocols/) → s17 → s18 → s19 → s20
> *"Agent 하나로는 부족하다, 팀을 꾸려라"* — 파일 기반 inbox + teammate thread.
>
> **Harness Layer**: Teams — 멀티 Agent 협업, message bus.

---

## 문제

"백엔드 전체를 리팩터링하라"는 작업은 인증, 데이터베이스 레이어, API 라우트, 테스트까지 건드린다. API 라우트를 작업하는 Agent 하나는 더 이상 인증 모듈의 세부 사항을 context에 담고 있지 않다. context window는 한정되어 있고, Agent 하나가 모든 모듈을 다 다룰 수는 없다.

s06의 sub-agent는 임시직이다. 한 가지 일을 위해 불려 왔다가 끝나면 사라진다. 어떤 작업은 서로 소통하고 협업할 수 있는 teammate가 필요하다.

---

## 해결책

![Agent Teams 개요](images/agent-teams-overview.en.svg)

교육용 코드는 S14의 기능들(prompt 조립, task 시스템, 백그라운드 실행, cron 스케줄링)을 그대로 이어받는다. 팀 메커니즘에 집중하기 위해 완전한 에러 복구, memory, skill 시스템은 생략한다. 추가된 것: **MessageBus**(파일 기반 inbox), **spawn_teammate_thread**(teammate thread 실행), **inbox injection**(Lead가 teammate 메시지를 받아 history에 주입).

Sub-agent vs Teammate:

| | s06 Sub-agent | s15 Teammate |
|---|---|---|
| 수명 | 일회성, 사용 후 폐기 | 멀티 턴(교육용: 10라운드; 실제 CC: idle loop) |
| 통신 | 결론만 반환 | 비동기 inbox, 언제든 통신 가능 |
| Context | 완전히 격리 | 메시지를 통해 공유 |
| 개수 | Lead 하나 + 가끔 sub-agent | Lead 하나 + 여러 teammate |

---

## 동작 방식

![Team Topology](images/team-topology.en.svg)

### MessageBus: 파일 기반 inbox

각 Agent(Lead와 teammate 포함)는 `.jsonl` inbox를 가진다. 보내기 = 대상의 파일에 JSON 한 줄을 append. 읽기 = 파일을 읽고 삭제(소비):

```python
class MessageBus:
    def send(self, from_agent: str, to_agent: str,
             content: str, msg_type: str = "message"):
        msg = {"from": from_agent, "to": to_agent,
               "content": content, "type": msg_type,
               "ts": time.time()}
        inbox = MAILBOX_DIR / f"{to_agent}.jsonl"
        with open(inbox, "a") as f:
            f.write(json.dumps(msg) + "\n")

    def read_inbox(self, agent: str) -> list[dict]:
        inbox = MAILBOX_DIR / f"{agent}.jsonl"
        if not inbox.exists():
            return []
        msgs = [json.loads(line) for line in inbox.read_text().splitlines()]
        inbox.unlink()  # consume: read + delete
        return msgs
```

왜 인메모리 큐가 아니라 파일을 쓰는가? 교육용 코드는 thread 간에 직관적이고 관찰하기 쉽기 때문에 파일을 사용한다. 실제 CC도 파일 inbox(`~/.claude/teams/{team}/inboxes/`)를 사용하지만, 동시 쓰기 안전성을 위해 `proper-lockfile`을 추가한다. 교육용 버전의 `read_inbox`는 읽기 + unlink 사이에 race condition이 있어 동시 읽기 시 메시지가 유실될 수 있지만, 교육 목적에서는 허용 가능하다.

### spawn_teammate_thread: Teammate 실행하기

Lead는 `spawn_teammate` tool을 호출해 teammate를 시작한다. teammate는 자체 system prompt, 메시지, 단순화된 tool 집합을 가지고 자신만의 daemon thread에서 실행된다:

```python
def spawn_teammate_thread(name: str, role: str, prompt: str) -> str:
    system = f"You are '{name}', a {role}. Use tools to complete tasks."

    def run():
        messages = [{"role": "user", "content": prompt}]
        sub_tools = [bash, read_file, write_file, send_message]
        for _ in range(10):           # max 10 rounds
            inbox = BUS.read_inbox(name)
            if inbox:
                messages.append({"role": "user",
                    "content": f"<inbox>{json.dumps(inbox)}</inbox>"})
            response = client.messages.create(
                model=MODEL, system=system, messages=messages[-20:],
                tools=sub_tools, max_tokens=8000)
            # ... execute tools, process results
        # Send final summary to Lead
        BUS.send(name, "lead", summary, "result")

    threading.Thread(target=run, daemon=True).start()
```

핵심 설계:
- **단순화된 tool 집합**: bash, read, write, send_message. 교육용 코드는 통신에 집중하기 위해 task와 cron을 생략한다. 실제 CC teammate는 TaskCreate, TaskUpdate 등도 가지고 있으며, task 시스템은 팀 전체에서 공유된다
- **교육용: 최대 10라운드**: 무한 루프를 방지한다. 실제 CC는 idle loop를 사용한다. 각 라운드 후 `idle_notification`을 보내고 inbox 메시지를 기다리며, 메시지가 도착하면 재개하고, `shutdown_request`가 올 때만 종료한다
- **완료 시 자동 보고**: `BUS.send(name, "lead", summary)`로 최종 결과를 Lead의 inbox에 보낸다

### Lead의 Inbox Injection

Lead는 main loop를 한 번 돌 때마다 inbox를 확인한다. teammate 메시지는 history에 주입되어 LLM이 이를 보고 반응할 수 있게 한다:

```python
# After main loop iteration
inbox = BUS.read_inbox("lead")
if inbox:
    inbox_text = "\n".join(
        f"From {m['from']}: {m['content'][:200]}" for m in inbox)
    history.append({"role": "user",
                    "content": f"[Inbox]\n{inbox_text}"})
```

교육용 코드는 사용자 입력 loop에서 주입한다. 실제 CC는 더 정교하다. Lead의 `useInboxPoller`가 1초마다 확인하며, 사용자 입력을 기다리지 않고 메시지를 새 턴으로 제출한다.

### Permission Bubbling

교육용 코드는 permission bubbling을 생략한다. 실제 CC의 흐름(`permissionSync.ts`, `useSwarmPermissionPoller.ts`):

1. Teammate가 승인이 필요한 작업을 만남 → Lead의 inbox로 `permission_request` 전송
2. Lead의 `useInboxPoller`가 요청 감지 → 승인 큐로 라우팅
3. 사용자가 승인 → Lead가 `permission_response`를 teammate에게 다시 전송
4. Teammate의 `useSwarmPermissionPoller`(500ms마다 polling)가 응답 수신 → 계속 진행하거나 거절

### 전체 그림

```
1. Lead: "Build the backend: one agent isn't enough, form a team"
2. Lead → spawn_teammate("alice", "backend dev", "Create database schema")
3. Lead → spawn_teammate("bob", "frontend dev", "Write API client")
4. Alice thread starts → her own LLM call → bash "python manage.py migrate"
5. Bob thread starts → his own LLM call → write_file("client.ts", ...)
6. Alice done → BUS.send("alice", "lead", "Schema done: users, orders tables")
7. Bob done → BUS.send("bob", "lead", "Client written with types")
8. Lead next iteration → inbox injected into history → LLM sees both results
```

두 teammate가 병렬로 작업한다.

---

## s14에서 바뀐 점

| 구성 요소 | 이전 (s14) | 이후 (s15) |
|-----------|-------------|-------------|
| Agent 개수 | 1 | Lead 1 + N개 teammate thread |
| 통신 | 없음 | MessageBus + .mailboxes/*.jsonl |
| 새 클래스 | — | MessageBus, active_teammates dict |
| 새 함수 | — | spawn_teammate_thread, run_send_message, run_check_inbox |
| Lead tool | 11 (s14) | + spawn_teammate, send_message, check_inbox (14) |
| Teammate tool | — | bash, read_file, write_file, send_message (4) |
| 권한 | 로컬에서 결정 | 교육용 코드는 생략(실제 CC는 bubbling 있음) |

---

## 직접 해보기

```sh
cd learn-claude-code
python s15_agent_teams/code.py
```

다음 prompt들을 시도해 보자:

1. `Spawn alice as a backend developer. Ask her to create a file called schema.sql with a users table.`
2. `Check your inbox for alice's result.`
3. `Spawn bob as a tester. Ask him to check if schema.sql exists and list its contents.`

관찰할 점: Lead는 teammate를 어떻게 spawn하는가? `.mailboxes/` JSONL 파일은 어떻게 생겼는가? teammate가 끝난 후 Lead의 inbox가 history에 주입되는가?

---

## 다음 단계

Teammate는 작업하고 소통할 수 있다. 하지만 Lead가 Alice를 종료시키고 싶을 때 thread를 곧바로 죽여 버리면 절반만 작성된 파일이 남을 수 있다. graceful shutdown 프로토콜이 필요하다: Lead가 shutdown_request를 보내고, teammate는 마무리한 뒤 종료한다.

s16 Team Protocols → Shutdown handshake와 메시지 규약.

<details>
<summary>CC 소스 코드 심층 분석</summary>

> 다음은 CC 소스 코드 `spawnMultiAgent.ts`, `useInboxPoller.ts`(969줄), `useSwarmPermissionPoller.ts`(330줄), `teammateMailbox.ts`, `teamHelpers.ts`를 바탕으로 한 완전한 분석이다.

### 1. 중앙 Message Bus는 없다, 파일시스템이다

교육용 코드는 메시지를 주고받기 위해 `MessageBus` 클래스를 사용한다. 실제 CC는 더 직접적이다. 각 Agent가 다른 Agent의 inbox 파일에 직접 쓴다.

Inbox 경로: `~/.claude/teams/{teamName}/inboxes/{agentName}.json`

쓰기는 동시 쓰기 안전성을 위해 `proper-lockfile`을 사용한다(최대 10회 재시도). 각 파일은 JSON 배열이며, append는 읽기 → append → 다시 쓰기로 이뤄진다.

### 2. 15가지 메시지 타입

CC 팀 통신에는 15가지의 구조화된 메시지 타입이 있다(`teammateMailbox.ts`):

| Type | 방향 | 용도 |
|------|-----------|---------|
| `plain text` | 양방향 | teammate 간 일반 통신 |
| `idle_notification` | Teammate→Lead | teammate가 한 턴을 끝내고 idle 상태가 됨 |
| `permission_request` | Teammate→Lead | teammate가 작업 승인을 필요로 함 |
| `permission_response` | Lead→Teammate | Lead의 승인 결과 |
| `plan_approval_request` | Teammate→Lead | teammate가 검토를 위해 plan 제출 |
| `plan_approval_response` | Lead→Teammate | Lead의 plan 검토 |
| `shutdown_request` | Lead→Teammate | graceful shutdown 요청 |
| `shutdown_approved` | Teammate→Lead | shutdown 확인 |
| `shutdown_rejected` | Teammate→Lead | shutdown 거절(사유 포함) |
| `task_assignment` | Lead→Teammate | task 할당 |
| `team_permission_update` | Lead→Teammate | 권한 변경 broadcast |
| `mode_set_request` | Lead→Teammate | teammate의 권한 mode 변경 |
| `sandbox_permission_*` | 양방향 | 네트워크 권한 요청/응답 |
| `teammate_terminated` | System | teammate 제거 알림 |

텍스트 메시지는 모델에 전달하기 위해 `<teammate-message>` XML 태그로 감싼다.

### 3. Permission Bubbling: 양방향 Polling

교육용 코드는 permission bubbling을 생략한다. 실제 CC의 흐름(`permissionSync.ts`):

1. **Teammate**가 승인이 필요한 작업을 만남 → Lead의 inbox로 `permission_request` 전송
2. **Lead**의 `useInboxPoller`(1초마다 polling)가 요청 감지 → `ToolUseConfirmQueue`로 라우팅
3. Lead의 UI가 teammate 이름과 색상이 표시된 승인 대화창을 보여줌
4. 사용자가 승인 → Lead가 `permission_response`를 teammate의 inbox로 다시 전송
5. **Teammate**의 `useSwarmPermissionPoller`(500ms마다 polling)가 응답 수신 → 계속 진행하거나 거절

### 4. Teammate 생명주기

CC teammate는 `spawnTeammate()`로 생성된다(`spawnMultiAgent.ts`):

1. **Spawn**: tmux pane 생성(또는 in-process), 색상 할당, 팀 config 작성
2. **Work**: `useInboxPoller`가 1초마다 inbox 확인 → 메시지 도착 시 새 턴으로 제출
3. **Idle**: Stop hook 발동 → Lead에게 `idle_notification` 전송
4. **Shutdown**: Lead가 `shutdown_request` 전송 → teammate가 `shutdown_approved` 응답 → Lead가 정리

### 5. 팀 Config

팀 registry는 `~/.claude/teams/{teamName}/config.json`에 있다(`teamHelpers.ts`):

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

Teammate는 중첩될 수 없다(`AgentTool.tsx:273`가 "teammate가 다른 teammate를 spawn하는 것"을 명시적으로 금지한다).

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
