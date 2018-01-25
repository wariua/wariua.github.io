---
layout: post
title: 확장 BPF
category: facility
tags: [bpf, seccomp, xdp, perf]
---
TL;DR: eBPF 프로그램을 작성해서 커널 내 여러 지점에서 실행할 수 있다. 네트워크와 실행 추적 쪽에서 주로 사용한다.

## BPF

eBPF 얘기를 시작하는 가장 진부한 방법은 역시나 BPF부터 소개하는 것이다.

BPF는 버클리 패킷 필터(Berkeley Packet Filter)의 줄임말이다. 이름 그대로 패킷을 걸러내는 필터이다. 그런데 원조집이라 할 [BSD에서의 BPF](https://www.freebsd.org/cgi/man.cgi?query=bpf)는 네트워크 탭(리눅스의 `PF_PACKET`)까지 아우르는 개념이다. 옛날 옛적에 유닉스에는 CSPF(CMU/Stanford Packet Filter)라는 게 있었는데 [BPF라는 새 구조](http://www.tcpdump.org/papers/bpf-usenix93.pdf)가 이를 대체했다. 이후 리눅스에서는 네트워크 탭을 나름의 방식으로 구현하고 패킷 필터 부분만 가져왔다. 리눅스의 패킷 필터를 리눅스 소켓 필터링(LSF: Linux Socket Filtering)이라고도 한다.

BSD의 BPF는 다음과 같이 사용한다. (간결함을 위해 오류 검사가 의도적으로 생략돼 있음.)

```c
int fd;
struct ifreq ifr;
struct bpf_insn def_insn = BPF_STMT(BPF_RET | BPF_K, 1500);
struct bpf_program def_prog = { 1, &def_insn };
unsigned char *buf;
unsigned int bufsize;
struct bpf_xhdr *bhp;

fd = open("/dev/bpf", O_RDONLY);

strncpy(ifr.ifr_name, ifname, sizeof(ifr.ifr_name));
ioctl(fd, BIOCSETLIF, &ifr);

ioctl(fd, BIOCGBLEN, &bufsize);
buf = malloc(bufsize);

ioctl(fd, BIOCSETF, &def_prog);

read(fd, buf, bufsize);

bhp = (struct bpf_xhdr *) buf;

printf("a packet captured: caplen=%u, datalen=%u\n",
       bhp->bh_caplen, bhp->bh_datalen);
```

열고, 장치에 결속시키고, 필터를 붙이고, 읽어온다.

리눅스의 `PF_PACKET`도 비슷하다.

```c
int fd;
struct sockaddr_ll sll = { 0 };
struct sock_filter def_insn = BPF_STMT(BPF_RET | BPF_K, 1500);
struct sock_fprog def_prog = { 1, &def_insn };
unsigned char buf[1500];
ssize_t datalen;

fd = socket(PF_PACKET, SOCK_RAW, ETH_P_ALL);

sll.sll_family = AF_PACKET;
sll.sll_ifindex = ifindex;
sll.sll_protocol = ETH_P_ALL;
bind(fd, (struct sockaddr *) &sll, sizeof(sll));

setsockopt(fd, SOL_SOCKET, SO_ATTACH_FILTER, &def_prog, sizeof(def_prog));

datalen = recv(fd, buf, sizeof(buf), MSG_TRUNC);

printf("a packet captured: datalen=%u\n", datalen);
```

BPF를 사용하는 대표적인 프로그램이 libpcap이다. [libpcap/pcap-bpf.c](https://github.com/the-tcpdump-group/libpcap/blob/master/pcap-bpf.c)와 [libpcap/pcap-linux.c](https://github.com/the-tcpdump-group/libpcap/blob/master/pcap-linux.c)에 제대로 된 코드가 있다. 데이터 복사를 줄인 인터페이스([Zero-Copy BPF](http://www.watson.org/~robert/freebsd/2007asiabsdcon/20070309-devsummit-zerocopybpf.pdf), [PACKET_MMAP](https://wariua.cafe24.com/wiki/Documentation/networking/packet_mmap.txt))도 지원한다.

위 코드에도 간단한 BPF 프로그램이 포함돼 있다. "`return 1500`"에 대응하는 한 인스트럭션짜리 프로그램이다. return만 있는 건 아니고 load, store, jump, add/sub/mul/div/and/or/xor/shift 등도 있다. 그걸로 32비트짜리 ALU 1개와 레지스터 1개, 작업용 메모리 슬롯 16개를 가지고 패킷 내용을 살펴보고서 정수 값을 반환하는 프로그램을 작성하면 된다. 반환 값은 패킷 데이터를 몇 바이트나 통과시킬지 나타내고 0이면 패킷을 버린다. 패킷 데이터 접근을 위한 특별한 주소 지정 방식(`BPF_ABS`)이 있으며 운영 체제에 따라 자체적인 BPF 확장이 있을 수 있다. 맨페이지에 [인스트럭션 설명](https://www.freebsd.org/cgi/man.cgi?query=bpf#FILTER_MACHINE)과 [예시 프로그램](https://www.freebsd.org/cgi/man.cgi?query=bpf#EXAMPLES)이 있다. 그리고 리눅스에서의 구현을 (확장까지 포함해서) 커널 소스 트리 [Documentation/networking/filter.txt](https://wariua.cafe24.com/wiki/Documentation/networking/filter.txt)에서 설명한다.

libpcap의 동반자인 `tcpdump`로 필터 문자열이 어떤 프로그램으로 바뀌는지 볼 수 있다.

```
# tcpdump -d ip
(000) ldh      [12]
(001) jeq      #0x800           jt 2    jf 3
(002) ret      #262144
(003) ret      #0
# tcpdump -dd ip
{ 0x28, 0, 0, 0x0000000c },
{ 0x15, 0, 1, 0x00000800 },
{ 0x6, 0, 0, 0x00040000 },
{ 0x6, 0, 0, 0x00000000 },
# tcpdump -ddd ip
4
40 0 0 12
21 0 1 2048
6 0 0 262144
6 0 0 0
```

필터 프로그램을 `setsockopt(SO_ATTACH_FILTER)`로 소켓에 붙이면 패킷 수신 과정에서 적용된다.

`linux/net/packet/af_packet.c`:
```c
static int packet_rcv(struct sk_buff *skb, struct net_device *dev,
                      struct packet_type *pt, struct net_device *orig_dev)
{
        struct sock *sk;
        unsigned int snaplen, res;
        ...

        snaplen = skb->len;

        res = run_filter(skb, sk, snaplen);
        if (!res)
                goto drop_n_restore;
        if (snaplen > res)
                snaplen = res;

        ...
        if (pskb_trim(skb, snaplen))
                goto drop_n_acct;
        ...
}

static unsigned int run_filter(struct sk_buff *skb,
                               const struct sock *sk,
                               unsigned int res)
{
        struct sk_filter *filter;

        rcu_read_lock();
        filter = rcu_dereference(sk->sk_filter);
        if (filter != NULL)
                res = bpf_prog_run_clear_cb(filter->prog, skb);
        rcu_read_unlock();

        return res;
}
```

소켓으로 패킷을 수신하는 곳이 `PF_PACKET`만 있는 건 아니다.

`linux/net/ipv4/udp.c`:
```c
static int udp_queue_rcv_skb(struct sock *sk, struct sk_buff *skb)
{
        ...

        if (sk_filter_trim_cap(sk, skb, sizeof(struct udphdr)))
                goto drop;
        ...
}
```

`linux/net/core/filter.c`:
```c
int sk_filter_trim_cap(struct sock *sk, struct sk_buff *skb, unsigned int cap)
{
        int err;
        struct sk_filter *filter;
        ...

        rcu_read_lock();
        filter = rcu_dereference(sk->sk_filter);
        if (filter) {
                struct sock *save_sk = skb->sk;
                unsigned int pkt_len;

                skb->sk = sk;
                pkt_len = bpf_prog_run_save_cb(filter->prog, skb);
                skb->sk = save_sk;
                err = pkt_len ? pskb_trim(skb, max(cap, pkt_len)) : -EPERM;
        }
        rcu_read_unlock();

        return err;
}
```

`linux/net/core/sock.c`:
```c
int __sk_receive_skb(struct sock *sk, struct sk_buff *skb,
                     const int nested, unsigned int trim_cap, bool refcounted)
{
	...

        if (sk_filter_trim_cap(sk, skb, trim_cap))
                goto discard_and_relse;
        ...
}
```

`linux/net/dccp/ipv6.c`:
```c
static int dccp_v6_rcv(struct sk_buff *skb)
{
        ...
        struct sock *sk;
        ...

        sk = __inet6_lookup_skb(&dccp_hashinfo, skb, __dccp_hdr_len(dh),
                                dh->dccph_sport, dh->dccph_dport,
                                inet6_iif(skb), 0, &refcounted);
        ...

        return __sk_receive_skb(sk, skb, 1, dh->dccph_doff * 4,
                                refcounted) ? -1 : 0;

        ...
}
```

[바](https://en.wikipedia.org/wiki/Java_bytecode)[이](http://files.catwell.info/misc/mirror/lua-5.2-bytecode-vm-dirk-laurie/lua52vm.html)[트](https://medium.com/dailyjs/understanding-v8s-bytecode-317d46c94775)[코](http://webassembly.org/docs/binary-encoding/)[드](https://swtch.com/~rsc/regexp/regexp2.html)가 등장하면 [제](https://www.ibm.com/support/knowledgecenter/en/SSYKE2_8.0.0/com.ibm.java.lnx.80.doc/diag/understanding/jit_overview.html)[이](http://luajit.org/)[아](https://wiki.mozilla.org/IonMonkey)[이](https://github.com/WebAssembly/wasm-jit-prototype)[티](https://www.pcre.org/original/doc/html/pcrejit.html)(JIT)가 뒤따르는 법이다. 어느 아키텍처에나 있을 법한 단순한 인스트럭션들이라 [변환](https://github.com/torvalds/linux/blob/master/arch/x86/net/bpf_jit_comp.c)이 쉽다. 보통은 JIT를 기본으로 사용하며 리눅스에서는 `/proc/sys/net/core/bpf_jit_enable`로 켜 줘야 한다.

그런데 패킷 필터가 꼭 이런 형태여야 하는 건 아니다. 예를 들어 패킷을 내용과 메타 정보에 따라 거른다는 기능은 같지만 리눅스 넷필터는 전혀 다른 형태이다. 넷필터를 통해 충분히 효율적인 필터링 기능을 손쉽게 구현할 수 있지만 새로운 뭔가가 필요할 때마다 커널을 변경해야 한다. 그런 귀찮음을 예상했는지 BPF는 그 선조 시절부터 [가상 머신](https://github.com/torvalds/linux/blob/v4.14/kernel/bpf/core.c#L770) 형태였다. 범용 메커니즘을 제공할 테니 알아서 하라는 것이다.

## 용도 확장 - seccomp

BPF의 핵심은 프로그램을 작성해서 커널 내 몆몆 지점에서 돌릴 수 있다는 것이다. (그래서 구글 프로젝트 제로의 [Spectre 설명](https://googleprojectzero.blogspot.kr/2018/01/reading-privileged-memory-with-side.html)에도 (e)BPF가 등장한다.) 유연성과 편의성 간 타협이라는 측면에서 커널 프로그래밍과 커널 이용 사이의 영역을 채워 준다. 꽤 편하면서도 유연한 메커니즘이 있는데 사람들이 가만 놔뒀을 리 없다. 패킷 대신 다른 선형 데이터를 프로그램 입력으로 주고 반환 값 해석 방식을 나름대로 정하면 다른 모듈에서도 BPF를 사용할 수 있다.

리눅스의 [seccomp](https://en.wikipedia.org/wiki/Seccomp)(SECure COMPuting mode)]은 프로세스가 자신에게 허용되는 시스템 호출을 제한할 수 있는 메커니즘이다. 설령 프로세스가 탈취되더라도 피해를 최소화 하기 위한 방어책인데, 서버 프로그램들이 초기화를 마치고 본격적인 동작을 시작하기 전에 `setuid()` 등으로 특권을 버리는 것과 통하는 면이 있다. 원래는 안전 컴퓨팅 모드로 들어가면 4가지 시스템 호출(`read()`, `write()`, `exit()`, `sigreturn()`)만 가능하다. 안전도 좋지만, 빡세다. 좀 더 유연하면 좋을 테고, 이왕이면 인자까지 보고 허용 여부를 결정할 수 있으면 좋을 것이다. 그래서 BPF가 도입됐다. 프로세스에 BPF 프로그램을 붙이면 그 프로세스가 시스템 호출을 할 때마다 BPF 프로그램이 실행된다. 프로그램 입력은 시스템 호출 번호와 인자들이고 반환 값에 따라 시스템 호출이 허용되거나 거절되거나 기타 방식으로 동작이 바뀐다.

`linux/arch/x86/entry/common.c`:
```c
static long syscall_trace_enter(struct pt_regs *regs)
{
        u32 arch = in_ia32_syscall() ? AUDIT_ARCH_I386 : AUDIT_ARCH_X86_64;
        ...

#ifdef CONFIG_SECCOMP
        /*
         * Do seccomp after ptrace, to catch any tracer changes.
         */
        if (work & _TIF_SECCOMP) {
                struct seccomp_data sd;

                sd.arch = arch;
                sd.nr = regs->orig_ax;
                sd.instruction_pointer = regs->ip;
#ifdef CONFIG_X86_64
                if (arch == AUDIT_ARCH_X86_64) {
                        sd.args[0] = regs->di;
                        sd.args[1] = regs->si;
                        sd.args[2] = regs->dx;
                        sd.args[3] = regs->r10;
                        sd.args[4] = regs->r8;
                        sd.args[5] = regs->r9;
                } else
#endif
                {
                        ...
                }

                ret = __secure_computing(&sd);
                if (ret == -1)
                        return ret;
        }
#endif

        ...
}
```

`linux/kernel/seccomp.c`:
```c
int __secure_computing(const struct seccomp_data *sd)
{
        int mode = current->seccomp.mode;
        int this_syscall;

        ...

        this_syscall = sd ? sd->nr :
                syscall_get_nr(current, task_pt_regs(current));

        switch (mode) {
        case SECCOMP_MODE_STRICT:
                __secure_computing_strict(this_syscall);  /* may call do_exit */
                return 0;
        case SECCOMP_MODE_FILTER:
                return __seccomp_filter(this_syscall, sd, false);
        default:
                BUG();
        }
}

static int __seccomp_filter(int this_syscall, const struct seccomp_data *sd,
                            const bool recheck_after_trace)
{
        u32 filter_ret, action;
        struct seccomp_filter *match = NULL;
        int data;

        ...

        filter_ret = seccomp_run_filters(sd, &match);
        data = filter_ret & SECCOMP_RET_DATA;
        action = filter_ret & SECCOMP_RET_ACTION_FULL;

        switch (action) {
        ...
        }
        ...
}

static u32 seccomp_run_filters(const struct seccomp_data *sd,
                               struct seccomp_filter **match)
{
        struct seccomp_data sd_local;
        u32 ret = SECCOMP_RET_ALLOW;
        /* Make sure cross-thread synced filter points somewhere sane. */
        struct seccomp_filter *f =
                        READ_ONCE(current->seccomp.filter);
        ...

        /*
         * All filters in the list are evaluated and the lowest BPF return
         * value always takes priority (ignoring the DATA).
         */
        for (; f; f = f->prev) {
                u32 cur_ret = BPF_PROG_RUN(f->prog, sd);

                if (ACTION_ONLY(cur_ret) < ACTION_ONLY(ret)) {
                        ret = cur_ret;
                        *match = f;
                }
        }
        return ret;
}
```

seccomp의 주된 용도는 샌드박스이다. 웹 브라우저나 서버가 제발로 들어갈 수도 있고 Docker나 LXD가 컨테이너를 집어넣을 수도 있다. 시스템 호출을 선별적으로 실패하게 할 수도 있으니 오류 주입(fault injection)에 쓰는 것도 가능하겠지만 사용자 공간만 보면 안타깝게도 `LD_PRELOAD`가 너무 강적이다.

[seccomp(2) 맨페이지](https://github.com/wariua/manpages-ko/wiki/seccomp%282%29) 말미에 예시 프로그램이 있는데, 숫제 기계어다. [linux/samples/seccomp/](https://github.com/torvalds/linux/tree/master/samples/seccomp) 디렉터리에 예시 프로그램들과 더불어 `bpf-helper.[ch]`가 있는데, 어셈블리어 정도로 만들어 준다. 한 걸음 더 올라가면 [libseccomp](https://lwn.net/Articles/494252/)가 있는데, 자주 쓰는 인스트럭션 패턴을 '규칙'으로 추상화하고 이를 조작하는 API를 제공한다. [테스트 프로그램](https://github.com/seccomp/libseccomp/tree/master/tests)을 보면 꽤 편리해 보인다. 많이 편해졌고 파이썬 바인딩까지 생겼으니 좋은데, 아직 배가 고프다. 남은 건 BPF 프로그램 자체를 고급 언어로 작성하는 것일 텐데, 그 전에 거쳐야 할 과정이 있다.

## 확장 BPF

BPF를 다양하게 써먹으려니 슬슬 한계점들이 보인다.

 * 함수 호출이 불가능하다. 가령 패킷의 어떤 헤더를 건너뛰는 함수나 현재 프로세스의 UID를 알려주는 함수가 있어서 프로그램에서 호출할 수 있다면 편리할 것이다. 거기 더해 다른 BPF 프로그램을 '호출'할 수 있다면 할 수 있는 게 더 많아질 것이다.
 * 데이터 저장 공간이 빈약하다. 더 큰 공간이 있어서 프로그램과 독립적으로 유지되기도 한다면 프로그램 반복 실행으로 얻은 통계 자료를 저장하거나 프로그램 간 데이터 전달 통로로 이용할 수 있을 것이다.
 * 32비트다. 그리고 지금은 64비트 시대이다. 덤으로 레지스터도 좀 많아지면 좋을 것이다.
 * 좀 특이한 인스트럭션들이 있다. 가령 점프 인스트럭션 인자로 참일 때 오프셋과 거짓일 때 오프셋이 함께 있다. 별로 유용하지 않으면서 JIT 컴파일을 복잡하게 만든다.
 * 프로그램 관리가 불편하다. 가령 프로그램들을 미리 준비해 뒀다가 필요할 때 골라서 실행하기만 된다면 편할 것이다.

많이 쓰는 64비트 아키텍처들과 비슷하도록 ISA를 확장해서 여러 문제를 해결할 수 있다. 근데 두 가지 ISA를 동시에 유지하기엔 부담이 되니까 기존 방식 프로그램을 실행할 때 내부적으로 새 인스트럭션 세트에 맞게 변환해서 돌리는 식이 좋을 것이다. 그 새로운 아키텍처를 확장 BPF(extended BPF => eBPF) 내지 내부 BPF(internal BPF)라고 하고, 이전 BPF 형식은 앞에 '전통적(classic)'이라는 수식어가 붙어서 cBPF가 된다. eBPF 프로그램을 작성할 때는 `#include <linux/bpf.h>` 하고 cBPF에서는 `#include <linux/filter.h>` 한다.

eBPF에는 [여러 헬퍼 함수들](https://github.com/torvalds/linux/blob/v4.14/include/uapi/linux/bpf.h#L255)이 있어서 프로그램에서 호출할 수 있고 다른 프로그램을 `exec()`(내지 꼬리 호출) 할 수도 있다. 또 스택이 생겼고 힙 내지 공유 메모리 역할을 하는 자료 구조(맵)도 추가됐다. 사용자 공간에서도 맵에 접근할 수 있어서 설정이나 동작 결과를 위아래로 주고받는 데 쓸 수 있다. 그리고 이 새로운 자료 구조와 프로그램을 다루기 위한 새 시스템 호출 `bpf()`가 생겼다. 그렇게 틀이 갖춰지고 나서는 프로그램 종류가 하나씩 늘고 (네트워킹 여기저기, 실행 추적, ...) 맵 종류가 함께 늘어난다 (해시, 배열, 프로그램, 스택 트레이스, 장치, 소켓, LRU, LPM, 맵의 맵, ...).

다음 문제는 프로그램과 맵 객체의 관리이다. 기본적으로 프로세스가 종료되면 (그래서 가령 소켓이 닫히면) 연계된 BPF 프로그램과 맵이 사라진다. 그런데 쓰이는 곳이 늘다 보면 프로세스와 독립적으로 객체가 유지돼야 하는 경우가 생기기 마련이다. 그래서 커널의 관련 서브시스템에서 참조를 유지해서 객체가 사라지는 걸 막기도 한다. 하지만 그걸로는 부족한 것이, 객체를 참조하는 파일 디스크립터가 닫히고 나면 사용자 공간에서 맵에 접근할 방법이 없다. 그래서 계속 도는 어떤 에이전트 프로세스에게 유닉스 도메인 소켓을 통해 파일 디스크립터를 넘겨서 보관하기도 한다. 번거로운 일이고, 그래서 객체를 [가상의 파일 시스템에 '저장'](https://lwn.net/Articles/664688/)할 수 있는 방법(`bpf(BPF_OBJ_{PIN,GET})`)이 생겼다. 그 bpf 타입 파일 시스템을 마운트 하면 셸에서 객체들을 관리할 수도 있다. 이런 최근 내용들은 [bpf(2) 맨페이지](https://github.com/wariua/manpages-ko/wiki/bpf%282%29)나 [커널 문서](https://wariua.cafe24.com/wiki/Documentation/networking/filter.txt)에도 아직 기록되지 않았다. `linux/include/linux/bpf.h` 파일과 `linux/kernel/bpf/` 내 파일들에서 정보를 얻을 수 있다. 또 ["eBPF 철저 소개"](https://lwn.net/Articles/740157/) 글에서 모든 프로그램 종류와 맵 타입을 간략히 설명해 준다.

맵은 커널 메모리를 소모하고 프로그램은 커널 문맥에서 동작한다. 당연히 접근 제어가 필요하다. `bpf()` 동작에 필요한 권한은 대략 다음과 같다.

 * 맵 생성할 때 `CAP_NET_ADMIN` 필요 (일부는 `CAP_SYS_ADMIN`)
 * 프로그램 적재할 때 `BPF_PROG_TYPE_SOCKET_FILTER`와 `BPF_PROG_TYPE_CGROUP_SKB` 제외하고 `CAP_SYS_ADMIN` 필요
 * cgroup이나 소켓 맵 등에 프로그램을 붙이거나 떼어 낼 때 `CAP_SYS_ADMIN` 필요
 * ID를 통한 객체 검색에 `CAP_SYS_ADMIN` 필요
 * 위에 해당하지 않아도 `/proc/sys/kernel/unprivileged_bpf_disabled`를 0 아닌 값으로 설정하면 모든 동작에 `CAP_SYS_ADMIN` 필요

요약하면, 기본적으로 특수 권한이 필요하지만 `BPF_PROG_TYPE_SOCKET_FILTER`는 일반 사용자도 돌릴 수 있다. 하지만 그마저 시스템 전역 설정으로 막을 수 있다. 그리고 리눅스 4.4 전에선 다 필요 없고 무조건 `CAP_SYS_ADMIN`이 필요하다.

프로그램 종류별로 입력(문맥) 데이터와 사용 가능한 헬퍼 함수들이 다르다. 따로 정리된 곳이 있는지는 모르겠고, 커널 소스에 프로그램 종류마다 `struct bpf_verifier_ops` 객체가 정의돼 있는데 `is_valid_access`와 `convert_ctx_access`, `get_func_proto` 콜백에 그 내용들이 구현돼 있다. 4.14 기준으로 두 파일(`linux/net/core/filter.c`, `linux/kernel/trace/bpf_trace.c`)만 보면 된다.

`linux/net/core/filter.c`:
```c
static const struct bpf_func_proto *
bpf_base_func_proto(enum bpf_func_id func_id)
{
        switch (func_id) {
        case BPF_FUNC_map_lookup_elem:
                return &bpf_map_lookup_elem_proto;
        case BPF_FUNC_map_update_elem:
                return &bpf_map_update_elem_proto;
        ...
        default:
                return NULL;
        }
}

static const struct bpf_func_proto *
sk_filter_func_proto(enum bpf_func_id func_id)
{
        switch (func_id) {
        case BPF_FUNC_skb_load_bytes:
                return &bpf_skb_load_bytes_proto;
        ...
        default:
                return bpf_base_func_proto(fund_id);
}

static bool sk_filter_is_valid_access(int off, int size,
                                      enum bpf_access_type type,
                                      struct bpf_insn_access_aux *info)
{
        switch (off) {
        case bpf_ctx_range(struct __sk_buff, tc_classid):
        case bpf_ctx_range(struct __sk_buff, data):
        case bpf_ctx_range(struct __sk_buff, data_end):
        case bpf_ctx_range_till(struct __sk_buff, family, local_port):
                return false;
        }

        ...
        return bpf_skb_is_valid_access(off, size, type, info);
}

static u32 bpf_convert_ctx_access(enum bpf_access_type type,
                                  const struct bpf_insn *si,
                                  struct bpf_insn *insn_buf,
                                  struct bpf_prog *prog, u32 *target_size)
{
        struct bpf_insn *insn = insn_buf;
        int off;

        switch (si->off) {
        case offsetof(struct __sk_buff, len):
                *insn++ = BPF_LDX_MEM(BPF_W, si->dst_reg, si->src_reg,
                                      bpf_target_off(struct sk_buff, len, 4,
                                                     target_size));
                break;
        ...
        }

        return insn - insn_buf;
}

const struct bpf_verifier_ops sk_filter_prog_ops = {
        .get_func_proto         = sk_filter_func_proto,
        .is_valid_access        = sk_filter_is_valid_access,
        .convert_ctx_access     = bpf_convert_ctx_access,
};
```

#### 상위 언어

사실 BPF 사용 확대에 있어 가장 큰 장애물은 프로그래밍 언어다. 어셈블리어 깨작거려서 작성할 수 있는 프로그램에는 한계가 있이니 고급 언어로 작성한 프로그램을 eBPF 기계어로 바꿔 주는 컴파일러가 필요하다. 그래서 LLVM 3.7에 [eBPF 백엔드가 추가](https://github.com/llvm-mirror/llvm/commit/4fe85c75482f9d11c5a1f92a1863ce30afad8d0d)됐다. 그리고 3년이 지났는데 GCC 쪽은 아직 소식이 없다.

다음은 (용감하게 IPv4를 가정해서) 패킷 출발 주소가 127.0.0.1이면 허용하고 아니면 버리게 하는 필터 프로그램이다.

`bpf-lo.c`:
```c
#include <stddef.h>
#include <linux/bpf.h>
#include <linux/filter.h> /* for BPF_NET_OFF */
#include <linux/ip.h>
#include <linux/in.h>

unsigned long long load_word(void *skb, unsigned long long off)
        asm("llvm.bpf.load.word");

__attribute__((section("socket_prog"), used))
int local_only(struct __sk_buff *skb)
{
        __u32 saddr = load_word(skb, BPF_NET_OFF + offsetof(struct iphdr, saddr));

        if (saddr == INADDR_LOOPBACK)
                return 0x40000;

        return 0;
}
```

(`BPF_NET_OFF`와 `BPF_LL_OFF`의 의미는 [처리 코드](https://github.com/torvalds/linux/blob/v4.14/kernel/bpf/core.c#L62) 참고. 내장 '함수' `llvm.bpf.load.word`는 [타겟 기술 파일](https://github.com/llvm-mirror/llvm/blob/master/lib/Target/BPF/BPFInstrInfo.td) 말미 참고.)

```
$ clang -O2 -Wall -I ... -target bpf -S bpf-lo.c -o -
        .text
        .section        socket_prog,"ax",@progbits
        .globl  local_only
        .p2align        3
local_only:                             # @local_only
# BB#0:
        r6 = r1
        r0 = *(u32 *)skb[-1048564]
        r1 = r0
        r1 <<= 32
        r1 >>= 32
        r0 = 1
        r2 = 2130706433
        if r1 == r2 goto LBB0_2
# BB#1:
        r0 = 0
LBB0_2:
        r0 <<= 18
        exit
```

```
$ clang -O2 -Wall -I ... -target bpf -c bpf-lo.c
$ readelf -x socket_prog bpf-lo.o

Hex dump of section 'socket_prog':
  0x00000000 bf160000 00000000 20000000 0c00f0ff ........ .......
  0x00000010 bf010000 00000000 67010000 20000000 ........g... ...
  0x00000020 77010000 20000000 b7000000 01000000 w... ...........
  0x00000030 b7020000 0100007f 1d210100 00000000 .........!......
  0x00000040 b7000000 00000000 67000000 12000000 ........g.......
  0x00000050 95000000 00000000                   ........

$ objcopy -I elf64-little --dump-section socket_prog=bpf-lo.insn bpf-lo.o
```

(생체 역어셈블러를 돌리고 싶으면 [인스트럭션 인코딩](https://github.com/iovisor/bpf-docs/blob/master/eBPF.md) 참고.)

타겟으로 `bpfeb`(빅엔디안)과 `bpfel`(리틀엔디안)도 가능하다. 즉, 그냥도 교차 컴파일인데 *더* 교차 컴파일도 가능하다. 교차이다 보니 아키텍처 의존적 헤더들(가령 `/usr/include/x86_64-linux-gnu/*`)을 못 찾아서 컴파일이 안 될 수 있다. 적당한 아키텍처별 경로나 커널 헤더 경로 등을 지정해 주면 된다.

`bpf-lo.insn` 파일에 들어있는 필터 프로그램을 소켓에 붙여 보자.

`quiet-server.c` (오류 검사 생략돼 있음):
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <sys/syscall.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <linux/bpf.h>

static int bpf(int cmd, union bpf_attr *attr, unsigned int size)
{
        return syscall(__NR_bpf, cmd, attr, size);
}

static void install_filter(const char *prog_path, int sock)
{
        struct stat st;
        void *insns;
        union bpf_attr attr;
        int fd;

        fd = open(prog_path, O_RDONLY);
        fstat(fd, &st);
        insns = mmap(0, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
        close(fd);

        memset(&attr, 0, sizeof(attr));
        attr.prog_type = BPF_PROG_TYPE_SOCKET_FILTER;
        attr.insns = (__u64) insns;
        attr.insn_cnt = st.st_size / sizeof(struct bpf_insn);
        attr.license = (__u64) "GPL";

        fd = bpf(BPF_PROG_LOAD, &attr, sizeof(attr));
        munmap(insns, st.st_size);

        setsockopt(sock, SOL_SOCKET, SO_ATTACH_BPF, &fd, sizeof(fd));
        close(fd);
}

int main(void)
{
        int sock, newsock;
        struct sockaddr_in laddr, raddr;
        socklen_t raddr_len;

        sock = socket(AF_INET, SOCK_STREAM, 0);

        install_filter("bpf-lo.insn", sock);

        laddr.sin_family = AF_INET;
        laddr.sin_addr.s_addr = INADDR_ANY;
        laddr.sin_port = htons(12345);

        bind(sock, (struct sockaddr *) &laddr, sizeof(laddr));
        listen(sock, 5);

        while (1) {
                raddr_len = sizeof(raddr);
                newsock = accept(sock, (struct sockaddr *) &raddr, &raddr_len);

                fprintf(stderr, "accepted from %s\n", inet_ntoa(raddr.sin_addr));

                close(newsock);
        }
}
```

위 서버 프로그램을 실행하고 `telnet localhost 12345` 하면 연결이 되지만 ``telnet 'ip route get 1 | head -1 | cut -d' ' -f7` 12345`` 하면 안 된다.

C로 작성할 수 있게 된 건 좋은데 아직 좀 불편하다. 게다가 맵 사용과 관련해 문제가 있다. BPF 프로그램 내에서 헬퍼 함수로 맵을 사용하려면 맵을 가리키는 파일 디스크립터가 있어야 한다. 그런데 맵을 생성하는 건 보통 프로그램 컴파일 때가 아니라 적재 때이다. 따라서 적재 전에 BPF 프로그램 내의 맵 FD 값 사용 위치를 찾아서 실제 FD 값으로 채워 줘야 한다. 즉, 일종의 재배치(relocation) 단계가 필요하다.

그걸 프로그램마다 구현하는 건 좀 그러니까 라이브러리 같은 게 있으면 좋을 것이다. 이왕이면 다른 편의 기능들도 좀 있어서 프로그램과 맵 스펙이 담긴 오브젝트 파일 하나 던져 주면 알아서 맵 생성하고 프로그램 패치 하고 적재까지 해 주면 좋을 것이다. 현재 그런 구현체가 적어도 네 가지 있다.

 * `linux/tools/lib/bpf/`의 libbpf
 * `linux/samples/bpf/bpf_load.[ch]`
 * [iproute2](https://www.kernel.org/pub/linux/utils/net/iproute2/)의 `lib/bpf/bpf.c`
 * [BCC](https://github.com/iovisor/bcc)의 `src/cc/libbpf.[ch]`

오브젝트 내 섹션 구조나 맵 선언 방식이 조금씩 다를 수 있다. 하지만 기능 면에선 고만고만하다. 라이브러리 형태이고 `perf`에서도 쓰는 libbpf를 한번 사용해 보자.

일단 libbpf를 빌드 해야 한다. 그 디렉터리 안에서 `make` 하면 `libbpf.a`와 `libbpf.so`가 생긴다. libelf가 필요하다.

`bpf-lo.c`에 라이선스와 커널 버전 정보를 추가해야 한다. 커널 버전은 원래 필요 없는 건데 (kprobe용 프로그램에만 필요) libbpf에서는 무조건 요구하고 있다. 뭐, 가르쳐 주자. 함수에서도 그렇지만 섹션 이름이 중요하지 심볼 이름은 중요하지 않다.

```c
...
#include <linux/version.h>

#define SEC(x) __attribute__((section(x), used))

unsigned long long load_word(void *skb, unsigned long long off)
        asm("llvm.bpf.load.word");

SEC("socket_prog")
int local_only(struct __sk_buff *skb)
{
        ...
}

char _licence[] SEC("license") = "GPL";
__u32 _version SEC("version") = LINUX_VERSION_CODE;
```

그리고 응용의 적재 루틴을 바꾸면 된다.

`quiet-server.c`:
```c
...
#include "libbpf.h"

static void install_filter(const char *prog_path, int sock)
{
        int fd;
        struct bpf_object *obj;

        bpf_prog_load(prog_path, BPF_PROG_TYPE_SOCKET_FILTER, &obj, &fd);

        setsockopt(sock, SOL_SOCKET, SO_ATTACH_BPF, &fd, sizeof(fd));
        close(fd);
}

...
```

많이 간단해졌다. 다만 맵 사용이 없는 건 아쉽다. [linux/samples/bpf/](https://github.com/torvalds/linux/tree/master/samples/bpf)에서 다양한 예시 프로그램들(`*_kern.c`)을 볼 수 있다.

한편으로 구현체 목록 마지막의 BCC는 좀 눈여겨볼 필요가 있다. BPF 프로그램을 좀 더 편하게 작성할 수 있게 해 주고 파이썬 및 Lua 바인딩을 제공한다. ([예시 프로그램](https://github.com/iovisor/bcc/tree/master/examples) 참고.) BCC는 [IO Visor](https://www.iovisor.org/)라는 더 큰 프로젝트의 일부인데, BCC를 이용해 XDP용 BPF 프로그램을 작성한다. (XDP는 아래에 다시 등장한다.) IO Visor의 하위 프로젝트로 [사용자 공간 eBPF 가상 머신](https://github.com/iovisor/ubpf)도 있다.

## 용도 확장 - 다종다양

작성 가능한 프로그램의 범위가 넓어지면 사용하는 곳도 늘어난다. 일부는 cBPF를 지원하다가 eBPF까지 지원하게 되었고 나머지 대다수는 eBPF만 지원한다. 한편 seccomp에서는 아직 cBPF만 지원한다.

기원이 소켓 필터이다 보니 네트워크 쪽 용도가 다수이다. 그래서 프로그램 입력도 `struct __sk_buff` 타입인 경우가 많다. `struct __sk_buff`와 기타 입력 데이터 타입들이 `linux/bpf.h` 파일에 정의돼 있다.

##### `BPF_PROG_TYPE_SOCKET_FILTER` - 패킷 필터링, 분류, 파싱, ...

소켓 필터에 eBPF를 사용할 수도 있다. `setsockopt(SO_ATTACH_FILTER)` 대신 `setsockopt(SO_ATTACH_BPF)`로 붙이면 된다. 프로그램 입력은 `struct __sk_buff`이다. 프로그램 반환 값의 해석 방식은 cBPF에서와 같다.

이 타입을 사용하는 또 다른 곳이 넷필터의 xt_bpf 매치이다. (필터끼리는 통하는 법이다.) [nfbpf_compile 프로그램](http://git.netfilter.org/iptables/tree/utils/nfbpf_compile.c)으로 생성한 cBPF 프로그램을 사용할 수도 있고 미리 컴파일 해 둔 eBPF 프로그램을 사용할 수도 있다. ([Cloudflare의 소개글](https://blog.cloudflare.com/introducing-the-bpf-tools/)도 참고.) 0 아닌 값을 반환하면 일치한 것으로 처리한다.

BPF 원조 사용처인 `PF_PACKET` 소켓에는 일종의 [부하 분산 기능](https://wariua.cafe24.com/wiki/Documentation/networking/packet_mmap.txt#AF_PACKET_.EB.B6.84.EC.82.B0_.EB.AA.A8.EB.93.9C)이 있다. 소켓을 여러 개 만들어서 묶어 놓으면 지정한 알고리즘에 따라 수신 패킷이 그 중 하나로 간다. 그 알고리즘을 BPF로 구현할 수 있으며 프로그램 반환 값을 `%` 해서 소켓을 정한다. `setsockopt(SOL_PACKET, PACKET_FANOUT_DATA)`로 프로그램을 붙인다. 일종의 패킷 스위치에서 경로 결정 알고리즘을 BPF로 구현하는 것인데, 이 패턴은 아래에서 여러 번 변주된다.

[KCM(Kernel Connection Multiplexer)](https://lwn.net/Articles/657999/)([커널 문서](https://wariua.cafe24.com/wiki/Documentation/networking/kcm.txt)) 소켓에서도 사용한다. KCM은 스트림 프로토콜인 TCP와 메시지 기반 응용 프로토콜 사이에 끼어들어서 메시지 처리와 다중화 서비스를 제공한다. 예를 들어 TCP 소켓 하나를 기반으로 KCM 소켓 여러 개를 만들어서 메시지 처리 스레드마다 하나씩 할당한다. TCP 소켓으로 데이터 스트림이 들어오면 KCM 모듈에서 메시지를 조립하고, 완성된 메시지를 KCM 소켓들 중 하나로 보낸다. 기존에 응용 계층에서 하던 작업 일부를 대신 해 주는 셈이다. 그런데 메시지를 조립하려니 프로토콜마다 형식이 다르고, 그래서 등장하는 게 BPF 프로그램이다. [메시지 헤더를 파싱 해서 메시지 길이를 반환](https://github.com/torvalds/linux/blob/v4.14/net/kcm/kcmsock.c#L380)하면 된다. 그런데 사실 KCM 모듈의 핵심은 다중화이고 메시지 파싱은 [스트림 파서(strparser)](https://lwn.net/Articles/695982/)([커널 문서](https://wariua.cafe24.com/wiki/Documentation/networking/strparser.txt))라는 다른 모듈을 통해 수행한다. strparser는 잠시 후 다시 등장한다.

여담으로, [KCM 패치 설명](https://lwn.net/Articles/657970/)을 보면 향후 확장 가능성 중에 "TLS와의 통합 (커널 내 TLS는 따로 진행)"이란 게 있다. `linux/net/tls/`에 있는 [TLS 레코드 계층](https://wariua.cafe24.com/wiki/Documentation/networking/tls.txt) 구현 과정에서 KCM을 이용하게 될까? 별로 그럴 것 같지 않다. 다중화가 유용할지도 모르겠고, 무엇보다 하드코딩 된 BPF 프로그램을 적재하는 커널 코드란 건 아무래도 이상하다.

##### `BPF_PROG_TYPE_SCHED_CLS`, `BPF_PROG_TYPE_SCHED_ACT` - 패킷 스케줄링

넷필터가 나왔는데 패킷 스케줄러가 안 나오면 섭섭하다. 분류자(classifier)와 행위(action)로 BPF 프로그램을 사용할 수 있다. 프로그램 입력은 마찬가지로 `struct __sk_buff`이되 더 많은 필드를 사용할 수 있다. 사용할 수 있는 헬퍼 함수도 훨씬 많다. 분류자는 classid를 반환하고 행위는 `TC_ACT_*`를 반환한다.

[tc-bpf(8) 맨페이지](https://github.com/wariua/manpages-ko/wiki/tc-bpf%288%29)에는 `tc`뿐 아니라 BPF 프로그래밍 일반에 대한 유용한 내용이 많다. 그리고 [Cilium 매뉴얼](http://docs.cilium.io/en/latest/bpf/)에는 더 풍부한 정보가 있다.

##### `BPF_PROG_TYPE_SK_SKB` - 소켓 간 메시지 전달

[4.14에서 추가](https://github.com/torvalds/linux/commit/b005fd189cec9407b700599e1e80e0552446ee79)된 따끈따끈한 타입이며 [소켓 맵](https://lwn.net/Articles/731133/)(`BPF_MAP_TYPE_SOCKMAP`)에서 사용한다. 소켓 맵은 이름처럼 (TCP) 소켓들의 배열이다. 소켓으로 세그먼트가 들어오면 strparser를 이용해 메시지를 조립하고, 완성되면 배열 내 소켓 하나를 골라서 그리로 보낸다. 또는 그냥 버릴 수도 있다. 메시지 조립과 처리 방식 결정에 BPF 프로그램을 사용하는데, `bpf(BPF_PROG_ATTACH)`로 소켓 맵에 프로그램을 붙일 때 `attach_type`이 각각 `BPF_SK_SKB_STREAM_PARSER`와 `BPF_SK_SKB_STREAM_VERDICT`이다. 입력은 둘 모두 `struct __sk_buff`이다. PARSER의 반환 값은 KCM에서처럼 메시지 길이(아직 모르면 0)이다. VERDICT의 반환 값은 `SK_DROP` 아니면 `SK_PASS`인데, 메시지가 전달되게 하려면 헬퍼 함수 `bpf_sk_redirect_map()`으로 대상 소켓을 지정한 후 `SK_PASS`를 반환하면 된다.

소켓 맵으로 뭘 만들 수 있을까? 간단한 필터링 기능을 가진 프록시를 만들 수 있을 것이다. 프로토콜에서 메시지 순서에 덜 민감하다면 로드 밸런서를 만드는 데 쓸 수도 있다. 사용자 공간에서 동작하는 L7 스위치보다 구현하기 불편하지만 문맥 전환이 없으니 훨씬 효율적으로 동작한다.

[소켓 맵 소스 코드](https://github.com/torvalds/linux/blob/master/kernel/bpf/sockmap.c)를 보면 저작권자가 "Covalent IO, Inc. http://covalent.io"라고 돼 있다. 앞서 슬쩍 등장했던 [Cilium](https://www.cilium.io/)이 바로 이 회사의 프로젝트다. Cilium은 컨테이너 환경 마이크로서비스를 주요 대상으로 하는 L7 스위치이다. 한편으로 같은 저작권자명이 [장치 맵 소스 코드](https://github.com/torvalds/linux/blob/master/kernel/bpf/devmap.c)에도 등장하는데, 이 맵은 XDP를 위한 것이다.

##### `BPF_PROG_TYPE_XDP` - 효율적인 대안 패킷 처리 경로

Cilium이 TCP 소켓 위에서 동작하는 L7 스위치라면 [XDP](http://prototype-kernel.readthedocs.io/en/latest/networking/XDP/introduction.html)는 장치 드라이버 근처에서 동작하는 패킷 스위치... 등을 만드는 데 쓸 수 있는 프레임워크이다. 수신 패킷 처리 경로의 아주 이른 지점에서 장치에 등록된 eBPF 프로그램을 실행하고 그 결과에 따라 패킷을 버리거나(`XDP_DROP`) 커널 네트워크 스택으로 넘기거나(`XDP_PASS`) 들어온 장치로 반사하거나(`XDP_TX`) 다른 장치로 보낸다(`XDP_REDIRECT`). 장치 가까이에서 동작하기 때문에 오버헤드가 작고, 그래서 DoS 방어처럼 성능이 중요한 [여러 용도](http://people.netfilter.org/hawk/presentations/xdp2016/xdp_intro_and_use_cases_sep2016.pdf)에 사용할 수 있다. REDIRECT 할 때 쓰는 게 장치 맵(`BPF_MAP_TYPE_DEVICE_MAP`)이고 IP 주소에 따른 동작을 위해 LPM 맵(`BPF_MAP_TYPE_LPM_TRIE`)을 사용할 수도 있다. Cilium 소스 트리의 [예시 프로그램](https://github.com/cilium/cilium/blob/master/bpf/bpf_xdp.c)을 참고할 수 있다.

프로그램 입력이 패킷인 건 마찬가지인데 타입이 단촐하다.

```c
struct xdp_md {
        __u32 data;
        __u32 data_end;
};
```

`struct sk_buff`를 만들기도 전에 XDP 프로그램을 실행하기 때문이다. `tc`에서와 비슷하게 `ip link set dev ... xdp ...` 명령으로 장치에 프로그램을 붙이는데, 장치 드라이버에 XDP 지원이 구현돼 있으면 [수신 루틴 초입](https://github.com/torvalds/linux/blob/v4.14/drivers/net/ethernet/intel/i40e/i40e_txrx.c#L2124)에서 프로그램을 실행하고 아니면 [좀 더 위](https://github.com/torvalds/linux/blob/v4.14/net/core/dev.c#L3993)에서 실행한다.

장치 드라이버마다 XDP 관련 루틴을 구현해야 하고 `struct net_device`에 연산이 세 가지(`ndo_xdp`, `ndo_xdp_xmit`, `ndo_xdp_flush`)나 추가됐다. 이런 아름답지 못한 모양새를 감수하면서까지 얻으려는 건 결국 [효율적인 대체 네트워크 스택 구현]({{ site.baseurl }}{% post_url 2017-11-30-efficient-network-device %}) 가능성이다. 그런 점에서 XDP는 DPDK와 겹치는 면이 있는데, 더 쉽게 진입할 수 있지만 첫 걸음 너머가 좁고 험하다. BPF가 가능성인 만큼 제약이기도 하기 때문이다. 효율성과 가변성이 중요한 단순한 기능을 빠르게 구현해야 할 때 좋은 선택일 수 있다.

##### `BPF_PROG_TYPE_SOCK_OPS` - TCP 스택 동작 조정

`BPF_PROG_TYPE_XDP`와 `BPF_PROG_TYPE_SK_SKB`가 BPF를 주재료 삼아 새로 뭔가를 만드는 거라면 `BPF_PROG_TYPE_SOCK_OPS`는 기존에 가렵던 지점에서 BPF를 영리하게 이용하는 것이다. TCP 같은 프로토콜에는 동작 환경이나 상황에 따라 적당한 값이 다른 매개변수들이 있기 마련인데, 한 예가 망 환경 발전에 따라 [야](https://tools.ietf.org/html/rfc2001)[금](https://tools.ietf.org/html/rfc2414)[야](https://tools.ietf.org/html/rfc3390)[금](https://tools.ietf.org/html/rfc6928) 커지고 있는 최초 윈도 크기이다. 이런 매개변수를 조정하는 여러 방법들이 있지만 각기 한계나 불편함이 있다 ([커밋 메시지](https://github.com/torvalds/linux/commit/40304b2a1567fecc321f640ee4239556dd0f3ee0) 참고). BPF 프로그램으로 매개변수를 조정할 수 있으면 가령 데이터 센터 내 연결에만 실험적인 성능 지향 매개변수 값을 사용하는 게 가능할 것이다. 더 나아가 연결의 주요 상태 변화 지점에서 BPF 프로그램을 실행할 수 있고 거기서 `setsockopt()`를 호출할 수 있으면 더 다양한 조작이 가능하다.

프로그램 입력은 `struct bpf_sock_ops`이고 반환 값의 해석 방식은 동작 위치(`BPF_SOCK_OPS_*`)에 따라 다르다. `linux/samples/bpf/tcp_*_kern.c` 파일들을 참고할 수 있다. 그런데 프로그램을 붙이는 대상이 cgroup이다. 즉, 원하는 프로세스들의 그룹을 만들어서 거기 BPF 프로그램을 붙여 놓으면 그 프로세스들이 생성한 소켓에서 프로그램이 실행된다. 상당히 편리한 대상 지정 방식인데, 이어지는 세 종류도 cgroup 기반이다.

##### `BPF_PROG_TYPE_CGROUP_SKB` - IP 패킷 필터링

`BPF_PROG_TYPE_SOCKET_FILTER`의 개선판이다. 일단 이름처럼 cgroup 단위로 대상을 지정할 수 있다. `AF_INET`/`AF_INET6` 소켓에만 적용되는데, 붙이는 위치(`attr->attach_type`)가 두 가지(`BPF_CGROUP_INET_{INGRESS,EGRESS}`)이다. INGRESS는 실행 위치가 기본적으로 `BPF_PROG_TYPE_SOCKET_FILTER`와 같지만 반환 값 의미론이 다른데, 1이면 통과이고 아니면 버린다. EGRESS로 붙인 프로그램은 패킷 출력 경로 중간쯤(넷필터 POST_ROUTING 직후)에서 실행된다. 마찬가지로 1을 반환하면 통과이고 아니면 버린다. 프로그램 입력은 둘 모두 `struct __sk_buff`이다.

특정 프로세스들이 소켓으로 주고받는 패킷을 통제할 수 있으니 개인 방화벽 만드는 데 써먹는 걸 생각해 볼 수 있다. 일단 대상 프로세스 관리는 넷필터보다 편할 것 같은데 사용자 공간으로 비동기 알림을 보낼 방법이 마땅찮다. 여담으로 세션별 데이터 사용이 가능해지면 (가령 소켓이나 conntrack마다 따로 할당된 워드가 있어서 읽기와 쓰기가 가능하다면) 더 다양한 네트워크 응용이 가능할 것 같다.

##### `BPF_PROG_TYPE_CGROUP_SOCK` - 소켓 생성 제어

`AF_INET`/`AF_INET6` 소켓 생성 과정 마지막에 실행되는 프로그램이다. 1을 반환하면 생성을 허용하고 아니면 막는다. 프로그램 입력(`struct bpf_sock`)에 담긴 정보가 적어서 세밀한 필터링은 어려워 보인다.

##### `BPF_PROG_TYPE_CGROUP_DEVICE` - 장치 파일 접근 제어

간만에 네트워크 외 분야이다. [커널 4.15에 추가](https://github.com/torvalds/linux/commit/ebc614f687369f9df99828572b1d85a7c2de3d92)될 타입이다. 장치 파일 생성/읽기/쓰기를 추가적으로 통제할 수 있는 프로그램이다. 프로그램 입력은 `struct bpf_cgroup_dev_ctx`이며, 1을 반환하면 허용이고 아니면 거부이다.

##### `BPF_PROG_TYPE_LWT_IN`, `BPF_PROG_TYPE_LWT_OUT`, `BPF_PROG_TYPE_LWT_XMIT` - 경량 터널링

[경량 터널링](https://lwn.net/Articles/650778/)(lightweight tunneling)을 얘기하려면 먼저 기존 터널 얘기를 해야 한다. 전통적 터널에서는 가상 장치를 만들어서 그 장치로 트래픽을 라우팅 한다. 장치 출력 루틴에서 캡슐화가 이뤄지고 결과 패킷을 다시 라우팅 해서 물리적 장치로 내보낸다. 수신 쪽도 비슷하고, 그래서 패킷 송수신 때 네트워크 스택을 두 번 거치는 셈이 된다. 하지만 캡슐화/역캡슐화 과정이 간단한 헤더 붙이고 떼는 게 전부라면, 그리고 결과 패킷을 내보낼 물리적 장치가 미리 정해져 있다면 훨씬 간단한 동작 구조가 가능하다. 가상 장치 같은 건 잊어 버리고 라우트 입출력 함수(`dst->input`, `dst->output`)를 이용하는 것이다.

```
ip route add 10.1.1.0/30 encap mpls 200 via inet 10.1.1.1 dev swp1
```

사용자 공간에서 위와 같은 식으로 '터널 설정'을 한다. 그러면 커널 내 해당 터널 모듈에서 `struct lwtunnel_encap_ops`로 등록해 둔 핸들러 함수들이 다음 위치에서 호출된다.

 * `.input`: `dst_input()`
 * `.output`: `dst_output()`
 * `.xmit`: 네트워크 계층 떠나기 직전

경량 터널링이 가능할 정도로 단순한 캡슐화/역캡슐화 로직이라면 BPF 프로그램으로 구현하는 게 가능할 수도 있다. 위의 세 지점이 각각 IN/OUT/XMIT 타입에 대응한다. 프로그램 입력은 `struct __sk_buff`이고, 반환 값에 따라 패킷 처리를 계속하거나(`BPF_OK`), 중단하거나(`BPF_DROP`), 출력 장치를 바꾼다(`BPF_REDIRECT`).

패킷 처리 경로 중간에서 패킷을 조작한다는 면에서 xfrm 프레임워크와도 통한다. LWT 프로그램으로 IPsec을 구현할 수 있을까? 암호 알고리즘은 커널 안에 있는 걸 헬퍼 함수 형태로 쓰면 될 테고 SA는 맵에 저장하면 된다. (동시 접근이 신경 쓰이지만 모른 척하자.) 수신 윈도 처리 같은 것도 무난하게 구현할 수 있을 테고 자잘한 몇몇 기능들은 포기하면 그만이다. 좋다. 가능성을 확인했으니 이제 xfrm을 잘 이용하면 된다.

##### `BPF_PROG_TYPE_KPROBE`, `BPF_PROG_TYPE_TRACEPOINT`, `BPF_PROG_TYPE_PERF_EVENT` - 실행 추적

[Kprobes](https://lwn.net/Articles/132196/)를 이용하면 [커널 디버깅/추적/계측](https://www.ibm.com/developerworks/library/l-kprobes/index.html)을 할 수 있다. 프루브 지점을 설정하면, 그래서 그 위치의 인스트럭션이 [중단점 내지 점프 인스트럭션으로 교체](https://wariua.cafe24.com/wiki/Documentation/kprobes.txt#kprobe.EB.8A.94_.EC.96.B4.EB.96.BB.EA.B2.8C_.EB.8F.99.EC.9E.91.ED.95.98.EB.8A.94.EA.B0.80.3F)되고 나면 커널 실행 흐름이 거길 지날 때 미리 등록해 둔 핸들러가 호출된다. 그러면 자연스럽게 뒤따르는 확장은 [BPF 핸들러를 실행](https://github.com/torvalds/linux/blob/master/kernel/trace/bpf_trace.c)할 수 있게 하는 것이다.

커널을 대상으로 하는 Kprobe와 사용자 프로세스를 대상으로 하는 Uprobe에서 핸들러로 쓸 수 있는 프로그램이 `BPF_PROG_TYPE_KPROBE`이다. 커널 모듈 작성 없이 미리 정해둔 지점들을 간편하게 조사할 수 있는 Tracepoint와 시스템 호출 추적 메커니즘에 사용할 수 있는 프로그램이 `BPF_PROG_TYPE_TRACEPOINT`이다. 그리고 `perf` 등으로 이벤트 발생 통계를 얻는 데 쓸 수 있는 프로그램이 `BPF_PROG_TYPE_PERF_EVENT`이다. 좀 뜬금없어 보이는 맵 타입 `BPF_MAP_TYPE_STACK_TRACE`와 헬퍼 함수 `bpf_get_stackid()`가 쓰이는 게 이쪽이기도 하다. <tt>[perf_event_open()](https://github.com/wariua/manpages-ko/wiki/perf_event_open%282%29)</tt>으로 얻은 디스크립터에 `ioctl(PERF_EVENT_IOC_SET_BPF)`로 프로그램을 붙인다. `linux/samples/bpf/`에 간단한 예시가 있다.
