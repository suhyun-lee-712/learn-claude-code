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

> 실제 CC도 파일 inbox(`~/.claude/teams/{teamName}/inboxes/{agentName}.json`)를 사용한다.  
> 동시 쓰기 안전을 위해 `proper-lockfile`을 추가한다는 점이 교육용 버전과 다르다.

### 통신은 항상 inbox를 통해서

모든 통신은 inbox가 유일한 채널이다.

```
Lead     → Teammate:  teammate inbox에 append
Teammate → Lead:      lead inbox에 append
Teammate → Teammate:  상대방 inbox에 append
```

메시지 타입에 상관없이 **항상 받는 사람의 inbox 파일에 append**하는 방식으로 통일돼 있다.  
Agent들이 서로의 존재를 직접 알 필요가 없다는 점이 핵심 — 이를 **느슨한 결합(loose coupling)** 이라고 한다.

---

## 2. spawn_teammate_thread: 팀원 생성

Lead가 `spawn_teammate` tool을 호출하면, teammate가 **별도의 background thread**에서 실행된다.

```python
def spawn_teammate_thread(name, role, prompt):
    system = f"You are '{name}', a {role}. Use tools to complete tasks."

    def run():
        messages = [{"role": "user", "content": prompt}]
        sub_tools = [bash, read_file, write_file, send_message]  # 4개만

        for _ in range(10):                      # 최대 10라운드 (교육용)
            inbox = BUS.read_inbox(name)         # ① 편지 확인
            if inbox:
                messages.append(...)             # ② history에 주입
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

---

## Permission Bubbling (실제 CC)

교육용 코드는 생략하지만, 실제 CC에서는 teammate가 위험한 작업을 만나면 사용자에게 승인을 요청하는 흐름이 있다.

```
1. Teammate → Lead inbox:  permission_request 전송
2. Lead의 useInboxPoller (1초마다):  감지 → UI에 승인 대화창 표시
3. 사용자 승인
4. Lead → Teammate inbox:  permission_response 전송
5. Teammate의 useSwarmPermissionPoller (500ms마다):  응답 수신 → 작업 재개
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
