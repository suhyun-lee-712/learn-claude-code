# s12: Task System — 큰 목표를 작은 task로 쪼개기

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → ... → s10 → s11 → `s12` → [s13](../s13_background_tasks/) → s14 → ... → s20

> *"큰 목표를 작은 task로 쪼개고, 순서를 정하고, 영속화한다"* — 파일에 영속화되는 task 그래프, 멀티 에이전트 협업의 토대.
>
> **Harness Layer**: Tasks — 영속화된 목표, 복구 가능한 진행 상황.

---

## 문제

에이전트가 프로젝트 하나를 받는다. 데이터베이스를 세팅하고, API를 작성하고, 테스트를 추가하는 일이다. 에이전트는 s05의 TodoWrite로 체크리스트를 만든 다음 API부터 작성하기 시작하는데, 절반쯤 진행하다가 데이터베이스 테이블이 없다는 걸 깨닫고 다시 돌아가 고친다. 테스트를 추가할 때가 되니 이번엔 API 인터페이스 시그니처가 또 바뀌어 있다...

기초를 다지기도 전에 지붕부터 올릴 수는 없다. task에는 순서가 있다. task 의존성은 방향성 비순환 그래프(DAG, Directed Acyclic Graph)를 이루어야 한다. 다만 교육용 버전에서는 `blockedBy` 체크만 보여줄 뿐, 순환 탐지(cycle detection)는 다루지 않는다.

s05의 TodoWrite는 현재 task에 대한 실행 체크리스트로, 세션 메모리에 유지된다. 여기서 필요한 것은 **task 시스템**이다. 각 task가 JSON 파일이고, task 간에는 `blockedBy` 의존성이 있으며, 세션을 넘어 디스크에 영속화된다.

---

## 해결책

![Task System 개요](images/task-system-overview.en.svg)

교육용 코드는 기본적인 agent loop를 유지하면서, task 시스템에 집중하기 위해 S11의 전체 에러 복구 기능(RecoveryState, backoff, escalation, reactive compact, fallback model)은 생략한다. 추가된 것: 새로운 task tool 5개 + 영속화를 위한 `.tasks/` 디렉터리 + `blockedBy` 의존성 체크. task 시스템과 에러 복구는 독립적인 레이어다. CC 소스에서 `utils/tasks.ts`는 CRUD만 담당하고, 에러 복구는 `query.ts`의 with_retry/RecoveryState가 담당하며, 둘 사이에는 결합이 없다.

TodoWrite vs Task System:

| | TodoWrite (s05) | Task System (s12) |
|---|---|---|
| 역할 | 현재 task에 대한 실행 체크리스트 | 복구 가능한 task 시스템 |
| 저장소 | 프로세스 내 / 세션 상태 | `.tasks/{id}.json` |
| 의존성 | 없음 | `blockedBy` / `blocks` 그래프 |
| 라이프사이클 | 현재 세션 / 현재 task | 세션을 넘나듦 |
| 협조 | task 클레임 없음 | `owner` / claim |
| 상태 | pending / in_progress / completed | pending / in_progress / completed |
| 단위 | 에이전트 자신의 단계 | 클레임하고, 추적하고, 차단 해제할 수 있는 task |

---

## 동작 방식

![Task DAG](images/task-dag.en.svg)

### Task: 데이터 구조

각 task는 JSON 파일 하나로, `.tasks/` 디렉터리에 저장된다:

```python
@dataclass
class Task:
    id: str
    subject: str
    description: str
    status: str          # pending | in_progress | completed
    owner: str | None    # Agent name (multi-agent scenarios)
    blockedBy: list[str] # List of dependency task IDs
```

ID는 `timestamp + random hex`로 생성한다. 단순하지만 충분하다. CC는 순차적인 ID + highwatermark 파일을 사용해 ID 재사용을 막는데, 이는 더 엄격한 설계다.

### create_task: task 생성

```python
def create_task(subject: str, description: str = "",
                blockedBy: list[str] | None = None) -> Task:
    task = Task(
        id=f"task_{int(time.time())}_{random_hex(4)}",
        subject=subject, description=description,
        status="pending", owner=None,
        blockedBy=blockedBy or [],
    )
    save_task(task)
    return task
```

생성 시 자동으로 `save_task`를 호출해 `.tasks/{id}.json`을 기록한다. `blockedBy`는 의존성을 선언한다. 예를 들어 "API 작성"은 `blockedBy: ["task_schema"]`를 갖는다.

### can_start: 의존성 체크

task는 자신의 `blockedBy` 의존성이 모두 **completed** 된 뒤에야 시작할 수 있다:

```python
def can_start(task_id: str) -> bool:
    task = load_task(task_id)
    for dep_id in task.blockedBy:
        if not _task_path(dep_id).exists():
            return False  # missing dependency = blocked
        dep = load_task(dep_id)
        if dep.status != "completed":
            return False
    return True
```

`can_start`는 `claim_task`의 선행 체크다. `blockedBy` 의존성 중 하나라도 completed가 아니면 그 task는 클레임할 수 없다. 존재하지 않는 의존성은 차단된 것으로 취급해, 잘못된 ID를 참조하다 크래시가 나는 일을 방지한다.

### claim_task: task 클레임

에이전트가 task 작업을 시작할 때 `claim_task`를 호출한다. `owner`를 설정하고 상태를 `pending` → `in_progress`로 바꾼다. `owner` 필드는 누가 그 task를 작업 중인지 기록해, 멀티 에이전트 시나리오에서 중복 클레임을 방지한다:

```python
def claim_task(task_id: str, owner: str = "agent") -> str:
    task = load_task(task_id)
    if task.status != "pending":
        return f"Task {task_id} is {task.status}, cannot claim"
    if not can_start(task_id):
        deps = [d for d in task.blockedBy
                if load_task(d).status != "completed"]
        return f"Blocked by: {deps}"
    task.owner = owner
    task.status = "in_progress"
    save_task(task)
    return f"Claimed {task_id} ({task.subject})"
```

이미 다른 누군가가 클레임한 task이거나(`status != "pending"`), 의존성이 충족되지 않았다면(`can_start`가 False 반환) 클레임은 거부된다.

### complete_task: 완료 및 차단 해제

task가 끝나면 `completed`로 설정한다. 동시에 다른 모든 task를 스캔해 **방금 차단이 해제된** 하위 task를 찾는다:

```python
def complete_task(task_id: str) -> str:
    task = load_task(task_id)
    task.status = "completed"
    save_task(task)
    # Find newly unblocked downstream tasks
    unblocked = [t.subject for t in list_tasks()
                 if t.status == "pending" and t.blockedBy
                 and can_start(t.id)]
    msg = f"Completed {task_id} ({task.subject})"
    if unblocked:
        msg += f"\nUnblocked: {', '.join(unblocked)}"
    return msg
```

"schema"를 완료하면 "endpoints"와 "docs"에 대해 `can_start`가 True를 반환한다. 이제 이들이 시작될 수 있다.

### get_task: 전체 상세 보기

`list_tasks`는 한 줄짜리 요약만 보여준다. `get_task`는 description과 의존성 상세를 포함한 전체 task JSON을 반환한다. 세션을 넘어 복구할 때, 에이전트는 작업을 이어가기 위해 전체 description을 읽어야 한다:

```python
def get_task(task_id: str) -> str:
    task = load_task(task_id)
    return json.dumps(asdict(task), indent=2)
```

### 상태 머신: 두 개의 액션, 세 개의 상태

```
pending ──claim──→ in_progress ──complete──→ completed
```

여기서 `claim` / `complete`는 액션이고, `pending` / `in_progress` / `completed`는 상태다:

- **claim_task**: `pending` → `in_progress`. owner를 설정하고 작업을 시작한다.
- **complete_task**: `in_progress` → `completed`. task를 완료로 표시하고 하위 task를 차단 해제한다.

CC에는 `in_progress → pending` 해제 경로가 없다. 팀원(teammate)이 종료되거나 셧다운되면, CC는 그 팀원이 끝내지 못한 task를 할당 해제(owner를 비움)하고 상태를 `pending`으로 되돌려, 다른 에이전트가 다시 클레임할 수 있게 한다. 교육용 버전은 이 복구 경로를 생략한다.

### 종합하기

```python
# Create tasks with dependencies
schema = create_task("setup database schema")
endpoints = create_task("create API endpoints", blockedBy=[schema.id])
tests = create_task("write tests", blockedBy=[endpoints.id])
docs = create_task("write docs", blockedBy=[schema.id])

# Agent claims the first available task
claim_task(schema.id)       # ✓ Claimed (no dependencies)
complete_task(schema.id)    # ✓ Completed → unblocks endpoints, docs

claim_task(endpoints.id)    # ✓ Claimed (schema completed)
complete_task(endpoints.id) # ✓ Completed → unblocks tests

claim_task(docs.id)         # ✓ Claimed (schema completed)
complete_task(docs.id)      # ✓ Completed

claim_task(tests.id)        # ✓ Claimed (endpoints completed)
complete_task(tests.id)     # ✓ Completed
```

`create_task` 하나마다 JSON 파일을 기록하고, `claim_task` / `complete_task` 하나마다 그 파일을 갱신한다. 세션을 넘어서 `.tasks/` 디렉터리는 그대로 남는다. 에이전트는 이 파일들을 읽어 진행 상황을 복구한다.

---

## s11에서 바뀐 점

| 구성 요소 | 이전 (s11) | 이후 (s12) |
|-----------|-------------|-------------|
| task 관리 | 없음 | Task dataclass + tool 5개 |
| 새 타입 | — | Task (id, subject, description, status, owner, blockedBy) |
| 저장소 | 영속화 없음 | `.tasks/{id}.json` 세션 간 영속화 |
| 의존성 | 없음 | `blockedBy` 그래프 + `can_start` 체크 |
| Tools | bash, read_file, write_file (3) | + create_task, list_tasks, get_task, claim_task, complete_task (8) |
| 라이프사이클 | — | pending → in_progress → completed (해제 롤백 없음) |

---

## 직접 해보기

```sh
cd learn-claude-code
python s12_task_system/code.py
```

다음 prompt를 시도해보자:

1. `Create tasks: setup database schema, create API endpoints (depends on schema), write tests (depends on endpoints), write docs (depends on schema)`
2. `List all tasks and their statuses`
3. `Claim the first unblocked task and complete it`
4. `List tasks again — which ones are now unblocked?`

관찰할 점: `.tasks/` 디렉터리에 JSON 파일이 생성되는가? task를 완료한 뒤, 차단되어 있던 task들이 차단 해제되는가?

---

## 다음 단계

task 그래프가 갖춰졌다. 하지만 어떤 task는 시간이 오래 걸린다. 전체 테스트 스위트를 돌리거나 서버에 배포하는 일처럼 말이다. 에이전트는 토큰 단위로 과금되는 LLM을 호출하므로, 느린 작업을 마냥 기다릴 여유가 없다.

s13 Background Tasks → 느린 작업은 백그라운드로 보낸다. 에이전트는 다른 task를 계속 처리하다가, 백그라운드 작업이 끝나면 알림을 받는다.

<details>
<summary>CC 소스 심층 분석</summary>

> 다음은 CC 소스 코드 `utils/tasks.ts`(862줄), `tools/TaskCreateTool/TaskCreateTool.ts`(138줄), `tools/TaskUpdateTool/TaskUpdateTool.ts`(406줄), `tools/TaskGetTool/TaskGetTool.ts`(128줄), `tools/TaskListTool/TaskListTool.ts`(116줄), `hooks/useTaskListWatcher.ts`(221줄)를 바탕으로 한 완전한 분석이다.

### 1. TaskRecord의 전체 필드

튜토리얼은 id, subject, status, owner, blockedBy만 다룬다. CC에는 실제로 9개의 필드가 있다(`utils/tasks.ts:76-89`):

| 필드 | 타입 | 용도 |
|------|------|---------|
| `id` | string | 증가하는 정수 ID |
| `subject` | string | 짧은 제목 |
| `description` | string | 자유 형식 설명 |
| `activeForm` | string? | 현재진행형 표현, in_progress일 때 스피너에 표시 |
| `owner` | string? | 할당된 에이전트 ID |
| `status` | pending/in_progress/completed | 라이프사이클 |
| `blocks` | string[] | 이 task가 차단하는 task ID들 (하위) |
| `blockedBy` | string[] | 이 task를 차단하는 task ID들 (상위) |
| `metadata` | Record? | 임의의 확장 키-값 쌍 |

저장 위치: `~/.claude/tasks/{taskListId}/{id}.json`. task당 파일 하나.

### 2. TodoWrite의 업그레이드가 아니라 — 두 개의 독립 시스템

CC에서 Task System과 TodoWrite는 **공존**하며, `isTodoV2Enabled()`(`utils/tasks.ts:133`)로 전환된다. 인터랙티브 세션은 기본적으로 Task(V2)를, 비인터랙티브/SDK 세션은 기본적으로 TodoWrite를 사용한다. `CLAUDE_CODE_ENABLE_TASKS` 환경 변수로 Task를 강제 활성화할 수 있다. Task에는 TodoWrite에 없는 것들이 있다: 파일 락 동시성 보호, 의존성 강제(enforcement), 소유권(ownership), fs.watch 기반 reactive 모니터링, 라이프사이클 hook.

### 3. 동시 클레임 락킹

`claimTask()`(`utils/tasks.ts:541-612`)는 경쟁(race)을 막기 위해 이중 락킹을 사용한다:

**task 파일 락**: `proper-lockfile`이 `{taskId}.json`을 잠근다(최대 30회 재시도, 지수 백오프 5-100ms). 락 안에서:
1. task 재읽기(TOCTOU 방지)
2. 이미 다른 곳에서 클레임됐는지 체크 → `already_claimed`
3. 이미 완료됐는지 체크 → `already_resolved`
4. 상위가 completed가 아닌지 체크 → `blocked`
5. owner 설정

**리스트 레벨 락**(에이전트 busy 체크): `.lock` 파일. 모든 task를 원자적으로 스캔해 해당 에이전트가 이미 열려 있는 다른 task를 갖고 있는지 확인한다.

참고: 교육용 버전은 클레임과 작업 시작을 한 단계로 합친다(claim = owner 설정 + in_progress). 실제 CC의 `claimTask`는 주로 owner 경쟁을 해소한다 — 상태는 바꾸지 않고 owner만 설정한다. 상태 갱신은 `TaskUpdate`가 처리한다.

### 4. ID 재사용을 막는 High-Water Mark

`.highwatermark` 파일은 지금까지 할당된 가장 높은 task ID를 기록한다. task가 삭제되더라도 그 ID는 재사용되지 않는다.

### 5. 네 개의 task tool

CC의 task 시스템에는 네 개의 tool이 있다(튜토리얼의 단일 범용 Task tool이 아니다): `TaskCreate`, `TaskGet`, `TaskUpdate`, `TaskList`. 모두 `isConcurrencySafe: true`와 `shouldDefer: true`로 설정되어 있다(tool 스키마가 초기 prompt에 없고, ToolSearch 이후에만 보인다).

교육용 버전의 `create_task(blockedBy=...)`는 생성 시점에 의존성을 선언하는데, 이는 합리적인 단순화다. 실제 CC의 `TaskCreate`는 subject/description/activeForm/metadata만 받는다 — 의존성은 `TaskUpdate`의 `addBlocks/addBlockedBy`로 관리한다.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
