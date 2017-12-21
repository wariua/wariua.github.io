---
layout: post
title: 인스트럭션 캐시와 성능 - 간단 실험
category: performance
tags: [cache, perf]
---
[VPP](https://wiki.fd.io/view/VPP)에서는 [인스트럭션 캐시 활용률을 높여서 스루풋을 높이려 한다]({{ site.baseurl }}{% post_url 2017-12-06-vpp-graph %}). 그런데 과연 얼마나 성능을 개선할 수 있을까? icache 적중률이 프로그램 성능에 어느 정도나 영향을 줄까?

## 도구

캐시 적중률 정보를 얻는 도구로 Valgrind와 `perf` 유틸리티가 있다. Valgrind는 일종의 시뮬레이터다. 이식성이 높고 유연하지만 외부(커널, 다른 프로세스)의 영향을 반영하지 못하기 때문에 오차가 있을 수 있다. 그리고, 느리다. `perf stat` 명령은 다양한 하드웨어/소프트웨어 카운터를 보여 준다. 빠르고 더 정확하지만 리눅스 전용이고 아키텍처 제약이 있으며 실행 머신에서의 동작 특성만 확인할 수 있다.

```
$ valgrind --tool=cachegrind ./a.out blahblah
==22094== Cachegrind, a cache and branch-prediction profiler
==22094== Copyright (C) 2002-2017, and GNU GPL'd, by Nicholas Nethercote et al.
==22094== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==22094== Command: ./a.out blahblah
==22094== 
--22094-- warning: L3 cache found, using its data for the LL simulation.
...
==22094== 
==22094== I   refs:      676,764,718,345
==22094== I1  misses:        509,693,762
==22094== LLi misses:             74,319
==22094== I1  miss rate:            0.08%
==22094== LLi miss rate:            0.00%
==22094== 
==22094== D   refs:      280,031,965,007  (177,119,405,573 rd   + 102,912,559,434 wr)
==22094== D1  misses:        602,678,071  (    333,944,605 rd   +     268,733,466 wr)
==22094== LLd misses:        275,784,902  (     84,688,742 rd   +     191,096,160 wr)
==22094== D1  miss rate:             0.2% (            0.2%     +             0.3%  )
==22094== LLd miss rate:             0.1% (            0.0%     +             0.2%  )
==22094== 
==22094== LL refs:         1,112,371,833  (    843,638,367 rd   +     268,733,466 wr)
==22094== LL misses:         275,859,221  (     84,763,061 rd   +     191,096,160 wr)
==22094== LL miss rate:              0.0% (            0.0%     +             0.2%  )
```

```
$ perf stat -e cycles,instructions,...,L1-icache-load-misses,... ./a.out blahblah
...

 Performance counter stats for './a.out blahblah':

   643,290,014,620      cycles                    #    3.974 GHz                      (25.00%)
   923,267,837,233      instructions              #    1.44  insn per cycle           (33.34%)
     6,734,099,803      cache-references          #   41.600 M/sec                    (41.67%)
           792,285      cache-misses              #    0.012 % of all cache refs      (41.67%)
   373,886,460,354      L1-dcache-loads           # 2309.675 M/sec                    (41.66%)
   178,169,758,991      L1-dcache-stores          # 1100.639 M/sec                    (16.67%)
     5,888,113,355      L1-dcache-load-misses     #    1.57% of all L1-dcache hits    (16.67%)
    13,433,907,445      L1-icache-load-misses                                         (16.66%)

     161.878356343 seconds time elapsed
```

["What every programmer should know about memory" 7장](https://lwn.net/Articles/257209/)에서 더 많은 도구들을 소개해 준다.

## 어림셈

프로그램이 작다면 코드 워킹 셋이 icache에 상주할 수도 있다. 그런 프로그램에서 벡터 처리 구조는 무의미하다. 어느 정도 크기부터 icache 미스가 성능에 영향을 끼칠까?

현재 대부분의 x86-64 프로세서에서 L1 icache 크기는 32KiB다. associativity는 보통 8-way, 라인 크기는 64B다. 프로그램이 데이터 한 단위를 처리하는 동안 실행하는 코드가 주소 공간에 균일하게 분포돼 있다고 (그래서 실행 코드 크기가 32KiB여도 퇴출이 발생하지 않는다고) 하자. 또 한 인스트럭션이 평균 4바이트이고 처리 과정에서 같은 코드를 여러 번 실행하는 경우는 무시할 만하다고 하자. 그럼 데이터 한 단위 처리에 실행하는 인스트럭션이 8천 개 이하이면 icache 미스가 없게 된다.

패킷 한 개에 8천 인스트럭션을 생각해 보면, 4GHz 코어에서 [사이클 당 인스트럭션](https://en.wikipedia.org/wiki/Instructions_per_cycle)이 2라고 할 때 1Mpps다. L2 수준의 간단한 응용을 제외하면 미스가 충분히 발생할 수 있다. 그리고 다른 분야의 응용들은 대부분 해당될 것이다.

패킷당 8천 인스트럭션이라는 기준으로 보면 VPP의 노드는 상당히 잘게 쪼개져 있다. 프레임워크 자체에서 실행하는 인스트럭션을 산입하고 icache가 작은 시스템까지 고려하다는 의미도 있겠지만, 노드를 모듈화의 한 방법으로 사용하는 게 주된 이유인 것 같다.

## 간단 실험

성능이 중요한 프로그램을 하나 골라서 실행 구조를 바꾼 다음 성능 변화를 확인해 보려고 했는데 적당한 프로그램 찾기가 쉽지 않아서 그냥 편하게 가기로 했다. [SQLite](https://www.sqlite.org/)로 다음 동작을 하는 짧은 프로그램을 만들었다.

 1. 메모리에 두 컬럼(key, value)짜리 테이블 생성
 2. 레코드 10k개 INSERT
 3. 다음을 1000번 반복
    1. 10k개 레코드 각각에 대해 SELECT 및 UPDATE

3-i 단계에서 각 세부 단계를 여러 데이터를 가지고 실행해 봤다. 말하자면 다음 코드를 N 값을 바꿔가며 실행해 본 것이다. 컴파일러 최적화의 영향을 피하기 위해 N을 런타임에 입력받는다.

```c
for (i = 0; i < 1000; i++) {
    for (j = 0; j < 10*1024; j += N) {
        sqlite3_stmt stmt[N];

        for (k = 0; k < N; k++) {
            snprintf(sql, sizeof(sql),
                     "SELECT value FROM tbl WHERE key = ...", ...);
            sqlite3_prepare_v2(db, sql, -1, &stmt[k], NULL);
        }

        for (k = 0; k < N; k++)
            sqlite3_step(stmt[k]);

        for (k = 0; k < N; k++)
            sqlite3_finalize(stmt[k]);

        /* UPDATE에 대해 마찬가지로 prepare, step, finalize... */
    }
}
```

캐시가 성능에 끼치는 영향을 설명할 때 커다란 다차원 배열을 여러 순서로 접근하면서 실행 시간을 비교하는 경우가 많다 (예: [여기](https://lwn.net/Articles/255364/) 6.2절). 위 코드도 본질적으로 비슷한 코드이다. icache가 대상이란 게 다를 뿐이다.

Valgrind 등으로 프로파일링 해 보면 `sqlite3_prepare_v2()`, `sqlite3_step()`, `sqlite3_finalize()`의 실행 인스트럭션 수가 10:9:1 정도이다. 당연하지만 SQL 문 내용이나 테이블 스키마, 레코드 개수 등에 따라 비율이 달라진다.

`perf stat` 출력을 비교해 보면,

기본 버전 (N=1):
```
   646,137,594,818      cycles                    #    3.972 GHz                      (25.00%)
   924,169,673,767      instructions              #    1.43  insn per cycle           (33.34%)
     7,120,560,561      cache-references          #   43.768 M/sec                    (41.67%)
           876,687      cache-misses              #    0.012 % of all cache refs      (41.67%)
     162688.582487      cpu-clock (msec)          #    1.000 CPUs utilized
               395      cs                        #    0.002 K/sec
               215      faults                    #    0.001 K/sec
   374,147,452,013      L1-dcache-loads           # 2299.777 M/sec                    (41.66%)
   178,225,239,216      L1-dcache-stores          # 1095.499 M/sec                    (16.67%)
     5,870,432,921      L1-dcache-load-misses     #    1.57% of all L1-dcache hits    (16.66%)
    13,605,255,870      L1-icache-load-misses                                         (16.67%)
       841,136,295      LLC-loads                 #    5.170 M/sec                    (25.00%)
           520,871      LLC-load-misses           #    0.06% of all LL-cache hits     (33.34%)
       222,899,926      LLC-stores                #    1.370 M/sec                    (16.67%)
            11,736      LLC-store-misses          #    0.072 K/sec                    (16.67%)

     162.687561004 seconds time elapsed
```

벡터 버전 (N=8):
```
   539,316,315,129      cycles                    #    3.973 GHz                      (25.00%)
   917,758,487,341      instructions              #    1.70  insn per cycle           (33.34%)
     2,393,149,449      cache-references          #   17.628 M/sec                    (41.67%)
           792,502      cache-misses              #    0.033 % of all cache refs      (41.67%)
     135761.760116      cpu-clock (msec)          #    1.000 CPUs utilized
                37      cs                        #    0.000 K/sec
               216      faults                    #    0.002 K/sec
   372,158,761,095      L1-dcache-loads           # 2741.264 M/sec                    (41.66%)
   176,734,587,695      L1-dcache-stores          # 1301.799 M/sec                    (16.67%)
     3,368,440,286      L1-dcache-load-misses     #    0.91% of all L1-dcache hits    (16.67%)
     4,578,741,521      L1-icache-load-misses                                         (16.67%)
       393,741,945      LLC-loads                 #    2.900 M/sec                    (25.00%)
           458,660      LLC-load-misses           #    0.12% of all LL-cache hits     (33.34%)
       194,934,128      LLC-stores                #    1.436 M/sec                    (16.67%)
             7,236      LLC-store-misses          #    0.053 K/sec                    (16.67%)

     135.759370665 seconds time elapsed
```

실행한 인스트럭션 수는 거의 같다. 하지만 실행에 걸린 사이클 수는 (그래서 경과 시간은) 꽤 차이가 난다. 15% 정도 줄었다. 캐시 미스를 살펴보면, 최상위 캐시(LLC) 미스 횟수에 차이가 있지만 100 정도 곱해 봐야 조용히 묻히는 정도이다. L1 dcache 미스 횟수가 3/5 정도로 줄었지만 icache 미스 횟수는 1/3로 줄었다. 그리고 그 두 캐시 미스 차이의 합과 소모 사이클 수 차이의 비율이 대략 1:10이다. L1 캐시 미스 발생 시 적어도 몇 사이클 버리는 걸 생각하면 사이클 수 차이의 상당 부분이 캐시 미스로 설명된다.

icache 미스 비율을 단순하게 계산하면 1.5%와 0.5%다. 도긴개긴 아닌가 싶기도 하다. 하지만 캐시 라인 하나에는 인스트럭션이 여러 개 들어간다. 즉, 한번 미스가 발생하서 라인 하나를 가져오면 이어지는 여러 인스트럭션들은 자동으로 히트다. 이것저것 가정해서 대략 16을 곱하면 캐시 라인 단위로 본 미스율이 나온다. 24%와 8%, 이렇게 보니 성능에 상당히 영향을 줄 것 같다.

prepare-step-finalize 조합을 한 번 실행하는 데 얼마나 많은 인스트럭션을 쓰는 건까? 인스트럭션 수를 1000 * 10 * 1024 * 2로 나누면 40k가 좀 넘게 나온다. finalize는 무시하고, prepare와 step에서 각각 20k 개 정도 실행하는 셈이다. 각각을 더 작은 연산으로 나눌 수 있다면 icache 효율을 더 높일 여지가 있다는 얘기다.

N을 더 높이면 성능이 더 개선될까? 그게 또 그렇지는 않았다. N을 높이면 실행 인스트럭션 수가 마구 증가하면서 스루풋이 떨어지는데, 동적 메모리 관리 오버헤드가 아닐까 싶다.

## 요약

 * 단순한 프로그램들을 제외하면 효과를 기대할 수 있음.
 * 일단 프로파일링.
 * 데이터를 최대 몇 개씩 처리하는 게 최선인지는 케바케.
