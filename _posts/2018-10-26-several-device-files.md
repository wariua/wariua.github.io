---
layout: post
title: 장치 파일 몇 가지
category: facility
tags: [cpu]
---
`/dev` 디렉터리에는 여러 장치 파일들이 모여 산다. 새 얼굴이 좀 있나 둘러보면,

### 네트워크 쪽인 듯 아닌 듯

```
crw------- 1 root root 10, 59 10월  2 15:14 /dev/cpu_dma_latency
crw------- 1 root root 10, 56 10월  2 15:14 /dev/memory_bandwidth
crw------- 1 root root 10, 58 10월  2 15:14 /dev/network_latency
crw------- 1 root root 10, 57 10월  2 15:14 /dev/network_throughput
```

부번호가 이어져 있으니 한 세트인 게 분명한데 도대체 어디서 나온 녀석들인지 짐작이 안 간다. 그래서 찾아 보면 전원 관리 서브시스템 안에 [qos.c](https://github.com/torvalds/linux/blob/master/kernel/power/qos.c) 파일이 있고 `drivers` 디렉터리 안에 [qos.c](https://github.com/torvalds/linux/blob/master/drivers/base/power/qos.c) 파일이 있다. 그리고 커널 문서 [pm_qos_interface.txt](https://www.kernel.org/doc/Documentation/power/pm_qos_interface.txt)도 있다. v2.6.25에서 [등장](https://github.com/torvalds/linux/commit/d82b351)했으니까 사실 새 얼굴은 아니다.

간단히 말해 "요 성능을 이 수준 이상으로 유지해 줘~"라고 요청할 수 있는 메커니즘이다. 가령 `cpu_dma_latency`에 100 값을 기록하고 파일을 계속 열고 있으면 그동안 DMA 동작 수행 지연을 100마이크로초 이하로 유지해 (정확하게는 유지하려고 노력해) 준다.

요청하는 쪽은 사용자 프로세스나 커널 내 모듈(가령 사운드 서브시스템)이고, 요청을 받아서 서비스 품질을 유지하려고 애쓰는 쪽은 항목마다 다르다. 가령 `cpu_dma_latency`는 CPU 유휴 상태 관리 모듈에서 담당하는데, 가능한 [CPU C-state](https://www.dell.com/support/article/kr/ko/krbsd1/qna41893/c-state%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9E%85%EB%8B%88%EA%B9%8C-?lang=ko) 범위를 제한하는 방식으로 품질을 유지한다. C-state가 깊을수록 C0(active) 상태로 복귀하는 데 긴 시간이 걸리고, 그게 DMA 지연으로 이어질 수 있기 때문이다. (`/sys/devices/system/cpu/cpu0/cpuidle/state*/latency` 참고.)

간단히 경험을 해 보려면,

1. `powertop` 명령을 실행해서 "Idle stats" 탭을 연다. 보통은 가장 깊은 상태(C6나 C7)의 비중이 가장 높다.

2. 루트 권한으로 다음을 실행한다.

   ```
   # (echo "0x00000000"; sleep 60) > /dev/cpu_dma_latency
   ```

잠시 후 C0 상태 비율이 100%가 되는 걸 볼 수 있다. 즉, 전력 소모 따위 신경쓰지 않고 CPU를 최고 성능으로 유지하는 거다. `powertop`에서 "Frequency stats" 탭을 보면 최고 클럭이 유지되는 걸 볼 수 있고 센서 값을 확인하면 CPU 온도가 쑥쑥 올라가는 걸 볼 수 있다. 당연히 전기 요금도 함께 올라간다. 그렇게 극단적인 걸 원하는 건 아니라면 수십 us 정도로 지정할 수도 있다.

단위가 `cpu_dma_latency`는 us, `memory_bandwidth`는 Mbps, `network_latency`는 us, `network_throughput`은 kbps다. 지연은 지정한 값(들 중 최솟값) 이하를 유지하는 거고 스루풋은 지정한 값(들 중 최댓값) 이상을 유지하는 거고 대역폭은 지정한 값(들의 합) 이상을 유지하는 거다. 근데 `cpu_dma_latency` 말고 다른 항목들은 이름만 있고 QoS 적용 주체가 없다. ([어느 글](https://lwn.net/Articles/386139/)에 따르면 한때 mac80211 계층에서 `network_latency`를 다뤘던 모양이다.) 파일을 읽으면 종합해서 현재 적용 중인 값이 (`int`로) 나온다.

앞에 `dev_`가 붙은 API는 장치(가령 GPU)별로 각기 QoS 요청을 받아서 적용할 수 있도록 하는 것이다.

### 인생샷

```
crw------- 1 root root 10, 231 10월  2 15:14 /dev/snapshot
```

일단 이름은 메모리 스냅샷을 뜻한다. 근데 `/dev/mem` 같은 게 아니라 하이버네이션용 스냅샷 얘기다. 그리고 `ioctl()`을 통해 suspend/hibernate 전환 과정의 세부 동작들을 수행할 수 있다.

관련 소스는 전력 관리 서브시스템 안의 [user.c](https://github.com/torvalds/linux/blob/master/kernel/power/user.c)다. 소스 파일 이름처럼 사용자 공간에서 직접 모드 전환을 제어하려고 할 때를 위한 장치 파일이다. 근데 훨씬 편한 인터페이스(`/sys/power/state`)가 있기 때문에 쓸 일이 별로 없다. 자세한 건 커널 문서 [userland-swsusp.txt](https://www.kernel.org/doc/Documentation/power/userland-swsusp.txt) 참고.

### 그대 입력

```
crw------- 1 root root 10, 223 10월  2 15:14 /dev/uinput
```

가상 입력 장치(키보드, 마우스, ...)를 등록해서 가상 입력을 줄 수 있는 특수 파일이다. 가령 매크로 프로그램을 만드는 데 쓸 수 있겠다.

관련 소스는 [uinput.c](https://github.com/torvalds/linux/blob/master/drivers/input/misc/uinput.c)이다. [커널 문서](https://www.kernel.org/doc/html/v4.19/input/uinput.html)에 예시 프로그램이 있다.

### 사용자 입출력

```
crw------- 1 root root 10, 240 10월  2 15:14 /dev/userio
```

사실 "User I/O"는 아니고 "User-land Serial I/O"다. 직렬 입출력 장치(가령 노트북 터치패드)를 에뮬레이트 할 수 있다.

관련 소스는 [serio.c](https://github.com/torvalds/linux/blob/master/drivers/input/serio/userio.c), [커널 문서](https://www.kernel.org/doc/html/v4.19/input/userio.html) 참고.

### 문자 장치를 만드는 문자 장치

```
crw------- 1 root root 10, 203 10월  2 15:14 /dev/cuse
```

FUSE(Filesystem in Userspace)를 통해 사용자 공간에서 파일 시스템을 구현할 수 있는 것처럼 CUSE(Character device in Userspace)를 통해 사용자 공간에서 문자 장치를 구현할 수 있다. 범용적인 에뮬레이션인 셈인데, [소개 글](https://lwn.net/Articles/308445/)에 따르면 초기 용도는 구식 OSS 인터페이스를 좀 편하게 제공하는 거였다고 한다.

관련 소스는 [cuse.c](https://github.com/torvalds/linux/blob/master/fs/fuse/cuse.c)인데 주석에 보면 괜스레 반가운 이름이 보인다. libfuse의 [문서](https://libfuse.github.io/doxygen/cuse_8c.html)와 [소스](https://github.com/libfuse/libfuse/blob/fuse-3.2.6/lib/cuse_lowlevel.c)를 참고할 수 있다.
