---
layout: post
title: virtio 암호 장치
category: facility
tags: [virtio, crypto]
---
전가상화 분야에서 연산 쪽은 네이티브와 다를 바 없게 만들 수 있었지만 I/O 쪽에선 오버헤드가 너무 컸다. 여러 해결 노력이 이어졌는데 그 중 한 방향은 게스트 운영체제에 포함된 어떤 가상 장치 드라이버와 하이퍼바이저가 효율적으로 통신할 수 있게 만드는 것이고, 널리 쓰이는 게 [Rusty Russell이 만들고](https://lwn.net/Articles/239238/) [표준화](http://docs.oasis-open.org/virtio/virtio/v1.0/virtio-v1.0.html)까지 된 [virtio](https://www.ibm.com/developerworks/library/l-virtio/index.html)다. 여러 하이퍼바이저뿐 아니라 [DPDK](http://doc.dpdk.org/guides/nics/virtio.html) 같은 플랫폼에서도 지원한다. 적용 대상으로 쉽게 떠오르는 게 디스크와 NIC인데, 또 어떤 게 가능한지는 서브시스템 ID 목록을 보면 된다.

`linux/include/uapi/linux/virtio_ids.h`:
```c
#define VIRTIO_ID_NET           1 /* virtio net */
#define VIRTIO_ID_BLOCK         2 /* virtio block */
#define VIRTIO_ID_CONSOLE       3 /* virtio console */
#define VIRTIO_ID_RNG           4 /* virtio rng */
#define VIRTIO_ID_BALLOON       5 /* virtio balloon */
#define VIRTIO_ID_RPMSG         7 /* virtio remote processor messaging */
#define VIRTIO_ID_SCSI          8 /* virtio scsi */
#define VIRTIO_ID_9P            9 /* 9p virtio console */
#define VIRTIO_ID_RPROC_SERIAL 11 /* virtio remoteproc serial link */
#define VIRTIO_ID_CAIF         12 /* Virtio caif */
#define VIRTIO_ID_GPU          16 /* virtio GPU */
#define VIRTIO_ID_INPUT        18 /* virtio input */
#define VIRTIO_ID_VSOCK        19 /* virtio vsock transport */
#define VIRTIO_ID_CRYPTO       20 /* virtio crypto */
```

최근에야 겨우 [목록에 ID를 올린](https://lwn.net/Articles/707096/) 게 [virtio 암호 장치](https://privatewiki.opnfv.org/_media/dpacc/a_new_framework_of_cryptography_virtio_driver.pdf)다. QEMU에서 [지원](https://wiki.qemu.org/Features/VirtioCrypto)한다. 제공하는 서비스는 CIPHER, MAC, HASH, AEAD다. RANDOM을 추가하고 위 목록의 `VIRTIO_ID_RNG`를 없애면 좋겠지만, 뭐 모든 게 완벽할 수는 없는 법이다. 비대칭 암호도 현재는 지원하지 않는다.

게스트 CPU에서도 할 수 있을 암호 연산을 호스트에게 떠넘기는 게 의미가 있으려면 호스트에서 더 효율적으로 수행할 방법이 있어야 된다. 즉 호스트에서 기껏 AES-NI 같은 전용 인스트럭션을 쓸 거라면 virtio-crypto를 이용하는 건 순수한 CPU 사이클 낭비다. ([이전 글]({{ site.baseurl }}{% post_url 2018-10-09-async-crypto %}) 참고.)

게스트 내 사용자 공간의 응용에서 요청한 암호 연산이 호스트의 암호 장치까지 닿으려면 긴 경로를 따라가야 한다.

게스트 응용 <-> 게스트 커널 crypto subsys <-> virtio-crypto 장치 <-> 호스트 하이퍼바이저 <-> ...

하이퍼바이저에서 암호 장치까지는 바로 연결돼 있을 수도 있고, 암호 라이브러리나 호스트 커널 crypto 서브시스템을 (또는 둘 모두를) 추가로 거칠 수도 있다. 단계마다 나름의 역할이 있기는 한데, 여튼 길다.

이와 달리 PCI passthrough를 통해 게스트가 호스트의 장치(의 [일부](https://en.wikipedia.org/wiki/Single-root_input/output_virtualization))에 직접 접근하는 방식도 가능하다. 보드라운 클라우드 속에서 그렇게까지 해야 하나 싶기는 하지만 당연히 성능은 이쪽이 훨씬 낫다. virtio-crypto가 성능을 상당 부분 포기하고 얻는 건 추상화와 캡슐화다. AWS 같은 데서 부가 서비스로 만들기에 좋은 형태이고, 또 아래에서 조용하게 가성비 높은 암호 장치를 자체 제작해 볼 수도 있을 거다.
