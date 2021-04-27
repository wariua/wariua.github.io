---
layout: post
title: 숨은 헤더
category: general
tags: [redis, talloc]
---
연두에는 헤더 얘기가 제맛이다.

네트워크 프로토콜이든 다른 어디서든 데이터 앞에 붙은 헤더는 일반적으로 메타 데이터를 위한 자리다. 그런데 그 메타 데이터가 사용자에게 필요치 않다면 굳이 헤더를 드러낼 이유가 없다. 이번 글은 메모리 할당을 맴돌며 숨은 헤더를 주제로 하는 변주들이다.

## Redis의 <tt>sds</tt>

[Redis](https://redis.io/)의 근간을 이루는 두 가지 자료 구조를 꼽으라면 해시 기반 딕셔너리인 <tt>[dict](https://github.com/antirez/redis/blob/unstable/src/dict.h)</tt>, 그리고 이진 문자열인 <tt>[sds](https://github.com/antirez/redis/blob/unstable/src/sds.h)</tt>이다. `sds`는 본 데이터인 `char` 배열 앞에 가변 길이 헤더를 붙이고 뒤에 NULL 종료자(`\0`)를 붙인 것이다. `sds` 타입은 `char *`로 정의되어 있어서 본 데이터의 시작 위치를 가리킨다.

[redis/src/sds.h](https://github.com/antirez/redis/blob/unstable/src/sds.h):
```c
typedef char *sds;

/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
}
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
...
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
...
#define SDS_TYPE_64 4
#define SDS_TYPE_MASK 7
#define SDS_TYPE_BITS 3
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void *)((s)-(sizeof(struct sdshdr##T)));
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
#define SDS_TYPE_5_LEN(f) ((f)>>SDS_TYPE_BITS)

static inline size_t sdslen(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_8:
            return SDS_HDR(8,s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->len;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->len;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->len;
    }
    return 0;
}
```

헤더 버전이 여러 개라서 좀 산만하지만 결국 문자열 시작점 바로 앞에 메타 정보를 담은 헤더를 붙이는 것이다. 문자열을 생성할 때 헤더까지 합쳐서 메모리를 할당하고 헤더를 건너뛴 위치를 반환한다.

[redis/src/sds.c](https://github.com/antirez/redis/blob/unstable/src/sds.c):
```c
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen);
    int hdrlen = sdsHdrSize(type);
    ...

    sh = s_malloc(hdrlen+initlen+1);
    if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    if (sh == NULL) return NULL;
    s = (char*)sh+hdrlen;
    ...
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}
```

(`s_malloc()` 바로 다음의 `memset()`이 심히 거슬리더라도 그냥 넘어가자.)

`sds` 메모리를 해제할 때는 반대로 문자열 시작점에서 헤더 위치를 역산해서 그 주소를 `free()`에게 건네게 된다.

## talloc

데이터를 위한 메모리 공간 앞에 숨겨진 헤더를 만들 수 있다면 거기에 길이와 플래그만 담으라는 법은 없다. 더 많은 걸 하기 위해 더 많은 걸 집어넣을 수도 있다.

[talloc](https://talloc.samba.org/)은 Samba에 포함된 메모리 관리 모듈이다. 성능 때문에 메모리 관리자를 따로 만든 건가 싶을 수도 있지만 talloc의 목표는 효율적 동작이 아니라 편리한 메모리 관리이다. 핵심은 메모리 블록들 간에 위계 구조를 (즉 트리를) 만들어서 한 블록을 해제하면 자식들까지 자동으로 해제되는 것이다. 그래서 다음과 같이 사용자 데이터를 만들었을 때,

```c
struct user {
    uid_t uid;
    char *username;
    size_t num_groups;
    char **groups;
};

/* create new top level context */
struct user *user = talloc(NULL, struct user);

user->uid = 1000;
user->num_groups = N;

/* make user the parent of following contexts */
user->username = talloc_strdup(user, "Test user");
user->groups = talloc_array(user, char*, user->num_groups);

for (i = 0; i < user->num_groups; i++) {
    /* make user->groups the parent of following context */
    user->groups[i] = talloc_asprintf(user->groups,
                                      "Text group %d", i);
}
```

사용이 끝난 데이터를 다음 호출 하나로 정리할 수 있다. `user`가 가리키는 메모리 블록뿐 아니라 `user`를 문맥으로 해서 생성한 `N`+2개 블록들이 함께 해제된다.

```c
talloc_free(user);
```

딱 봐도 편할 것 같고 메모리 누수 걱정을 덜어 줄 것 같다. 대신 치러야 하는 비용은 블록들 간의 관계를 관리하는 것, 즉 링크를 위한 추가 공간과 연산이다.

[talloc/talloc.c](https://github.com/samba-team/samba/blob/master/lib/talloc/talloc.c):
```c
struct talloc_chunk {
    /*
     * flags includes the talloc magic, which is randomised to
     * make overwrite attacks harder
     */
    unsigned flags;

    /*
     * If you have a logical tree like:
     *
     *           <parent>
     *           /   |   \
     *          /    |    \
     *         /     |     \
     * <child 1> <child 2> <child 3>
     *
     * The actual talloc tree is:
     *
     *  <parent>
     *     |
     *  <child 1> - <child 2> - <child 3>
     *
     * The children are linked with next/prev pointers, and
     * child 1 is linked to the parent with parent/child
     * pointers.
     */

    struct talloc_chunk *next, *prev;
    struct talloc_chunk *parent, *child;
    ...
    size_t size;
    ...
};

/* 16 byte alignment seems to keep everyone happy */
#define TC_ALIGN16(s) (((s)+15)&~15)
#define TC_HDR_SIZE TC_ALIGN16(sizeof(struct talloc_chunk))
#define TC_PTR_FROM_CHUNK(tc) ((void *)(TC_HDR_SIZE + (char*)tc))

...

static inline void *__talloc_with_prefix(const void *context,
                                         size_t size,
                                         size_t prefix_len,
                                         struct talloc_chunk **tc_ret)
{
    struct talloc_chunk *tc = NULL;
    struct talloc_memlimit *limit = NULL;
    size_t total_len = TC_HDR_SIZE + size + prefix_len;
    struct talloc_chunk *parent = NULL;

    ...
    if (likely(context != NULL)) {
        parent = talloc_chunk_from_ptr(context);
        ...
    }

    if (tc == NULL) {
        char *ptr;

        ...
        ptr = malloc(total_len);
        if (unlikely(ptr == NULL)) {
            return NULL;
        }
        tc = (struct talloc_chunk *)(ptr + prefix_len);
        ...
    }

    tc->limit = limit;
    tc->size = size;
    ...

    if (likely(context != NULL)) {
        if (parent->child) {
            parent->child->parent = NULL;
            tc->next = parent->child;
            tc->next->prev = tc;
        } else {
            tc->next = NULL;
        }
        tc->parent = parent;
        tc->prev = NULL;
        parent->child = tc;
    } else {
        tc->next = tc->prev = tc->parent = NULL;
    }

    *tc_ret = tc;
    return TC_PTR_FROM_CHUNK(tc);
}
```

[left-child right-sibling 이진 트리](https://en.wikipedia.org/wiki/Left-child_right-sibling_binary_tree)를 확장한 자료 구조로 [로즈 트리](https://en.wikipedia.org/wiki/Rose_tree)를 표현한다.

## malloc

`sds`나 talloc 아래에는 malloc 계열 메모리 관리자가 있다. 그런데 <tt>[malloc_usable_size()](https://wariua.github.io/man-pages-ko/malloc_usable_size%283%29)</tt> 같은 함수가 가능한 건 `malloc()`이 반환하는 메모리 블록에도 숨은 헤더가 있기 때문이다.

[glibc/malloc/malloc.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=malloc/malloc.c):
```c
struct malloc_chunk {

  INTERNAL_SIZE_T      mchunk_prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      mchunk_size;       /* Size in bytes, including overhead. */

  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;

  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};

/*
   malloc_chunk details:

    ...

    An allocated chunk looks like this:


    chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of previous chunk, if unallocated (P clear)  |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of chunk, in bytes                     |A|M|P|
      mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             User data starts here...                          .
            .                                                               .
            .             (malloc_usable_size() bytes)                      .
            .                                                               |
nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             (size of chunk, but used for application data)    |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of next chunk, in bytes                |A|0|1|
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 
    Where "chunk" is the front of the chunk for the purpose of most of
    the malloc code, but "mem" is the pointer that is returned to the
    user.  "Nextchunk" is the beginning of the next contiguous chunk.
 
    ...
 
    Free chunks are stored in circular doubly-linked lists, and look like this:
 
    chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of previous chunk, if unallocated (P clear)  |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    `head:' |             Size of chunk, in bytes                     |A|0|P|
      mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Forward pointer to next chunk in list             |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Back pointer to previous chunk in list            |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Unused space (may be 0 bytes long)                .
            .                                                               .
            .                                                               |
nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    `foot:' |             Size of chunk, in bytes                           |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Size of next chunk, in bytes                |A|0|0|
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

    ...
*/

/* conversion from malloc headers to user pointers, and back */

#define chunk2mem(p)   ((void*)((char*)(p) + 2*SIZE_SZ))
#define mem2chunk(mem) ((mchunkptr)((char*)(mem) - 2*SIZE_SZ))

/* The smallest possible chunk */
#define MIN_CHUNK_SIZE        (offsetof(struct malloc_chunk, fd_nextsize))
```

이것저것 복잡하지만 결국은 사용자에게 보이지 않는 헤더에 대한 이야기이다.

사용자 공간은 밝고 다채롭다. 이제 커널로 내려갈 차례이다.

## <tt>struct sk_buff</tt>

BSD에 `struct mbuf`가 있고 DPDK에 `struct rte_mbuf`가 있다면 리눅스에는 `struct sk_buff`가 있다. 메타 데이터를 담는 `struct sk_buff` 구조체의 `head` 필드가 패킷 데이터 버퍼를 가리킨다. ([구글 이미지 검색](https://www.google.co.kr/search?q=sk_buff&tbm=isch) 참고.) 그런데 리눅스 커널에서 패킷을 저장할 메모리를 할당할 때 `sk_buff`와 데이터 버퍼를 한 덩어리로 할당하지는 않는다. 그러면 `sk_buff` 사용 방식에 제약이 생기기 때문이다. 즉, 여기엔 숨은 헤더가 없다.

얘기를 좀 돌려서, NIC가 패킷을 수신했을 때 전통적인 장치 드라이버의 동작은 다음과 같다.

 1. `sk_buff`를 (그리고 그에 딸린 데이터 버퍼를) 새로 할당한다.
 2. NIC 수신 버퍼에서 패킷 데이터를 읽어와서 데이터 버퍼로 복사한다.
 3. 몇 가지 기본적인 처리(가령 이더넷 헤더 검사)를 한다.
 4. 적당한 함수(`netif_rx()`, `netif_receive_skb()`)를 호출해서 패킷을 네트워크 스택으로 보낸다.

2번 단계는 딱 봐도 CPU를 많이 소모하게 생겼다. 데이터 복사를 NIC가 대신(DMA) 해 주면 좋을 텐데, 그러자면 버퍼를 NIC에게 미리 제공해 줘야 한다. 이건 비교적 간단하다. 장치 초기화 때 데이터 버퍼를 미리 몇 개 만들어서 하드웨어에게 제공하고, 장치를 제거할 때 그 버퍼들을 받아서 해제하면 된다. 그러면 NIC이 사전 할당 버퍼들의 풀을 관리하면서 패킷이 들어오면 버퍼 하나를 골라 데이터를 복사한 후 커널에게 알린다. 그럼 ISR에서는 데이터 버퍼에 대한 포인터만 하드웨어에게서 얻으면 된다. 그리고 패킷 수신 때 풀에서 버퍼를 꺼내는 것처럼 출력 완료 후 풀에 버퍼를 채워넣으면 된다. 그리고 드라이버 코드에서 계속 현재 풀 크기를 확인해서 빈 슬롯을 채우면 된다. 하드웨어와 소프트웨어 각각에 추가 구현이 필요하지만 데이터 버퍼 메모리 할당/해제 횟수가 상당히 줄어든다. 그래서 여러 NIC들이 이런 사전 할당 버퍼 풀(링) 방식을 사용한다.

근데 사람 욕심이란 게, 이렇게 하고 나면 1번 단계도 없애고 싶기 마련이다. 데이터 버퍼를 미리 할당해서 재사용할 수 있다면 `sk_buff` 구조체도 그리 하면 되지 않겠는가. 다행히 하드웨어가 데이터 버퍼와 연계된 정보(`sk_buff` 주소 값)를 함께 관리해 준다면 좋겠지만 모두가 그런 건 아니다.

문제를 정리하면 이렇다.

 1. `sk_buff` 메모리 공간과 데이터 버퍼 메모리 공간이 있다.
 2. `sk_buff` 공간의 주소를 알면 데이터 버퍼의 주소를 알 수 있다. (`skb->head`)
 3. NIC에게 데이터 버퍼 주소를 제공한다. (사전 할당)
 4. NIC에게서 받은 데이터 버퍼 주소로부터 `sk_buff` 주소를 알아내야 한다. (패킷 수신)

"연계 `sk_buff` 주소"라는 것도 결국 몇 개의 비트, 즉 일종의 메타 정보이다. 그리고 NIC는 데이터 버퍼 메모리 공간 사용자이다. 그래서 문제를 다시 쓰면,

 1. 어떤 메모리 공간이 있다. 그에 딸린 메타 정보가 있다.
 2. 사용자에게 메모리 공간의 주소를 제공한다.
 3. 사용자에게 받은 메모리 공간 주소로부터 메타 정보를 얻어야 한다.

malloc 같은 메모리 관리자가 해결해야 하는 문제와 동일하며 그렇다면 같은 방식으로 해결할 수 있다. 데이터 버퍼 앞쪽 일부를 헤더로 전용해서 `sk_buff`에 대한 주소를 저장하고, NIC에게는 그 헤더 다음의 주소를 주면 된다. 즉, 숨은 헤더다.

[linux/drivers/net/ethernet/tile/tilepro.c](https://github.com/torvalds/linux/blob/master/drivers/net/ethernet/tile/tilepro.c):
```c
static int tile_net_open_inner(struct net_device *dev)
{
    ...
    tile_net_register(dev);
    ...

    if (priv->intr_id == 0) {
        ...
        tile_irq_activate(irq, TILE_IRQ_PERCPU);
        ...
        tile_net_provide_needed_buffers(info);
        ...
    }

    ...
    netif_start_queue(dev);

    return 0;
}

static void tile_net_provide_needed_buffers(struct tile_net_cpu *info)
{
    while (info->num_needed_small_buffers != 0) {
        if (!tile_net_provide_needed_buffer(info, true))
            goto oops;
        info->num_needed_small_buffers--;
    }

    while (info->num_needed_small_buffers != 0) {
        if (!tile_net_provide_needed_buffer(info, false))
            goto oops;
        info->num_needed_large_buffers--;
    }

    return;

oops:
    ...
}

static bool tile_net_provide_needed_buffer(struct tile_net_cpu *info,
                                           bool small)
{
    ...
    struct sk_buff *skb;
    void *va;

    sturct sk_buff **skb_ptr;

    /* Request 96 extra bytes for alignment purposes. */
    skb = netdev_alloc_skb(info->napi.dev, len + padding);
    if (skb == NULL)
        return false;

    /* Skip 32 or 96 bytes to align "data" mod 128. */
    align = -(long)skb->data & (128 - 1);
    BUG_ON(align > padding);
    skb_reserve(skb, align);

    /* This address is given to IPP. */
    va = skb->data;

    ...

    /* Save a back-pointer to 'skb'. */
    skb_ptr = va - sizeof(*skb_ptr);
    *skb_ptr = skb;

    ...

    /* Provide the new buffer. */
    tile_net_provide_linux_buffer(info, va, small);

    return true;
}

...

static int tile_net_poll(struct napi_struct *napi, int budget)
{
    ...

    while (priv->active) {
        ...
        if (tile_net_poll_aux(info, index)) {
            if (++work >= budget)
                goto done;
        }
    }

    napi_complete_done(&info->napi, work);

    ...

done:

    if (priv->active)
        tile_net_provide_needed_buffers(info);

    return work;
}

static bool tile_net_poll_aux(struct tile_net_cpu *info, int index)
{
    ...
    netio_pkt_t *pkt = (netio_pkt_t *)((unsigned long) &qsp[1] + index);
    ...

    /* Extract the packet size.  FIXME: Shouldn't the second line */
    /* get substracted?  Mostly moot, since it should be "zero". */
    unsigned long len =
        (NETIO_PKT_CUSTOM_LENGTH(pkt) +
         NET_IP_ALIGN - NETIO_PACKET_PADDING);

    /* Extract the "linux_buffer_t". */
    unsigned int buffer = pkt->__packet.word;

    /* Extract "small" (vs "large"). */
    bool small = ((buffer & 1) != 0);

    /* Convert "linux_buffer_t" to "va". */
    void *va = __va((phys_addr_t)(buffer >> 1) << 7);

    ...

    if (filter != 0) {
        ...
    } else {

        /* Acquire the associated "skb". */
        struct sk_buff **skb_ptr = va - sizeof(*skb_ptr);
        struct sk_buff *skb = *skb_ptr;

        ...

        netif_receive_skb(skb);

        ...
    }

    ...
}
```

NIC에게 버퍼를 줄 때 숨겨뒀던 `sk_buff`에 대한 포인터를 패킷 수신 때 얻어내서 사용한다.

근데 사실은 Tilera의 이런 방식이 특이한 쪽에 속한다. 일반적으로는 NIC에게 버퍼를 줄 때 함께 제공한 어떤 추가 정보(가령 배열 색인이나 주소 자체)를 패킷 수신 때 함께 받아서 대응하는 `sk_buff`를 알아낸다. 거기엔 어떤 헤더도 숨어 있지 않다.

[linux/drivers/net/ethernet/cisco/enic/enic_main.c](https://github.com/torvalds/linux/blob/master/drivers/net/ethernet/cisco/enic/enic_main.c):
```c
static int enic_rq_alloc_buf(struct vnic_rq *rq)
{
    ...
    struct sk_buff *skb;
    unsigned int len = netdev->mtu + VLAN_ETH_HLEN;
    unsigned int os_buf_index = 0;
    dma_addr_t dma_addr;
    struct vnic_rq_buf *buf = rq->to_use;

    ...
    skb = netdev_alloc_skb_ip_align(netdev, len);
    if (!skb)
        return -ENOMEM;

    dma_addr = pci_map_single(enic->pdev, skb->data, len,
                              PCI_DMA_FROMDEVICE);
    if (unlikely(enic_dma_map_check(enic, dma_addr))) {
        dev_kfree_skb(skb);
        return -ENOMEM;
    }

    enic_queue_rq_desc(rq, skb, os_buf_index,
        dma_addr, len);

    return 0;
}

...

static int enic_rq_service(struct vnic_dev *vdev, struct cq_desc *cq_desc,
    u8 type, u16 q_number, u16 completed_index, void *paque)
{
    struct enic *enic = vnic_dev_priv(vdev);

    vnic_rq_service(&enic->rq[q_number], cq_desc,
        completed_index, VNIC_RQ_RETURN_DESC,
        enic_rq_indicate_buf, opaque);

    return 0;
}

static void enic_rq_indicate_buf(struct vnic_rq *rq,
    struct cq_desc *cq_desc, struct vnic_rq_buf *buf,
    int skipped, void *opaque)
{
    ...
    struct sk_buff *skb;
    ...

    skb = buf->os_buf;
    ...

    if (eop && bytes_written > 0) {
        ...
        if (!(netdev->features & NETIF_F_GRO))
            netif_receive_skb(skb);
        else
            napi_gro_receive(&enic->napi[q_number], skb);
        ...
    } else {
        ...
    }
}
```

[linux/drivers/net/ethernet/cisco/enic/enic_res.h](https://github.com/torvalds/linux/blob/master/drivers/net/ethernet/cisco/enic/enic_res.h):
```c
static inline void enic_queue_rq_desc(struct vnic_rq *rq,
    void *os_buf, unsigned int os_buf_index,
    dma_addr_t dma_addr, unsigned int len)
{
    ...
}
```

[linux/drivers/net/ethernet/cisco/enic/vnic_rq.h](https://github.com/torvalds/linux/blob/master/drivers/net/ethernet/cisco/enic/vnic_rq.h):
```c
static inline void vnic_rq_service(struct vnic_rq *rq,
    struct cq_desc *cq_desc, u16 completed_index,
    int desc_return, void (*buf_service)(struct vnic_rq *rq,
    struct cq_desc *cq_desc, struct vnic_rq_buf *buf,
    int skipped, void *opaque), void *opaque)
{
    struct vnic_rq_buf *buf;
    int skipped;

    buf = rq->to_clean;
    while (1) {

        skipped = (buf->index != completed_index);

        (*buf_service)(rq, cq_desc, buf, skipped, opaque);

        ...
        buf = rq->to_clean;
    }
}
```
