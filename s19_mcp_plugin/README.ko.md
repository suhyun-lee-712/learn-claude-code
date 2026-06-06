# s19: MCP Tools — 외부 도구, 표준 프로토콜

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → ... → s17 → s18 → `s19` → [s20](../s20_comprehensive/)

> *"외부 도구, 표준 프로토콜"* — 발견하고, 조립하고, 호출한다. Agent는 누가 만들었는지 알 필요가 없다.
>
> **Harness 레이어**: Plugin — 표준 프로토콜을 통한 외부 기능.

---

## 문제

s01부터 s18까지, agent가 사용하는 모든 tool은 직접 손으로 작성한 것이었다 — bash, read, write, task, worktree. 입력 검증, 실행 로직, 에러 처리 모두 한 줄 한 줄 작성했다.

이제 통합해야 할 외부 서비스가 3개 생겼다. 회사의 Jira API(이슈 조회, 티켓 생성), 사내 배포 시스템(배포 트리거, 로그 조회), 그리고 팀의 Notion 지식 베이스(문서 검색, 페이지 생성)다. 서비스마다 tool 코드를 새로 작성하고 싶지는 않다.

표준 프로토콜이 필요하다 — 외부 서비스가 이 프로토콜만 구현한다면, 그 서비스가 어떤 언어로 작성되었든 agent가 그 tool을 바로 호출할 수 있다.

---

## 해결책

![MCP 아키텍처](images/mcp-architecture.en.svg)

MCP(Model Context Protocol)는 agent가 외부 tool을 어떻게 발견하고 호출하는지를 정의한다. 핵심 개념은 다음과 같다.

| 개념 | 목적 |
|------|------|
| MCPClient | agent 측 클라이언트 — 서버에 연결하고, tool을 발견하고, tool을 호출한다 |
| MCP Server | 외부 서비스 — `tools/list` + `tools/call`을 구현한다 |
| assemble_tool_pool | 내장 tool과 MCP tool을 하나의 tool pool로 조립한다 |
| mcp\_\_server\_\_tool 네이밍 | 서로 다른 서버 간의 tool 이름 충돌을 방지한다 |

s18의 교육용 버전 worktree 격리, 자율적 claiming, idle 폴링, protocol 시스템을 이어받는다. 이 챕터에서는 `connect_mcp` tool을 추가한다 — 외부 서비스에 연결하고, tool을 발견하고, tool pool에 추가한다.

이 튜토리얼은 mock handler를 사용해 외부 서버를 시뮬레이션한다. 실제 버전이라면 서브프로세스를 띄우고 stdin/stdout JSON-RPC로 통신할 것이다. mock을 쓰면 외부 의존성 없이 전체 흐름을 실행해 볼 수 있다. 그 대신 실제 네트워크 통신이나 프로세스 관리는 볼 수 없다는 트레이드오프가 있다.

---

## 동작 방식

### MCPClient: 발견 + 호출

```python
class MCPClient:
    def __init__(self, name: str):
        self.name = name
        self.tools: list[dict] = []
        self._handlers: dict[str, callable] = {}

    def register(self, tool_defs, handlers):
        """Simulates tools/list discovery."""
        self.tools = tool_defs
        self._handlers = handlers

    def call_tool(self, tool_name: str, args: dict) -> str:
        """Simulates tools/call."""
        handler = self._handlers.get(tool_name)
        if not handler:
            return f"MCP error: unknown tool '{tool_name}'"
        return handler(**args)
```

이 튜토리얼은 Python 함수를 사용해 서버 tool 구현을 시뮬레이션한다. 실제 버전은 stdio JSON-RPC로 서브프로세스와 통신한다.

### connect_mcp: 연결 + 발견

```python
def connect_mcp(name: str) -> str:
    if name in mcp_clients:
        return f"MCP server '{name}' already connected"
    factory = MOCK_SERVERS.get(name)
    if not factory:
        return f"Unknown server '{name}'. Available: ..."
    mcp_client = factory()
    mcp_clients[name] = mcp_client
    return f"Connected to '{name}'. Discovered: ..."
```

연결이 끝나면 그 서버의 tool을 바로 사용할 수 있다.

### normalize_mcp_name: 이름 정규화

```python
_DISALLOWED_CHARS = re.compile(r'[^a-zA-Z0-9_-]')

def normalize_mcp_name(name: str) -> str:
    return _DISALLOWED_CHARS.sub('_', name)
```

`[a-zA-Z0-9_-]`에 해당하지 않는 모든 문자는 `_`로 치환된다. 서버나 tool 이름에 들어간 특수 문자가 네이밍 충돌이나 injection 문제를 일으키지 않도록 방지한다.

### assemble_tool_pool: tool pool 조립

```python
def assemble_tool_pool() -> tuple[list[dict], dict]:
    tools = list(BUILTIN_TOOLS)
    handlers = dict(BUILTIN_HANDLERS)
    for server_name, mcp_client in mcp_clients.items():
        safe_server = normalize_mcp_name(server_name)
        for tool_def in mcp_client.tools:
            safe_tool = normalize_mcp_name(tool_def["name"])
            prefixed = f"mcp__{safe_server}__{safe_tool}"
            tools.append(...)
            handlers[prefixed] = (
                lambda *, c=mcp_client, t=tool_def["name"], **kw:
                    c.call_tool(t, kw))
    return tools, handlers
```

`mcp__{server}__{tool}` 접두사는 서로 다른 서버 간의 tool 이름 충돌을 방지한다. 이름은 `normalize_mcp_name`을 거쳐 정규화된다.

MCP tool 설명에는 `(readOnly)` 또는 `(destructive)` 어노테이션이 포함된다 — 이 튜토리얼은 텍스트 어노테이션을 사용하지만, 실제 CC는 권한 시스템을 위해 구조화된 tool 어노테이션을 사용한다.

### 캐시 없음: tool pool이 바뀌면 prompt도 바뀐다

s10-s18의 agent_loop는 prompt 캐싱을 사용해 재직렬화를 피했다. s19에서는 캐시를 제거한다.

```python
def agent_loop(messages, context):
    tools, handlers = assemble_tool_pool()     # Rebuild every time
    system = assemble_system_prompt(context)    # Regenerate every time
    ...
    if any(b.name == "connect_mcp" ...):
        tools, handlers = assemble_tool_pool()  # Rebuild after connection
        system = assemble_system_prompt(context)
```

이유: `connect_mcp` 이후에는 tool pool이 바뀐다 — `mcp__docs__search` 같은 새로운 tool이 추가된다. 캐시된 tool 목록은 stale 상태가 되고, 이를 계속 사용하면 모델이 새 tool을 호출할 수 없다. 이 튜토리얼은 직렬화 시간이 약간 늘어나는 비용을 감수하고 단순히 캐싱을 제거한다.

### MCP tool: Lead 전용

이 튜토리얼에서 `connect_mcp`는 Lead의 tool이며, `assemble_tool_pool`은 Lead의 agent_loop에만 사용된다. teammate는 여전히 고정된 8개 tool 서브셋(bash, read_file, write_file, send_message, submit_plan, list_tasks, claim_task, complete_task)을 사용한다.

이는 교육용 단순화다. 실제 CC에서는 MCP tool을 main agent와 sub-agent 모두 사용할 수 있다 — sub-agent는 부모의 MCP 설정을 상속한다.

---

## s18에서 바뀐 점

| 구성 요소 | 이전 (s18) | 이후 (s19) |
|------|-----------|-----------|
| tool 출처 | 전부 직접 작성한 내장 tool | 직접 작성 + 동적으로 발견하는 MCP 외부 tool |
| tool pool | 고정된 BUILTIN_TOOLS | assemble_tool_pool이 mcp\_\_ 접두사 tool을 동적으로 조립 |
| 이름 안전성 | 없음 | normalize_mcp_name 정규화 |
| 새 타입 | — | MCPClient 클래스 (tools/list + tools/call 시뮬레이션) |
| 네임스페이스 | — | mcp\_\_server\_\_tool로 충돌 방지 |
| tool 설명 | 어노테이션 없음 | (readOnly)/(destructive) 어노테이션 |
| prompt 캐시 | 있음 (s10부터) | 제거 — tool pool이 동적이라 캐시가 stale 됨 |
| Lead tool | 17개 (s18) | 18개 (+connect_mcp) |
| teammate tool | 8개 (s18) | 8개 (변경 없음, MCP tool은 Lead 전용) |
| 확장 방식 | tool 추가를 위해 코드 작성 | 표준 프로토콜, 어떤 언어로든 서버 구현 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s19_mcp_plugin/code.py
```

다음 prompt를 시도해 보자.

1. `Connect to the docs MCP server and search for something`
2. `Connect to the deploy server and trigger a deployment`
3. `Connect both servers — what tools are now available?`

관찰할 점: MCP 서버에 연결한 후, tool 이름에 `mcp__docs__`나 `mcp__deploy__` 접두사가 붙는가? 두 서버의 tool을 동시에 사용할 수 있는가? MCP tool 설명에 (readOnly)/(destructive) 어노테이션이 포함되는가?

---

## 다음 단계

이제 Agent는 표준 프로토콜을 통해 외부 tool을 연결할 수 있다. 하지만 앞선 19개 챕터는 각각 하나의 메커니즘을 따로따로 추가한 것이다. 실제 Agent는 19개의 별도 데모로 동작하지 않는다.

tool, 권한, hook, todo, task graph, memory, compact, 백그라운드 작업, cron, team, worktree, 그리고 MCP는 모두 별개의 예제로 존재하는 것이 아니라 하나의 같은 loop에 붙어야 한다.

s20 Comprehensive Agent → 앞선 19개 챕터를 하나의 완전한 harness로 결합한다. 여러 메커니즘, 하나의 loop.

<details>
<summary>CC 소스 심층 분석</summary>

> 아래 내용은 CC 소스 분석에 기반한다: `services/mcp/client.ts`, `auth.ts`, `config.ts`, `channelNotification.ts`.

### 1. 6가지 Transport 타입

이 튜토리얼은 stdio mock 하나만 보여준다. CC는 6가지 transport 타입을 지원한다 (`types.ts:23-25`).

| Transport | 통신 방식 |
|-----------|---------|
| `stdio` | 서브프로세스 stdin/stdout (크로스 플랫폼 기본값) |
| `sse` | HTTP Server-Sent Events |
| `http` | Streamable HTTP (POST/SSE 양방향) |
| `ws` | WebSocket |
| `sse-ide` | IDE 내장 SSE transport |
| `sdk` | 인-프로세스 SDK transport |

연결 시 로컬(stdio) 서버와 원격(http/sse/ws) 서버는 동시에 배치 처리된다. 로컬은 3개 배치, 원격은 20개 배치다.

### 2. tool pool 병합 알고리즘

`assembleToolPool()` (`tools.ts:345-364`):

```typescript
// Dedup with priority: built-in tools win on name collision (sorted first)
return uniqBy(
  [...builtInTools.sort(byName), ...filteredMcpTools.sort(byName)],
  'name',
)
```

내장 tool과 MCP tool은 함께 정렬되지 않고 따로 정렬된다. 그 이유는 CC의 `claude_code_system_cache_policy`가 마지막 내장 tool 뒤의 특정 위치에 전역 캐시 breakpoint를 두기 때문이다 — 정렬을 섞으면 이 설계가 깨진다.

### 3. 네이밍 규칙: `mcp__server__tool`

`buildMcpToolName()` (`mcpStringUtils.ts:50-52`):

```
mcp__<normalizedServerName>__<normalizedToolName>
```

`[a-zA-Z0-9_-]`에 해당하지 않는 모든 문자는 `_`로 치환된다 (`normalization.ts:17-23`). 이 튜토리얼의 `normalize_mcp_name`도 같은 규칙을 사용한다.

### 4. 권한 검사

CC는 MCP tool을 위한 별도의 권한 시스템을 가지고 있다. `checkPermissions()`는 MCP tool에 대해 내장 tool과는 다른 로직을 적용한다 — MCP tool은 자신만의 권한 요구사항(readOnly, destructive 등)을 선언할 수 있고, CC는 그 선언을 바탕으로 사용자 확인이 필요한지 결정한다. 이 튜토리얼은 설명에 텍스트 어노테이션 `(readOnly)` / `(destructive)`만 사용하며, 권한 강제는 하지 않는다.

### 5. 설정 출처와 우선순위

MCP 서버 설정은 여러 출처에서 온다. CC의 우선순위는 낮은 것부터 높은 것 순으로 다음과 같다.

```
claude.ai connectors < plugin < user settings.json < approved project .mcp.json < local settings.local.json
```

`claude.ai` connector는 별도로 가져와서, 내용 시그니처로 중복 제거되고, 가장 낮은 우선순위로 병합된다 (`config.ts:1267-1289`). 엔터프라이즈 `managed-mcp.json`이 존재하면 다른 모든 설정은 제외된다.

이 튜토리얼은 서버 이름을 `MOCK_SERVERS` dict에 직접 전달하며, 설정 병합은 하지 않는다.

### 6. Channel 알림: 서버가 메시지를 다시 푸시한다

이 튜토리얼은 agent → MCP Server 단방향 호출만 다룬다. CC는 역방향 알림도 지원한다 (`channelNotification.ts`).

1. 서버가 `capabilities.experimental['claude/channel']`을 선언한다
2. 서버가 MCP notification `notifications/claude/channel`을 통해 agent에 메시지를 보낸다
3. 메시지는 `<channel source="serverName">...</channel>` XML 태그로 감싸진다
4. agent가 SleepTool에 의해 깨어난다 (1초 이내)

서버는 권한도 요청할 수 있다: `notifications/claude/channel/permission_request` → Agent가 `notifications/claude/channel/permission`으로 응답한다. 사용자는 5글자 짧은 ID로 승인/거부한다.

### 7. OAuth 인증 흐름

CC의 MCP 인증(`auth.ts`)은 완전한 OAuth 2.0 + PKCE 흐름을 지원한다.
- public client + PKCE를 통한 OAuth 메타데이터 발견 (RFC 8414 / RFC 9728)
- 로컬 콜백 서버가 authorization code를 수신
- `getSecureStorage()`를 통한 토큰 영속화 (macOS Keychain / Linux 암호화 파일 / Windows Credential Manager)
- 만료 5분 전 자동 갱신
- 크로스 애플리케이션 액세스(XAA): 브라우저가 id_token 획득 → RFC 8693 + RFC 7523 교환 → 브라우저 팝업 반복 없음

### 8. 연결 라이프사이클 에러 처리

CC는 MCP 연결에 대해 세밀한 에러 분류와 재시도를 한다 (`client.ts:1266-1402`).
- 종료성 에러(ECONNRESET, ETIMEDOUT, EPIPE 등): 연속 3회 실패 → 종료 + 재연결
- tool 호출 401: 토큰 만료 → `McpAuthError` throw → 재인증 트리거
- tool 호출 타임아웃: `Promise.race` 타임아웃 (설정 가능, 기본값 약 28시간)
- Stdio 연결 끊김: SIGINT → SIGTERM → SIGKILL 순서로 프로세스 종료

### 이 튜토리얼의 단순화

- 6가지 transport 타입 → 1개 (mock stdio): 다룰 개념 수를 관리 가능하게
- Channel 역방향 알림 → 생략: 튜토리얼 agent는 항상 호출 주체
- OAuth 흐름 → 생략: 튜토리얼은 서버에 인증이 필요 없다고 가정
- 다층 설정 우선순위 → 생략: 튜토리얼은 서버 이름을 직접 전달
- 복잡한 에러 분류 → 생략: 튜토리얼은 try/except를 fallback으로 사용
- MCP tool Lead 전용 → sub-agent 상속 생략: 코드 구조를 단순화

</details>

<!-- translation-sync: zh@v2, en@v2, ja@v0 -->
