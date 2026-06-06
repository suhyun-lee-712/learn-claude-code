# s02: Tool Use — 도구 추가는 한 줄이면 끝

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → `s02` → [s03](../s03_permission/) → s04 → ... → s20
> *"도구를 추가하면, 핸들러 하나만 추가하면 된다"* — loop는 그대로다. dispatch map에 새 도구를 등록하기만 하면 끝.
>
> **Harness Layer**: Tool Dispatch — 모델의 영향 범위를 넓히기.

---

## 도구는 단 하나: Bash

s01 Agent는 도구가 하나뿐이다: bash. 파일을 읽으려면 `cat`, 쓰려면 `echo "..." > file.py`, 편집하려면 `sed`.

모델은 "이 파일을 읽어라"라고 생각하지만, 그것을 `cat path/to/file`로 일일이 풀어 써야 한다. 토큰을 낭비하고 오류를 부르는 불필요한 번역 계층이 하나 더 끼는 셈이다.

---

## 개요: Tool Dispatch

![Tool Dispatch](images/tool-dispatch.en.svg)

s01의 loop는 완전히 보존된다 (LLM 호출, stop_reason 체크, message append — 단 한 단어도 바뀌지 않았다). 바뀐 것은 도구 실행 그 한 줄뿐이다: `run_bash()`가 `TOOL_HANDLERS[block.name]()` dispatch 조회로 대체된다.

Agent에 도구를 추가하려면 딱 두 가지만 하면 된다:

1. **도구 정의**: `TOOLS` 배열에 항목 하나 추가
2. **핸들러 등록**: `TOOL_HANDLERS` dict에 매핑 하나 추가

---

## 도구 1개에서 5개로

s01에는 bash 하나뿐이었다:

```python
TOOLS = [{"name": "bash", ...}]

def run_bash(command): ...
```

s02에서는 5개의 도구로 확장되며, 각각 독립적으로 정의된다:

```python
TOOLS = [
    {"name": "bash",       "description": "Run a shell command.", ...},
    {"name": "read_file",  "description": "Read file contents.",  ...},
    {"name": "write_file", "description": "Write content to file.", ...},
    {"name": "edit_file",  "description": "Replace text in file once.", ...},
    {"name": "glob",       "description": "Find files by pattern.", ...},
]
```

각 도구는 자체 구현 함수를 가진다:

```python
def run_read(path, limit=None):
    lines = safe_path(path).read_text().splitlines()
    if limit:
        lines = lines[:limit]
    return "\n".join(lines)

def run_write(path, content):
    safe_path(path).write_text(content)
    return f"Wrote {len(content)} bytes to {path}"

def run_edit(path, old_text, new_text):
    text = safe_path(path).read_text()
    if old_text not in text:
        return "Error: text not found"
    safe_path(path).write_text(text.replace(old_text, new_text, 1))
    return f"Edited {path}"

def run_glob(pattern):
    import glob as g
    return "\n".join(g.glob(pattern, root_dir=WORKDIR))
```

---

## Tool Dispatch

```python
TOOL_HANDLERS = {
    "bash":       run_bash,
    "read_file":  run_read,
    "write_file": run_write,
    "edit_file":  run_edit,
    "glob":       run_glob,
}

# Only one line changed in the loop — from hardcoded run_bash to dispatch lookup:
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS[block.name]    # lookup
        output = handler(**block.input)         # call
        results.append(...)
```

도구 추가 = `TOOLS` 배열에 항목 하나 + `TOOL_HANDLERS` dict에 한 줄. loop는 그대로다.

---

## 여러 도구 호출

모델은 종종 한 번에 여러 tool_use 호출을 반환한다 — "a.py와 b.py를 읽고, 모든 .py 파일을 나열해라".

교육용 버전은 원래의 `response.content` 순서대로 하나씩 실행한다. CC의 방식은 더 복잡하다: 원래 순서를 연속된 배치(batch)로 잘라내어, 배치 내의 동시성 안전(concurrency-safe) 도구는 병렬로 실행하고, 배치들은 엄격히 순차적으로 처리한다 (부록 참조).

---

## 빠른 참고

| 개념 | 한 줄 요약 |
|---------|-----------|
| TOOL_HANDLERS | 도구 이름 → 핸들러 함수 dict. 도구 추가 = 매핑 한 줄 추가 |
| Tool Definition | 모델에게 "내가 할 수 있는 것"을 알려주는 JSON schema |
| 여러 도구 호출 | 모델이 한 번에 여러 tool_use를 반환할 수 있음; 교육용 버전은 원래 순서대로 실행 |
| Loop 그대로 | s01의 `while True` loop — 단 한 줄도 바뀌지 않음 |

---

## s01에서 바뀐 점

| 구성 요소 | 이전 (s01) | 이후 (s02) |
|-----------|-------------|-------------|
| 도구 개수 | 1개 (bash) | 5개 (+read, write, edit, glob) |
| 도구 실행 | 하드코딩된 `run_bash()` | TOOL_HANDLERS dispatch 조회 |
| 경로 안전성 | 없음 | safe_path 검증 (파일 도구에 한함) |
| Loop | `while True` + `stop_reason` | s01과 동일 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s02_tool_use/code.py
```

다음 prompt들을 시도해보자:

1. `Read the file README.md and tell me what this project is about`
2. `Create a file called test.py that prints "hello", then read it back`
3. `Find all Python files in this directory`
4. `Read both README.md and requirements.txt, then create a summary file`

관전 포인트: 모델은 언제 도구를 하나만 호출하고, 언제 한 번에 여러 개를 호출하는가? 여러 도구 호출이 올바른 순서로 실행되는가?

---

## 다음 단계

이제 Agent는 5개의 특화된 도구를 가진다. 파일 도구는 `safe_path`로 보호되지만, bash는 아무 제약이 없다 — `rm -rf /`도 그대로 실행된다.

→ s03 Permission: 도구 실행 앞에 게이트를 추가한다 — 이 작업은 안전한가? 사용자 승인이 필요한가?

<details>
<summary>CC 소스 코드 깊이 파고들기</summary>

> 다음 내용은 CC 소스 코드 `Tool.ts`, `tools.ts`, `toolOrchestration.ts`, `toolExecution.ts`, `StreamingToolExecutor.ts`를 검토한 결과를 바탕으로 한다.

### 1. 도구 정의 방식

**교육용 버전**: `TOOLS` 배열 + `TOOL_HANDLERS` dict. 정의와 구현이 분리되어 있다.
**CC**: 각 도구는 `buildTool()`로 생성되는 독립적인 객체이며, schema, 검증, 권한, 실행을 모두 포함한다. `getAllBaseTools()`가 모든 도구를 집계한다.

교육용 버전의 분리 방식은 교육 목적에 더 명확하다 — 독자가 "도구 추가 = 두 개의 정의"를 즉시 이해할 수 있다.

### 2. 동시성 안전성: isConcurrencySafe()

![Tool Concurrency](images/concurrency-comparison.en.svg)

교육용 버전은 동시성 없이 원래 순서대로 도구를 하나씩 실행한다. CC는 `isConcurrencySafe(input)`로 동시성을 판단하는데 — 이것은 단순히 "읽기 전용 vs 쓰기"가 아니라, 구체적인 input에 따라 판단한다는 점에 주목하자:

| | isReadOnly | isConcurrencySafe |
|---|---|---|
| FileRead | true | true |
| Glob | true | true |
| Bash `ls` | true | **true** ← 핵심 차이 |
| Bash `rm` | false | false |
| TaskCreate | false | **true** ← 상태를 변경하지만 동시 실행 가능 (s12에서 도입) |

CC의 Bash 도구의 `isConcurrencySafe`는 `isReadOnly`와 같다 — 읽기 전용 명령은 동시 실행 가능하고, 쓰기 명령은 불가능하다. TaskCreate는 task 파일을 수정하지만, 각각 다른 파일에 쓰기 때문에 동시 실행이 가능하다.

### 3. 분할(Partition) 알고리즘

CC의 `partitionToolCalls()` (`toolOrchestration.ts:91-115`)는 두 그룹으로 나누는 것이 아니라 — 도구 호출을 **연속된 블록 단위**로 배치(batch)화한다:

```
[read A, read B, glob *.py, bash "rm x", read C]
  → batch1(concurrent): [read A, read B, glob *.py]
  → batch2(serial): [bash "rm x"]
  → batch3(concurrent): [read C]
```

연속된 동시성 안전 호출들은 같은 배치로 묶여 실제로 병렬 실행된다 (`toolOrchestration.ts:152-176`, 동시성 제한 있음). 동시성 안전하지 않은 호출을 만나면 새 배치를 시작하여 순차 실행한다. 배치들은 엄격히 순차적이다.

### 4. 검증 파이프라인

CC에서 각 도구 호출은 엄격한 5단계 검증을 거친다 (`toolExecution.ts`):

1. **Zod schema 검증** (`614-680`, 교육용 버전은 JSON Schema 사용): 파라미터 타입/구조 검사
2. **도구 레벨 validateInput()** (`682-733`): 파라미터 값 검증 (예: 경로가 작업 디렉터리 내에 있는지)
3. **PreToolUse hooks** (`800-862`, s04에서 다룸): hook은 메시지를 반환하거나 input을 수정하거나 실행을 차단할 수 있음
4. **권한 검사** (`921-931`, s03의 핵심 주제): canUseTool + checkPermissions → allow/deny/ask
5. **tool.call() 실행** (`1207-1222`)

교육용 버전은 Zod를 생략하고(JSON Schema 사용), validateInput을 생략하지만(safety 함수 사용), 권한 검사와 hook 개념은 보존한다.

### 5. 스트리밍 도구 실행

CC의 `StreamingToolExecutor` (`StreamingToolExecutor.ts`)는 모델이 아직 생성 중일 때 도구를 시작한다 — 모델이 끝날 때까지 기다리지 않는다. 모델이 아직 "Let me analyze"를 출력하는 중에 `read_file`이 완료될 수도 있다. 교육용 버전은 이를 구현하지 않으며, 이는 s01의 목표와 일치한다 — 최고 성능이 아니라 개념적 명료함.

### 6. 도구 결과 영속화

각 도구는 `maxResultSizeChars` 필드를 가진다. 이 임계값을 초과하는 결과는 디스크에 영속화되며, 모델은 미리보기 + 파일 경로를 보게 된다. FileRead는 특별하다 — `Infinity`로 설정되어 파일 읽기 출력이 다시 영속화되는 것을 막는다. 구체적으로, FileRead의 결과가 임계값을 초과해 영속화되면, 모델이 그 영속화된 파일을 다시 읽을 때 또 한 번 영속화를 유발한다 → 무한 루프 (파일 읽기 → 영속화 → 재읽기 → 재영속화 → ...).

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
