---
layout: post
title: 레이블, 정적 키
category: general
tags: [gcc, perf]
---
리눅스 커널에서는 실행 흐름 단순화를 위해 다음 코드 패턴을 종종 사용한다.

```c
int do_something(...)
{
        int err;

        lock_something();

        err = do_step_1();
        if (err)
                goto out;

        err = do_step_2();
        if (err)
                goto err_step_2;

        goto out;

err_step_2:
        undo_step_1();
out:
        unlock_something();
        return err;
}
```

하지만 일반 코드에서는 `goto`를 잘 쓰지 않고, 그래서 레이블을 만나기도 쉽지 않다. 하지만 만나려고 하면 또 얼마든 만날 수 있다.

```
$ gcc -S -Os -xc -o - - << EOF
int toeven(int n) { if (n & 1) n <<= 1; return n; }
EOF                                   

        .file   ""
        .text
        .globl  toeven
        .type   toeven, @function
toeven:
.LFB0:
        .cfi_startproc
        movl    %edi, %eax
        testb   $1, %al
        je      .L2
        addl    %eax, %eax
.L2:
        ret
        .cfi_endproc
.LFE0:
        .size   toeven, .-toeven
        .ident  "GCC: (Ubuntu 7.2.0-8ubuntu3) 7.2.0"
        .section        .note.GNU-stack,"",@progbits
```

## 레이블 확장

C 언어를 [온갖 방식으로 확장](https://gcc.gnu.org/onlinedocs/gcc/C-Extensions.html)하는 GCC에서 레이블이라고 그냥 놔뒀을 리 없다.

리눅스 커널에 이런 코드가 있다.

`linux/include/linux/kernel.h`:
```c
#define _THIS_IP_  ({ __label__ __here; __here: (unsigned long)&&__here; })
```

한 줄에서 GNU 확장 세 가지를 사용하고 있다. [문으로 식 만들기](https://gcc.gnu.org/onlinedocs/gcc/Statement-Exprs.html)야 워낙 자주 쓰이는 것이고, `__label__`과 `&&`가 눈에 띈다.

`__label__`은 레이블을 '선언'하기 위한 키워드다. 로컬 변수 선언하듯 [로컬 레이블](https://gcc.gnu.org/onlinedocs/gcc/Local-Labels.html)을 선언한다. 매크로 안에서 레이블을 사용하면 매크로 호출 코드의 레이블과 충돌할 가능성이 있는데, 로컬 레이블 선언으로 충돌을 피할 수 있다. [함수 속 함수](https://gcc.gnu.org/onlinedocs/gcc/Nested-Functions.html)에서 바깥 함수로 `goto` 할 때도 사용한다.

객체에 `&`를 붙이면 객체의 주소이듯 레이블에 `&&`를 붙이면 [레이블의 주소](https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html)가 된다. 결과를 `void *` 타입 변수에 저장했다가 `goto *ptr;` 식으로 사용할 수도 있다. 배열에 저장할 수도 있으니 상태 머신에서 점프 테이블로 사용하기에 좋다.

그렇게 해서 `_THIS_IP_`는 로컬 레이블 `__here`의 주소, 즉 이 매크로를 호출한 지점의 주소가 된다. 이름 그대로 인스트럭션 포인터 값이다.

```
$ gcc -S -O2 -xc -o - - << EOF
#define _THIS_IP_ ({ __label__ __here; __here: &&__here; })
void *here(void) { return _THIS_IP_; }
EOF

	.file	""
	.text
	.p2align 4,,15
	.globl	here
	.type	here, @function
here:
.LFB0:
	.cfi_startproc
.L2:
	leaq	.L2(%rip), %rax
	ret
	.cfi_endproc
.LFE0:
	.size	here, .-here
	.ident	"GCC: (Ubuntu 7.2.0-8ubuntu3) 7.2.0"
	.section	.note.GNU-stack,"",@progbits
```

위 코드에는 등장하지 않지만 레이블 관련 확장이 한 가지 더 있다. 레이블에도 속성(`__attribute__(...)`)을 붙일 수 있다. `unused`를 붙여서 "label '...' defined but not used" 경고를 없앨 수 있고 `hot`/`cold`로 `__builtin_expect()`(`likely()`/`unlikely()`) 효과를 줄 수도 있다.

그런데 [레이블 속성 설명](https://gcc.gnu.org/onlinedocs/gcc/Label-Attributes.html)의 예시 코드를 보면 `asm goto`라는 게 등장한다.

```c
   asm goto ("some asm" : : : : NoError);

/* This branch (the fall-through from the asm) is less commonly used */
ErrorHandling:
   __attribute__((cold,unused)); /* Semi-colon is required here */
   printf("error\n");
   return 0;

NoError:
   printf("no error\n");
   return 1;
```

`asm goto()`는 GCC의 [어셈블리어 확장](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html)인 `asm()`을 다시 확장한 것이다. 끝에 추가된 매개변수가 레이블 목록인데, 실행 중에 그 중 한 곳으로 점프할 수도 있다고 컴파일러에게 알려주는 역할을 한다.

리눅스 커널에서 `asm goto`를 사용하는 곳이 두어 곳 있는데, 그 중 하나가 정적 키이다.

## 정적 키

커널 소스 [Documentation/static-keys.txt](https://wariua.cafe24.com/wiki/Documentation/static-keys.txt)에 설명이 있다. 간단히 말해 특정 조건에서만 (가령 모니터링을 켰을 때만) 실행하는 루틴으로 향하는 브랜치 인스트럭션을 런타임에 바꿔치기하는 것이다. 평상시엔 브랜치 인스트럭션 자리에 no-op를 채워 두며, 그래서 실행이 직선으로 이어진다. 그러다가 런타임에 키 값을 바꾸면 (즉 '키를 돌리면') 그 키를 사용하는 지점들을 모두 브랜치 수행 인스트럭션으로 바꾼다. 즉, 런타임에 메모리 상의 코드를 수정한다.

예를 들어 `perf` 프로그램으로 정보를 수집하는 동안에만 실행해야 하는 루틴이 있다. 평상시에는 그 루틴이 없는 것처럼 돌다가 시스템 호출 등을 통해 동작을 바꾼다.

`linux/kernel/events/core.c`:
```c
DEFINE_STATIC_KEY_FALSE(perf_sched_events);
...

static void account_event(struct perf_events *events)
{
        bool inc = false;

        if (events->parent)
                return;

        if (events->attach_state & PERF_ATTACH_TASK)
                inc = true;
        ...

        if (inc) {
                if (atomic_inc_not_zero(&perf_sched_count))
                        goto enabled;

                mutex_lock(&perf_sched_count);
                if (!atomic_read(&perf_sched_count)) {
                        static_branch_enable(&perf_sched_events);
                        ...
                }
                ...
                mutex_unlock(&perf_sched_mutex);
        }
        ...
}

static void perf_sched_delayed(struct work_struct *work)
{
        mutex_lock(&perf_sched_mutex);
        if (atomic_dec_and_test(&perf_sched_count))
                static_branch_disable(&perf_sched_events);
        mutex_unlock(&perf_sched_mutex);
}
```

그리고 스케줄러에서 문맥 전환 때마다 다음 두 함수를 호출한다. `perf`와 관련된 복잡한 내용은 `__perf_event_task_sched_{in,out}()` 안에 들어있다.

`linux/include/linux/perf_event.h`:
```c
static inline void perf_event_task_sched_in(struct task_struct *prev,
                                            struct task_struct *task)
{
        if (static_branch_unlikely(&perf_sched_events))
                __perf_event_task_sched_in(prev, task);
        ...
}

static inline void perf_event_task_sched_out(struct task_struct *prev,
                                             struct task_struct *next)
{
        perf_sw_event_sched(PERF_COUNT_SW_CONTEXT_SWITCHES, 1, 0);

        if (static_branch_unlikely(&perf_sched_events))
                __perf_event_task_sched_out(prev, next);
}
```

키의 기본 값과 likely/unlikely의 조합에 따라 어떤 인스트럭션 구조가 생성되는지를 [linux/include/linux/jump_label.h](https://github.com/torvalds/linux/blob/v4.14/include/linux/jump_label.h#L326)에서 보여 준다. 보통은 true/likely 조합과 false/unlikely 조합을 사용하게 된다.

네트워크 서브시스템에서도 넷필터를 포함한 몇몇 곳에서 정적 키를 사용한다. 패킷 처리 초반에도 (구식 API지만) 사용 코드가 있다.

`linux/net/core/dev.c`:
```c
static struct static_key generic_xdp_needed __read_mostly;

static int generic_xdp_install(struct net_device *dev, struct netdev_xdp *xdp)
{
        ...
        switch(xdp->command) {
        case XDP_SETUP_PROG:
                rcu_assign_pointer(dev->xdp_prog, new);
                if (old)
                        bpf_prog_put(old);

                if (old && !new) {
                        static_key_slow_dec(&generic_xdp_needed);
                } else if (new && !old) {
                        static_key_slow_inc(&generic_xdp_needed);
                        dev_disable_lro(dev);
                }
                break;
        ...
        }

        return ret;
}

static int netif_receive_skb_internal(struct sk_buff *skb)
{
        ...

        if (static_key_false(&generic_xdp_needed)) {
                ...
                ret = do_xdp_generic(rcu_dereference(skb->dev->xdp_prog), skb);
                ...

                if (ret != XDP_PASS)
                        return NET_RX_DROP;
        }
        ...
}
```

네트워크 장치에 어떤 BPF 프로그램을 설치하면 패킷 수신 처리 초반에 그 프로그램을 실행한다. 즉 대체 스택을 만들 수 있는 것이다. BPF와 XDP는 조만간 다른 글에서 다시 살펴본다.

커널의 정적 키 코드 일부를 복붙하면 다음과 같다. x86-64 기준이다.

```c
typedef int bool;
enum { false, true };


static inline bool arch_static_branch(bool *key, bool branch)
{
        asm goto("1:"
                ".byte 0x0f,0x1f,0x44,0x00,0 \n\t"
                ".pushsection __jump_table,  \"aw\" \n\t"
                ".balign 8 \n\t"
                ".quad 1b, %l[l_yes], %c0 + %c1 \n\t"
                ".popsection \n\t"
                : : "i" (key), "i" (branch) : : l_yes);

        return false;
l_yes:
        return true;
}

#define static_branch_unlikely(key) \
        ({ bool branch = arch_static_branch(&(key), false); branch; })


bool tell_a_lie = false;

int toeven(int n)
{
        if (n & 1)
                n <<= 1;

        if (static_branch_unlikely(tell_a_lie))
                return 3;

        return n;
}
```

`arch_static_branch()`의 `asm goto` 코드는 no-op 인스트럭션(`0x0f,0x1f,...`)을 생성하고 나서 `__jump_table`이라는 섹션에 레이블 `1:`의 주소, 레이블 `l_yes:`의 주소, `key`의 주소(+`branch` 값)을 추가한다. 이 세 값이 있으면 양방향으로 인스트럭션을 바꿀 수 있다.

컴파일 해서 파일 내용을 살펴보자.

```
$ gcc toeven.c -c -Wall -O2

$ objdump -D -j .text toeven.o

toeven.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <toeven>:
   0:   89 f8                   mov    %edi,%eax
   2:   a8 01                   test   $0x1,%al
   4:   74 02                   je     8 <toeven+0x8>
   6:   01 c0                   add    %eax,%eax
   8:   0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)
   d:   c3                      retq   
   e:   b8 03 00 00 00          mov    $0x3,%eax
  13:   c3                      retq   

$ objdump -s -j __jump_table toeven.o

toeven.o:     file format elf64-x86-64

Contents of section __jump_table:
 0000 00000000 00000000 00000000 00000000  ................
 0010 00000000 00000000                    ........        
```

어랏, 점프 테이블에 내용이 없다! 링크 때 재배치 해야 하는 주소 값들이기 때문이다.

```
$ objdump -r toeven.o

toeven.o:     file format elf64-x86-64

RELOCATION RECORDS FOR [__jump_table]:
OFFSET           TYPE              VALUE 
0000000000000000 R_X86_64_64       .text+0x0000000000000008
0000000000000008 R_X86_64_64       .text+0x000000000000000e
0000000000000010 R_X86_64_64       tell_a_lie


RELOCATION RECORDS FOR [.eh_frame]:
OFFSET           TYPE              VALUE 
0000000000000020 R_X86_64_PC32     .text
```

.text 세그먼트의 오프셋 0x08과 0x0e를 위의 디스어셈블 코드에서 확인해 보면 각각 브랜치 위치와 점프 목적지인 걸 확인할 수 있다. 하지만 런타임에 0x08에 있는 인스트럭션을 바꾸는 것까지 따라해 볼 수는 없다. 텍스트 세그먼트에 쓰기를 하는 건 커널이나 되니까 가능한 일이다. 췟.

----

2018-02-24:

**정정**: 사용자 공간에서도 코드 영역을 쓰기 가능하게 바꿀 수 있고, 그래서 [정적 키를 사용자 공간에서 구현]({{ site.baseurl }}{% post_url 2018-02-24-static-key-in-userspace %})할 수 있다.
