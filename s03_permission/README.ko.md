# s03: Permission — 실행 전에 권한을 확인하기

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → s02 → `s03` → [s04](../s04_hooks/) → s05 → ... → s20
> *"실행하기 전에 권한을 확인하라"* — permission 파이프라인은 어떤 작업에 승인이 필요한지 결정한다.
>
> **Harness Layer**: Permission — tool 실행 앞단의 게이트.

---

## 문제

s02의 Agent는 5개의 tool을 가지고 있다. 파일 관련 tool은 `safe_path`로 보호되지만, bash는 아무런 제약이 없다. "프로젝트를 정리해줘"라고 시키면 `rm -rf /`를 실행해버릴 수도 있다.

안전성은 모델을 신뢰하는 것에 기댈 수 없다 — 코드가 필요하다. 모든 tool 실행 전에 수행되는 검사 말이다.

---

## 해결책

![Permission 개요](images/permission-overview.en.svg)

s02의 loop는 그대로 보존된다. 유일한 변화는 tool 실행 전에 `check_permission()`을 끼워 넣는 것이다 — 각 tool 호출은 정해진 순서대로 세 개의 게이트를 통과한다. 먼저 hard deny, 그다음 soft ask, 둘 다 매칭되지 않으면 allow.

세 게이트는 세 가지 결정에 대응한다:

| 게이트 | 목적 | 매칭 시 |
|------|---------|----------|
| 1. Deny List | 영구적으로 금지된 작업 (`rm -rf /`, `sudo`) | 즉시 거부, 실행되지 않음 |
| 2. Rule Matching | 맥락에 따라 달라지는 작업 (워크스페이스 밖에 쓰기, `rm` 파일) | 게이트 3으로 넘김 |
| 3. User Approval | 게이트 2가 매칭된 후, 사용자 확인을 위해 멈춤 | 사용자가 allow 또는 deny를 결정 |

세 게이트 중 어느 것도 매칭되지 않으면 → 바로 실행한다. 대부분의 일상적인 작업은 이 경로를 탄다.

---

## 동작 방식

![Permission 파이프라인](images/permission-pipeline.en.svg)

**게이트 1**: hard deny 리스트. 가장 먼저 확인하고, 매칭되면 차단 메시지를 반환한다. (교육용 데모: 단순 문자열 매칭은 신뢰할 수 있는 보안 메커니즘이 아니다 — 명령어 변형이나 shell 확장으로 우회할 수 있다. CC의 접근 방식은 부록에 있다.)

```python
DENY_LIST = [
    "rm -rf /", "sudo", "shutdown", "reboot",
    "mkfs", "dd if=", "> /dev/sda",
]

def check_deny_list(command: str) -> str | None:
    for pattern in DENY_LIST:
        if pattern in command:
            return f"Blocked: '{pattern}' is on the deny list"
    return None
```

**게이트 2**: rule matching — "언제 사용자에게 물어볼지"를 기술한다. 각 rule은 tool과 검사 조건을 명시한다.

```python
PERMISSION_RULES = [
    {
        "tools": ["write_file", "edit_file"],
        "check": lambda args: not (WORKDIR / args.get("path", "")).resolve().is_relative_to(WORKDIR),
        "message": "Writing outside workspace",
    },
    {
        "tools": ["bash"],
        "check": lambda args: any(kw in args.get("command", "") for kw in ["rm ", "> /etc/", "chmod 777"]),
        "message": "Potentially destructive command",
    },
]

def check_rules(tool_name: str, args: dict) -> str | None:
    for rule in PERMISSION_RULES:
        if tool_name in rule["tools"] and rule["check"](args):
            return rule["message"]
    return None
```

**게이트 3**: rule이 매칭된 후, 사용자 입력을 위해 멈춘다.

```python
def ask_user(tool_name: str, args: dict, reason: str) -> str:
    print(f"\n⚠  {reason}")
    print(f"   Tool: {tool_name}({args})")
    choice = input("   Allow? [y/N] ").strip().lower()
    return "allow" if choice in ("y", "yes") else "deny"
```

**세 게이트를 모두 연결한 것**, tool 실행 전에 끼워 넣는다:

```python
def check_permission(block) -> bool:
    # Gate 1: Hard deny
    if block.name == "bash":
        reason = check_deny_list(block.input.get("command", ""))
        if reason:
            print(f"\n⛔ {reason}")
            return False

    # Gate 2 + 3: Rule matching → User approval
    reason = check_rules(block.name, block.input)
    if reason:
        decision = ask_user(block.name, block.input, reason)
        if decision == "deny":
            return False

    return True

# In agent_loop — s02's loop with just one line added:
for block in response.content:
    if block.type == "tool_use":
        if not check_permission(block):           # ← NEW
            results.append({... "content": "Permission denied."})
            continue
        output = TOOL_HANDLERS[block.name](**block.input)  # s02 original
        results.append(...)
```

---

## s02로부터의 변경점

| 구성 요소 | 이전 (s02) | 이후 (s03) |
|-----------|-------------|-------------|
| 보안 모델 | 없음 (모델을 신뢰) | 세 게이트 permission 파이프라인 |
| 추가된 함수 | — | check_deny_list, check_rules, ask_user, check_permission |
| Loop | 모든 tool을 바로 실행 | 실행 전에 check_permission()을 삽입 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s03_permission/code.py
```

다음 prompt들을 시도해보라:

1. `Create a file called test.txt in the current directory` (그대로 통과해야 함)
2. `Delete all temporary files in /tmp` (bash + rm이 게이트 2를 발동)
3. `What files are in the current directory?` (읽기 전용, 모두 통과)
4. `Try to write a file to /etc/something` (워크스페이스 밖 쓰기가 게이트 2를 발동)

무엇을 주목할 것인가: 어떤 작업이 그대로 통과하는가? 어떤 작업이 확인을 요구하는가? 어떤 작업이 바로 거부되는가?

---

## 다음은

permission 검사가 자리를 잡았다 — 하지만 모든 검사가 loop 안에 `check_permission()`으로 하드코딩되어 있다. 각 tool 실행 전후에 로깅을 추가하고 싶다면? 특정 작업 이후에 git commit을 자동으로 트리거하고 싶다면? 이런 확장 로직을 loop 전체에 흩뿌리면 loop가 비대해진다.

→ s04 Hooks: loop에 hook을 추가한다. 확장 로직은 hook에 매달리고, loop는 깨끗하게 유지된다.

<details>
<summary>CC 소스 코드 들여다보기</summary>

> 다음 내용은 CC 소스 코드 `types/permissions.ts`, `utils/permissions/permissions.ts`, `toolExecution.ts`, `utils/permissions/yoloClassifier.ts`, `tools/AgentTool/forkSubagent.ts`에 대한 리뷰를 바탕으로 한다.

### 1. PermissionResult: 3개가 아니라 4개

교육용 버전의 세 게이트(deny → ask → allow)는 CC와 완전히 대응되지 않는다. CC의 `PermissionResult`는 4가지 behavior를 가진다 (`types/permissions.ts:241-266`):

| behavior | 의미 | 교육용 버전 대응 |
|----------|---------|---------------------------|
| `allow` | 바로 허용 | 게이트 3 통과 |
| `deny` | 바로 거부 | 게이트 1 매칭 |
| `ask` | 사용자에게 다이얼로그 표시 | 게이트 2 매칭 |
| `passthrough` | tool이 의견을 표명하지 않고 일반 파이프라인으로 넘김 | 교육용 버전에 없음 |

### 2. 프로덕션 검증 단계

CC의 tool 호출은 세 게이트를 거치지 않는다 — `checkPermissionsAndCallTool()`(`toolExecution.ts:599-1745`), hook, `hasPermissionsToUseToolInner()`(`utils/permissions/permissions.ts:1158-1310`), 그리고 classifier 로직에 분산된 여러 단계를 거친다:

1. **Zod schema 검증** (`toolExecution.ts:614-680`) — 파라미터 타입 검사
2. **validateInput()** (`toolExecution.ts:682-733`) — tool 수준의 의미 검증
3. **backfillObservableInput()** (`toolExecution.ts:784`) — 레거시 필드 백필
4. **PreToolUse hook** (`toolExecution.ts:800-862`) — hook이 allow/deny/ask를 반환할 수 있음
5. **resolveHookPermissionDecision()** (`toolExecution.ts:921-931`) — hook + 파이프라인 결정 조율
6. **hasPermissionsToUseToolInner()** (`permissions.ts:1158-1310`) — 다층 rule 검사:
   - deny rule에 의해 tool 전체가 비활성화 → `deny`
   - ask rule에 의해 tool 전체가 플래그됨 → `ask`
   - `tool.checkPermissions()` tool 자체의 판단
   - tool 자체가 deny를 반환 → `deny`
   - `requiresUserInteraction()` → `ask`
   - 콘텐츠 관련 ask rule → `ask` (우회 불가)
   - 보안 검사 위반 → `ask` (우회 불가)
   - bypassPermissions 모드 → `allow`
   - allow rule에 의해 tool 전체가 허용됨 → `allow`
   - passthrough → `ask`로 변환

### 3. Deny List: 파일 하나가 아니라 8개의 출처

CC에는 단일 deny list가 없다. permission rule은 8개의 출처에서 온다 (`types/permissions.ts:54-62`):

| 출처 | 설정 위치 |
|--------|----------------------|
| `userSettings` | `~/.claude/settings.json` |
| `projectSettings` | `.claude/settings.json` |
| `localSettings` | `settings.local.json` |
| `flagSettings` | Feature flag |
| `policySettings` | 엔터프라이즈 관리 정책 |
| `cliArg` | `--allowedTools` / `--deniedTools` |
| `command` | 인라인 명령어 |
| `session` | 세션 내 임시 인가 |

각 rule 형식: `{ toolName: "Bash", ruleBehavior: "deny", ruleContent: "npm publish:*" }`. 여러 출처의 rule은 병합되며, 우선순위가 높은 출처가 낮은 출처를 덮어쓴다 (낮은 순에서 높은 순: user < project < local < flag < policy, 그리고 cliArg, command, session).

### 4. isDestructive()는 무엇인가

CC에서 `isDestructive`(`Tool.ts:405-406`)는 **순수하게 UI 표시용**이다 — tool 목록에 `[destructive]` 라벨을 표시하는 것. permission 결정에는 참여하지 않는다. 모든 tool은 기본적으로 `false`를 반환한다. ExitWorktree(remove 시)와 MCP tool(`annotations.destructiveHint`에 따라)만 이를 오버라이드한다.

### 5. YoloClassifier (자동 승인)

CC의 auto 모드에서는 매번 다이얼로그를 띄우지 않는다. `classifyYoloAction`(`utils/permissions/yoloClassifier.ts:1012`)은 tool 호출 + 대화 context를 classifier LLM에 보내 안전성을 판단한다. 먼저 acceptEdits 모드 시뮬레이션을 시도하고(`permissions.ts:620-656`, acceptEdits가 허용하면 → 자동 승인), 그다음 safe tool whitelist를 확인하고(`permissions.ts:658-686`), 마지막으로 classifier를 호출한다. classifier가 연속으로 너무 많이 거부하면 → 수동 승인으로 폴백한다.

### 6. Permission Bubbling

(AgentTool을 통해 fork된) 서브 Agent의 `permissionMode`는 `'bubble'`로 설정된다 (`forkSubagent.ts:50`). 이는 서브 Agent에서 조용히 거부되는 대신, permission 다이얼로그가 **부모 Agent의 터미널로 버블링되어 올라간다**는 뜻이다. 이 과정에서 Bash classifier는 계속 동작한다 — permission 다이얼로그를 표시하면서 백그라운드에서 자동 승인이 가능한지 판단한다.

### 교육용 버전의 단순화는 의도적이다

- 다단계 파이프라인 → 3개 게이트: 이해의 진입 장벽을 크게 낮춤
- 8개 rule 출처 → 1개 로컬 DENY_LIST: 다룰 수 있는 개념 수
- isDestructive → 생략 (교육용 버전에는 UI 레이어가 없고, CC에서도 permission 결정에 참여하지 않음)
- YoloClassifier → 생략 (추가 LLM 호출과 telemetry에 의존)
- Permission bubbling → 생략 (s15에서 멀티 Agent를 다룸)

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
