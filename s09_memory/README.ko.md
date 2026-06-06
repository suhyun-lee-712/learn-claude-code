# s09: Memory — 압축은 디테일을 잃는다, 잃지 않는 레이어를 따로 두자

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → ... → s07 → s08 → `s09` → [s10](../s10_system_prompt/) → s11 → ... → s20
> *"압축은 디테일을 잃는다, 잃지 않는 레이어를 따로 두자"* — 파일 스토어 + 인덱스 + 온디맨드 로딩, compaction을 넘어, 세션을 넘어.
>
> **Harness Layer**: Memory — compaction과 세션을 넘어 살아남는 지식.

---

## 문제

s08의 autoCompact는 현재 목표, 남은 작업, 사용자 제약 조건을 요약에 보존하지만, 디테일은 사라진다. "스페이스 말고 탭을 써"가 "사용자는 코드 스타일 선호가 있음" 정도로 단순화될 수 있다. 게다가 새 세션을 시작하면 그 요약조차 사라진다.

LLM은 영속 상태를 갖지 않는다. 모든 정보는 context window 안에 산다. context가 가득 차면 압축되고, 압축은 손실이 있다(lossy). 필요한 것은 압축에 참여하지 않으면서 세션을 넘어 영속하는 스토리지 레이어다.

---

## 해결책

![메모리 개요](images/memory-overview.en.svg)

s08의 압축 파이프라인은 그대로 유지하고, memory에 집중한다. 스토리지는 파일시스템을 사용한다. `.memory/` 디렉터리에 각 메모리가 YAML frontmatter(`name` / `description` / `type`)를 가진 `.md` 파일로 들어간다. 파일이 쌓이면 인덱스가 필요하다. `MEMORY.md`가 한 줄에 하나의 링크를 담고 SYSTEM에 주입된다.

핵심 설계: 인덱스는 SYSTEM prompt에 남고(prompt cache로 캐싱 가능), 파일 내용은 온디맨드로 주입된다(현재 대화에 파일명/description으로 매칭, 캐시를 깨지 않음). 쓰기에는 두 가지 경로가 있다. 사용자가 명시적으로 "기억해"라고 말하거나, 각 턴이 끝난 뒤 백그라운드에서 추출(extraction)이 동작한다. 파일이 쌓이면 주기적인 통합(consolidation)이 중복을 제거한다.

네 가지 memory 타입은 각각 서로 다른 질문에 답한다:

| 타입 | 답하는 질문 | 예시 |
|------|---------|---------|
| user | 당신이 누구인가 | "스페이스 말고 탭을 써" |
| feedback | 어떻게 일할 것인가 | "데이터베이스를 mock하지 마" |
| project | 무슨 일이 벌어지고 있는가 | "Auth 재작성은 컴플라이언스 때문" |
| reference | 어디서 찾을 수 있는가 | "파이프라인 버그는 Linear INGEST에 있음" |

---

## 동작 방식

![메모리 서브시스템](images/memory-subsystems.en.svg)

### 스토리지: Markdown 파일 + 인덱스

각 메모리는 메타데이터용 YAML frontmatter를 가진 `.md` 파일이다:

```markdown
---
name: user-preference-tabs
description: User prefers tabs for indentation
type: user
---

User prefers using tabs, not spaces, for indentation.
**Why:** Consistency with existing codebase conventions.
**How to apply:** Always use tabs when writing or editing files.
```

`MEMORY.md`는 인덱스이며, 한 줄에 하나의 링크다:

```markdown
- [user-preference-tabs](user-preference-tabs.md) — User prefers tabs for indentation
```

새 메모리를 쓰면 인덱스가 자동으로 다시 빌드된다:

```python
def write_memory_file(name, mem_type, description, body):
    slug = name.lower().replace(" ", "-")
    filepath = MEMORY_DIR / f"{slug}.md"
    filepath.write_text(
        f"---\nname: {name}\ndescription: {description}\ntype: {mem_type}\n---\n\n{body}\n"
    )
    _rebuild_index()
```

### 로딩: 두 가지 경로

**경로 1: 인덱스를 SYSTEM에.** `build_system()`은 각 사용자 요청 시작 시 `MEMORY.md`를 한 번 읽어 메모리 카탈로그를 SYSTEM prompt에 주입한다. memory 추출과 통합은 턴이 끝날 때만 동작하므로, 같은 사용자 요청 안에서 SYSTEM을 반복해서 다시 빌드할 필요가 없다.

**경로 2: 관련 메모리를 온디맨드로.** 각 사용자 요청 시작 시 `load_memories()`가 최근 대화와 메모리 카탈로그(name + description)를 가벼운 사이드 쿼리(side-query)로 LLM에 보내 관련 파일명을 고르고, 그 내용을 읽어 주입한다. 비용을 통제하기 위해 최대 5개로 제한한다.

```python
def select_relevant_memories(messages, max_items=5):
    files = list_memory_files()
    if not files:
        return []

    # Build catalog: "0: user-preference-tabs — User prefers tabs..."
    catalog = "\n".join(f"{i}: {f['name']} — {f['description']}" for i, f in enumerate(files))

    response = client.messages.create(model=MODEL, messages=[{"role": "user",
        "content": f"Select relevant memory indices. Return JSON array.\n\n"
                   f"Recent conversation:\n{recent}\n\nMemory catalog:\n{catalog}"}],
        max_tokens=200)
    indices = json.loads(re.search(r'\[.*?\]', response.content[0].text).group())
    return [files[i]["filename"] for i in indices if 0 <= i < len(files)]
```

사이드 쿼리가 실패하면(API 오류, JSON 파싱 실패) name + description에 대한 키워드 매칭으로 폴백한다.

### 쓰기: 각 턴 이후의 추출

사용자가 항상 "이거 기억해"라고 말하지는 않는다. 선호는 보통 평범한 대화 곳곳에 흩어져 있다: "탭이 스페이스보다 낫다", "이제부터 작은따옴표를 쓰자".

`extract_memories()`는 각 턴이 끝날 때 동작하며, 모델이 tool_use 없이 멈출 때(대화가 자연스러운 단락에 도달했음을 의미) 트리거된다:

```python
# In agent_loop:
if response.stop_reason != "tool_use":
    extract_memories(messages)   # Extract new memories from recent dialogue
    consolidate_memories()       # Check if consolidation is needed
    return
```

추출 전에 기존 메모리를 확인해 중복을 피한다. 추출 prompt는 LLM에게 `{name, type, description, body}`의 JSON 배열을 반환하도록 요청하며, 진짜로 새로운 정보를 찾았을 때만 파일을 쓴다.

```python
def extract_memories(messages):
    dialogue = format_recent_messages(messages[-10:])
    existing = "\n".join(f"- {m['name']}: {m['description']}" for m in list_memory_files())

    prompt = (
        "Extract user preferences, constraints, or project facts.\n"
        "Return JSON array: [{name, type, description, body}].\n"
        "If nothing new or already covered, return [].\n\n"
        f"Existing memories:\n{existing}\n\nDialogue:\n{dialogue[:4000]}"
    )
    # ... parse response, write files ...
```

### 통합: 저빈도 중복 제거

메모리 파일은 쌓인다. `consolidate_memories()`는 파일 수가 임계값(기본 10)에 도달하면 트리거되어, LLM에게 중복 제거, 모순 병합, 낡은 메모리 정리를 요청한다:

```python
CONSOLIDATE_THRESHOLD = 10

def consolidate_memories():
    files = list_memory_files()
    if len(files) < CONSOLIDATE_THRESHOLD:
        return  # Too few, not worth consolidating
    # Send all memories to LLM, get back deduplicated list
    # Replace all files with consolidated results
```

CC는 이 과정을 **Dream**이라 부르며, 실제로는 네 개의 게이트를 둔다: 시간 간격, 스캔 스로틀, 세션 수, 파일 락. 교육용 버전은 파일 수 임계값으로 단순화한다.

### Memory가 저장하는 것

Memory는 세션을 넘어 계속 유용한 정보를 저장한다: 사용자 선호, 반복되는 피드백, 프로젝트 배경, 자주 쓰는 진입점, 조사 단서. "나중에 유용할 것"에 집중하며, 인덱스와 온디맨드 로딩을 통해 그 정보를 다시 가져온다.

Session memory는 하나의 세션 내부의 연속성에 집중한다: compaction 이후에 어떤 context가 살아남아야 하는가. 둘은 함께 동작한다. Memory는 장기 지식을 다루고, session memory는 compaction을 넘어 현재 세션을 다룬다.

---

## s08과의 변경점

| 컴포넌트 | 이전 (s08) | 이후 (s09) |
|-----------|-------------|-------------|
| Memory 능력 | 없음 (선호가 compaction과 함께 퇴화) | 스토리지 + 로딩 + 추출 + 통합 |
| 새 함수 | — | write_memory_file, select_relevant_memories, load_memories, extract_memories, consolidate_memories |
| 스토리지 | — | .memory/MEMORY.md 인덱스 + .memory/*.md 파일 |
| Tool | bash, read, write, edit, glob, todo_write, task, load_skill, compact (9) | bash, read_file, write_file, edit_file, glob, task (6) |
| Loop | 매 턴 압축만 | Memory 주입 + 압축 + 턴 이후 추출 + 주기적 통합 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s09_memory/code.py
```

다음 prompt들을 시도해 보자(여러 턴에 걸쳐 입력하며, 메모리가 쌓이고 로딩되는 것을 관찰):

1. `I prefer using tabs for indentation, not spaces. Remember that.`
2. `Create a Python file called test.py` (Agent가 탭을 쓰는지 관찰)
3. `What did I tell you about my preferences?` (Agent가 기억하는지 관찰)
4. `I also prefer single quotes over double quotes for strings.`

관찰 포인트: 각 턴 이후 `[Memory: extracted N new memories]`가 나타나는가? `.memory/`에 `.md` 파일이 생성되는가? `MEMORY.md` 인덱스가 업데이트되는가? 새 대화에서 Agent가 이전 메모리를 자동으로 로딩하는가?

---

## 다음 단계

Memory, 압축, tool이 모두 자리 잡았다. 하지만 system prompt는 여전히 하드코딩된 문자열이다. 새 tool을 추가하려면 description을 수동으로 추가해야 하고, 프로젝트를 바꾸면 prompt 전체를 다시 써야 한다. Prompt는 런타임에 조립되어야 한다.

s10 System Prompt → 세그먼트 + 런타임 조립. 다른 프로젝트, 다른 tool, 다른 prompt.

<details>
<summary>CC 소스 코드 깊이 파보기</summary>

> 아래 내용은 `src/` 아래 `memdir/`, `services/`, `utils/`, `query/`의 CC 소스 코드 분석에 기반한다. 줄 번호는 소스와 대조 검증했다.

### 소스 코드 경로

| 파일 | 줄 | 책임 |
|------|-------|---------------|
| `memdir/memdir.ts` | 507 | 코어: MEMORY.md 정의 (`34-38`), memory/plan/tasks를 구분하는 memory 동작 지시문 (`199-266`), `loadMemoryPrompt()` 세 경로 (`419-490`) |
| `memdir/findRelevantMemories.ts` | 141 | Sonnet 사이드 쿼리 memory 선택 (`18-24` system prompt, `97-122` 호출 로직) |
| `memdir/memoryTypes.ts` | 271 | 타입 정의, frontmatter 필드 |
| `memdir/memoryScan.ts` | — | .md 파일 스캔, MEMORY.md 제외, frontmatter 읽기, 최대 200개 파일, mtime 내림차순 정렬 (`35-94`) |
| `services/extractMemories/extractMemories.ts` | 615 | Forked agent 추출, 제한된 권한, `skipTranscript: true`, `maxTurns: 5` (`371-427`) |
| `services/autoDream/autoDream.ts` | 324 | Dream 통합, 4계층 게이팅 (`63-66` 기본값, `130-190` 게이팅, `224-233` forked agent) |
| `services/SessionMemory/sessionMemory.ts` | 495 | 세션 레벨 memory 관리 |
| `services/compact/sessionMemoryCompact.ts` | — | Session memory 경량 요약, 임계값 10K/5/40K (`56-61`) |
| `utils/attachments.ts` | — | 주입 예산: 파일당 200줄 / 4096바이트, 세션당 60KB (`269-288`); query로 관련 memory 찾기 (`2196-2241`) |
| `query.ts` | — | 각 사용자 턴 시작 시 memory 프리페치 (`301-304`), 논블로킹 수집 (`1592-1614`) |
| `query/stopHooks.ts` | — | Stop hook의 fire-and-forget가 추출과 Dream을 트리거 (`141-155`) |

### Memory 선택: 임베딩이 아니라 LLM

CC는 임베딩 벡터 유사도가 아니라 **Sonnet 자체로 선택**한다(`findRelevantMemories.ts`):

1. `memoryScan.ts`가 `.memory/`의 모든 `.md` 파일을 스캔(MEMORY.md 제외), 최대 200개, mtime 내림차순 정렬
2. 모든 memory 파일의 `name` + `description`을 카탈로그로 나열
3. Sonnet 사이드 쿼리로 전송: "name과 description으로 진짜 유용한 memory를 선택(최대 5개). 확신이 없으면 건너뛰어라."
4. Sonnet이 `{ selected_memories: ["file1.md", ...] }`를 반환
5. 선택된 파일의 전체 내용을 읽어(파일당 ≤ 200줄 / 4096바이트) 주입. 세션 총 예산: 60KB

각 사용자 턴 시작 시 `query.ts:301-304`가 memory 프리페치(비동기)를 시작하고, tool 실행 후 `1592-1614`가 완료된 결과를 논블로킹으로 수집한다.

### 추출 타이밍: autoCompact 이후가 아니라 Stop Hook

트리거 위치(`stopHooks.ts:141-155`): `handleStopHooks()` 내부에서 fire-and-forget가 추출과 Dream을 트리거한다. 교육용 버전은 추출을 `stop_reason != "tool_use"` 분기에 두어 방향성을 맞춘다.

CC의 추출은 forked agent로 동작한다(`extractMemories.ts:371-427`): 제한된 권한, `skipTranscript: true`, `maxTurns: 5`. 중복 보호도 있다: 메인 Agent가 이미 memory 파일을 썼다면 추출은 건너뛴다.

### Memory 파일 포맷

CC는 Markdown + YAML frontmatter를 사용하며, 교육용 버전과 일관된다. 네 가지 타입: `user`, `feedback`, `project`, `reference`.

`memdir.ts:34-38`은 인덱스 제약을 정의한다: `MEMORY.md`는 최대 200줄 / 25KB. `memdir.ts:199-266`은 memory 동작 지시문을 빌드하며, memory를 plan 및 tasks와 명시적으로 구분한다. 저장 위치: `~/.claude/projects/<sanitized-git-root>/memory/`.

### Dream: 4계층 게이팅

"유휴 상태일 때 트리거"나 "개수가 충분하면 통합"이 아니라, 네 개의 게이트다(`autoDream.ts`, 기본값 `63-66`, 게이팅 로직 `130-190`):

1. **시간 게이트**: 마지막 통합 이후 ≥ 24시간
2. **스캔 스로틀**: 잦은 파일시스템 스캔 방지
3. **세션 게이트**: 마지막 통합 이후 수정된 세션 transcript ≥ 5개
4. **락 게이트**: 현재 통합 중인 다른 프로세스 없음 (`.consolidate-lock` 파일)

병합 자체는 forked agent로 동작한다(`224-233`): 탐색 → 최근 신호 수집 → 병합 및 파일 쓰기 → 정리 및 인덱스 업데이트. 락 파일의 mtime이 lastConsolidatedAt 역할을 한다. 크래시 복구: 락은 1시간 후 자동 만료.

### User Memory vs Session Memory

| | User Memory | Session Memory |
|---|---|---|
| 영속성 | 세션 간 | 단일 세션 |
| 스토리지 | `memory/` 안의 여러 .md 파일 | `session-memory/<id>/memory.md` |
| 로딩 대상 | system prompt | compact 요약 |
| 목적 | 세션 간 지식 축적 | compact 간 context 연속성 |

sessionMemoryCompact(s08에서 언급)은 Session Memory를 사용한다: autoCompact 전에 session memory 파일을 읽고, 충분하면(≥ 10K tokens, ≥ 5개 텍스트 메시지, ≤ 40K tokens, `sessionMemoryCompact.ts:56-61`) LLM 호출 없이 그것을 요약으로 사용한다.

### 실제 구현이 더 복잡한 지점

- **Feature flags**: Memory 기능에는 여러 feature gate 레이어가 있다
- **Team memory**: 팀 공유 memory, `loadMemoryPrompt()`에 전용 경로가 있다(교육용 버전에서는 다루지 않음)
- **KAIROS**: 타이밍을 인지하는 memory 추출 전략, `loadMemoryPrompt()`의 daily-log 모드
- **Prompt cache**: Memory 주입은 prompt cache TTL을 고려해, 매 턴 전체 system prompt를 다시 쓰는 것을 피해야 한다
- **File locks**: 멀티 프로세스 시나리오를 위한 동시성 제어
- **Memory prefetch**: 비동기 프리페치, 메인 플로우 논블로킹

### 교육용 버전의 단순화는 의도적이다

- LLM 사이드 쿼리 → LLM 사이드 쿼리 + 키워드 폴백: 교육용 버전은 LLM 선택을 유지하면서 폴백 경로를 추가
- Memory JSON → Markdown + frontmatter: 교육용 버전이 CC와 일치
- Stop hook 트리거 → `stop_reason != "tool_use"` 분기: 같은 방향성
- 4계층 게이팅 → 파일 수 임계값: 교육용 버전에는 transcript 시스템과 멀티 세션 개념이 없음
- Forked agent + 제한된 권한 → 직접 호출: 교육용 버전에는 서브프로세스 격리가 없음

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
