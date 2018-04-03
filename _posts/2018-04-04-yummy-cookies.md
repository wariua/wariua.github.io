---
layout: post
title: 쿠키 냠냠
category: security
tags: [ddos]
---
4월 초 현재 보안 쪽 새 소식은 [BranchScope](https://arstechnica.com/gadgets/2018/03/its-not-just-spectre-researchers-reveal-more-branch-prediction-attacks/)지만 멜트다운/스펙터가 대표하는 CPU 투기 실행 기반 취약성의 또 다른 사례일 뿐이라서 위키백과 항목이나 따로 생길지 모르겠다. 네트워크 보안 쪽으로 범위를 좁혀 보면, TLS 1.3이 정식 RFC가 되기 직전이다. 8346번은 놓쳤고 8446번은 먼데 과연 몆 번을 받을까? 한편 한 달 전에는 Github을 필두로 몇몇 사이트에 memcached를 통한 대규모 증폭형 [DDoS 공격](https://it.slashdot.org/story/18/03/10/0521250/massive-ddos-attacks-are-now-targeting-google-amazon-and-the-nra)이 있었다. ([Cloudflare의 설명](https://blog.cloudflare.com/memcrashed-major-amplification-attacks-from-port-11211/) 참고.)

[증폭 공격](https://en.wikipedia.org/wiki/Denial-of-service_attack#Amplification) 자체야 새로울 게 없고 등장인물이 새로울 뿐이다. DNS와 NTP를 이용하는 공격을 뉴스에서 본 기억이 있는데 그 외에 BitTorrent, Kad, SNMP, NetBIOS 등도 이용됐던 모양이다. 모든 공격이 그렇듯 몇 가지 조건이 동시에 성립해야 공격이 성공할 수 있고, 그래서 각 조건은 방어/완화 기회를 나타내기도 한다.

 1. 패킷 출발 주소를 속일 수 있다.
 2. 중간 호스트(가령 memcached 서버)에 대한 접근 통제가 없다.
 3. 공격자가 보내는 패킷보다 중간 호스트가 응답으로 보내는 패킷이 더 크다. 즉 증폭 비율이 높다.

1번을 해결하기 위해 TCP 같은 연결 지향 프로토콜로 갈아타는 방법이 있다. 무방비로 노출된 mongodb나 etcd가 문제가 돼도 어쨌든 증폭 공격에 악용되지는 않는다. 하지만 여러 현실적 이유 때문에 선택 불가능한 경우가 많다. 한편으로 모든 망 운영자가 [ingress 필터링](https://wariua.cafe24.com/wiki/RFC2827)을 도입하면 다 끝나는 문제이기도 하다.

2번은 서버 운용 측면의 문제다. memcached는 비공개 망에서의 사용을 가정하고 있기 때문에 프로그램이나 프로토콜에 어떤 방어 메커니즘도 없다. (redis에는 원시적이나마 [인증 기제](https://redis.io/commands/auth)가 있다. 사람들이 그걸 얼마나 쓰느냐는 또 다른 문제지만.) 따라서 공개된 환경에서 돌릴 때는 앞에 방화벽 같은 걸 둬야 한다. 근데 memcached는 그러면 된다 쳐도 DNS나 NTP 같은 경우에는 앞에서 막을 수도 없다. 한편으로 memcached의 기본 설정이 좀 더 조심스럽기만 했어도 (가령, 따로 지정하지 않으면 localhost에 바인드 하기) 문제가 발생하지 않았을 것이다.

3번은 응용 프로토콜 설계상의 문제인데, 딱 봐도 범용적인 대응책이 없다.

다시 돌아가서, 1번 조건에 대한 응용 프로토콜 수준의 해법이 [쿠키](https://en.wikipedia.org/wiki/Magic_cookie)다. 첫 번째 요청 패킷을 받았을 때 서버가 "자, 따라해 봐, '덈텇꾩궙룄린쑝'."이라고 응답을 보내고, 클라이언트가 잘 따라하면 이후 처리를 계속한다. 발음이나 억양이 약간이라도 틀리거나 응답이 없으면 무시한다.

쿠키를 본격적으로 쓰는 프로토콜은 [IKEv2](https://tools.ietf.org/html/rfc7296#section-2.6)나 [DTLS](https://wariua.cafe24.com/wiki/RFC6347#4.2.1._.EC.84.9C.EB.B9.84.EC.8A.A4_.EA.B1.B0.EB.B6.80_.EB.8C.80.EC.B1.85) 같은 UDP 기반 보안 프로토콜이다. 사실 여기서 쿠키로 막으려는 건 네트워크 회선 포화보다는 서버 CPU/메모리 자원 고갈이다.

```
개시자                            응답자
-------------------------------------------------------------------
HDR(A,0), SAi1, KEi, Ni  -->
                             <--  HDR(A,0), N(COOKIE)
HDR(A,0), N(COOKIE), SAi1,
    KEi, Ni  -->
                             <--  HDR(A,B), SAr1, KEr,
                                      Nr, [CERTREQ]
HDR(A,B), SK {IDi, [CERT,]
    [CERTREQ,] [IDr,] AUTH,
    SAi2, TSi, TSr}  -->
                             <--  HDR(A,B), SK {IDr, [CERT,]
                                      AUTH, SAr2, TSi, TSr}
```

```
클라이언트                                 서버
----------                                 ----
ClientHello           ------>

                      <----- HelloVerifyRequest
                             (쿠키 담고 있음)

ClientHello           ------>
(쿠키 있음)

[핸드셰이크 나머지]
```

서버로 최초 요청이 오면 세션 식별자와 쿠키만 담은 응답을 보내고, 클라이언트는 앞서 보냈던 요청에 쿠키만 추가해서 다시 보낸다. 쿠키 값은 클라이언트의 요청 메시지 내용과 서버만 아는 비밀 값을 이용해 계산한다. 재전송된 요청이 쿠키 값 검증을 통과하면 그제서야 서버에서 연결 상태를 새로 만든다.

(이걸로 주소 위조는 얼추 막을 수 있지만 보안 쪽 프로토콜에는 그 이상이 필요하다. 프로토콜 내에 아주 비싼 연산(비대칭키 암호 연산)이 있기 때문에 이를 노린 서비스 거부 공격이 가능하다. 그런 공격을 완화하려면 클라이언트도 만만찮게 비싼 연산을 수행한 후에야 서버가 비싼 연산을 수행하도록 하면 된다. 그게 [퍼](https://wariua.cafe24.com/wiki/RFC8019)[즐](http://www.csl.sri.com/users/ddean/papers/usenix01b.pdf) 메커니즘이다.)

더 오래된 사례로 TCP SYN 플러딩 공격 방어 메커니즘인 [SYN 쿠키](http://cr.yp.to/syncookies.html)가 있다. 목표가 같으니 해법도 비슷하다. 하지만 여러 현실적 제약 때문에 프로토콜 확장이 어렵기에 새로운 패킷 교환이나 옵션 도입 없이 일련 번호 필드에 쿠키를 넣는 방식을 택했고, 그래서 SYN 패킷에 있던 일부 옵션 정보(MSS)는 쿠키에 포함시켜서 보존하지만 나머지는 포기한다. [어떤 구현](https://github.com/torvalds/linux/blob/master/net/ipv4/syncookies.c)에서는 타임스탬프 값에 다른 옵션 값들을 집어넣기도 한다.

```
클라이언트               서버
----------               ----
SYN,           ----->
seq=ISN,
ack=0
               <-----    SYN/ACK,
                         seq=COOKIE,
                         ack=ISN+1
ACK,           ----->
seq=ISN+1,
ack=COOKIE+1             (세션 데이터(TCB) 생성)
```

SYN 쿠키에서는 TCP 옵션 일부를 포기해야 하고 IKEv2/DTLS에서는 메시지 교환이 한 번 추가된다. 가벼운 비용이 아니고, 그래서 평상시에는 쿠키 없이 동작하다가 특정 상황에서만 (가령 미완료 연결 개수가 기준치를 넘었을 때에만) 동작하게 할 수 있다.

많이 쓰는 프로토콜일수록 새 메커니즘을 추가하기가 까다롭다. 그런 면에서 TCP에 못지 않은 게 DNS다. 사실 주소 위조 방지만 놓고 보면 [DNS over TCP](https://tools.ietf.org/html/rfc7766)가 오래 전부터 표준 후보였으니 그걸 쓰면 된다. [1.1.1.1](https://1.1.1.1/) 같은 (캐싱) DNS 서버 대부분은 TCP로도 연결이 가능하다.

```
$ host -T wariua.github.io 1.1.1.1
Using domain server:
Name: 1.1.1.1
Address: 1.1.1.1#53
Aliases: 

wariua.github.io is an alias for sni.github.map.fastly.net.
sni.github.map.fastly.net has address 151.101.229.147
sni.github.map.fastly.net has IPv6 address 2a04:4e42:36::403
```

문제는 TCP 연결 수립으로 인한 지연이다. TLS 1.3에서도 RTT를 줄이려고 그렇게 노력하는 시대인데 왕복 1회 추가는 너무하다. [TCP Fast Open](https://wariua.cafe24.com/wiki/RFC7413)을 쓰면 다시 줄일 수 있지만 아직 실험 단계인 프로토콜이다. 결국 기존 UDP 기반 프로토콜에 옵션 형태로 쿠키를 집어넣는 수밖에 없고, 그게 [DNS COOKIE 옵션](https://tools.ietf.org/html/rfc7873)이다.

DNS는 앞서 언급한 IKEv2/DTLS와는 여러 점에서 다른데, 완전한 무상태 프로토콜이고 트랜잭션이 아주 가볍다. 그리고 일반적으로 응답 메시지가 더 크다. 그래서 완화의 주안점도 서버 자원 고갈이 아니라 증폭을 통한 네트워크 포화이다. 또한 워낙에 단순한 프로토콜이기 때문에 추가 왕복이 가능하면 없어야 한다. 그래서 DNS 쿠키에서는 TCP Fast Open처럼 캐싱을 한다. 즉 첫 번째 통신 때는 추가 메시지 교환을 하며 새 쿠키를 얻고, 그걸 클라이언트가 로컬에 저장해 뒀다가 다음 요청 때 사용한다. 그 사이 별일(클라이언트 IP 주소 변경, 긴 시간 경과)이 없어서 쿠키가 유효하면 서버가 바로 응답을 보낸다. 보통 각 클라이언트가 한두 대의 DNS 서버와 통신하므로 대부분은 추가 왕복 없이 통신이 이뤄진다. 여담으로 서버가 구워 준 쿠키를 클라이언트가 고이 간직하고 있다가 다시 보낸다는 점에선 뜬금없지만 HTTP의 쿠키와 비슷한 면이 있다.

```
클라이언트               서버
----------               ----
REQUEST         ----->
Client Cookie
                <-----   REPLY, RCODE=BADCOOKIE
                         Client Cookie
                         Server Cookie
REQUEST         ----->
Client Cookie
Server Cookie
                <-----   REPLY, RCODE=NOERROR
                         Client Cookie
                         Server Cookie

                  ...

REQUEST         ----->
Client Cookie
Server Cookie
                <-----   REPLY, RCODE=NOERROR
                         Client Cookie
                         Server Cookie
```

이걸 기본으로 해서 서버의 동작을 조정할 수 있다. 가령 서버 부하가 낮은 동안에는 클라이언트가 유효한 서버 쿠키를 보내지 않은 경우에도 바로 응답을 보내줄 수 있을 것이다. 위 그림에서 두 번째와 세 번째 단계를 생략하는 셈이다. 반대로 부하가 많이 올라갔을 때는 유효한 쿠키가 담겨 있지 않은 요청을 일정 확률로 무시할 수 있을 것이다.

NTP 쪽은 어떨까? NTP를 이용한 증폭 공격은 MONLIST라는 "그런 게 있었어?" 싶은 명령을 악용했던 거라서 그 명령을 막는 걸로 해결됐다. ([Cloudflare의 설명](https://www.cloudflare.com/learning/ddos/ntp-amplification-ddos-attack/) 참고.) 한편으로 [NTS](https://tools.ietf.org/html/draft-ietf-ntp-using-nts-for-ntp)라는 게 준비 중인데, NTP용 간단 DTLS다. 여기선 "절대 요청보다 큰 응답을 보내지 않는다"는 원칙을 통해 증폭 공격을 방지한다. NTS에도 "쿠키"라는 게 등장하는데 TLS의 [세션 티켓](https://wariua.cafe24.com/wiki/RFC5077)에 대응한다. 마찬가지로 소중히 간직하다가 내용을 들여다보지도 않고 그대로 돌려보낸다. 매정한 프로토콜들 같으니라고.
