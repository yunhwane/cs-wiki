# 프로세스 (Process)

> 개요. 점진 확장 예정.

실행 중인 프로그램. 디스크 위 정적 바이너리 = 프로그램, 메모리 적재되어 CPU 점유 중 = 프로세스.

---

## 1. CPU 와 프로세스 — 왜 전환 필요한가

CPU 코어 1개 = 한 순간 1개 명령어만 실행. 그러나 동시에 실행 중처럼 보이는 프로세스 다수 (브라우저, IDE, 음악 플레이어 ...).

→ OS 가 짧은 시간 단위 (time slice, 보통 수 ms) 로 프로세스 번갈아 CPU 할당. 빠르게 전환되어 사용자 눈엔 동시 실행.

이 전환 과정 = **Context Switch**. 전환 위해 각 프로세스 상태 보존 필요 → **PCB**.

---

## 2. PCB (Process Control Block)

OS 가 프로세스마다 1개씩 유지하는 자료구조. 프로세스 정체성·상태·자원 정보 전부 모음. 커널 공간 (kernel space) 에 존재 — 사용자 코드 직접 접근 불가.

> Linux 구현체 = `struct task_struct` (`include/linux/sched.h`). 크기 약 10KB 내외, 필드 수백 개.

### 2.1 PCB 필드 카테고리

#### (a) 식별자

| 필드 | 의미 |
|------|------|
| PID | 프로세스 식별자 (Linux: `pid_t pid`) |
| TGID | Thread Group ID. 멀티스레드 시 메인 스레드 PID = 모든 스레드 공유. `getpid()` 가 반환하는 값 |
| PPID | 부모 프로세스 PID |
| PGID / SID | 프로세스 그룹, 세션 ID (시그널·터미널 제어용) |
| UID / GID | 소유자 / 그룹 (real, effective, saved 3종) |

→ Linux 에서 스레드 = "PID 다른데 TGID 같은 task". 커널은 스레드도 task_struct 1개 가짐.

#### (b) 상태 (state)

```c
volatile long state;   // TASK_RUNNING, TASK_INTERRUPTIBLE,
                       // TASK_UNINTERRUPTIBLE, TASK_STOPPED,
                       // TASK_TRACED, EXIT_ZOMBIE, EXIT_DEAD ...
```

- `TASK_RUNNING`: 실행 중 또는 ready (둘 다 같은 상태). 실제 CPU 점유 여부는 런큐 위치로 결정.
- `TASK_INTERRUPTIBLE`: sleep 중. 시그널로 깨움 가능 (예: `read()` 대기).
- `TASK_UNINTERRUPTIBLE`: sleep 중. 시그널 무시. 디스크 I/O 대기 등 (`ps` 에서 `D` state).
- `EXIT_ZOMBIE`: 종료 후 부모가 `wait()` 하기 전. 종료 코드만 보존, 자원 대부분 해제.

#### (c) CPU 컨텍스트 (Context Switch 핵심)

```c
struct thread_struct thread;   // 아키텍처별
// x86_64 예: rsp, rip, fs/gs base, debug regs, FPU/SIMD state, ...
```

- 범용 레지스터 (RAX~R15)
- 명령어 포인터 (RIP / PC)
- 스택 포인터 (RSP), 베이스 포인터 (RBP)
- 플래그 레지스터 (RFLAGS)
- 세그먼트 / TLS 베이스 (FS, GS)
- FPU·SSE·AVX 상태 (lazy save 가능)
- 페이지 테이블 베이스 (CR3) — 메모리 디스크립터 통해 간접 보관

→ Context Switch 시 이 부분이 PCB 에 저장/복원되는 영역. 다른 필드 (PID, UID 등) 는 안 건드림.

#### (d) 메모리 디스크립터

```c
struct mm_struct *mm;          // 사용자 주소 공간
struct mm_struct *active_mm;   // 커널 스레드용
```

`mm_struct` 안에:
- `pgd` (page global directory) → CR3 에 적재될 페이지 테이블 루트
- `mmap` → VMA (Virtual Memory Area) 리스트: 코드/데이터/힙/스택/매핑 파일 각각
- `start_code`, `end_code`, `start_data`, `end_data`, `start_brk`, `brk`, `start_stack` — 영역 경계
- `arg_start/end`, `env_start/end` — 명령행 인자, 환경변수 위치
- RSS, VSS 등 사용량

→ 같은 프로세스 내 스레드 = `mm` 공유. fork 자식 = COW (Copy-on-Write) 로 공유 후 분기.

#### (e) 스케줄링

```c
int prio, static_prio, normal_prio;
unsigned int policy;              // SCHED_NORMAL, SCHED_FIFO, SCHED_RR, SCHED_DEADLINE
struct sched_entity se;           // CFS 용 (vruntime, load weight, ...)
struct sched_rt_entity rt;        // 실시간용
cpumask_t cpus_allowed;           // affinity
```

- vruntime: CFS 스케줄러 가상 실행 시간. 작은 값부터 실행.
- nice (-20 ~ 19): 우선순위 가중치.
- affinity: 특정 CPU 코어에만 묶기.

#### (f) 파일·I/O

```c
struct files_struct *files;       // fd 테이블
struct fs_struct *fs;             // cwd, root, umask
struct nsproxy *nsproxy;          // 네임스페이스 (mnt, pid, net, ipc, uts, user, cgroup)
```

- `files->fd_array[]` 인덱스 = fd 번호. fd 0=stdin, 1=stdout, 2=stderr.
- fork 시 fd 테이블 복사. exec 시 close-on-exec 플래그 fd 만 닫힘.

#### (g) 시그널

```c
struct signal_struct *signal;     // 프로세스 전체 공유
struct sighand_struct *sighand;   // 시그널 핸들러 테이블
sigset_t blocked, real_blocked;   // 마스크
struct sigpending pending;        // 대기 시그널
```

#### (h) 친족 관계 (트리)

```c
struct task_struct __rcu *real_parent;
struct task_struct __rcu *parent;
struct list_head children;        // 자식 리스트
struct list_head sibling;         // 형제 리스트
struct task_struct *group_leader; // 스레드 그룹 리더
```

→ 트리 형태. PID 1 (`init` / `systemd`) 가 루트. 부모 죽은 고아 = init 이 입양.

#### (i) 자원 사용량·제한

- `utime`, `stime`: 사용자/커널 모드 CPU 시간
- `start_time`: 생성 시각
- `rlim[]`: ulimit (스택 크기, 파일 수, CPU 시간 등)
- 페이지 폴트, 컨텍스트 스위치 카운트

#### (j) 보안·자격 증명

```c
const struct cred __rcu *cred;    // uid, gid, capabilities, SELinux ctx
```

setuid 시 effective uid 변경.

---

### 2.2 PCB 메모리 배치 — task_struct ↔ 커널 스택

Linux 에서 각 task 마다 커널 스택 (보통 8KB, 16KB) 별도 할당. 과거엔 스택 끝에 `thread_info` 박혀 있어서 `current` 매크로가 `RSP & ~(THREAD_SIZE-1)` 로 빠르게 task_struct 찾았음.

```
높은 주소
┌──────────────────┐
│   kernel stack   │ ← 인터럽트/시스템 콜 시 사용
│        ↓         │
│                  │
│        ↑         │
│   thread_info    │ (구버전. 최신은 task_struct 안으로 이동)
└──────────────────┘ 낮은 주소
                     ↓ struct task_struct *task   ─→ PCB 본체 (별도 slab 할당)
```

최신 커널: `CONFIG_THREAD_INFO_IN_TASK` 로 `thread_info` 가 task_struct 첫 멤버. `current` = per-CPU 변수로 직접 가리킴.

### 2.3 PCB 검색 자료구조

수천~수만 프로세스 중 PID 로 빠르게 찾기:

- **PID 해시 테이블** (`pid_hash`): PID → task_struct 매핑
- **IDR / radix tree**: PID 할당·재사용 관리 (`alloc_pid`)
- **task list** (doubly linked list): 모든 프로세스 순회용. `for_each_process(p)` 매크로

### 2.4 PCB 생성·소멸

| 시점 | 동작 |
|------|------|
| `fork()` / `clone()` | 부모 task_struct 대부분 복사 (`copy_process`). PID 새로 할당. mm/files/sighand 등은 플래그에 따라 공유 또는 복사 (COW) |
| `exec()` | task_struct 유지. mm 교체 (새 바이너리 매핑), fd 일부 close-on-exec, 시그널 핸들러 기본값 |
| `exit()` | 자원 해제 (mm, files, ...). 상태 = ZOMBIE. 종료 코드만 남김 |
| `wait()` (부모) | ZOMBIE task_struct 최종 free. 부모 없으면 init 이 회수 |

### 2.5 ps 로 PCB 일부 보기

```bash
ps -eo pid,ppid,tgid,stat,nice,pri,rss,vsz,wchan,comm
# stat 예: R(run) S(sleep) D(uninterruptible) Z(zombie) T(stopped) + 추가 플래그(<, N, s, l)
cat /proc/$$/status   # 현재 셸 PCB 상당 부분 텍스트로
cat /proc/$$/maps     # mm_struct VMA 덤프
```

`/proc/[pid]/` = 커널이 PCB 내용 노출하는 가상 파일시스템.

---

→ Context Switch 시 현재 PCB 의 (c) CPU 컨텍스트 저장, 다음 PCB 의 (c) 복원 + (d) 의 페이지 테이블 (CR3) 교체. 그게 전환 핵심 비용.

---

## 3. Context Switch

**현재 프로세스 상태 PCB 에 저장 → 다음 프로세스 PCB 에서 복원 → 실행 재개**.

CPU 보는 단순한 사실: 한 순간 1개 프로세스만 실행. 나머지 환상 = 빠른 전환의 결과.

### 3.1 Mode Switch ≠ Context Switch

혼동 잦음. 둘 다른 개념.

| 구분 | 무엇 | 비용 |
|------|------|------|
| **Mode Switch** | user mode ↔ kernel mode 전환. CPL (Current Privilege Level) 변경. CR3 안 바꿈, 같은 프로세스 안에서 발생 | 작음 (~수십 ns) |
| **Context Switch** | 프로세스(또는 스레드) A → B 로 실제 교체. PCB 저장/복원 | 큼 (~µs ~ 수 µs) |

시스템 콜 (`read()`, `write()` 등) → **mode switch만** 발생할 수도 있고, 블록되면 context switch 까지 이어짐.

### 3.2 발생 계기 — voluntary vs involuntary

**Voluntary (자발적):**
- I/O 대기 (`read()`, `recv()`, `wait()` 블록)
- 락 대기 (mutex, semaphore)
- `sleep()`, `yield()`, `select()`/`epoll_wait()`

**Involuntary (강제):**
- 타임 슬라이스 만료 (timer interrupt → 스케줄러 호출)
- 우선순위 더 높은 task 도착 (preemption)
- 인터럽트 처리 후 `need_resched` 플래그 설정됨

```bash
# 카운터 확인
cat /proc/$$/status | grep -E 'voluntary'
# voluntary_ctxt_switches:    123
# nonvoluntary_ctxt_switches: 45
```

- voluntary 多 = I/O 바운드
- nonvoluntary 多 = CPU 바운드 + 경쟁 심함

### 3.3 전환 시퀀스 (Linux 기준)

```
[사용자 모드 실행 중 task A]
        │
        │  ① trap (timer IRQ / syscall / page fault)
        ▼
[커널 진입: mode switch]
   - CPU 가 RSP → 커널 스택으로 교체
   - 사용자 레지스터 일부 자동 push (IRET 프레임)
        │
        │  ② 커널이 인터럽트 처리. need_resched 검사
        ▼
[schedule() 호출]
   - ③ pick_next_task(): 런큐에서 다음 task B 선택 (CFS)
   - ④ context_switch(prev=A, next=B)
        │
        ├─ switch_mm(A->mm, B->mm):
        │     CR3 ← B->mm->pgd  (페이지 테이블 교체)
        │     → TLB flush 또는 PCID 로 보존
        │
        └─ switch_to(A, B):
              prev 의 RBP/RSP/콜리세이브 레지스터 → A->thread 저장
              B->thread → 레지스터 복원
              RSP ← B 의 커널 스택
              [여기서 RIP 가 B 가 마지막에 멈췄던 위치로 점프]
        │
        ▼
[B 의 커널 스택에서 깨어남]
   - ⑤ return-to-user 경로
   - IRET / SYSRET → 사용자 모드 복귀
        │
        ▼
[사용자 모드 task B 실행 재개]
```

핵심 지점: `switch_to` 후 **돌아오는 흐름은 next 의 과거 호출 컨텍스트**. 마치 함수가 다른 시간으로 점프한 것처럼 보임.

### 3.4 무엇이 저장·복원되는가

**저장 (caller-saved 는 trap 시점에 이미 스택에 있음):**

| 영역 | 저장 위치 |
|------|----------|
| 사용자 레지스터 (RAX~R15, RIP, RFLAGS) | trap 프레임 (커널 스택) |
| 커널 callee-saved (RBX, RBP, R12~R15, RSP) | `task->thread` |
| FPU/SSE/AVX 상태 | `task->thread.fpu` (lazy: XSAVE/XRSTOR, 필요 시점까지 미룸) |
| 세그먼트 베이스 (FS for TLS) | `task->thread.fsbase` |

**전환 시 추가 동작:**
- CR3 (페이지 디렉터리) 교체 → 가상 주소 공간 변경
- TLB 처리 (아래 §3.5)
- per-CPU `current` 포인터 업데이트
- LDT, IO bitmap 등 (드물게)

→ 모든 범용 레지스터 매번 저장하지는 않음. ABI 의 callee-saved 만 명시 저장. caller-saved 는 trap 진입 시 이미 스택에 보존됨.

### 3.5 TLB / 캐시 — 진짜 비싼 부분

직접 비용 (레지스터 교체) = 작음. 간접 비용 = 큼.

#### TLB (Translation Lookaside Buffer)

가상→물리 주소 변환 캐시. 페이지 테이블 (CR3) 바뀌면 무효화 필요.

| 상황 | TLB 처리 |
|------|---------|
| 같은 프로세스 내 스레드 전환 | mm 공유 → CR3 안 바꿈 → **TLB 보존** ✅ |
| 다른 프로세스 전환 (PCID 미사용) | CR3 교체 → 전체 flush. 이후 TLB miss 폭증 |
| **PCID/ASID** (x86 Process Context ID, ARM ASID) | 각 PCB 에 ID 부여. CR3 비트 63 = "no flush" → TLB 엔트리에 PCID 태그 → 보존 |

→ 스레드 전환 << 프로세스 전환 비용. PCID 로 격차 줄어듦.

#### CPU 캐시 (L1/L2/L3)

CR3 교체해도 캐시 자체는 물리 주소 기반 (PIPT) → 자동 flush 안 됨. 그러나:
- 새 프로세스 working set 이 캐시 라인 밀어냄 → **cache pollution**
- 코어 이동 (다른 코어로 스케줄링) = L1/L2 통째로 cold start
- 결국 수 µs ~ 수십 µs 동안 느린 메모리 접근

**브랜치 예측기, prefetcher 도 학습 무효화.** 측정 안 보이지만 IPC (instructions per cycle) 떨어뜨림.

### 3.6 비용 정리

| 항목 | 대략 |
|------|------|
| 직접 비용 (레지스터 + 스케줄러 로직) | ~1 µs |
| TLB miss 회복 (PCID 없을 때) | ~수 µs |
| 캐시 cold | ~수십 µs (워크로드 의존) |
| 코어간 마이그레이션 (NUMA cross) | 더 비쌈 |

**경험치**: 같은 코어 스레드 전환 ~1µs, 다른 프로세스 ~3-5µs, NUMA 노드 넘으면 +α.

### 3.7 측정

```bash
# 시스템 전체 컨텍스트 스위치 율
vmstat 1
# cs 컬럼 = 초당 ctx switch 수

# perf 로 직접
perf stat -e context-switches,cpu-migrations,cache-misses ./myapp

# pidstat 으로 프로세스별
pidstat -w 1

# 마이크로 벤치마크 (lmbench)
lat_ctx -P 1 2 4 8
```

코드로 1회 ctx switch 측정:
```c
// 두 프로세스 파이프 ping-pong → RTT / 2 ≈ 1 ctx switch
```

### 3.8 줄이는 방법

- **스레드 사용** (mm 공유 → TLB 보존)
- **CPU affinity** (`taskset`, `sched_setaffinity`) — 코어 고정으로 캐시 보존
- **배치 I/O** (epoll, io_uring) — 시스템 콜 묶어서 호출 횟수 감소
- **busy-wait** 짧은 락 (스핀락) — 컨텍스트 스위치 회피
- **타임 슬라이스 조정** (`sched_min_granularity_ns`) — 스루풋 우선이면 길게
- **NUMA-aware 배치** — 메모리·코어 같은 노드

### 3.9 너무 잦으면 / 너무 드물면

- **잦음**: 사용자 코드 시간 < 오버헤드. throughput 저하. 원인 = 락 경쟁, 잘못된 스레드 풀 크기, 인터럽트 폭주.
- **드묾**: 한 task 가 CPU 독점. 응답성 저하 (UI 끊김, 네트워크 timeout).

스케줄러 (CFS) 가 vruntime 으로 균형. nice / chrt 로 튜닝.

---

## 4. 프로세스 메모리 영역

프로세스 가상 주소 공간 구성 (낮은 주소 → 높은 주소):

```
┌─────────────────┐ 높은 주소
│     Stack       │ ↓ 함수 호출 시 자라남 (위→아래)
│       │         │
│       ▼         │
│                 │
│   (free)        │
│                 │
│       ▲         │
│       │         │
│      Heap       │ ↑ malloc/new 시 자라남 (아래→위)
├─────────────────┤
│   BSS (uninit)  │ 초기화 안 한 전역/static = 0
├─────────────────┤
│   Data (init)   │ 초기화 된 전역/static
├─────────────────┤
│   Code (.text)  │ 기계어, read-only
└─────────────────┘ 낮은 주소
```

### 4.1 Code 영역 (.text)

- 기계어 명령어 저장.
- **read-only**. 쓰기 시도 시 segfault.
- 같은 프로그램 여러 인스턴스 = 같은 코드 페이지 공유 가능 (메모리 절약).

### 4.2 Data 영역

- **Data (.data)**: 초기화된 전역 변수, static 변수. 예: `int g = 5;`
- **BSS (.bss)**: 초기화 안 한 전역/static. 예: `int g;` → 0 으로 채움. 바이너리 파일엔 크기만 기록 (디스크 절약).

수명: 프로그램 시작 ~ 종료.

### 4.3 Heap

- 동적 할당 영역. `malloc`, `new`, `mmap` 등.
- **아래 → 위 (낮은 주소 → 높은 주소)** 로 성장.
- 해제 (`free`, `delete`) 명시적 필요. 안 하면 leak.
- 단편화 (fragmentation) 발생 가능.

### 4.4 Stack

- 함수 호출 프레임 (지역 변수, 매개변수, 리턴 주소, 이전 BP).
- **위 → 아래 (높은 주소 → 낮은 주소)** 로 성장.
- LIFO. 함수 return 시 자동 해제.
- 크기 제한 (보통 8MB, `ulimit -s`). 초과 = **stack overflow**.

### 4.5 왜 Heap/Stack 반대로 자라나

가운데 빈 공간 공유. 한쪽만 많이 써도 다른 쪽 영역 침범 전까지 사용 가능. 충돌 = 오버플로우.

---

## 5. 프로세스 상태 (생명주기)

```
   new ──→ ready ⇄ running ──→ terminated
              ▲       │
              │       ▼
            waiting ←─┘  (I/O, event)
```

- **new**: 생성 중
- **ready**: CPU 할당 대기
- **running**: CPU 점유 실행 중
- **waiting (blocked)**: I/O 등 이벤트 대기
- **terminated**: 종료, 자원 회수 대기 (zombie 가능)

---

## 6. fork / exec / wait — 프로세스 생성 3종 세트

Unix 프로세스 생성 모델 핵심. **새 프로세스 = fork (복제) → exec (이미지 교체) → 부모 wait (회수)**.

Windows 의 `CreateProcess` 와 다름. Unix 는 "복제 후 다른 프로그램으로 변신" 2단계.

### 6.1 fork() — 프로세스 복제

```c
#include <unistd.h>
pid_t fork(void);
```

**호출 1번, 리턴 2번.** 부모와 자식 모두 fork() 다음 줄부터 실행 재개.

| 리턴 값 | 누구에게 |
|--------|---------|
| `> 0` (자식 PID) | 부모 프로세스 |
| `0` | 자식 프로세스 |
| `-1` | 실패 (에러 = 부모만 받음) |

```c
pid_t pid = fork();
if (pid < 0) {
    perror("fork");
    exit(1);
} else if (pid == 0) {
    // 자식 코드
    printf("child pid=%d, ppid=%d\n", getpid(), getppid());
    _exit(0);
} else {
    // 부모 코드
    printf("parent pid=%d, child=%d\n", getpid(), pid);
}
```

#### 자식이 부모로부터 복제하는 것

- 메모리 영역 (code, data, heap, stack) — **COW (Copy-on-Write)** 로 공유 후 분기
- 파일 디스크립터 테이블 (각 fd 의 reference count 증가, 같은 file description 가리킴)
- 시그널 핸들러
- 환경 변수, 작업 디렉터리, umask
- nice 값, affinity
- 자원 제한

#### 자식이 다르게 가지는 것

- PID, PPID
- 자원 사용량 0 으로 초기화 (utime, stime, page fault 카운터)
- 미해결 시그널 (pending) = 빈 상태
- 파일 락 상속 안 함
- `times()` 0 으로 리셋

#### COW (Copy-on-Write)

복제 시 메모리 전체 복사하면 비싸고 낭비. 대부분 자식은 곧 exec 해서 메모리 버림.

→ 페이지 테이블만 복사. 모든 페이지 read-only 마킹. 한쪽이 쓰기 시도 = page fault → 커널이 그 페이지만 복사 후 R/W 복원.

→ exec 까지 가면 거의 복사 없이 끝남. fork 비용 작음.

#### vfork() — 옛 최적화

```c
pid_t vfork(void);
```

자식이 부모 mm 공유 (COW 도 안 함). 부모는 자식이 exec/_exit 할 때까지 정지. 매우 빠르지만 위험. 현대엔 거의 안 씀 (COW 로 충분).

#### clone() — Linux 의 일반화

```c
long clone(unsigned long flags, void *stack, ...);
```

`fork`, `pthread_create` 둘 다 내부적으로 `clone` 호출. flags 로 무엇을 공유할지 세부 제어:

| flag | 효과 |
|------|------|
| `CLONE_VM` | mm 공유 (스레드 핵심) |
| `CLONE_FILES` | fd 테이블 공유 |
| `CLONE_FS` | cwd, root 공유 |
| `CLONE_SIGHAND` | 시그널 핸들러 공유 |
| `CLONE_THREAD` | 같은 스레드 그룹 (TGID) |
| `CLONE_PARENT` | 부모를 호출자의 부모로 |
| `CLONE_NEWNS/PID/NET/...` | 새 네임스페이스 (컨테이너) |

스레드 = `CLONE_VM|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|...`. 프로세스 = 거의 다 끔.

---

### 6.2 exec() — 프로세스 이미지 교체

```c
int execve(const char *path, char *const argv[], char *const envp[]);
```

**현재 프로세스의 메모리 이미지를 새 프로그램으로 덮어씀.** PID 유지. 성공 시 **리턴 안 함** (돌아갈 코드가 사라졌음). 실패 시만 `-1` 리턴.

```c
char *args[] = {"ls", "-l", "/tmp", NULL};
execve("/bin/ls", args, environ);
perror("execve");   // 여기 도달 = 실패
exit(127);
```

#### exec 후에 무엇이 남고 무엇이 사라지나

**바뀜 (새 프로그램 것):**
- 코드, 데이터, BSS, 힙, 스택 — 전부 새로 매핑
- 시그널 핸들러 → 기본값 (SIG_DFL). 단, ignored (SIG_IGN) 는 유지

**유지됨:**
- PID, PPID
- 열린 fd (단, **close-on-exec** (`FD_CLOEXEC`, `O_CLOEXEC`) 플래그 있는 것은 자동 close)
- nice, affinity, 자원 제한
- 작업 디렉터리, umask, 세션·프로세스 그룹
- 미처리 시그널 (pending), 시그널 마스크
- 자원 사용량 (utime, stime 누적)

#### exec 패밀리 6종

libc 래퍼들. 마지막 글자 = 옵션:

| 함수 | l/v | p | e |
|------|-----|---|---|
| `execl` | list (가변 인자) | - | - |
| `execlp` | list | PATH 검색 | - |
| `execle` | list | - | env 명시 |
| `execv` | vector (배열) | - | - |
| `execvp` | vector | PATH 검색 | - |
| `execvpe` | vector | PATH 검색 | env 명시 |

전부 내부적으로 `execve` 시스템 콜 호출.

```c
execlp("ls", "ls", "-l", NULL);     // PATH 검색
execv("/bin/ls", (char*[]){"ls","-l",NULL});
```

---

### 6.3 wait() — 자식 종료 회수

자식이 종료하면 → ZOMBIE 상태. 자원 (mm, files) 해제됐지만 task_struct + 종료 코드만 남음. 부모가 회수 안 하면 PID 누수.

```c
#include <sys/wait.h>
pid_t wait(int *wstatus);
pid_t waitpid(pid_t pid, int *wstatus, int options);
int   waitid(idtype_t, id_t, siginfo_t *, int options);
```

#### wait()

아무 자식이나 1개 종료 대기. 블록.

```c
int status;
pid_t child = wait(&status);
```

#### waitpid()

```c
waitpid(pid, &status, options);
```

| pid 인자 | 의미 |
|---------|------|
| `> 0` | 특정 PID 자식 |
| `0` | 같은 프로세스 그룹 자식 |
| `-1` | 모든 자식 (= `wait()` 동등) |
| `< -1` | 특정 PGID 자식 |

| options | 의미 |
|---------|------|
| `WNOHANG` | 비블록. 종료된 자식 없으면 즉시 0 리턴 (polling) |
| `WUNTRACED` | stop 된 자식도 보고 |
| `WCONTINUED` | continue 된 자식도 보고 |

#### status 매크로

```c
WIFEXITED(status)   // 정상 종료?
WEXITSTATUS(status) // exit() 인자 (하위 8비트만)
WIFSIGNALED(status) // 시그널로 종료?
WTERMSIG(status)    // 종료 시그널 번호
WCOREDUMP(status)   // core dump 했나?
WIFSTOPPED(status)  // SIGSTOP 등으로 정지?
WSTOPSIG(status)
```

```c
if (WIFEXITED(s))   printf("exit %d\n", WEXITSTATUS(s));
else if (WIFSIGNALED(s)) printf("killed by sig %d\n", WTERMSIG(s));
```

#### waitid()

POSIX, 더 깔끔한 인터페이스. `siginfo_t` 로 풍부한 정보. `WNOWAIT` (회수 안 하고 엿보기) 가능.

---

### 6.4 Zombie vs Orphan

| 용어 | 정의 | 부모 | 결과 |
|------|------|------|------|
| **Zombie** | 자식 종료, 부모 wait 안 함 | 살아있음 | task_struct 누수. PID 고갈 위험 |
| **Orphan** | 부모가 먼저 종료 | 죽음 | init (PID 1) 이 입양. init 이 주기적으로 wait → 자동 회수 |

좀비 청소 패턴:

```c
// 1) SIGCHLD 핸들러
void sigchld(int sig) {
    while (waitpid(-1, NULL, WNOHANG) > 0) ;
}
signal(SIGCHLD, sigchld);

// 2) 손주 트릭 (double fork): 자식이 또 fork 후 즉시 종료
//    → 손주는 init 의 자식이 되어 자동 회수

// 3) 시그널 무시 (Linux 한정)
signal(SIGCHLD, SIG_IGN);   // 자식 자동 회수
```

`ps aux | grep '<defunct>'` → 좀비 확인.

---

### 6.5 SIGCHLD

자식 종료/정지/재개 시 부모에게 발송. 기본 동작 = 무시. 핸들러 등록하면 비동기 회수 가능.

주의:
- 시그널 합쳐짐 (coalesce). SIGCHLD 1번에 여러 자식 종료 가능 → 핸들러에서 **루프로** `WNOHANG` waitpid.
- async-signal-safe 함수만 사용.

---

### 6.6 fork-exec-wait 패턴 (셸 동작)

```c
pid_t pid = fork();
if (pid == 0) {
    // 자식: 새 프로그램으로 변신
    execvp(argv[0], argv);
    _exit(127);              // exec 실패만 도달
}
// 부모: 자식 종료 대기
int status;
waitpid(pid, &status, 0);
```

이게 셸이 명령어 1개 실행할 때 하는 일.

파이프 (`a | b`) = fork 2번 + pipe + dup2 + exec 2번 + wait 2번.

---

### 6.7 fork 흔한 함정

#### (a) 출력 버퍼 중복

```c
printf("hi\n");      // 버퍼만 채움 (라인 버퍼링이 아닌 경우)
fork();
// → 자식·부모 둘 다 같은 버퍼 보유. 둘 다 flush → "hi\n" 두 번
```

해결: fork 전에 `fflush(stdout)`.

#### (b) 멀티스레드 + fork

`fork()` 는 **호출한 스레드만** 자식에 복제. 다른 스레드가 잡고 있던 락은 잠긴 채로 자식에 들어감 → 데드락.

→ POSIX 권고: 멀티스레드 환경에선 fork 후 즉시 exec. 그 사이엔 async-signal-safe 함수만.

#### (c) atexit 핸들러

자식도 등록된 atexit 함수 상속. 자식에서 `exit()` 부르면 부모 자원 (예: 임시 파일) 까지 지워질 수 있음.

→ 자식은 `_exit()` (atexit 안 부름) 사용 권장.

#### (d) fd 누수

부모가 connect 한 소켓을 자식이 자동 상속. 자식이 exec 해도 안 닫힘. 자식이 종료해도 부모가 fd 안 닫으면 파일/소켓 못 닫힘.

→ `O_CLOEXEC` 또는 `FD_CLOEXEC` 사용.

---

### 6.8 posix_spawn — 대안

```c
int posix_spawn(pid_t *pid, const char *path, ...);
```

fork+exec 합친 함수. 멀티스레드 환경에서 안전. 임베디드·MMU 없는 시스템에서도 동작. 셸·런타임 (Java ProcessBuilder 등) 내부 구현.

---

### 6.9 strace 로 실제 시스템 콜 보기

```bash
strace -f -e trace=clone,execve,wait4,exit_group -- bash -c 'ls'
```

bash 가 fork (= clone), execve("/bin/ls"), wait4 호출하는 흐름 그대로 보임.

---

## 7. 확장 예정 항목

- [ ] 스레드 vs 프로세스
- [ ] 가상 메모리, 페이지 테이블, MMU
- [ ] 스케줄링 알고리즘 (FCFS, SJF, RR, MLFQ)
- [ ] IPC (파이프, 시그널, 공유 메모리, 메시지 큐, 소켓)
- [ ] 컨테이너 (네임스페이스, cgroup)
