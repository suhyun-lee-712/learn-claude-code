# s18: Worktree Isolation — 디렉터리 분리, 충돌 없음

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → ... → s16 → s17 → `s18` → [s19](../s19_mcp_plugin/) → s20

> *"디렉터리 분리, 충돌 없음"* — Task는 목표를 소유하고, worktree는 디렉터리를 소유하며, 둘은 ID로 묶인다.
>
> **Harness Layer**: Isolation — 분리된 디렉터리에서의 병렬 실행.

---

## 문제

s17에서는 Alice와 Bob이 모두 같은 디렉터리에서 작업합니다. Alice의 task는 "auth 모듈 리팩터링", Bob의 task는 "UI 로그인 페이지 리팩터링"입니다.

Alice가 `write_file("config.py", ...)`를 호출합니다. Bob도 `write_file("config.py", ...)`를 호출합니다. 둘 다 같은 파일을 수정하면서 서로를 덮어쓰게 됩니다. 게다가 깔끔한 롤백도 불가능합니다 — 어느 변경이 누구의 것인지 구분할 수 없습니다.

s15-s17은 "누가 무엇을 하는가"(task 시스템)와 "어떻게 통신하는가"(message bus)는 해결했지만, "어디서 작업하는가"는 해결하지 못했습니다.

---

## 해결책

![Worktree 개요](images/worktree-overview.en.svg)

Git worktree를 사용하면 같은 repo에서 각자 고유한 브랜치를 가진 독립적인 작업 디렉터리를 여러 개 만들 수 있습니다. Alice는 `.worktrees/auth-refactor/`에서, Bob은 `.worktrees/ui-login/`에서 작업하므로 충돌이 없습니다.

S17의 교육용 버전 MessageBus, 프로토콜, 자율적 claiming을 그대로 이어받습니다. 이번 장에서는 다음을 추가합니다:

| 기능 | 목적 |
|------------|---------|
| create_worktree | task를 위한 격리된 디렉터리 + 브랜치 생성 |
| bind_task_to_worktree | task와 디렉터리를 바인딩 (상태 변경 없음) |
| remove_worktree / keep_worktree | 완료 후 정리 또는 보존 |
| validate_worktree_name | 경로 탐색(path traversal)과 잘못된 문자 거부 |

---

## 동작 방식

### 생성: Task-Worktree 바인딩

```python
def create_worktree(name: str, task_id: str = "") -> str:
    validate_worktree_name(name)       # Only [A-Za-z0-9._-]{1,64}
    path = WORKTREES_DIR / name
    ok, result = run_git(["worktree", "add", str(path), "-b", f"wt/{name}", "HEAD"])
    if not ok:
        return f"Git error: {result}"
    if task_id:
        bind_task_to_worktree(task_id, name)
    log_event("create", name, task_id)
    return f"Worktree '{name}' created at {path}"

def bind_task_to_worktree(task_id: str, worktree_name: str):
    task = load_task(task_id)
    task.worktree = worktree_name       # Write worktree field only
    save_task(task)                     # Status stays pending, waits for teammate claim
```

바인딩 규칙: 하나의 task는 하나의 worktree에 바인딩됩니다. 바인딩은 task 상태를 변경하지 *않습니다* — task는 `pending` 상태를 유지하며, 팀원이 claim할 때에만 `in_progress`로 진행됩니다. 이렇게 하면 Lead가 task와 worktree를 미리 생성해 둘 수 있고, 팀원들은 유휴 시간에 자연스럽게 worktree가 바인딩된 task를 claim하게 됩니다.

### 팀원 도구의 Cwd 전환

교육용 버전은 팀원별로 `wt_ctx` dict를 유지하며 현재 worktree 경로를 추적합니다. 팀원이 worktree가 바인딩된 task를 claim하면 `wt_ctx`가 자동으로 worktree 경로로 설정되고, 팀원의 `bash`, `read_file`, `write_file`은 해당 worktree 디렉터리에서 실행됩니다:

```python
# Inside teammate thread
wt_ctx = {"path": None}

def _run_claim_task(task_id):
    result = claim_task(task_id, owner=name)
    if "Claimed" in result:
        task = load_task(task_id)
        if task.worktree:
            wt_ctx["path"] = str(WORKTREES_DIR / task.worktree)
    return result

def _run_bash(command):
    return run_bash(command, cwd=wt_ctx["path"])  # Execute in worktree
```

이것은 교육용으로 단순화한 것입니다. 실제 CC의 EnterWorktree는 `process.chdir()`를 사용해 프로세스 전체의 디렉터리를 전환하고, AgentTool isolation은 `cwdOverride`를 사용해 sub-agent 실행을 감쌉니다.

### 정리: 보존 또는 제거

task가 완료된 후에는 두 가지 선택지가 있습니다:

```python
def remove_worktree(name: str, discard_changes: bool = False) -> str:
    # Safety check: refuse by default if changes exist
    if not discard_changes:
        files, commits = _count_worktree_changes(path)
        if files > 0 or commits > 0:
            return "Has uncommitted changes. Use discard_changes=true to force, or keep_worktree"
    ok, _ = run_git(["worktree", "remove", str(path), "--force"])
    if not ok:
        return "Remove failed"
    run_git(["branch", "-D", f"wt/{name}"])
    log_event("remove", name)

def keep_worktree(name: str) -> str:
    log_event("keep", name)
    return f"Worktree '{name}' kept for review (branch: wt/{name})"
```

Keep = 수동 리뷰와 머지를 위해 브랜치를 보존합니다. Remove = 커밋되지 않은 변경이 있으면 기본적으로 거부하며, 확인을 위해 `discard_changes=true`가 필요합니다. task를 자동으로 완료시키지 *않습니다* — task 완료는 팀원의 `complete_task`로 명시적으로 트리거됩니다.

### 이벤트 로그: 감사 가능

각 라이프사이클 작업은 감사를 위해 로그에 기록됩니다:

```python
def log_event(event_type: str, worktree_name: str, task_id: str = ""):
    event = {"type": event_type, "worktree": worktree_name,
             "task_id": task_id, "ts": time.time()}
    # append to .worktrees/events.jsonl
```

이벤트 타입: `create`, `remove`, `keep`. 교육용 버전은 수동 감사를 위해 이벤트를 로깅합니다. 완전한 복구를 위해서는 인덱스나 `git worktree list` 스캐닝이 필요할 것입니다.

### run_git: 성공/실패 반환

```python
def run_git(args: list[str]) -> tuple[bool, str]:
    r = subprocess.run(["git"] + args, cwd=WORKDIR, ...)
    return r.returncode == 0, output
```

`create_worktree`와 `remove_worktree`는 git 명령이 성공한 후에만 이벤트 로그를 기록하므로, 로그가 실제 상태를 반영하도록 보장합니다.

---

## s17과의 차이점

| 구성 요소 | 이전 (s17) | 이후 (s18) |
|-----------|-------------|-------------|
| 작업 디렉터리 | 모든 agent가 WORKDIR 공유 | 각 task가 git worktree에 바인딩 가능 |
| Task 데이터 | id/subject/status/owner/blockedBy | + worktree 필드 |
| 팀원 도구 cwd | 항상 WORKDIR | worktree가 바인딩된 task를 claim할 때 자동 전환 |
| 새 함수 | — | create_worktree, bind_task_to_worktree, remove_worktree, keep_worktree, validate_worktree_name |
| Worktree 안전성 | 없음 | 이름 검증 + 변경이 있을 때 제거 거부 |
| 이벤트 로그 | 없음 | events.jsonl 라이프사이클 감사 |
| Lead 도구 | 14 (s17) | + create_worktree, remove_worktree, keep_worktree (17) |
| 팀원 도구 | 8 (s17) | 8 (bash/read/write가 worktree cwd에서 실행) |

---

## 직접 해보기

```sh
cd learn-claude-code
python s18_worktree_isolation/code.py
```

다음 prompt를 시도해 보세요:

`Create two tasks, then create worktrees for each (bind with task_id). Spawn alice and bob. Watch them auto-claim and work in isolated directories.`

관찰할 점: `git status`에서 두 worktree가 서로 다른 브랜치를 보여주나요? worktree가 바인딩된 task를 claim한 후, 팀원의 bash가 worktree 디렉터리에서 실행되나요? 변경이 있을 때 `remove_worktree`가 거부하나요? 바인딩 후에도 task 상태가 여전히 `pending`인가요?

---

## 다음 단계

Agent 팀이 이제 격리된 작업 공간에서 스스로 조직화할 수 있게 되었습니다. 하지만 Agent의 능력은 우리가 작성한 도구 — bash, read, write, task... — 에 한정되어 있습니다.

만약 사용자가 이미 자신만의 도구를 가지고 있다면 어떨까요? 사내 Jira API나 커스텀 배포 시스템 같은 것 말입니다.

s19 MCP Plugin → Agent에게 플러그인 시스템을 제공합니다. 외부 도구는 표준 프로토콜을 통해 연결되며, Agent는 그것을 누가 작성했는지 알 필요가 없습니다.

<details>
<summary>CC 소스 심층 분석</summary>

CC의 worktree 시스템에는 두 가지 경로가 있습니다: **EnterWorktree** (현재 세션이 전환해 들어감) 와 **AgentTool isolation** (sub-agent 격리).

### EnterWorktree: 현재 세션 전환

`EnterWorktreeTool.ts:92-97`은 worktree를 생성한 후 즉시 `process.chdir(worktreePath)`, `setCwd()`, `setOriginalCwd()`, `saveWorktreeState()`를 호출합니다. 현재 세션의 작업 디렉터리가 worktree로 직접 전환됩니다 — prompt 힌트가 아니라 프로세스 수준의 디렉터리 변경입니다.

`ExitWorktreeTool.ts:261-320`에서 keep과 remove 모두 `restoreSessionToOriginalCwd()`를 호출해 원래 디렉터리를 복원합니다. Remove는 커밋되지 않은 변경을 확인하고(`ExitWorktreeTool.ts:190-220`), `discard_changes: true` 없이는 거부합니다.

### AgentTool Isolation: Sub-Agent 격리

`AgentTool.tsx:590-641`은 `isolation: "worktree"`일 때 `createAgentWorktree()`를 호출해 worktree를 생성하고, `cwdOverridePath`를 사용해 sub-agent 실행을 감쌉니다. 모든 sub-agent 작업은 자동으로 worktree 디렉터리에서 실행됩니다. `AgentTool/prompt.ts:272`는 모델에게 알려줍니다: 이것은 임시 worktree이며, 변경이 없으면 자동 정리하고, 변경이 있으면 경로와 브랜치를 반환하라고.

`worktree.ts:902-951`의 `createAgentWorktree()`는 전역 세션 cwd를 수정하지 *않으며*, 오직 sub-agent 용도로만 쓰입니다. `worktree.ts:961-1020`의 `removeAgentWorktree()`는 메인 repo 루트에서 삭제합니다.

### 이름 검증

`worktree.ts:76-84`는 slug를 검증합니다: `.`/`..`는 거부하고, `[a-zA-Z0-9._-]`는 허용합니다. `worktree.ts:48`은 `VALID_WORKTREE_SLUG_SEGMENT`를 정의합니다. 교육용 버전의 `validate_worktree_name`은 동일한 규칙을 사용합니다.

### 경로와 브랜치 네이밍

실제 경로는 `.claude/worktrees/`이고, 브랜치 이름은 `worktree-{slug}`입니다(`worktree.ts:204-227`, 슬래시는 `+`로 대체됨). 교육용 버전은 단순화를 위해 `.worktrees/`와 `wt/{name}`을 사용합니다.

생성 시 `git worktree add -B`를 사용하며(`worktree.ts:326-328`), 현재 HEAD보다 `origin/<defaultBranch>`를 선호합니다.

### 상태 관리

CC에는 task-worktree 바인딩이 없습니다. Worktree 상태는 `PersistedWorktreeSession`(`worktree.ts:756-768`)을 통해 관리되며, `originalCwd`, `worktreePath`, `worktreeName`, `worktreeBranch`, `originalBranch`, `originalHeadCommit`, `sessionId` 등의 필드를 포함합니다 — taskId 필드는 없습니다. `saveWorktreeState()`(`sessionStorage.ts:2883-2920`)는 세션 transcript에 `type: 'worktree-state'`로 기록합니다.

교육용 버전은 바인딩을 위해 task의 `worktree` 필드를 사용하는데, 이는 교육용 단순화입니다. CC는 worktree와 task를 두 개의 독립적인 시스템으로 취급하며, Agent의 context 이해를 통해 둘을 연결합니다.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v0 -->
