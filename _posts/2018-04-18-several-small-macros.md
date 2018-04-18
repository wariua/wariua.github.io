---
layout: post
title: 자잘한 매크로들
category: general
---
여느 대형 프로그램과 마찬가지로 리눅스 커널에서는 코딩 편의성과 코드 가독성을 위해 매크로를 많이 활용한다. 기발해 보이는 작은 매크로(주의: 웃음 포인트)들을 옮겨 보자면,

## `SYSCALL_DEFINEn()`

이름 그대로 시스템 호출 핸들러를 정의할 때 쓰는 매크로다. `n`은 매개변수 개수다.

`linux/kernel/bpf/syscall.c`:
```c
SYSCALL_DEFINE3(bpf, int, cmd, union bpf_attr __user *, uattr, unsigned int size)
{
        union bpf_attr attr = {};
        int err;

        if (sysctl_unprivilieged_bpf_disabled && !capable(CAP_SYS_ADMIN))
                return -EPERM;

        err = check_uarg_tail_zero(uattr, sizeof(attr)
        if (err)
                return err;
        size = min_t(u32, size, sizeof(attr));

        ...

        return err;
}
```

매크로 정의는 이렇다.

`linux/include/linux/syscalls.h`:
```c
/*
 * __MAP - apply a macro to syscall arguments
 * __MAP(n, m, t1, a1, t2, a2, ..., tn, an) will expand to
 *    m(t1, a1), m(t2, a2), ..., m(tn, an)
 * The first argument must be equal to the amount of type/name
 * pairs given.  Note that this list of pairs (i.e. the arguments
 * of __MAP starting at the third one) is in the same format as
 * for SYSCALL_DEFINE<n>/COMPAT_SYSCALL_DEFINE<n>
 */
#define __MAP0(m,...)
#define __MAP1(m,t,a) m(t,a)
#define __MAP2(m,t,a,...) m(t,a), __MAP1(m,__VA_ARGS__)
#define __MAP3(m,t,a,...) m(t,a), __MAP2(m,__VA_ARGS__)
#define __MAP4(m,t,a,...) m(t,a), __MAP3(m,__VA_ARGS__)
#define __MAP5(m,t,a,...) m(t,a), __MAP4(m,__VA_ARGS__)
#define __MAP6(m,t,a,...) m(t,a), __MAP5(m,__VA_ARGS__)
#define __MAP(n,...) __MAP##n(__VA_ARGS__)

#define __SC_DECL(t, a) t a
#define __TYPE_AS(t, v) __same_type((__force t)0, v)
#define __TYPE_IS_L(t)  (__TYPE_AS(t, 0L))
#define __TYPE_IS_UL(t) (__TYPE_AS(t, 0UL))
#define __TYPE_IS_LL(t) (__TYPE_AS(t, 0LL) || __TYPE_AS(t, 0ULL))
#define __SC_LONG(t, a) __typeof(__builtin_choose_expr(__TYPE_IS_LL(t), 0LL, 0L)) a
#define __SC_CAST(t, a) (__force t) a
#define __SC_ARGS(t, a) a
#define __SC_TEST(t, a) (void)BUILD_BUG_ON_ZERO(!__TYPE_IS_LL(t) && sizeof(t) > sizeof(long))

...

#define SYSCALL_METADATA(sname, nb, ...)        /* 생략 */

...

#define SYSCALL_DEFINE0(sname)                                  \
        SYSCALL_METADATA(_##sname, 0);                          \
        asmlinkage long sys_##sname(void)

#define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE5(name, ...) SYSCALL_DEFINEx(5, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE6(name, ...) SYSCALL_DEFINEx(6, _##name, __VA_ARGS__)

#define SYSCALL_DEFINE_MAXARGS  6

#define SYSCALL_DEFINEx(x, sname, ...)                          \
        SYSCALL_METADATA(sname, x, __VA_ARGS__)                 \
        __SYSCALL_DEFINEx(x, sname, __VA_ARGS__)

#define __PROTECT(...) asmlinkage_protect(__VA_ARGS__)
#define __SYSCALL_DEFINEx(x, name, ...)                                 \
        asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))       \
                __attribute__((alias(__stringify(SyS##name))));         \
        static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__));  \
        asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__));      \
        asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__))       \
        {                                                               \
                long ret = SYSC##name(__MAP(x,__SC_CAST,__VA_ARGS__));  \
                __MAP(x,__SC_TEST,__VA_ARGS__);                         \
                __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));       \
                return ret;                                             \
        }                                                               \
        static inline SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```

다채로운 정찬이다. 게다가 맵이라니...

* `__VA_ARGS__`: [가변 인자 매크로](https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html)
* `__force` (`__attribute__((force))`): [man sparse(1)](https://linux.die.net/man/1/sparse). 정적 분석기에게 주는 힌트.
* `__same_type`: [내장 함수 __builtin_types_compatible_p](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#index-_005f_005fbuiltin_005ftypes_005fcompatible_005fp). 호환되는 타입인가?
* `__builtin_choose_expr()`: [내장 함수 __builtin_choose_expr](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#index-_005f_005fbuiltin_005fchoose_005fexpr). 삼항 연산자 `?:`와 비슷.
* `__attribute__((alias()))`: [alias 속성](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html#index-alias-function-attribute). 별명 붙이기.
* `__stringify()`: `linux/include/linux/stringify.h`

  ```c
  /* Indirect stringification.  Doing two levels allows the parameter to be a
   * macro iteself.  For example, compile with -DFOO=bar, __stringify(FOO)
   * converts to "bar".
   */
  
  #define __stringify_1(x...)     #x
  #define __stringify(x...)       __stringify_1(x)
  ```

인자 타입 검사를 하는 `__SC_TEST()` 매크로 정의에는 `BUILD_BUG_ON_ZERO()`라는 게 있다. 음... 제로의 영역에서 버그 만들기?

## `BUILD_BUG_ON*()`

컴파일 타임 `assert()`다.

`linux/include/linux/build_bug.h`:
```c
/*
 * Force a compilation error if condition is true, but also produce a
 * result (of value 0 and type size_t), so the expression can be used
 * e.g. in a structure initializer (or where-ever else comma expressions
 * aren't permitted).
 */
#define BUILD_BUG_ON_ZERO(e) (sizeof(struct { int:(-!!(e)); }))

...
#define BUILD_BUG_ON_MSG(cond, msg) compiletime_assert(!(cond), msg)

/**
 * BUILD_BUG_ON - break compile if a condition is true.
 * @condition: the condition which the compiler should know is false.
 *
 * ...
 */
#ifndef __OPTIMIZE__
#define BUILD_BUG_ON(condition) ((void)sizeof(char[1 - 2*!!(condition)]))
#else
#define BUILD_BUG_ON(condition) \
        BUILD_BUG_ON_MSG(condition, "BUILD_BUG_ON failed: " #condition)
#endif
```

멋지다. 근데 오류 메시지를 찍자면 결국 `compiletime_assert()`를 써야 된다.

`linux/include/linux/compiler.h`:
```c
#ifndef __compiletime_error
# define __compiletime_error(message)
/*
 * Sparse complains of variable sized arrays due to the temporary variable in
 * __compiletime_assert. Unfortunately we can't just expand it out to make
 * sparse see a constant array size without breaking compiletime_assert on old
 * versions of GCC (e.g. 4.2.4), so hide the array from sparse altogether.
 */
# ifndef __CHECKER__
#  define __compiletime_error_fallback(condition) \
        do { ((void)sizeof(char[1 - 2 * condition])); } while (0)
# endif
#endif
#ifndef __compiletime_error_fallback
# define __compiletime_error_fallback(condition) do { } while (0)
#endif

#ifdef __OPTIMIZE__
# define __compiletime_assert(condition, msg, prefix, suffix)           \
        do {                                                            \
                bool __cond = !(condition);                             \
                extern void prefix ## suffix(void) __compiletime_error(msg); \
                if (__cond)                                             \
                        prefix ## suffix();                             \
                __compiletime_error_fallback(__cond);                   \
        } while (0)
#else
# define __compiletime_assert(condition, msg, prefix, suffix) do { } while (0)
#endif

#define _compiletime_assert(condition, msg, prefix, suffix) \
        __compiletime_assert(condition, msg, prefix, suffix)

/**
 * compiletime_assert - break build and emit msg if condition is false
 * @condition: a compile-time constant condition to check
 * @msg:       a message to emit if condition is false
 *
 * ...
 */
#define compiletime_assert(condition, msg) \
        _compiletime_assert(condition, msg, __compiletime_assert_, __LINE__)
```

`linux/include/linux/compiler-gcc.h`:
```c
#ifndef __CHECKER__
# define __compiletime_warning(message) __attribute__((warning(message)))
# define __compiletime_error(message) __attribute__((error(message)))
#endif /* __CHECKER__ */
```

`#ifdef`이 난무하는데, 결국은 `error` 속성 사용 방식이 우선이라는 얘기다.

* `__attribute__((error()))`: [error 속성](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html#index-error-function-attribute). 호출하면 컴파일 오류.

`BUILD_BUG_ON()`에는 컴파일 타임에 상수로 평가되는 식을 써야 된다. 근데 커널 구성 옵션 등에 따라 상수 식이 될 수도 있고 아닐 수도 있는 식이 있을 수 있다.

`linux/include/linux/bug.h`:
```c
#define MAYBE_BUILD_BUG_ON(cond)                        \
        do {                                            \
                if (__builtin_constant_p((cond)))       \
                        BUILD_BUG_ON(cond);             \
                else                                    \
                        BUG_ON(cond);                   \
        } while (0)
```

* `__builtin_constant_p(...)`: [내장 함수 __builtin_constant_p](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#index-_005f_005fbuiltin_005fconstant_005fp). 상수인가?

## `IS_ENABLED()`

`#define`이 있으면 `#ifdef`도 있다. `#ifdef`을 많이 쓰는 경우가 커널 구성 옵션 확인할 때다.

`linux/include/net/net_namespace.h`:
```c
struct net {
        ...
#if defined(CONFIG_IP_SCTP) || defined(CONFIG_IP_SCTP_MODULE)
        struct netns_sctp       sctp;
#endif
#if defined(CONFIG_IP_DCCP) || defined(CONFIG_IP_DCCP_MODULE)
        struct netns_dccp       dccp;
#endif
        ...
} __randomize_layout;
```

* `__randomize_layout`: [구조체 필드 배치 무작위화](https://lwn.net/Articles/722293/), [randomize_layout_plugin.c](https://github.com/torvalds/linux/blob/master/scripts/gcc-plugins/randomize_layout_plugin.c), [designated_init 속성](https://gcc.gnu.org/onlinedocs/gcc/Common-Type-Attributes.html#index-designated_005finit-type-attribute)

모듈 사용 여부에 의존적인 코드마다 `#ifdef ... #endif` 쓰는 거, 지겨운 데다 실수하기도 쉽다. 다른 방법 없냐고 하면, 있다.

`linux/net/ipv4/tcp_input.c`:
```c
static void tcp_openreq_init(...)
{
        struct inet_request_sock *ireq = inet_rsk(req);

        ...

        ireq->ir_mark = inet_request_mark(sk, skb);
#if IS_ENABLED(CONFIG_SMC)
        ireq->smc_ok = rx_opt->smc_ok;
#endif
}
```

일단 짧아졌다. 그리고 이런 것도 된다.

`linux/net/ipv4/tcp_input.c`:
```c
int tcp_conn_request(...)
{
        ...

        if (want_cookie && !tmp_opt.saw_tstamp)
                tcp_clear_options(&tmp_opt);

        if (IS_ENABLED(CONFIG_SMC) && want_cookie)
                tmp_opt.smc_ok = 0;

        ...
}
```

커널 구성 항목 심볼(`CONFIG_SMC`, `CONFIG_SMC_MODULE`)은 정의돼 있지 않거나 1로 정의돼 있다. 각 경우에 0과 1로 평가되는 매크로가 필요하다. 그리고 내부적으로 `||` 처리도 필요하다.

`linux/include/linux/kconfig.h`:
```c
#define __ARG_PLACEHOLDER_1 0,
#define __take_second_arg(__ignored, val, ...) val

#define __and(x, y)                     ___and(x, y)
#define ___and(x, y)                    ____and(__ARG_PLACEHOLDER_##x, y)
#define ____and(arg1_or_junk, y)        __take_second_arg(arg1_or_junk y, 0)

#define __or(x, y)                      ___or(x, y)
#define ___or(x, y)                     ____or(__ARG_PLACEHOLDER_##x, y)
#define ____or(arg1_or_junk, y)         __take_second_arg(arg1_or_junk 1, y)

#define __is_defined(x)                 ___is_defined(x)
#define ___is_defined(val)              ____is_defined(__ARG_PLACEHOLDER_##val)
#define ____is_defined(arg1_or_junk)    __take_second_arg(arg1_or_junk 1, 0)

#define IS_BUILTIN(option) __is_defined(option)

#define IS_MODULE(option) __is_defined(option##_MODULE)

#define IS_REACHABLE(option) __or(IS_BUILTIN(option), \
                                __and(IS_MODULE(option), __is_defined(MODULE)))

#define IS_ENABLED(option) __or(IS_BUILTIN(option), IS_MODULE(option))
```

평소에 뭘 먹으면 이런 생각을 할 수 있게 되는 걸까.

이 방식의 또 다른 장점은 비활성화 코드도 일단 컴파일은 하고서 제거한다는 점이다. 즉 이 경우 저 경우 따로 컴파일 하지 않아도 컴파일 오류 정도는 잡을 수 있다.

복잡한 매크로들을 봤으니 이제 진짜로 작고 귀여운 걸 봐야겠다.

## 가변 인자

[가변 인자 매크로](https://en.wikipedia.org/wiki/Variadic_macro)를 이용해 `printk()` 비슷한 뭔가를 만들 때는 꼭 콜론이 문제가 된다.

```
$ cpp << EOF
> #define dprintf(fmt, ...) printf("DBG: " fmt, __VA_ARGS__)
> 
> dprintf("val=%d\n", val);
> dprintf("fmt-only\n");
> EOF
# 1 "<stdin>"
# 1 "<built-in>"
# 1 "<command-line>"
# 31 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 32 "<command-line>" 2
# 1 "<stdin>"


printf("DBG: " "val=%d\n", val);
printf("DBG: " "fmt-only\n", );
```

게다가 컴파일러에 따라선 `dprintf("fmt-only\n", );`라고 해야 매크로가 확장되기도 한다. 복잡하다. 특별한 이유 없으면 [가변 인자](https://github.com/wariua/manpages-ko/wiki/stdarg%283%29) 인라인 함수가 답이다.

리눅스 커널에서는 [gcc 확장](https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html)(`##__VA_ARGS__`)을 쓴다.

`linux/include/linux/printk.h`:
```c
#ifndef pr_fmt
#define pr_fmt(fmt) fmt
#endif

#define pr_emerg(fmt, ...) \
        printk(KERN_EMERG pr_fmt(fmt), ##__VA_ARGS__)
```

나중에는 [\_\_VA_OPT\_\_](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0306r2.html)라는 걸 쓰게 될지도 모르겠다.

## 한 번만

좀 더 아래를 보면 재밌어 보이는 매크로가 있다.

`linux/include/linux/printk.h`:
```c
#define printk_once(fmt, ...)                                   \
({                                                              \
        static bool __print_once __read_mostly;                 \
        bool __ret_print_once = !__print_once;                  \
                                                                \
        if (!__print_once) {                                    \
                __print_once = true;                            \
                printk(fmt, ##__VA_ARGS__);                     \
        }                                                       \
        unlikely(__ret_print_once);                             \
})
```

말하자면 <tt>[pthread_once()](https://github.com/wariua/manpages-ko/wiki/pthread_once%283p%29)</tt> 같은 건데, 어쩌다 여러 번 찍는다고 큰일 나는 건 아니니까 설렁설렁 구현해 놨다. 제대로 하자면 락을 쓰면 된다. 그런데 실행 여부를 나타내는 플래그(위의 `__print_once`) 검사에 드는 CPU 클럭이 아까울 수도 있다.

`linux/include/linux/once.h`:
```c
#define DO_ONCE(func, ...)                                                   \
        ({                                                                   \
                bool ___ret = false;                                         \
                static bool ___done = false;                                 \
                static DEFINE_STATIC_KEY_TRUE(___once_key);                  \
                if (static_branch_unlikely(&___once_key)) {                  \
                        unsigned long ___flags;                              \
                        ___ret = __do_once_start(&___done, &___flags);       \
                        if (unlikely(___ret)) {                              \
                                func(__VA_ARGS__);                           \
                                __do_once_done(&___done, &___once_key,       \
                                               &___flags);                   \
                        }                                                    \
                }                                                            \
                ___ret;                                                      \
        })

#define get_random_once(buf, nbytes)                                         \
        DO_ONCE(get_random_bytes, (buf), (nbytes))
```

기본 구조는 비슷하다. 그리고 쪼잔한 클럭 절약 방법은 [정적 키]({{ site.baseurl }}{% post_url 2018-01-15-labels-and-static-key %}) 사용이다.

## 이름 바꾸기

`__RENAME()`이라는 간단한 매크로가 하나 있다.

`linux/include/linux/string.h`:
```c
#ifndef __HAVE_ARCH_STRNLEN
extern __kernel_size_t strnlen(const char *,__kernel_size_t);
#endif
...

#define __FORTIFY_INLINE extern __always_inline __attribute__((gnu_inline))
#define __RENAME(x) __asm__(#x)

void fortify_panic(const char *name) __noreturn __cold;
...

#if !defined(__NO_FORTIFY) && defined(__OPTIMIZE__) && defined(CONFIG_FORTIFY_SOURCE)
...

extern __kernel_size_t __real_strnlen(const char *, __kernel_size_t) __RENAME(strnlen);
__FORTIFY_INLINE __kernel_size_t strnlen(const char *p, __kernel_size_t maxlen)
{
        size_t p_size = __builtin_object_size(p, 0);
        __kernel_size_t ret = __real_strnlen(p, maxlen < p_size ? maxlen : p_size);
        if (p_size <= ret && maxlen != ret)
                fortify_panic(__func__);
        return ret;
}

...

#endif
```

* `__attribute__((gnu_inline))`: [gnu_inline 속성](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html#index-gnu_005finline-function-attribute). 절대 단독 함수로 컴파일 하지 않음.
* `__builtin_object_size(...)`: [내장 함수 __builtin_object_size](https://gcc.gnu.org/onlinedocs/gcc/Object-Size-Checking.html#index-_005f_005fbuiltin_005fobject_005fsize-1).

매크로를 확장해 보자.

```c
extern __kernel_size_t __real_strnlen(const char *, __kernel_size_t) __asm__("strnlen");
```

함수 선언 뒤에 `__asm__`이 붙고 그 안에는 함수 이름만 덜렁 있다. 그리고 커널 소스에는 `__real_strnlen()`이라는 함수가 없고 대신 `strnlen()` 구현이 (또!) 있다.

`linux/lib/string.c`:
```c
#ifndef __HAVE_ARCH_STRNLEN
/**
 * ...
 */
size_t strnlen(const char *s, size_t count)
{
        const char *sc;

        for (sc = s; count-- && *sc != '\0'; ++sc)
                /* nothing */;
        return sc - s;
}
EXPORT_SYMBOL(strnlen);
#endif
```

특이한 문법이 대부분 그렇듯 컴파일러 확장이다. 어셈블러 코드에서 쓸 [변수/함수 이름을 지정](https://gcc.gnu.org/onlinedocs/gcc/Asm-Labels.html)하는 방법이다. 즉 컴파일 중에는 `__real_strnlen`이라는 이름을 쓰다가 어셈블러 코드를 생성할 때는 `strnlen`를 쓰는 식이다.

```
$ cat onetwo.c
int one asm("two") = 2;

int two(void) asm("one");

int two(void)
{
        return 1;
}
$ gcc -S -O2 onetwo.c -o -
        .file   "onetwo.c"
        .text
        .p2align 4,,15
        .globl  one
        .type   one, @function
one:
.LFB0:
        .cfi_startproc
        movl    $1, %eax
        ret
        .cfi_endproc
.LFE0:
        .size   one, .-one
        .globl  two
        .data
        .align 4
        .type   two, @object
        .size   two, 4
two:
        .long   2
        .ident  "GCC: (Ubuntu 7.2.0-8ubuntu3.2) 7.2.0"
        .section        .note.GNU-stack,"",@progbits
```

비슷한 문법을 써서 [레지스터를 특정 변수 전용으로 고정](https://gcc.gnu.org/onlinedocs/gcc/Explicit-Register-Variables.html)할 수도 있다. 아키텍처 의존성이 있긴 하지만 유용할 때가 가끔은 있을 것 같다.
