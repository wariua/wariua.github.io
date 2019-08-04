---
layout: post
title: 유령 용융 잡담
category: security
last_modified_at: 2019-05-26
---
예고 대로 [Meltdown/Spectre](https://meltdownattack.com/)가 공개되고서 인터넷 세상이 시끌시끌하다. 두 취약점에 대한 설명은 [프로젝트 제로 블로그](https://googleprojectzero.blogspot.kr/2018/01/reading-privileged-memory-with-side.html)와 두 논문([Meltdown](https://meltdownattack.com/meltdown.pdf), [Spectre](https://spectreattack.com/spectre.pdf))부터 여러 설명 글까지 많이 있으니 곁다리 잡담이나 해야겠다.

## 맥락

스토리를 좀 거슬러 올라가면 커널에서 임의 코드를 실행하려는 여러 공격들에 대한 광역 완화 기술인 [커널 ASLR](https://lwn.net/Articles/569635/)에 닿는다. 커널 코드와 데이터를 임의 주소에 배치하고 `/proc/kallsyms` 등을 통한 주소 노출을 막으면 [ROP류 공격](https://en.wikipedia.org/wiki/Return-oriented_programming)을 더 어렵게 만들 수 있다. 하지만 문맥 전환 오버헤드를 줄이기 위해 사용자 공간에 커널 메모리를 매핑 해 두는 운영체제 동작 때문에 [KASLR을 무력화](https://www.blackhat.com/docs/us-16/materials/us-16-Jang-Breaking-Kernel-Address-Space-Layout-Randomization-KASLR-With-Intel-TSX.pdf)할 수 있는 방법들이 속속 발견됐고, 그래서 제시된 게 [KAISER](https://gruss.cc/files/kaiser.pdf)다. 오버헤드를 줄이기 위한 기법들이 있지만 결국은 사용자 공간과 커널 공간의 페이지 테이블을 분리하는 것이다. 이름 그대로, 효율적으로 부채널을 없애기 위한 커널 주소 격리 방법(Kernel Address Isolation to have Side-channels Efficiently Removed)이다.

어떤 커널 주소에 내용 불명의 뭔가가 매핑 돼 있다는 걸 알아내는 건 한 걸음 전진일 뿐이다. 다음 단계는 특정 주소에 어떤 바이트가 있는지 알아내는 것이다. 그러면 괜찮은 인스트럭션 열(가젯)을 찾아내서 다른 취약점을 통해 실행하거나 운이 좋으면 중요 데이터를 얻을 수 있다. 그리고 반년 전 Anders Fogh라는 연구원이 이를 [시도해서 가능성을 확인](https://cyber.wtf/2017/07/28/negative-result-reading-kernel-memory-from-user-mode/)했다. 대부분 CPU에 다양한 형태로 구현돼 있는 투기적(speculative) 실행을 통해 읽으려는 비트 값에 따라 시스템 상태(가령 캐시 내용물)가 달라지게 만들고 메모리 접근 타이밍을 측정해서 그 상태를 알아내는 방식인데, 이게 다듬어진 게 [CVE-2017-5754](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-5754), 즉 멜트다운이다.

Fogh 씨 글의 결론부 제목이 "판도라의 상자"이다. 투기적 실행 악용이라는 상자가 열리자 연구가 이어졌고, 브랜치에서의 투기적 실행에 집중해서 나온 게 [CVE-2017-5753](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-5753)과 [CVE-2017-5715](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-5715)이고, 묶어서 스펙터이다. 두 취약점은 브랜치라는 주제를 공유하지만 메커니즘과 영향력이 꽤 다르다.

취약점들에 대한 정보가 물밑에서 CPU 벤더와 운영체제 제작자들에게 공유되어 해법 탐색이 이뤄졌다. 리눅스에서는 앞서 등장한 KAISER를 통해 멜트다운을 해결하기로 방향을 잡았는데, 내용을 다듬고 [정치적으로 올바른](https://lkml.org/lkml/2017/12/4/709) 이름을 붙인 게 [KPTI](https://en.wikipedia.org/wiki/Kernel_page-table_isolation)이다. 이름 그대로 커널 페이지 테이블 격리이다. (조금 더 자세한 설명은 [여기](https://lkml.org/lkml/2017/12/18/1523).) 멜트다운만 놓고 보면 결국 돌고돌아 다소의 비용과 함께 커널 메모리가 다시 안전하게 감춰지게 됐다는 훈훈한 얘기이다. 하지만 그 과정에서 유령이 붙었다.

## 해법

이번 문제는 원인과 공격 방법, 해법이 모두 간단 명확한 [goto fail](https://en.wikipedia.org/wiki/Unreachable_code#goto_fail_bug) 같은 문제와 다르다. 공격이 더 어렵지만 영향 받는 범위가 크고 현실적으로 가능하면서 확실한 해법이 없다. CPU에서 투기적 실행을 되돌릴 때 남는 부대 효과가 있다는 게 근본 원인인데, 캐시 상태까지 포함해 모든 걸 완벽하게 되돌리는 건 현실적으로 가능하지 않다. 그렇다고 투기적 실행을 포기하고 CPU 성능을 몇 년 전으로 되돌릴 수도 없다. 결국 공격 성공을 위한 다른 조건들을 공략하는 식으로 해결할 수밖에 없다. 멜트다운에 필요한 전제 조건(사용자 공간에 커널 메모리가 매핑 돼 있음)을 제거하는 게 KPTI이고, 쿠션을 먹여서 조작된 브랜치 예측기의 영향을 피하는 게 구글의 [Retpoline](https://support.google.com/faqs/answer/7625886)이다. 투기적 실행으로 인한 시스템 상태 변경을 측정하기 어렵게 만드는 게 [Firefox의 단기 대응책](https://www.mozilla.org/en-US/security/advisories/mfsa2018-01/)이고, 범위를 벗어난 메모리 접근을 유도당하더라도 다른 방벽으로 막는 게 CVE-2017-5753에 대한 다른 여러 해결책들이다. CPU 벤더에서 마이크로코드 패치가 나온다 해도 이런 식으로 주변 지점을 공략하지, 투기적 실행을 포기하지는 못할 것이다.

## 영향

세 문제 모두 악의적 코드가 로컬에서 실행되는 걸 전제로 한다. 다행히도 중요한 정보가 모여 있는 컴퓨터에서는 일반적으로 아무 코드나 실행하지 않는다. PC에서는 믿을 수 없는 코드 실행이 더 빈번하겠지만 (관리자 권한까지 아니어도) 일단 코드를 실행할 수 있다면 멜트다운/스펙터 말고도 사용자 정보를 훔칠 효율적인 방법이 많다. 그리고 그간 많은 취약점들이 거쳐 간 자바스크립트 코드 실행 문제는 웹 브라우저들이 무난히 방어할 것이다. 결국 기존 이상으로 특별히 신경써야 할 곳은 귀중한 정보가 있을 수 있으면서 임의 코드 실행이 가능한 서버들인데, 가장 먼저 떠오르는 게 클라우드이다.

KPTI는 멜트다운만이 아니라 KASLR 무력화에 대한 해법이기도 하다. [성능 저하](https://www.phoronix.com/scan.php?page=article&item=linux-kpti-kvm)가 수반되니 단기적으로는 경우에 따라 피할 방법을 찾으려 할 수도 있겠지만 장기적으로는 당연한 것으로 받아들일 수밖에 없다. 거기에 어쩌면 어느 스펙터 대응책의 영향까지 합쳐져서 향후 운영체제들은 지금보다 커널-사용자 문맥 전환 비용이 꽤 높을 것이다. 패킷 송수신이나 디스크 I/O가 많은 응용이 크게 영향을 받을 텐데, 네트워크 쪽은 [자잘한 기법들](https://blog.cloudflare.com/how-to-receive-a-million-packets/)부터 [커널 우회]({{ site.baseurl }}{% post_url 2017-11-30-efficient-network-device %})까지 방안들을 고민하면 된다지만 디스크 쪽은 어떤 방법이 있을지 잘 모르겠다. 사용자 공간 파일 시스템은 이미 존재하고 잘하면 UIO로 저장 장치를 직접 다룰 수 있게 될 수도 있겠지만 디스크 스케줄링을 사용자 공간에서 돌리는 건 좀 상상하기 버겁다. 디스크 I/O에 공유 메모리를 사용해서 시스템 호출 횟수와 데이터 복사를 줄일 수 있을까?

## eBPF, Javascript

CVE-2017-5753과 관련해 eBPF나 자바스크립트 얘기가 등장하는 건 JIT 컴파일 때문이다. 문제의 인스트럭션 패턴을 공격 대상이 생성하여 실행하게끔 유도하는 게 상대적으로 쉽다. 하지만 원칙적으로는 외부에서 코드를 받아서 실행하는 모든 경우가 검토 대상이 된다. 또 음악/영상 데이터나 심지어 어느 프로토콜의 메시지를 통해 대상 머신에서 문제 인스트럭션 패턴이 실행되도록 만드는 게 가능할 수도 있다. 다만 투기적 실행을 유도하는 게 가능하다 해도 그로 인한 시스템 상태 변경을 측정하는 건 또 다른 문제이다.

[소켓 필터](https://www.kernel.org/doc/Documentation/networking/filter.txt)로 쓰이다 확장되고서 용도가 다양해진 eBPF는 조만간 `perf` 유틸리티와 관련해 다시 등장할 예정이다.

## 부채널

KASLR 무력화 기법들부터 멜트다운/스펙터에 두루 등장하는 게 부채널(side channel)이다. 이전에 [TEMPEST](https://en.wikipedia.org/wiki/Tempest_(codename)) 얘기 처음 읽었을 땐 좀 오버 아닌가 싶기도 했는데, 뭐 가능한 건 가능한 거다. 대신 속도나 정확도에 한계가 있을 수도 있고, 그래서 이번 취약점들을 통한 정보 추출 속도는 (컴퓨터 기준으로) 느린 편이다.

주로 등장하는 부채널이 타이밍 채널인데, 전통적으로 암호학 쪽에도 이와 관련된 [이슈가](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-0147) [많다](http://cr.yp.to/antiforgery/cachetiming-20050414.pdf). 심지어 Redis에도 타이밍 공격을 막기 위한 [상수 시간 문자열 비교 함수](https://github.com/antirez/redis/blob/4.0.6/src/server.c#L2613)가 있다!

----

2019-05-26:

이후 큰 틀에서 비슷한 CPU 취약점들이 더 발견됐고 대응하는 소프트웨어 패치와 마이크로코드 업데이트가 이뤄졌다. 15년 전통의 벤치마크 명가 Phoronix에서 그 대응책들이 [응용 성능에 끼치는 영향을 측정](https://www.phoronix.com/scan.php?page=article&item=mds-zombieload-mit)했다. 당연히 사용자 공간에서 주로 동작하는 응용에는 영향이 적고 문맥 전환이 빈번한 응용에서 영향이 크다.
