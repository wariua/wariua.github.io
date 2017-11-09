---
layout: post
title: 코루틴
category: general
tags: coroutine
---
[코루틴(coroutine)](https://en.wikipedia.org/wiki/Coroutine)은 비선점 멀티태스킹을 위해 서브루틴을 일반화 한 것...이라고 한다. 제너레이터(generator)와 가까운 친척, [고루틴(goroutine)](https://tour.golang.org/concurrency/1)과도 미묘한 친척. 핵심은 지속하는 실행 문맥(스택 + 현재 실행 위치 + 기타 레지스터 + ...)들 사이를 오가며 실행하는 것이다.

운영체제가 "그만! 이번엔 거기까지-"라며 강제로 문맥을 전환하는 식이 아니라 프로그램에서 자발적으로 다른 루틴에게 양보(`yield`)하니까 비선점이다. 운영체제에 들르지 않아도 되니 문맥 전환이 효율적이고 스케줄러가 없는 환경(bare metal 등)에서 이용할 수 있...을지도 모른다.

실행을 양보하는 즉시 실행 문맥이 사라진다는 제약을 추가하면 서브루틴이 된다. 루틴이 한 번 돌면 사라질 실행 문맥을 굳이 계속 유지할 필요가 없고, 그래서 서브루틴을 부를 때는 caller의 스택에 프레임을 하나 쌓아서 callee를 위한 임시 문맥을 만들었다가 되돌아 올 때 없앤다.

멀티태스킹이래도 결국 한 CPU를 돌려 쓰는 거니까 여러 루틴이 동시에 실행(concurrency)되기는 하지만 병렬로 실행(parallelism)되는 건 아니다. 그리고 실행 흐름 넘기는 지점을 직접 프로그래밍 해야 하니 복잡한 프로그램에서 여기저기 쓰기에는 골치 아플 수 있다.

C에도 코루틴이 있으면 좋겠지만 저수준 언어한테 그런 거 요구하면 안 된다. 하지만 코루틴이란 게 결국은 실행 문맥 만들고 전환할 수 있으면 되는 거라서 요래조래 비슷하게 만들 수는 있다.

코루틴보다 살짝 더 직관적인 게 제너레이터니까 다음 파이썬 코드를:

{% highlight python %}
def fib(n):
    a, b = 0, 1
    for i in xrange(n):
        yield a
        a, b = b, a+b

for i in fib(10):
    print i
{% endhighlight %}

C로 대강 흉내내면 이런 식이다.

{% highlight c %}
#include <stdio.h>
#include <ucontext.h>

static ucontext_t uctx_main, uctx_fib;
static char uctx_fib_stack[1024];

static struct fib_arg {
    /* in args */
    int n;

    /* out result */
    int generated;
    int ret;
} fib_arg;

static void
fib(void)
{
    int i, n;
    int a = 0, b = 1;

    fib_arg.generated = 1;

    for (i = 0; i < fib_arg.n; i++) {
        fib_arg.ret = a;
        swapcontext(&uctx_fib, &uctx_main);

        n = a + b;
        a = b;
        b = n;
    }

    fib_arg.generated = 0;
}

int
main(void)
{
    getcontext(&uctx_fib);
    uctx_fib.uc_stack.ss_sp = uctx_fib_stack;
    uctx_fib.uc_stack.ss_size = sizeof(uctx_fib_stack);
    uctx_fib.uc_link = &uctx_main;
    makecontext(&uctx_fib, fib, 0);

    fib_arg.n = 10;
    while (swapcontext(&uctx_main, &uctx_fib),
           fib_arg.generated)
        printf("%d\n", fib_arg.ret);

    return 0;
}
{% endhighlight %}

너저분하다. 캡슐화 잘 하면 좀 더 깔끔해지겠지만 이식성 우려 때문에 POSIX에서 퇴출된 함수를 가지고 그렇게 애쓸 필요까지야.

개념적으로 [makecontext()](https://github.com/wariua/manpages-ko/wiki/makecontext%283%29)/[setcontext()](https://github.com/wariua/manpages-ko/wiki/getcontext%283%29)는 [setjmp()/longjmp()](https://github.com/wariua/manpages-ko/wiki/setjmp%283%29)를 일반화 한 거라고 볼 수도 있다. 스택을 되감는 쪽으로만 문맥을 바꿀 수 있느냐 더 자유롭게 전환할 수 있느냐의 차이. 핵심은 결국 주요 레지스터들을 저장하거나 복원하는 거라서 [glibc의 소스 코드](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/unix/sysv/linux/i386/swapcontext.S)를 봐도 꽤 짧고 단순하다.
