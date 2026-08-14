# A Job Queue for Paid, Non-Idempotent Work

**Work unit** · One call to an external AI generation provider
**Cost** · Real money, per attempt
**Duration** · Minutes, not milliseconds
**Failure modes** · Some transient, some permanent, and telling them apart is the whole job

## Context

A capture session produces a job. The job calls an external provider to generate a video and a sticker, waits for it, downloads the result, and stores it. A customer is standing near a kiosk, or has already walked away with a QR code, waiting for that to finish.

Each attempt costs money. Retrying is not free, and retrying a request that already succeeded on the provider's side is worse than free — it is paying twice for one output.

## Problem

**Retry is the default answer to failure, and here it is sometimes the wrong one.** A network timeout should be retried. An authentication failure should not — it will fail identically, three more times, at cost. An invalid upload will never succeed no matter how many attempts it gets. A billing failure retried aggressively is a way to generate a large number of failed charges.

**Provider jobs can outlive our patience.** A generation request can be accepted by the provider and then stall. Requeuing it produces a second generation running in parallel with a first that may still complete — two charges, two outputs, one of which is discarded.

**Configuration drifts under a retrying job.** Operators tune models and prompts through a dashboard. If a job fails and is retried after an operator has changed something, the retry runs against different configuration than the original attempt. The result is a system where "we retried it" doesn't mean what it sounds like, and a failure can't be reproduced because the inputs no longer exist.

**The provider was expected to change.** The external service was explicitly a candidate rather than a commitment. Anything that hardcoded its identity would have to be unpicked later, under time pressure, in a system handling money.

## Approach

**The queue is a Postgres table.** Jobs are rows with a state, a configuration snapshot, and an attempt count. This is not the fastest queue available and it was never going to need to be — the work unit takes minutes, so queue overhead is irrelevant, and what matters instead is that job state is queryable, transactional with the rest of the data, and inspectable by an operator with SQL. A dedicated queue system would have added an operational component and moved job state somewhere it couldn't be joined against sessions and orders.

**Failures are classified, not counted.** Each failure is either retryable or terminal, by category:

| Category | Behaviour | Why |
|---|---|---|
| Transient provider or network failure | Retry with the automatic budget | Likely to succeed on another attempt |
| Authentication failure | Terminal | Will fail identically; retrying burns budget and time |
| Configuration error | Terminal | The job is wrong, not unlucky |
| Invalid upload | Terminal | The input cannot produce a result |
| Billing failure | Terminal | Retrying generates repeated failed charges |
| Storage failure | Terminal | Ours to fix, not the provider's to retry |
| Stale provider job | Never automatically requeued | A parallel generation would double-charge |

The last row is the one that matters most and is easiest to get wrong. A stalled job *looks* exactly like a case for a retry, and requeuing it is the intuitive fix. It is also how you pay twice.

**The automatic retry budget is small and fixed.** An initial attempt plus a bounded number of automatic retries, against the same configuration snapshot. When it is exhausted, the job stops and waits for a human. This is a deliberate cost ceiling: past a few attempts, the failure is probably not transient, and continuing to spend without anyone noticing is the failure mode of an unbounded retry loop.

**Beyond the budget, a retry is an explicit operator action.** An endpoint exists for it. Someone decides, and there is a record of them deciding.

**Every job carries an immutable configuration snapshot, and there are two layers of settings.** Each service has a baseline model and prompt configuration that cannot be edited, plus an operator-editable active configuration. A job records what it ran with. When an operator retries a job that failed on the active configuration, the retry runs against the **baseline** — not against whatever the dashboard currently holds.

This is subtle and it is the most important rule in the system. Without it, "retry with a known-good configuration" is impossible, because the only configuration available is the one someone has been adjusting while trying to diagnose the failure. The immutable baseline is the fixed point that makes recovery meaningful.

**The provider is named only in configuration.** Backend types, class names, and queue contracts describe *what kind of generation path* is being used, never which vendor supplies it. Endpoints and credentials are settings. Switching providers is an adapter and a configuration change, with no migration and no rename.

## Guarded destructive operations

Deleting a store or a device removes only rows that nothing references. If a device has credentials, orders, generation attempts, session history, or scoped configuration, the delete is refused with a conflict and the operator is directed to deactivate instead. Operational history is never cascaded away.

The reasoning is the same as everywhere else in this system: the data that explains what happened and what was charged is the data you need most when something has gone wrong, and it is precisely what a convenient cascade delete destroys.

---

## 한국어 요약

촬영 세션 하나가 잡 하나를 만듭니다. 잡은 외부 provider를 호출해 영상과 스티커를 생성하고, 기다리고, 결과를 내려받아 저장합니다. **시도마다 돈이 나갑니다.** 재시도는 공짜가 아니고, provider 쪽에서 이미 성공한 요청을 재시도하는 건 공짜보다 나쁩니다 — **출력 하나에 두 번 지불**하는 것입니다.

**어려웠던 지점**

- **실패의 기본 답은 재시도인데, 여기서는 종종 틀린 답입니다.** 네트워크 타임아웃은 재시도해야 합니다. 인증 실패는 아닙니다 — 똑같이, 세 번 더, 비용을 쓰며 실패합니다. 잘못된 업로드는 몇 번을 해도 성공하지 않습니다. 결제 실패를 공격적으로 재시도하는 건 **실패한 청구를 대량 생성하는 방법**입니다.
- **Provider 잡이 우리 인내심보다 오래 살 수 있습니다.** 생성 요청이 provider에 접수된 뒤 멈출 수 있습니다. 이걸 다시 큐에 넣으면 아직 완료될지 모르는 첫 번째와 **병렬로 두 번째 생성**이 돕니다 — 청구 둘, 출력 둘, 그중 하나는 버려집니다.
- **재시도하는 잡 아래에서 설정이 흘러갑니다.** 운영자가 대시보드로 모델과 프롬프트를 조정합니다. 잡이 실패하고, 운영자가 뭔가 바꾼 뒤에 재시도되면, 재시도는 **원래 시도와 다른 설정**으로 돌아갑니다. 결과적으로 "재시도했다"가 들리는 대로의 의미가 아니게 되고, **입력이 더 이상 존재하지 않아서 실패를 재현할 수 없습니다.**
- **Provider는 바뀔 것으로 예상됐습니다.** 외부 서비스는 확정이 아니라 후보였습니다. 그 정체를 하드코딩한 모든 것은 나중에, 시간 압박 속에서, 돈을 다루는 시스템에서 풀어내야 했을 겁니다.

**접근**

- **큐는 Postgres 테이블입니다.** 잡은 상태·설정 스냅샷·시도 횟수를 가진 행입니다. 가장 빠른 큐가 아니고 그럴 필요도 없었습니다 — 작업 단위가 분 단위라 큐 오버헤드는 무의미하고, 대신 중요한 건 **잡 상태가 조회 가능하고, 나머지 데이터와 트랜잭션을 공유하고, 운영자가 SQL로 들여다볼 수 있다**는 점입니다. 전용 큐 시스템은 운영 구성요소를 하나 늘리고 잡 상태를 세션·주문과 조인할 수 없는 곳으로 옮겼을 겁니다.
- **실패는 세는 게 아니라 분류합니다.**

| 분류 | 동작 | 이유 |
|---|---|---|
| 일시적 provider·네트워크 실패 | 자동 예산 내 재시도 | 다음 시도에 성공할 가능성 |
| 인증 실패 | 종료 | 동일하게 실패. 예산과 시간만 소모 |
| 설정 오류 | 종료 | 운이 나쁜 게 아니라 잡이 틀린 것 |
| 잘못된 업로드 | 종료 | 그 입력으로는 결과가 나오지 않음 |
| 결제 실패 | 종료 | 재시도는 실패 청구를 반복 생성 |
| 스토리지 실패 | 종료 | provider가 재시도할 게 아니라 우리가 고칠 것 |
| 멈춘 provider 잡 | **절대 자동 재큐잉 안 함** | 병렬 생성 = 이중 과금 |

마지막 줄이 가장 중요하고 가장 틀리기 쉽습니다. 멈춘 잡은 **정확히 재시도 대상처럼 보이고**, 다시 큐에 넣는 게 직관적인 해결책입니다. 그리고 그게 두 번 지불하는 방법입니다.

- **자동 재시도 예산은 작고 고정입니다.** 최초 시도 + 제한된 횟수의 자동 재시도, 모두 **같은 설정 스냅샷**으로. 소진되면 잡은 멈추고 사람을 기다립니다. 의도적인 비용 상한입니다 — 몇 번을 넘어가면 실패는 대개 일시적이지 않고, **아무도 모르게 계속 지출하는 것**이 무제한 재시도 루프의 실패 모드입니다.
- **예산 너머의 재시도는 명시적 운영자 행위입니다.** 그를 위한 엔드포인트가 있습니다. 사람이 결정하고, 결정한 기록이 남습니다.
- **모든 잡이 불변 설정 스냅샷을 갖고, 설정은 2계층입니다.** 각 서비스에 **편집 불가능한 baseline** 모델·프롬프트 설정과 **운영자 편집 가능한 active** 설정이 있습니다. 잡은 무엇으로 실행됐는지 기록합니다. active 설정에서 실패한 잡을 운영자가 재시도하면, 재시도는 **baseline으로** 돕니다 — 대시보드가 현재 들고 있는 값이 아니라.

  이건 미묘하고, **이 시스템에서 가장 중요한 규칙**입니다. 이게 없으면 "알려진 정상 설정으로 재시도"가 불가능합니다. 유일하게 사용 가능한 설정이 **누군가 장애를 진단하느라 계속 만지고 있던 그 설정**이기 때문입니다. 불변 baseline이 복구를 의미 있게 만드는 고정점입니다.
- **Provider 이름은 설정에만 등장합니다.** 백엔드 타입·클래스명·큐 계약은 **어떤 종류의 생성 경로**인지를 기술하지, 어느 벤더가 제공하는지를 기술하지 않습니다. 엔드포인트와 자격증명은 설정값입니다. 교체는 어댑터 하나와 설정 변경이며, 마이그레이션도 이름 변경도 없습니다.

**보호된 파괴적 연산** — 스토어나 디바이스 삭제는 **아무것도 참조하지 않는 행만** 제거합니다. 자격증명·주문·생성 시도·세션 이력·범위 설정이 걸린 디바이스는 409로 거부되고 운영자는 비활성화로 안내됩니다. **운영 이력은 절대 cascade되지 않습니다.** 이 시스템의 다른 모든 곳과 같은 논리입니다 — **무슨 일이 있었고 얼마가 청구됐는지 설명하는 데이터는 뭔가 잘못됐을 때 가장 필요한 데이터**이고, 편리한 cascade delete가 정확히 그걸 파괴합니다.
