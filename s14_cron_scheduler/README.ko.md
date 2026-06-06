# s14: Cron Scheduler — 정해진 일정에 따라 작업 생성하기

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → ... → s12 → s13 → `s14` → [s15](../s15_agent_teams/) → s16 → ... → s20
> *"정해진 일정에 따라 작업을 생성하고, 스케줄링과 실행을 분리한다"* — Cron 스케줄링, durable 또는 세션 단위.
>
> **Harness Layer**: 스케줄링 — 독립적인 thread가 시간을 확인하고, queue가 트리거를 전달한다.

---

## 문제

알람 시계는 당신이 지켜볼 필요가 없다. 7시로 맞춰 두면 7시에 울린다 — 자고 있든, 샤워 중이든, 요리 중이든 상관없이 울린다.

s13은 agent가 느린 작업을 백그라운드에서 실행할 수 있게 해주지만, 모든 작업은 여전히 수동으로 트리거된다. 당신이 무언가를 말하면 agent가 동작한다. "매일 아침 9시에 테스트 실행", "30분마다 CI 상태 확인" — 이런 반복 작업은 매번 사람이 직접 밀어붙일 필요가 없어야 한다.

---

## 해결책

![Cron Scheduler 개요](images/cron-scheduler-overview.en.svg)

교육용 코드는 S13의 단순화된 task 시스템, 백그라운드 실행, prompt 조립을 그대로 이어받는다. scheduler에 집중하기 위해 완전한 에러 복구, 메모리, skill 시스템은 생략한다. 추가된 것: 매초 폴링하는 독립적인 cron scheduler thread, 매칭된 job을 `cron_queue`에 넣고, agent가 idle 상태일 때 그것을 전달하는 queue processor.

수동 vs 스케줄:

| | 수동 (s13) | 스케줄 (s14) |
|---|---|---|
| 트리거 주체 | 사용자 입력 | Scheduler thread |
| 트리거 타이밍 | 언제든지 | cron 표현식으로 지정 |
| 사람 개입 | 있음 | 없음 (scheduler가 자동 enqueue, idle agent가 자동 전달) |
| 영속성 | — | Durable, 재시작 후에도 유지 |

---

## 동작 원리

### 4계층 모델

Cron 스케줄링에는 네 개의 계층이 있다:

1. **Scheduler**: daemon thread, 매초 폴링하며 지금이 그 시간인지 확인
2. **Queue**: `cron_queue`, scheduler가 발화된(fired) job을 기록
3. **Queue Processor**: queue가 비어 있지 않고 agent가 idle임을 확인하면 agent_loop 턴을 한 번 시작
4. **Consumer**: agent_loop가 queue를 소비하여 messages에 주입

교육용 버전은 최소한의 queue processor를 구현한다: `agent_lock`이 agent가 idle인지 알려주고, queue에 들어간 cron 작업이 자동으로 전달된다. 실제 CC의 `useQueueProcessor.ts`는 UI 블로킹, queue 우선순위, 다양한 message 모드까지 처리한다.

### CronJob: 데이터 구조

각 cron 작업은 하나의 `CronJob` 객체다:

```python
@dataclass
class CronJob:
    id: str
    cron: str        # "0 9 * * *" (5-field cron expression)
    prompt: str      # Message injected to the agent when fired
    recurring: bool  # True=recurring, False=one-shot
    durable: bool    # True=write to disk, survives sessions
```

Cron 표현식, 5개 필드, Unix에서 50년간 사용되어 왔다:

```
min  hour  dom  month  dow
 *    *     *     *     *      Every minute
 0    9     *     *     *      Every day at 9:00
*/5    *     *     *     *      Every 5 minutes
 0    9     *     *    1-5     Weekdays at 9:00
```

`*`, `*/N`, `N`, `N-M`, `N,M,...`를 지원한다.

### cron_matches: 5-Field 매칭

표준 cron 의미론: minute, hour, month는 모두 일치해야 한다. day-of-month(DOM)와 day-of-week(DOW)는 둘 다 제약(constrained)이 걸려 있을 때 OR로 처리한다:

```python
def cron_matches(cron_expr: str, dt: datetime) -> bool:
    fields = cron_expr.strip().split()
    if len(fields) != 5:
        return False
    minute, hour, dom, month, dow = fields
    dow_val = (dt.weekday() + 1) % 7  # Python Monday=0 → cron Sunday=0

    m = _cron_field_matches(minute, dt.minute)
    h = _cron_field_matches(hour, dt.hour)
    dom_ok = _cron_field_matches(dom, dt.day)
    month_ok = _cron_field_matches(month, dt.month)
    dow_ok = _cron_field_matches(dow, dow_val)

    if not (m and h and month_ok):
        return False
    # DOM and DOW: both constrained → either matching is enough (OR)
    dom_unconstrained = dom == "*"
    dow_unconstrained = dow == "*"
    if dom_unconstrained and dow_unconstrained:
        return True
    if dom_unconstrained:
        return dow_ok
    if dow_unconstrained:
        return dom_ok
    return dom_ok or dow_ok
```

### 독립적인 Scheduler Thread: 1초 폴링

scheduler는 agent_loop가 실행 중인지 여부와 무관하게 독립적인 daemon thread에서 동작한다. 개별 job의 에러가 전체 thread를 죽이지 않는다:

```python
def cron_scheduler_loop():
    while True:
        time.sleep(1)
        now = datetime.now()
        minute_marker = now.strftime("%Y-%m-%d %H:%M")
        with cron_lock:
            for job in list(scheduled_jobs.values()):
                try:
                    if cron_matches(job.cron, now):
                        if _last_fired.get(job.id) != minute_marker:
                            cron_queue.append(job)
                            _last_fired[job.id] = minute_marker
                        if not job.recurring:
                            scheduled_jobs.pop(job.id, None)
                            if job.durable:
                                save_durable_jobs()
                except Exception as e:
                    print(f"[cron error] {job.id}: {e}")
```

핵심 설계:
- **agent_loop와 독립**: agent_loop가 실행 중이 아니어도 scheduler는 백그라운드에서 시간을 확인한다
- **날짜를 포함한 minute_marker**: `"YYYY-MM-DD HH:MM"`를 사용해 같은 분(minute) 내 중복 발화를 막으면서도 다음 날에는 건너뛰지 않는다
- **job별 try/except**: 잘못된 job 하나가 scheduler thread를 죽이지 않는다
- **One-shot job**: 발화 후 scheduled_jobs에서 자동으로 제거된다

### Queue Processor + agent_loop: 전달

queue processor는 시간을 확인하지 않는다. queue에 작업이 있고 agent가 idle일 때만 턴을 시작한다:

```python
def queue_processor_loop():
    while True:
        time.sleep(0.2)
        if not has_cron_queue():
            continue
        if not agent_lock.acquire(blocking=False):
            continue
        try:
            if has_cron_queue():
                run_agent_turn_locked()
        finally:
            agent_lock.release()
```

agent_loop 또한 시간을 확인하지 않는다. `cron_queue`에서 발화된 작업만 꺼내서 messages에 주입한다:

```python
fired = consume_cron_queue()
for job in fired:
    messages.append({"role": "user",
                     "content": f"[Scheduled] {job.prompt}"})
```

생산자(scheduler thread), 전달자(queue processor), 소비자(agent_loop)는 `cron_queue`, `cron_lock`, `agent_lock`을 통해 분리(decouple)되어 있다.

### 검증: 잘못된 Cron이 Scheduler를 죽이지 못하게

`schedule_job`은 등록하기 전에 cron 표현식을 검증하고, 잘못된 입력에 대해서는 에러를 반환한다:

```python
def schedule_job(cron, prompt, recurring=True, durable=True):
    err = validate_cron(cron)
    if err:
        return err
    # ... register job
```

디스크에서 durable job을 로드할 때도 잘못된 표현식은 건너뛰어, 단 하나의 잘못된 작업이 시작(startup)을 망가뜨리지 않게 한다.

### Durable vs Session-only

- **Durable**: 작업 정의를 `.scheduled_tasks.json`에 기록한다. agent 재시작 시 로드된다.
- **Session-only**: 메모리에만 존재한다. agent가 종료되면 사라진다.

> **중요한 주의사항**: cron scheduler는 반드시 agent 프로세스 내부에서 실행되어야 한다. 프로세스가 종료되면 scheduler도 멈춘다. Durable이 의미하는 것은 작업 정의가 재시작 후에도 살아남는다는 것뿐이다 — 다음에 agent가 시작될 때 scheduler가 "발화해야 한다"는 것을 발견하고 발화한다. "앱이 닫혀 있어도 실행"이 필요하다면 시스템 crontab이나 systemd timer를 사용하라.

### 종합

```
1. On startup:
   load_durable_jobs() → restore durable tasks from .scheduled_tasks.json
   Thread(cron_scheduler_loop, daemon=True).start() → scheduler begins polling
   Thread(queue_processor_loop, daemon=True).start() → processor waits to deliver

2. Register a task:
   schedule_cron(cron="*/2 * * * *", prompt="run date", durable=True)
   → CronJob written to scheduled_jobs + .scheduled_tasks.json

3. Every 2 minutes:
   Scheduler checks → cron_matches returns True → cron_queue.append(job)
   → queue processor sees idle agent → agent_loop consume_cron_queue
   → injects "[Scheduled] run date"
   → LLM receives message, runs date command

4. Process shutdown:
   Scheduler thread stops (daemon=True)
   .scheduled_tasks.json stays on disk
   Next startup → load_durable_jobs → tasks restored
```

---

## s13에서 달라진 점

| 컴포넌트 | 이전 (s13) | 이후 (s14) |
|-----------|-------------|-------------|
| 트리거 방식 | 사용자 수동 트리거 | Scheduler thread가 자동 enqueue |
| 새 타입 | — | CronJob dataclass (id, cron, prompt, recurring, durable) |
| 새 함수 | — | cron_matches, validate_cron, schedule_job, cancel_job, cron_scheduler_loop, queue_processor_loop |
| 새 저장소 | — | .scheduled_tasks.json (durable) + memory (session-only) |
| Thread | 백그라운드 실행 thread | + Scheduler thread (daemon, 1초 폴링) + queue processor thread |
| Queue | background_results | + cron_queue (scheduler가 기록, queue processor가 전달, agent_loop가 소비) |
| Tools | 8개 (s12/s13) | + schedule_cron, list_crons, cancel_cron (11개) |

---

## 직접 해보기

```sh
cd learn-claude-code
python s14_cron_scheduler/code.py
```

다음 prompt들을 시도해 보라:

1. `Schedule a task to print the current date every 2 minutes`
2. `List all cron jobs`
3. `Create a one-shot reminder in 1 minute to check the build status`
4. `Cancel the recurring job and verify with list_crons`

관찰할 점: scheduler thread가 독립적으로 동작하는가? cron 작업이 정확한 시간에 발화하는가? 새 prompt 없이도 `[queue processor]`와 자동 실행이 보이는가? durable job이 `.scheduled_tasks.json`에 기록되는가?

---

## 다음 단계

이제 하나의 agent가 많은 일을 할 수 있다: 계획, 압축, 백그라운드, 스케줄링. 하지만 어떤 작업은 한 agent가 감당하기엔 너무 크다.

"백엔드 전체를 리팩토링하라" — 인증, 데이터베이스 계층, API 라우트, 테스트를 전면 개편. 한 agent의 주의력에는 한계가 있다. 이건 팀이 필요하다.

s15 Agent Teams → 한 agent로는 부족하다, 팀을 구성하자. 지속적인 동료(persistent teammates) + 비동기 inbox.

<details>
<summary>CC 소스 코드 심층 분석</summary>

> 다음은 CC 소스 코드 `CronCreateTool.ts`, `cronScheduler.ts`, `cron.ts`, `cronTasks.ts`, `cronTasksLock.ts`, `useScheduledTasks.ts` (139 lines)에 기반한 완전한 분석이다.

### 1. 세 개의 Cron Tool

CC는 모델에게 세 개의 cron tool을 노출한다: `CronCreate`, `CronDelete`, `CronList`. 모두 컴파일 타임 게이트 `feature('AGENT_TRIGGERS')`와 런타임 GrowthBook 플래그 `tengu_kairos_cron`으로 제어된다. 로컬 오버라이드를 위한 `CLAUDE_CODE_DISABLE_CRON` 환경 변수도 있다.

### 2. 저장소: `.claude/scheduled_tasks.json`

```json
{ "tasks": [{ "id": "abc12345", "cron": "0 9 * * *", "prompt": "...", "recurring": true, "durable": true, "createdAt": 1714567890000 }] }
```

Durable 작업은 디스크에 기록되고, session-only 작업은 `STATE.sessionCronTasks` 메모리 배열에 존재한다(프로세스 재시작 시 소실). `.scheduled_tasks.lock` 파일은 동일 프로젝트의 여러 세션에 걸친 중복 발화를 방지한다.

### 3. Scheduler: 1초 폴링

`cronScheduler.ts`는 매초(`CHECK_INTERVAL_MS = 1000`) 확인한다. lock을 보유한 쪽이 파일 작업을 트리거하고, 모든 세션은 session-only 작업을 트리거한다. `chokidar` 파일 watcher가 `scheduled_tasks.json` 변경을 감시한다.

### 4. Cron 표현식: 표준 5개 필드

Minute, hour, day, month, weekday. `*`, `*/N`, `N`, `N-M`, `N-M/S`, `N,M,...`를 지원한다. `L`, `W`, `?`는 지원하지 않는다. 모든 시간은 로컬 타임존으로 해석된다. day-of-month와 day-of-week는 둘 다 제약이 걸려 있을 때 OR 의미론을 사용한다.

### 5. Jitter (Thundering Herd 방지)

- Recurring 작업: 주기의 최대 10%까지 트리거 지연(최대 15분), task ID 기반의 결정적(deterministic) 해시
- One-shot 작업: 발화 시간이 `:00` 또는 `:30`에 떨어질 때 최대 90초 일찍
- Jitter 설정은 GrowthBook로 조정 가능하며 60초마다 갱신된다

### 6. 자동 만료(Auto-Expiration)

Recurring 작업은 7일 후 자동 만료된다(설정 가능, 최대 30일). 만료 직전에 마지막으로 한 번 발화한 뒤 자동 삭제된다.

### 7. Job 한도

`MAX_JOBS = 50` (`CronCreateTool.ts:25`). 초과 시 에러를 반환한다: "Too many scheduled jobs (max 50). Cancel one first."

### 8. Trigger 주입

발화 후, `enqueuePendingNotification()`를 통해 `priority: 'later'`로 command queue에 enqueue된다. `workload: WORKLOAD_CRON`으로 태깅되며 — 용량이 빠듯할 때 API는 cron으로 시작된 요청을 더 낮은 QoS로 처리한다.

### 9. Queue Processor: 자동 전달

실제 CC는 활성 query가 없고, UI가 블로킹되지 않았으며, queue가 비어 있지 않을 때 `useQueueProcessor.ts:48-60`를 통해 처리를 자동 트리거한다. `queueProcessor.ts:52-87`은 queue 우선순위에 따라 command를 `handlePromptSubmit()`로 디스패치한다. 교육용 버전은 `queue_processor_loop`로 핵심 동작을 유지한다: queue에 작업이 있고 agent가 idle일 때 agent_loop 턴을 한 번 자동으로 시작한다.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
