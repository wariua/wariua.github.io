---
layout: post
title: 정적 키 in userland
category: facility
---
[정적 키 소개 글]({{ site.baseurl }}{% post_url 2018-01-15-labels-and-static-key %})의 마무리는 이랬다.

> .text 세그먼트의 오프셋 0x08과 0x0e를 위의 디스어셈블 코드에서 확인해 보면 각각 브랜치 위치와 점프 목적지인 걸 확인할 수 있다. 하지만 런타임에 0x08에 있는 인스트럭션을 바꾸는 것까지 따라해 볼 수는 없다. 텍스트 세그먼트에 쓰기를 하는 건 커널이나 되니까 가능한 일이다. 췟.

근데 정말 불가능할까?

[man mprotect(2)](https://github.com/wariua/manpages-ko/wiki/mprotect%282%29):

> ## CONFORMING TO
>
> `mprotect()`: POSIX.1-2001, POSIX.1-2008, SVr4. POSIX에서는 `mmap(2)`을 통해 얻은 것이 아닌 메모리 영역에 적용 시 `mprotect()`의 동작 방식이 명세되어 있지 않다고 한다.
>
> ...
>
> ## NOTES
>
> 리눅스에서는 프로세스 주소 공간 내의 (커널 vsyscall 영역을 제외한) 어느 주소에도 `mprotect()` 호출을 항상 허용한다. 특히 이를 이용해 기존의 코드 매핑을 쓰기 가능으로 바꿀 수 있다.

즉, 플랫폼에 달려 있는데 리눅스에서는 가능하다. 그렇다면 지난 글에서 함께했던 `toeven.c`를 데리고 좀 더 멀리까지 가 볼 수 있다. 브랜치 인스트럭션 패치 루틴(`static_key_enable()`)을 작성해 보자.

더 쌓기 전에 정리부터. 일단 코드를 정적 키 모듈(`static-key.[ch]`), 정적 키 사용 모듈(`toeven.c`), 테스트 드라이버(`toeven-driver.c`)로 나눴다. 뒤쪽 둘은 간단하다.

`toeven.c`:
```c
#include "static-key.h"

struct static_key tell_a_lie = { false };

int toeven(int n)
{
	if (n & 1)
		n <<= 1;

	if (static_branch_unlikely(&tell_a_lie))
		return 3;

	return n;
}
```

`static_key_enable(&tell_a_lie)` 호출 후에는 3을 반환하게 된다.

`toeven-driver.c`:
```c
#include <stdio.h>
#include "static-key.h"

extern struct static_key tell_a_lie;

extern int toeven(int n);

int main(void)
{
        printf("%d\n", toeven(42)); /* expected: 42 */

        static_key_enable(&tell_a_lie);

        printf("%d\n", toeven(42)); /* expected: 3 */

        return 0;
}
```

이제 본론이다. 기본적으로 리눅스 커널의 코드를 요약 복사한 것이다. x86_64 기준이다.

`static-key.h`:
```c
#ifndef STATIC_KEY_H_
#define STATIC_KEY_H_

typedef int bool;

enum {
	false,
	true,
};


struct static_key {
	int enabled;
};

static inline bool arch_static_branch(struct static_key *key, bool branch)
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
({ bool branch = arch_static_branch(key, false); branch; })


extern void static_key_enable(struct static_key *key);

#endif /* STATIC_KEY_H_ */
```

`static_key_enable()`이 생겼다. 짝이 되는 `static_key_disable()`은 생략.

`static-key.c`:
```c
#include <string.h>
#include <unistd.h>
#include <sys/mman.h>
#include "static-key.h"

struct jump_entry {
	unsigned long code;
	unsigned long target;
	unsigned long key;
};

static inline struct static_key *jump_entry_key(struct jump_entry *entry)
{
	return (struct static_key *)((unsigned long)entry->key & ~1UL);
}

static bool static_key_enabled(struct static_key *key)
{
	int n = key->enabled;

	return n >= 0 ? n : 1;
}

static bool jump_entry_branch(struct jump_entry *entry)
{
	return (unsigned long)entry->key & 1UL;
}

enum jump_label_type {
	JUMP_LABEL_NOP = 0,
	JUMP_LABEL_JMP,
};

static enum jump_label_type jump_label_type(struct jump_entry *entry)
{
	struct static_key *key = jump_entry_key(entry);
	bool enabled = static_key_enabled(key);
	bool branch = jump_entry_branch(entry);

	return enabled ^ branch;
}

static void text_poke(void *addr, const void *opcode, size_t len)
{
	unsigned long pagesize = getpagesize();
	unsigned long page_start;

	page_start = ((unsigned long)addr) & ~(pagesize - 1);

	mprotect((void *)page_start, 1, PROT_READ|PROT_WRITE|PROT_EXEC);
	memcpy(addr, opcode, len);
	mprotect((void *)page_start, 1, PROT_READ|PROT_EXEC);
}

#define JUMP_LABEL_NOP_SIZE 5

union jump_code_union {
	char code[JUMP_LABEL_NOP_SIZE];
	struct {
		char jump;
		int offset;
	} __attribute__((packed));
};

static void arch_jump_label_transform(struct jump_entry *entry,
				      enum jump_label_type type)
{
	union jump_code_union code;
	const unsigned char ideal_nop[] = { 0x0f, 0x1f, 0x44, 0x00, 0x00 };

	if (type == JUMP_LABEL_JMP) {
		code.jump = 0xe9;
		code.offset = entry->target -
				(entry->code + JUMP_LABEL_NOP_SIZE);
	} else {
		memcpy(&code, ideal_nop, JUMP_LABEL_NOP_SIZE);
	}

	text_poke((void *)entry->code, &code, JUMP_LABEL_NOP_SIZE);
}

void static_key_enable(struct static_key *key)
{
	extern struct jump_entry __start___jump_table;
	extern struct jump_entry __stop___jump_table;

	struct jump_entry *entry = &__start___jump_table;
	struct jump_entry *stop = &__stop___jump_table;

	key->enabled = -1;

	for (; (entry < stop) && (jump_entry_key(entry) == key); entry++)
		arch_jump_label_transform(entry, jump_label_type(entry));

	key->enabled = 1;
}
```

이것 저것 복잡하지만 결국 핵심은 `arch_static_branch()`와 `arch_jump_label_transform()`이다.

`__jump_table` 섹션에 인스트럭션 패치를 위한 정보가 담겨 있으니 섹션 주소를 알아야 한다. [리눅스 커널의 방식](https://github.com/torvalds/linux/blob/v4.15/include/asm-generic/vmlinux.lds.h#L233)을 따라하기는 번잡할 것 같아서 GNU 링커가 제공하는 특수 변수(`__start_*`, `__stop_*`)를 사용했다.

사용자 공간에 맞는 방식으로 패치를 수행하는 함수가 `text_poke()`이다. `mprotect()`로 쓰기를 풀고 인스트럭션을 덮어 쓴다. 그러고 나서 이전 보호 방식을 복원해야 하는데, 메모리 페이지 보호 값을 얻는 시스템 호출을 못 찾아서 그냥 `PROT_READ|PROT_EXEC`로 설정하게 했다. (`/proc/self/maps`를 열어 볼 수도 있겠지만 그렇게까지 할 정성은 없다.)

실행해 보자.

```
$ gcc -Wall -O2 -o toeven-test toeven.c toeven-driver.c static-key.c
$ ./toeven-test
42
3
```

좋다. 메모리 상의 인스트럭션이 정말 바뀌는지도 보자.

```
$ gdb ./toeven-test
GNU gdb (Ubuntu 8.0.1-0ubuntu1) 8.0.1
Copyright (C) 2017 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
...
Reading symbols from ./toeven-test...(no debugging symbols found)...done.
(gdb) break toeven
Breakpoint 1 at 0x820
(gdb) run
Starting program: /.../toeven-test 

Breakpoint 1, 0x0000555555554820 in toeven ()
(gdb) disassemble
Dump of assembler code for function toeven:
=> 0x0000555555554820 <+0>:	mov    %edi,%eax
   0x0000555555554822 <+2>:	lea    (%rdi,%rdi,1),%edx
   0x0000555555554825 <+5>:	test   $0x1,%al
   0x0000555555554827 <+7>:	cmovne %edx,%eax
   0x000055555555482a <+10>:	nopl   0x0(%rax,%rax,1)
   0x000055555555482f <+15>:	retq   
   0x0000555555554830 <+16>:	mov    $0x3,%eax
   0x0000555555554835 <+21>:	retq   
End of assembler dump.
(gdb) continue
Continuing.
42

Breakpoint 1, 0x0000555555554820 in toeven ()
(gdb) disassemble
Dump of assembler code for function toeven:
=> 0x0000555555554820 <+0>:	mov    %edi,%eax
   0x0000555555554822 <+2>:	lea    (%rdi,%rdi,1),%edx
   0x0000555555554825 <+5>:	test   $0x1,%al
   0x0000555555554827 <+7>:	cmovne %edx,%eax
   0x000055555555482a <+10>:	jmpq   0x555555554830 <toeven+16>
   0x000055555555482f <+15>:	retq   
   0x0000555555554830 <+16>:	mov    $0x3,%eax
   0x0000555555554835 <+21>:	retq   
End of assembler dump.
```

오프셋 10의 다섯 바이트, 잘 바뀐다.

정적 키의 핵심은 간단하다. 브랜치 위치를 기억해 뒀다가 런타임에 인스트럭션을 바꿔치기하는 것이다. 하지만 제대로 구현하려면 몇 가지 디테일이 필요하다.

첫 번째 이슈는 병렬 실행이다. 사용자 공간 다중 스레드 프로그램에서든 커널에서든, 한 CPU에서 패치 중인 인스트럭션을 다른 CPU에서 페치 하려 (pun intended) 할 수 있다. 따라서 패치 중에 다른 CPU들을 어디 안전한 코드 위에 붙잡아 두거나 해야 한다. 사용자 공간에서는 <tt>[pthread_barrier_wait()](https://github.com/wariua/manpages-ko/wiki/pthread_barrier_wait%283p%29)</tt> 같은 걸 쓰면 된다. 리눅스 커널에서는 아키텍처에 따라 [그냥 다른 CPU들을 뺑뺑이 돌게](https://github.com/torvalds/linux/blob/v4.15/arch/arm64/kernel/insn.c#L258) 하기도 하고 [트랩까지 이용한 정지 없는 점진적 패치 신공](https://github.com/torvalds/linux/blob/v4.15/arch/x86/kernel/alternative.c#L795)을 쓰기도 한다.

다음 이슈는 캐시이다. 인스트럭션을 바꿨는데 다른 코어의 인스트럭션 캐시에 이전 인스트럭션이 남아 있으면 곤란하다. 따라서 아키텍처별 방법으로 캐시를 날려야 한다. 사용자 공간에서는 이 문제에 신경 쓰지 않아도 되는데, `mprotect()` 내에서 캐시를 날려 주기 때문이다.
