# s07: Skill Loading — 필요할 때만 로드하기

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → s02 → s03 → s04 → s05 → s06 → `s07` → [s08](../s08_context_compact/) → s09 → ... → s20
> *"필요할 때 로드하고, prompt를 가득 채우지 말라"* — system prompt가 아니라 tool_result를 통해 주입한다.
>
> **Harness Layer**: Knowledge — 필요할 때 로드하고, context를 채우지 말라.

---

## 문제

당신의 프로젝트에는 React 컴포넌트 명세, SQL 스타일 가이드, API 설계 문서가 있다. 당신은 Agent가 이 명세들을 자동으로 따르기를 바란다. 가장 직관적인 발상은 — 이것들을 전부 system prompt에 욱여넣는 것이다:

```python
SYSTEM = (
    f"You are a coding agent. "
    + open("docs/react-style.md").read()       # 2000 lines
    + open("docs/sql-style.md").read()         # 1500 lines
    + open("docs/api-design.md").read()        # 3000 lines
)
```

6500줄짜리 system prompt다. Agent는 CSS 색상을 바꾸든 SQL 쿼리를 고치든, 매 LLM 호출마다 이 문서들을 짊어진다. 내용의 99%는 현재 작업과 무관하며, 아무 이유 없이 토큰을 태운다.

---

## 해결책

![Skill 개요](images/skill-overview.en.svg)

이전 장의 최소 hook 구조, `todo_write`, 그리고 sub-Agent는 그대로 유지된다. 이번 장은 새로운 `load_skill` tool에 집중한다. 시작 시점에 skill 카탈로그를 SYSTEM prompt에 주입하고, 런타임에는 전체 내용을 로드하는 tool을 하나 더 등록하여, 실제로 사용할 때만 토큰을 쓴다.

2단계 설계:

| 단계 | 위치 | 시점 | 비용 |
|-------|----------|--------|------|
| 1. 카탈로그 | system prompt | 시작 시 주입 (harness가 skills/를 스캔) | skill당 ~100 토큰, 매 턴마다 짊어짐 |
| 2. 내용 | tool_result | Agent가 load_skill을 호출할 때; SKILL.md가 이후의 read_file/bash 접근을 안내하여 추가 리소스에 접근할 수 있음 | skill당 ~2000 토큰, 필요할 때만 |

dispatch 메커니즘은 변경되지 않으며, `load_skill`은 `TOOL_HANDLERS[block.name]`을 통해 자동으로 dispatch된다.

---

## 동작 방식

**skills/ 디렉터리**, skill 하나당 서브디렉터리 하나, 각각 `SKILL.md` 파일을 포함한다:

```
skills/
  agent-builder/SKILL.md
  code-review/SKILL.md
  mcp-builder/SKILL.md
  pdf/SKILL.md
```

**Level 1: 시작 시 카탈로그 주입**: harness는 시작 시 `_scan_skills()`를 호출하여 skills/ 디렉터리를 스캔하고, 각 SKILL.md의 YAML frontmatter(`name`, `description`)를 파싱하여 `SKILL_REGISTRY` 딕셔너리에 넣는다. `list_skills()`는 이 registry로부터 카탈로그를 생성하며, 이를 SYSTEM prompt에 주입한다. Agent는 추가 API 호출 없이 매 턴마다 "내가 사용할 수 있는 skill이 무엇인지"를 본다:

```python
SKILL_REGISTRY: dict[str, dict] = {}

def _scan_skills():
    if not SKILLS_DIR.exists():
        return
    for d in sorted(SKILLS_DIR.iterdir()):
        if not d.is_dir():
            continue
        manifest = d / "SKILL.md"
        if manifest.exists():
            raw = manifest.read_text()
            meta, body = _parse_frontmatter(raw)
            name = meta.get("name", d.name)
            desc = meta.get("description", raw.split("\n")[0].lstrip("#").strip())
            SKILL_REGISTRY[name] = {"name": name, "description": desc, "content": raw}

_scan_skills()  # runs once at startup

def list_skills() -> str:
    return "\n".join(f"- **{s['name']}**: {s['description']}" for s in SKILL_REGISTRY.values())

def build_system() -> str:
    catalog = list_skills()
    return (
        f"You are a coding agent at {WORKDIR}. "
        f"Skills available:\n{catalog}\n"
        "Use load_skill to get full details when needed."
    )

SYSTEM = build_system()
```

**Level 2: load_skill**: Agent가 "SQL 스타일 가이드가 필요하다"고 판단하고 `load_skill("sql-style")`을 호출한다. 조회는 파일 경로가 아니라 registry를 거치므로 path traversal 위험을 제거한다. SKILL.md 내용은 `tool_result`를 통해 주입되며, 참조된 `references/`, `scripts/`, `assets/`에 대한 이후 접근을 기존 파일 및 bash tool을 통해 포함할 수 있다.

```python
def load_skill(name: str) -> str:
    skill = SKILL_REGISTRY.get(name)
    if not skill:
        return f"Skill not found: {name}"
    return skill["content"]
```

핵심적인 구분: skill 내용은 system prompt의 일부가 아니다. 그것은 tool result로서 현재 messages에 들어간다. 이후의 호출들은 context compaction, truncation, 또는 세션 종료 전까지 그것을 히스토리와 함께 짊어진다. 이는 자연스럽게 s08의 compact로 이어진다: on-demand 로딩은 "짊어지지 말아야 할 것을 짊어지지 않기"를 해결하고, compact는 "버려야 할 것을 어떻게 버릴 것인가"를 해결한다.

---

## s06에서 달라진 점

| 구성 요소 | 이전 (s06) | 이후 (s07) |
|-----------|-------------|-------------|
| Tool 개수 | 7개 (bash, read, write, edit, glob, todo_write, task) | 8개 (+load_skill) |
| Knowledge 로딩 | 없음 | 2단계: 시작 시 SYSTEM에 카탈로그 + 런타임 load_skill; SKILL.md가 이후 리소스 접근을 안내할 수 있음 |
| SYSTEM prompt | 정적 문자열 | 시작 시 skills/를 스캔하여 카탈로그 주입 |
| Skill registry | 없음 | SKILL_REGISTRY (시작 시 채워지며, path traversal 방지) |
| Loop | 변경 없음 | 변경 없음 (skill tool이 자동으로 dispatch됨) |

---

## 실행해 보기

```sh
cd learn-claude-code
python s07_skill_loading/code.py
```

다음 prompt들을 시도해 보라:

1. `What skills are available?`
2. `Load the code-review skill and follow its instructions`
3. `I need to do a code review -- load the relevant skill first`

무엇을 지켜볼 것인가: Agent가 SYSTEM 카탈로그로부터 사용 가능한 skill을 알고 있는가? 전체 지침이 필요할 때 `[HOOK] load_skill`이 나타나는가? 답변이 로드된 skill의 지침을 사용하는가?

---

## 다음 단계

On-demand 로딩은 "짊어지지 말아야 할 것을 짊어지지 않기"를 해결했다. 그러나 또 다른 문제가 다가온다: Agent가 30분간 작업하고 나면, messages 리스트가 중간 과정으로 가득 찬다. 오래된 tool_result, 낡은 파일 내용이 context를 차지하면서도 아무런 가치를 더하지 않는다.

→ s08 Context Compact: 4계층 compaction 전략. 저렴한 계층이 먼저 실행되고, 비싼 계층이 마지막에 실행된다.

<details>
<summary>CC 소스 코드 깊이 살펴보기</summary>

> 다음 내용은 CC 소스 코드 `loadSkillsDir.ts`, `SkillTool.ts`, `bundledSkills.ts`, `commands.ts`에 대한 분석을 기반으로 한다.

### 1. Skill 소스: 단 하나의 skills/ 디렉터리가 아니다

교육용 버전은 모든 skill이 `skills/` 디렉터리에 있다고 가정한다. CC는 여러 파일에 흩어진 다수의 소스로부터 로드한다: `loadSkillsDir.ts`는 user/project/`--add-dir` 디렉터리와 legacy command(`.claude/commands/`)를 처리하고, `bundledSkills.ts`는 내장 skill을 처리하며, `SkillTool.ts`는 MCP 원격 skill을 처리하고, `commands.ts`는 command 집계를 처리한다. 종류로는 managed/policy skill, user skill(`~/.claude/skills/`), project skill(`.claude/skills/`), `--add-dir` skill, legacy command, dynamic skill, conditional skill(`paths` frontmatter를 가지며 파일 경로로 활성화됨), bundled skill, plugin skill, MCP skill이 있다.

### 2. SKILL.md Frontmatter — 공통 필드

CC의 SKILL.md YAML frontmatter는 `loadSkillsDir.ts`의 `parseSkillFrontmatterFields()`로 파싱된다. 공통 필드는 다음과 같다:

| 필드 | 용도 |
|-------|---------|
| `name` / `description` | 표시 이름과 설명 |
| `when_to_use` | 모델이 언제 호출할지 안내 |
| `allowed-tools` | skill이 사용할 수 있는 tool의 자동 허용 목록 |
| `context` | `inline`(기본값) 또는 `fork`(sub-Agent로 실행) |
| `model` | 모델 오버라이드 (haiku/sonnet/opus/inherit) |
| `hooks` | skill 수준의 hook 설정 |
| `paths` | 조건부 활성화를 위한 glob 패턴 |
| `user-invocable` | 사용자가 `/name`으로 호출 가능 |

전체 필드 목록은 버전에 따라 달라진다. 위는 교육용 버전과 관련된 핵심 필드들이다.

### 3. 2단계 로딩의 정밀한 구현

1. **카탈로그 (시작 시)**: `getSkillDirCommands()`가 디렉터리를 스캔 → 메타데이터만 담은 `Command` 객체로 등록한다. `getSkillListingAttachments()`는 skill 목록을 attachment로 포맷하며, context window의 약 1%(최대 8000자)로 예산이 책정된다.
2. **로드 (호출 시)**: 모델이 `Skill` tool을 호출(입력 필드는 `skill` + 선택적 `args`; 교육용 버전은 `name`을 사용) → `getPromptForCommand()`가 전체 SKILL.md 내용을 펼침 → `SkillTool`이 표시 텍스트 `"Launching skill: {name}"`을 담은 tool_result를 반환하고, 실제 skill 내용은 `newMessages`를 통해 주입된다. 교육용 버전은 둘 다 "tool_result를 통한 주입"으로 합쳐 단순화했다. 로드된 SKILL.md는 여전히 기존 파일/bash tool을 통해 참조된 리소스에 대한 이후 접근을 안내할 수 있다.

### 교육용 버전의 단순화는 의도적이다

- 여러 파일과 소스 → 1개의 `skills/` 디렉터리: 2단계 로딩의 핵심 개념을 보여주기에 충분하다
- 여러 frontmatter 필드 → name/description만 파싱: 파싱 복잡도를 줄인다
- Forked skill(`context: 'fork'`) → 생략: 교육용 버전은 inline skill 로딩만 펼친다
- `Skill` tool 입력 `skill`+`args` → 교육용 버전은 `name` 사용: 추가 인자 파싱 복잡도를 피한다

</details>

<!-- translation-sync: zh@v2, en@v2, ja@v2 -->
