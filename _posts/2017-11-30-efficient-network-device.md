---
layout: post
title: 효율적인 네트워크 장치 만들기
category: performance
tags: [dpdk]
---
## 배경 흐름

스위치, 방화벽, 로드 밸런서, ... 이런 애들은 응용이 동작하는 환경의 한 요소인 망을 구성하는 장치이다. 빛 안 드는 구석에서 조용히 제 할 일 하면 되는 장치라서 비용 효율성에 대한 압력이 좀 있는 편이다. 물론 IoT 쪽 장치들에 비할 바는 아니고 돈 많은 데서는 비싸도 잘 쓴다. 일부 단순 스위치들을 제외하면 결국 하드웨어 위에서 소프트웨어가 도는 형태이다. 전통적으로는 펌웨어라는 덜 말랑한 형태지만 요새 제품들 중에는 [OpenWrt](https://openwrt.org/)/[LEDE](https://lede-project.org/)처럼 리눅스 배포판 형태인 것도 있다.

길지 않은 컴퓨팅 역사 동안 하드웨어의 MIPS당 가격은 기술 축적과 규모의 경제에 힘입어 극적으로 떨어졌다. 하지만 하드웨어든 소프트웨어든 새로운 걸 만들어내는 비용은 그리 줄어들지 않았다. 그렇다 보니 하드웨어를 새로 만들기보다 기성품 하드웨어를 활용하는 게 나은 경우가 많다. 한편으로 소프트웨어 공학의 바람에도 불구하고 소프트웨어는 기예에서 산업으로의 전환을 완전히 이루지 못했다. 그리고 아직은 인간의 범용 문제 해결 능력을 대체할 존재가 등장하지 못했고, 그렇다고 그간 인간의 인지 능력이 향상된 것도 아니다. 그래서 소프트웨어는 하드웨어에 비해 비싸진다. 하드웨어를 팔면서 끼워 주던 OS와 DBMS는 이제 서버 하드웨어보다 비싸다.

다른 한편으로 하드웨어 성능의 증가는 메인프레임 정도에서나 쓰던 가상화를 일반화시켰고, 점점 영역을 넓히는 클라우드 역시도 범용 하드웨어 상의 소프트웨어 구현을 선호한다. 기존 인프라 위에서 다른 VM들처럼 다룰 수 있는 VNF도 좋고, 기능을 뽑아내서 마음껏 만지작거릴 수 있게 만든 SDN도 좋다.

결국 박리다매형 제품을 만들려는 경우나 기존 시장의 선두 위치를 사수하려는 경우 정도를 제외하면, 그리고 반대로 판매량이 불확실한 신규 제품을 빠르게 만들고 싶다면, 전용 하드웨어를 만들려고 하기보다는 범용 하드웨어 상에서 소프트웨어로 네트워크 기능을 구현하는 게 합리적인 선택이다.

## 효율적인 네트워크 기능을 효율적으로 구현하기

전통적으로 네트워크 장치의 기능성을 데이터 플레인과 컨트롤 플레인으로 나눈다. 더 많은 기능을 담고, 그래서 더 복잡하고, 게다가 더 자주 바뀌는 컨트롤 플레인은 일반적인 소프트웨어 개발 방식에 따라 만들면 된다. 하지만 장치의 주요 성능을 결정하는 데이터 플레인에서는 동작 효율성을 많이 고민해야 한다. 그래서 질문은 이렇다: 효율적으로 동작하는 소프트웨어 데이터 플레인을 구현할 수 있는 방법은 무엇인가?

인스트럭션을 한땀 한땀 이어붙여서 만들면 몇 년 걸리더라도 괜찮은 성능의 결과물이 나올 것이고, 가능한 모든 인스트럭션 조합을 만들어서 실험해 본다면 열죽음 전에는 어렵겠지만 최고의 결과물이 나올 것이다. 그래서 다시 질문은 이렇다: 효율적인 소프트웨어 데이터 플레인을 쉽게 구현할 수 있는 방법은 무엇인가?

기존 운영체제의 커널을 활용할 수 있다. 이미 많은 게 갖춰져 있으니 시작이 편하다. 하지만 범용 운영체제는 수많은 타협의 결과물이라서 다른 가치를 위해 효율성을 희생시킨 설계가 있기 마련이다. 그리고 많은 게 갖춰져 있다는 건 뭔가를 바꿀 때 고려할 게 많다는 것이기도 하다. 더 구체적인 측면에서 보자면 작업 결과를 확인하기 위해 재부팅이 필요하고, 시스템이 죽었을 때 디버깅 정보를 수집하기 어렵고, 출시 된 시스템이 죽을 때를 대비해 와치독 장치가 필요하다. 커널 모듈 형태로 만들고 kgdb와 kdump 등을 활용할 수도 있지만 언제나 가능하지도 않고 번잡한 데다 이런저런 한계가 있다.

구현하려는 네트워크 기능이 패킷이 아니라 데이터 스트림에 대해 동작한다면 (가령 WAF를 만든다면) 또 다른 종류의 문제가 생긴다. 일단 간단하게는 소켓 API를 쓰는 사용자 공간 프로세스 형태로 만들 수 있을 것이다. 그러고 나니 문맥 전환이 빈번히 일어나고 커널 소켓 버퍼와 응용 버퍼 사이에서 데이터 복사가 필요하다. 패킷 입출력이 이뤄지는 공간과 응용이 동작하는 공간이 분리되어 있으니 제어와 데이터가 그 경계를 넘느라 오버헤드가 생기는 것이다. 어떻게 그 오버헤드를 없앨 수 있을까? 

응용을 커널 공간으로 옮기고 커널 버퍼에 직접 접근하게 할 수 있을 것이다. 한발 더 나아가 범용 운영체제를 버리고 [유](https://en.wikipedia.org/wiki/Unikernel)[니](https://github.com/rumpkernel)[커](https://github.com/sysml/clickos)[널](https://github.com/libos-nuse)을 사용할 수도 있을 것이다. 하지만 어느 쪽이든 결과는 낮은 개발 효율이다. 더 많이 얻을 수 있지만 더 많은 비용을 치러야 한다.

응용이 도는 사용자 공간으로 패킷 입출력 기능을 옮길 수도 있다. 가령 리눅스 커널의 [사용자 공간 I/O](https://wariua.cafe24.com/wiki/Documentation/driver-api/uio-howto.rst) 메커니즘을 이용하면 하드웨어 장치의 레지스터들을 `mmap()`으로 접근할 수 있고 장치 인터럽트를 `poll()`/`read()`/`write()` 할 수 있다. 즉, 사용자 공간에서 네트워크 장치를 제어할 수 있다. 그리고 응용 프로그램과 네트워크 장치가 공유하는 메모리를 이용하면 복사 없이 패킷 데이터를 주고받을 수 있다. UIO 내지 기타 메커니즘과 공유 메모리를 결합해서 사용자 공간에서 빠른 패킷 입출력을 가능하게 하는 기술들로 [DPDK](http://dpdk.org/), [Netmap](http://info.iet.unipi.it/~luigi/netmap/), [PF_RING](http://www.ntop.org/products/packet-capture/pf_ring/) [등등](https://blog.cloudflare.com/kernel-bypass/)이 있다. 적은 비용으로 많이 얻을 수 있는 방법들이다. 개발을 편리하게 해 주는 여러 메커니즘과 도구들을 그대로 이용할 수 있고 VM 없이도 테스트 자동화가 가능하다. 그러면서도 타이머 인터럽트 정도만 남기고 문맥 전환 발생을 완전히 없앨 수 있다.

개중 제일 잘 나가는 DPDK를 가지고 얘기를 해 보자. DPDK는 응용 프로그램에 링크 할 수 있는 라이브러리 형태이다. 여러 네트워크 장치들을 위한 드라이버, 장치와 패킷 등을 다루는 루틴, 타이머와 암호 연산 장치를 다루는 루틴, 기타 보조 루틴들을 제공한다. [지원하는 NIC들](http://dpdk.org/doc/nics)은 Intel과 Cavium, NXP의 여러 모델, 그리고 Mavell, Cisco, Amazon(?!) 등의 몇몇 모델이다. 또 Intel, Cavium, NXP 등의 암호 연산 하드웨어를 지원한다.

DPDK 사용 프로그램의 기본 뼈대는 다음과 같다. 초기화와 구성, 작업 스레드 시작, 각 스레드에서의 루프 돌기라는 평범한 구조다.

```c
int main(int argc, char *argv[])
{
    /* 각종 초기화 */
    rte_eal_init(argc, argv);
    pktmbuf_pool = rte_pktmbuf_pool_create("mbuf_pool", ...);

    for (portid = 0; portid < nb_ports; portid++) {
        rte_eth_dev_configure(portid, 1, 1, &port_conf);
        rte_eth_rx_queue_setup(portid, 0, nb_rxd, rte_eth_dev_socket_id(portid),
                               NULL, pktmbuf_pool);
        ...
    }
    ...

    rte_eal_mp_remote_launch(lcore_proc, NULL, CALL_MASTER);

    RTE_LCORE_FOREACH_SLAVE(lcore_id)
        rte_eal_wait_lcore(lcore_id);
    ...
}

static int lcore_proc(void *dummy)
{
    ...

    while (1) {
        for (i = 0; i < qconf->n_rx_port; i++) {
            nb_rx = rte_eth_rx_burst(portid, 0, pkts_burst, MAX_PKT_BURST);

            for (j = 0; j < nb_rx; j++) {
                struct rte_mbuf *m = pkts_burst[j];
                /* 패킷 처리... */
            }
        }
    }

    return 0;
}
```

기본적으로 폴링(즉 busy waiting) 방식으로 패킷을 수신하며 코어에 고정(affinity)된 작업 스레드에서 run-to-completion 방식으로 작업을 수행한다. 이 모두는 문맥 전환 오버헤드를 없애기 위해서다. 그리고 공유 메모리 사용을 통해 패킷 복사 오버헤드도 없앴다. 이제 구현하려는 기능과 무관한 오버헤드는 거의 사라졌다. 좋다, 그런데, 어떻게 하면 성능을 더 높일 수 있을까?

위 코드에서 볼 수 있듯 DPDK에서는 한번에 여러 패킷을 수신하고 송신할 수 있다. 보통 때는 `rte_eth_rx_burst()`를 호출해도 패킷 1개만 (더 빈번하게는 0개가) 올라오겠지만 패킷 도착 간격이 짧아지면 그 수가 늘어난다. 패킷마다 거치던 어떤 실행 경로를 모아서 한 번만 거치니 실행 인스트럭션 수가 줄어들고, 그러면 스루풋이 올라간다. 쳐낼 오버헤드가 없으니 실행 인스트럭션 수를 줄여서 성능을 높이려는 시도이다.

[TSO/GSO/LRO/GRO](https://lwn.net/Articles/358910/) 메커니즘도 비슷한 노력이라고 볼 수 있다. 소프트웨어 에뮬레이션에서도 성능 향상 효과가 있는 건 한번에 처리하는 데이터가 커지면서 단위 데이터 크기당 인스트럭션 수가 줄기 때문이다. 다른 종류의 시도로 [I/OAT](https://www.intel.com/content/www/us/en/wireless-network/accel-technology.html)의 QuickData Technology 같은 기법이 있다. `memcpy()`를 오프로드 하는 건데, 뭔가 아리까리한 적용 범위도 그렇지만 애초에 `memcpy()`가 필요 없게 만들 수 있으면 그게 더 낫다.

문맥 전환 같은 오버헤드를 없애는 작업은 구조를 다루는 것이라서 한번에 큰 효과를 얻는 경우가 많다. 하지만 실행 인스트럭션을 줄이는 건 쪼잔하고 지난한 작업인 데다가 기대할 수 있는 효과가 한정적이다. 그건 그렇고, 어떻게 하면 성능을 더 높일 수 있을까? `__builtin_expect()`를 적절히 사용하고 컴파일러에게 대상 아키텍처를 구체적으로 알려 주면 조금 더 최적화 된 코드를 얻을 수 있다. 또 NUMA 시스템이라면 가급적 로컬 메모리에만 접근하도록 해서 메모리 접근 비용을 낮출 수 있다.

다음은 캐시이다. 그간 CPU 클럭 속도가 열심히 올라가다 보니 [캐시 미스의 비용](https://people.eecs.berkeley.edu/~rcs/research/interactive_latency.html)이 커졌다. 어떻게 캐시 미스를 줄일 수 있을까? 당연하지만 프로그램 코드와 접근 데이터의 크기를 줄이고 시간적/공간적 지역성을 높이면 캐시 미스가 줄어든다. 하지만 캐시까지 고려할 시점이면 코드나 데이터를 줄일 수 있는 여지가 별로 남아 있지 않다.

네트워크 장치에서 캐시에 드나드는 바이트들을 크게 세 종류로 나눌 수 있다.

* 패킷 데이터 (수신 데이터와 메타 데이터)
* 정적 데이터 (패킷 처리 과정에서 접근하는 각종 lookup table, 상수, 등등)
* 프로그램 인스트럭션

패킷 데이터는 크기가 크지 않고 상당수 응용에선 캐시 라인 한두 개로 충분하다. 그래서 약간의 prefetch 수행으로 캐시 미스를 없앨 수 있다. 패킷과 무관한 정적인 데이터도 prefetching으로 캐시 미스를 어느 정도 없앨 수 있다. 그런데 프로그램 인스트럭션을 읽느라 발생하는 캐시 미스는 어떻게 줄일 수 있을까?

그건 다음 포스팅에서 얘기하기로 하고, 늘어진 서론은 DPDK 관련 잡담으로 마무리...

## DPDK 여담

네트워크 장치의 주요 성능 메트릭으로 처리율(throughput)과 더불어 지연(latency)이 있다. 트레이딩처럼 1ms가 중요한 분야를 대상으로 낮은 지연을 내세우는 제품들이 있기도 하다. 그런데 가령 `rte_eth_rx_burst()`로 여러 패킷을 가져와서 처리할 때는 먼저 도착해 있던 패킷들에 추가 지연이 발생한다. 즉, 패킷 처리가 완료되는 시점이 (가령 패킷이 출력 큐로 들어가는 시점이) 단독 처리 때보다 좀 늦어진다. 대강 퉁치면, 여러 패킷을 한번에 처리하는 구간에서의 평균 처리 시간(총 시간 / 패킷 수)이 패킷 1개 단독 처리 시간의 절반이어야 평균 지연이 그대로 유지된다. 예를 들어 `rte_eth_rx_burst()`가 패킷 1개를 얻어오는 데 100사이클이 걸린다면 10개를 얻어올 때 500사이클이 넘게 걸리지 않아야 일부 패킷이 지연 증가를 경험하더라도 평균적으로 지연이 늘지 않는다. 꽤 어려운 조건이다.

작업 부하가 높아질수록 패킷당 처리 비용이 살짝 감소하는 성능 특성은 리눅스 커널의 NAPI를 닮았다. 하지만 NAPI의 그 특성은 폴링 모드로 전환하면서 IRQ 수신과 그에 수반되는 문맥 전환 비용이 사라지기 때문이다. 그렇다면 평상시에도 폴링 모드로 동작해서 높은 효율을 유지하면 될 텐데 그러지 않는 건 리눅스가 범용 OS이기 때문이다. 여러 서브시스템이 (그리고 사용자 프로세스들도) 공유하는 자원인 CPU를 마구 쓸 수 없다. 또, 들어오지도 않는 패킷을 바쁘게 기다리느라 저전력 모드로 들어갈 기회를 놓치는 것도 곤란하다.

DPDK 기반 응용이 폴링 모드로 동작하면 CPU 사용률이 쑥쑥 올라간다. 그 응용만을 위한 독립 장치에서라면 괜찮을 수도 있겠지만 다른 VM들과 CPU를 공유하는 클라우드 환경에서는 완전 민폐이다. 다행히 DPDK에서는 인터럽트 모드로 패킷을 수신할 수 있는 메커니즘도 제공한다. 패킷 수신 인터럽트를 켜고, `epoll()`로 인터럽트를 기다리고, 패킷 수신 인터럽트를 끄고, 패킷을 가져와서 처리하고, 수신 큐에 패킷이 더 없으면 다시 수신 인터럽트를 켜고, ... 이런 식으로 NAPI처럼 동작하면 된다. ([예시 프로그램](http://dpdk.org/doc/api/examples_2l3fwd-power_2main_8c-example.html) 참고.) 그런데 이때의 지연은 NAPI와 비교할 수 없을 정도로 크다. softirq 핸들러가 아니라 잠들어 있는 프로세스를 깨워서 스케줄 해야 하기 때문이다. 그래서 높은 트래픽 부하에서는 (가령 최대 성능을 측정할 때는) 스루풋과 지연 모두 훌륭한 수치를 보이는데 일상 사용 시에는 이상하게 높은 지연을 보일 수 있다.

한편으로, 위에선 스리슬쩍 넘어갔지만 DPDK가 제공하는 기능은 꽤 단촐해서 리눅스 커널 소스의 `net/core/` 정도에 해당한다. 즉, 프로토콜 스택이 포함돼 있지 않다. 그래서 DPDK에 프로토콜 스택을 결합시키려는 [여러](https://github.com/rumpkernel/drv-netif-dpdk) [시도](https://github.com/takayukiu/lwip-dpdk)[들](https://github.com/ansyun/dpdk-ans)이 있다.

효율적인 소프트웨어 데이터 플레인이라는 목표는 동일하지만 접근 방식이 다른 [OpenDataPlane](https://www.opendataplane.org/)이라는 것도 있다. DPDK가 동작하는 걸 만드는 데 집중한 결과물이라면 ODP는 일반성과 이식성을 목표로 하는 API이다. 그 API 아래에 있는 게 DPDK나 Netmap, raw 소켓일 수도 있고, 하드웨어를 얇은 라이브러리 OS로 감싼 것일 수도 있다. ODP에도 프로토콜 스택은 들어 있지 않고, 그래서 ODP를 기반으로 ARP/IPv4/IPv6, ICMP/TCP/UDP/IGMP 스택을 구현한 [OpenFastPath](http://openfastpath.org/)가 있다. ODP와 OFP, 뜻은 멋진데, 범용적이면서 효율적이기는 힘든 법이다.

[FD.io](https://fd.io/)의 [VPP](https://wiki.fd.io/view/VPP/What_is_VPP%3F)라는 것도 있다. 시스코가 기여했고 [리눅스 재단](https://www.linuxfoundation.org/) 프로젝트가 된 네트워크 응용 프레임워크이다. 일반적인 프로토콜들에 더해 VXLAN이나 IKEv2 같은 프로토콜도 (전부 또는 일부) 포함하고 있으며, 역시 리눅스 재단 프로젝트인 [OPNFV](https://www.opnfv.org/)의 구조 안에서 데이터 플레인을 맡고 있다. VPP에서 쓰고 있는 재밌는 구현 기법 두 가지를 이어지는 포스팅에서 다룬다.
