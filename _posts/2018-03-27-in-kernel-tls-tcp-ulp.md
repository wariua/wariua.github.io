---
layout: post
title: 커널 내 TLS - TCP ULP
category: facility
tags: [tls, crypto]
---
## 커널 내 TLS

[BPF 얘기]({{ site.baseurl }}{% post_url 2018-01-25-extended-bpf %})에서 KCM을 짧게 설명하면서 [리눅스 커널 내 TLS 프로토콜 구현](https://wariua.cafe24.com/wiki/Documentation/networking/tls.txt)을 슬쩍 언급했다. 비슷한 프로토콜들처럼 TLS에도 세션을 수립하는 부분과 상위 계층 데이터를 주고받는 부분이 있는데, 이 구현의 목표는 응용 데이터 송수신을 (정확히는 레코드 계층 처리를) 커널에서 수행하는 것이다. 사용자 공간 응용에서는 이전처럼 TLS 핸드셰이크를 수행하고서 그 결과로 나온 암호화 키 등의 보안 매개변수를 `setsockopt()`로 커널로 전달하면 된다. 그러면 평문으로 `send()`/`recv()` 할 수 있게 된다. 리눅스 4.15 기준으로 송신 방향만 구현돼 있다. 암호 스위트로 CIPHER_AES_128_GCM만 지원하며 키 갱신을 지원하지 않는다.

모듈의 주 저자는 [Mellanox](http://www.mellanox.com/)라는 회사다.

`linux/net/tls/tls_main.c`:
```c
MODULE_AUTHOR("Mellanox Technologies");
MODULE_DESCRIPTION("Transport Layer Security Support");
MODULE_LICENSE("Dual BSD/GPL");

...

static int do_tls_setsockopt_tx(...)
{
        ...

        /* currently SW is default, we will have ethtool in future */
        rc = tls_set_sw_offload(sk, ctx);
        prot = &tls_sw_prot;
        if (rc)
                goto err_crypto_info;

        sk->sk_prot = prot;
        goto out;

        ...
}
```

이중 라이선스인 게 눈에 띈다. 그리고 코드를 보면 `tls_set_hw_offload()` 같은 함수가 어디 있을 것 같은 모양새인데, [여기에](https://github.com/Mellanox/tls-offload/blob/tls_device_v3/net/tls/tls_device.c) 있다. 그 구현은 Mellanox의 [일부 NIC 제품군](http://www.mellanox.com/page/programmable_network_adapters)에 있는 TLS 오프로드 기능([관련 브랜치](https://github.com/Mellanox/tls-offload/tree/tls_device_v3))을 이용한다. 레코드 계층 전체가 하드웨어로 내려가는 건 아니고, 형식 맞춰서 평문 레코드를 전송하면 NIC에서 암호화 하고 인증 값 채워서 내보낸다. 요컨데 어느 회사가 자기네 제품 팔려고 커널에 관련 코드를 슬쩍 집어넣은 걸로 볼 수도 있다. 한편 Mellanox는 DPDK 프로젝트의 [골드 멤버](http://dpdk.org/about)이고 mlx{4,5} 드라이버를 제공하고 있기도 하다.

TLS/SSL을 커널에 넣으려는 시도가 처음은 아니다. [커널 문서](https://wariua.cafe24.com/wiki/Documentation/networking/tls.txt) 말미의 링크를 따라가 보면 Mellanox의 프로젝트를 거쳐 [af_ktls](https://github.com/ktls/af_ktls)라는 프로젝트에 닿는다. 마찬가지로 레코드 계층을 커널에서 구현한 것인데, ULP를 쓰는 대신 `AF_KTLS`라는 주소 패밀리를 도입하며 (`SOCK_DGRAM`이면? DTLS!), 송신뿐 아니라 수신도 구현하고 있다. (수신 메시지 파싱에 [strparser](https://wariua.cafe24.com/wiki/Documentation/networking/strparser.txt)를 이용한다.) 이 프로젝트 개발이 멈추고 반 년 정도 후에 리눅스 커널에 TLS 모듈(이하 ktls)이 등장했는데 양쪽 개발자가 좀 겹친다. 그리고 비슷한 시기에 관련 프로젝트 [af_ktls-tool](https://github.com/ktls/af_ktls-tool)이 [Mellanox의 프로젝트](https://github.com/Mellanox/tls-af_ktls_tool)가 됐다. 즉, af_ktls는 ktls의 직계존속이다.

먼지내가 좀 나기는 하지만 [kssl](http://www.ksl.ci.kyutech.ac.jp/~kourai/research/kssl/kssl.html)이라는 것도 있다. `setsockopt()`를 호출해서 'TLS 모드'로 전환하는 방식은 ktls와 비슷한데, 좀 더 급진적이다. OpenSSL을 통째로 커널에 집어넣고 핸드셰이크까지 커널에서 수행한다. 키 쌍과 CA 인증서 등을 설정할 방법이 필요하고, 그래서 전용 시스템 호출도 하나 추가한다. "어떻게 하면 응용 코드 변경을 가급적 줄이면서 TLS 지원을 추가할 수 있는가?"라는 고민의 (실험적인) 결과물이며 다른 변주로 `LD_PRELOAD`를 이용하는 [libsslwrap](http://www.ksl.ci.kyutech.ac.jp/~kourai/research/libsslwrap/libsslwrap.html)도 있다.

비슷한 고민의 결과물이 정식으로 운영 체제에 들어간 경우도 있다. 솔라리스의 [kssl](https://docs.oracle.com/cd/E36784_01/html/E36883/kssl-5.html)은 로컬에서 도는 TLS termination proxy이다. [사용 방법](http://www.c0t0d0s0.org/archives/5575-Less-known-Solaris-Features-kssl.html)이 꽤 간단하다. 오라클 덕분에 최신 소스 코드는 못 찾겠고, 오픈솔라리스의 포크인 illumos에서 [소스 코드](http://src.illumos.org/source/xref/illumos-gate/usr/src/uts/common/inet/kssl/)를 볼 수 있다.

## TLS의 자리

왜 TLS를 커널에 넣으려고 할까?

간단하게는 운영 체제가 제공하는 서비스가 하나 더해지는 걸로 볼 수 있다. 사용자 공간 응용에서 간편하게 TLS 통신을 할 수 있게 해 주는 것이다. 전송 계층 위의 프로토콜이 커널에 들어가는 게 흔한 일은 아니지만 유구한 전통의 NFS가 있고 추억이 된 kHTTPd도 있다.

근데 TLS가 보안 프로토콜이고 협상을 하는 프로토콜이다 보니 매개변수가 많다. 제안/허용할 프로토콜/알고리즘/매개변수 조합, 신원 증명/검증을 위한 정보, 각종 확장 기능 사용 여부와 각각의 매개변수까지, 이를 커널에게 전달하기 위한 인터페이스는 복잡할 수밖에 없다. 다 떠나서, 핸드셰이크 로직 자체가 커널에 두기에는 너무 복잡(== 위험)하다.

그래서 나온 타협책이 레코드 계층만 커널에 두는 것인데, 애매한 구조만큼이나 장단점도 어정쩡하다. 물론 IPsec/IKE라는 전례가 있기는 하다. 하지만 IPsec은 그 위에 다시 네트워크 계층이 있다 보니 사용자 공간으로 올리는 게 네트워크 스택을 통째로 올리는 일이 되고, 그래서 일반적으로 커널 구현이 유일한 선택지다. 하지만 TLS 레코드 계층 위에는 다른 TLS 하위 계층과 응용 계층이 있을 뿐이다.

성능 쪽은 어떨까? 암호 연산을 소프트웨어로 구현한다면 어디서 돌든 소모 클럭 수가 다르지 않으며 응용에서 여전히 `recv()`/`send()`를 호출한다면 문맥 전환도 줄지 않는다. 뒤집어 생각하면, 암호 장치 사용을 위해 `/dev/crypto` 같은 인터페이스를 거쳐야 하는 경우에는 커널 내 구현으로 오버헤드를 없앨 수 있으며, TLS가 커널에 있으면 [sendfile()](https://wariua.github.io/man-pages-ko/sendfile%282%29) 같은 걸로 문맥 전환을 줄이는 게 가능해진다.

성능이 얼마나 좋아질까? [af_ktls 소개 논문](https://netdevconf.org/1.2/papers/ktls.pdf)을 보면 `sendfile()` 사용 시 CPU 사용량이 4~7% 정도 떨어진다고 한다. [AES-NI](https://ko.wikipedia.org/wiki/AES-NI)를 쓰는 경우가 그렇고 암호 연산 수행 방식에 따라 상대적 효과는 달라질 것이다.

TLS의 일부 내지 전체를 커널에 집어넣는 건 커널-사용자 공간 경계를 응용 쪽으로 옮기는 것이다. 문맥 경계가 (즉 오버헤드가) 아직 남아 있다. 응용과 NIC 사이 구간에서 경계를 아예 없애려면 응용까지 커널로 내리거나 반대로 드라이버와 네트워크 스택을 사용자 공간으로 올리면 된다. 개발 비용 관점에서 할부와 일시불에 해당하는 셈인데 (이율은 케바케), 후자의 예로는 요새 한창 [TLS 지원](https://wiki.fd.io/view/VPP/HostStack/TLS)을 추가하고 있는 VPP가 있다. ([관련](https://gerrit.fd.io/r/gitweb?p=vpp.git;a=tree;f=src/vnet/tls;h=1f2349ea297d7fdffd348383d69d0ab2913c12e8;hb=HEAD) [소](https://gerrit.fd.io/r/gitweb?p=vpp.git;a=tree;f=src/plugins/tlsmbedtls;h=b5806ecac0d1249be0c7995e6d62ce34054bbcaf;hb=HEAD)[스](https://gerrit.fd.io/r/gitweb?p=vpp.git;a=tree;f=src/plugins/tlsopenssl;h=50e37d2edea527af92e89e190076ee720a7648a4;hb=HEAD) 참고.)

## 하드웨어 가속의 자리

TLS 동작 성능에 가장 큰 영향을 끼치는 것은 암호 연산 수행 방식이다. AES-NI는 알고리즘이나 키 크기를 바꾼 것과 효과가 비슷하니 넘어가고, 그러면 전용 하드웨어 방식과 NIC 내장 방식이 남는다. 전자야 워낙에 전통적인 방식이고, 후자의 예로 앞서 등장했던 Mellanox 말고 [다른 벤더](http://interfacemasters.com/products/nics/encryption-nics/)의 제품도 있다.

NIC에서 암호 연산을 수행하는 구조의 장점은 버스 병목을 완화할 수도 있다는 것이다. 별도 하드웨어를 이용해 패킷을 내보낼 때는 데이터가 시스템 버스를 3번 (메모리 &rarr; 암호 장치 &rarr; 메모리 &rarr; NIC) 오간다. 반면 NIC에서 암호 연산을 수행하는 방식에서는 2번이다 (메모리 &rarr; NIC &rarr; 메모리). 데이터 흐름이 단순하다는 것은 소프트웨어가 단순해질 수 있다는 의미이기도 하다.

(NIC 내장 방식에서 데이터가 버스를 2번 거친다는 건 추정이다. 1번이 아니라 2번인 이유는 TCP 재전송 때문이다. 재전송을 NIC에서 처리하겠다고 송신 버퍼를 따로 유지하는 건 비현실적이니 암호 연산 수행 결과를 운영체제 TCP 스택의 송신 버퍼로 되먹이는 수밖에 없다. 한편으로 Mellanox 구현에서는 `setsockopt(ULP_TLS)` 수행 시 세션 식별자와 TCP 일련 번호, TLS 매개변수 등을 NIC에게 전달한다. 즉, NIC에서 송신 버퍼는 아니어도 세션 상태는 유지한다. 그래서 최초 전송 세그먼트와 재전송 세그먼트를 NIC에서 구별할 수 있고, 전자에만 암호 연산을 수행할 수 있다.)

NIC 내장 방식에 분명 장점이 있기는 한데 수신 시 복호화까지 가능할 것 같지는 않다. TCP 일련 번호 검사, IP 단편 처리를 생각하면 (게다가 두 데이터그램의 단편 내지 세그먼트가 겹쳐 있거나 하면...) 하드웨어에서 할 일이 아니다. NIC와의 사이에 IP 계층밖에 없는 IPsec과는 사정이 다르다. 그렇다면 NIC 내장 방식은 송신 데이터 양이 상대적으로 큰 경우(서버)에 더 매력적일 수 있다. 반대쪽 클라이언트에서야 AES-NI 정도면 충분할 테고, 미들박스에서는 이런저런 비용을 생각할 때 선뜻 손이 가지는 않는 선택지이다.

## ULP - Upper Layer Protocol

현재 ULP 프레임워크를 쓰는 모듈은 커널 TLS뿐이다. 사실 ULP 자체가 [ktls 작업 과정](https://github.com/Mellanox/tls-offload/commit/30ba54c)에서 추가된 것이다.

ULP는 프레임워크라고 부르기 민망할 정도로 단순하다. 이름을 키로 해서 콜백을 등록해 두면 이후 그 이름으로 `setsockopt(SOL_TCP, TCP_ULP, ulp_name)` 호출 시 해당 콜백이 호출된다. 이게 전부다. 나머지는 모두 콜백 안에서 알아서 해야 한다. 소켓 이벤트 콜백들(`sk_data_ready`, `sk_write_space`, `sk_error_report`)을 교체할 수도 있겠고, `struct inet_connection_sock`의 `icsk_ulp_data` 필드도 써 가며 필요한 대로 소켓을 조작하면 된다.

소켓 옵션 이름에서 알 수 있듯 TCP 전용이고 `/proc/sys/net/ipv4/tcp_available_ulp` 파일에 현재 사용 가능한 ULP 사용 모듈들의 이름이 나온다. 모듈이 아직 안 올라가 있을 때 특권(`CAP_NET_ADMIN`) 사용자가 `setsockopt(TCP_ULP)`를 호출하면 자동 적재를 시도한다.

ULP는 앞으로 어떻게 될까? ULP를 사용하는 또 다른 상위 프로토콜이 커널에 추가될 것 같지는 않다. 한편으로 Mellanox NIC을 쓰는 게 아니라면 ktls는 계륵이어서 머지않아 사라질 수도 있고, 그러면 ULP도 함께할 것이다. ULP를 ktls에 한정시킬 게 아니라 소켓 닫기 콜백 등을 더해서 [eBPF]({{ site.baseurl }}{% post_url 2018-01-25-extended-bpf %})의 `BPF_PROG_TYPE_SOCK_OPS` 같은 소켓 동작 오버라이드 메커니즘으로 발전시킬 수도 있겠다.
