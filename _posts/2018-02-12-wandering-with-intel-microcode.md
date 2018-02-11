---
layout: post
title: intel-microcode와 떠나는 유랑 대잔치
category: general
tags: [microcode]
---
[유령/용융](https://meltdownattack.com/)의 등골 서늘한 열기가 가시지 않은 어느 날이었다. 패키지 업데이트를 하는데 인텔 마이크로코드 어쩌고 하는 게 지나갔다. 오오, 스펙터 대응을 위한 마이크로코드 업데이트인가!? 근데 이렇게 간단하게 되는 건가? 막 부팅 할 때 Del 키 눌러야 되고 그런 거 아닌가? 한번 살펴보기로 했다.

(참고: 리눅스뿐 아니라 윈도우에서도 소프트웨어 업데이트 형태로 마이크로코드를 갱신한다.)

### 알맹이 쪽 식탁 고립

간만에 커널까지 업데이트 했으니 일단 재부팅을 하고, 에피타이저로 멜트다운 쪽을 슬쩍 살펴보자. KPTI가 Kernel Page Table Isolation이었던가...

```
$ grep ISOLATION /boot/config-`uname -r`
CONFIG_MEMORY_ISOLATION=y
CONFIG_PAGE_TABLE_ISOLATION=y
```

격리되는 게 페이지 테이블만은 아닌가보다. 다행이다. 조금은 덜 외로울 테니...

(참고: `CONFIG_MEMORY_ISOLATION`은 멜트다운과 관계없다. 여러 해 전에 [김민찬 님이 도입](https://github.com/torvalds/linux/commit/ee6f509c32740)한 구성 항목이다.)

```
$ cat /proc/cpuinfo
...
flags		: ... pti ...
bugs		: cpu_meltdown spectre_v1 spectre_v2
...
```

아, 버그투성이 CPU 같으니라고... 근데 멜트다운이 좀 밀리는 모양새다. 스펙터에는 무려 버전이 붙어 있다.

### 見跡

다시 마이크로코드로 돌아가서, 업데이트 중에 봤으니 첫 번째 단서는 apt 터미널 로그다.

`/var/log/apt/term.log`:
```
...
Preparing to unpack .../47-linux-image-4.13.0-32-generic_4.13.0-32.35_amd64.deb ...
Examining /etc/kernel/preinst.d/
run-parts: executing /etc/kernel/preinst.d/intel-microcode 4.13.0-32-generic /boot/vmlinuz-4.13.0-32-generic
...
Preparing to unpack .../64-intel-microcode_3.20180108.0+really20170707ubuntu17.10.1_amd64.deb ...
Unpacking intel-microcode (3.20180108.0+really20170707ubuntu17.10.1) over (3.20170707.1) ...
...
Setting up intel-microcode (3.20180108.0+really20170707ubuntu17.10.1) ...
update-initramfs: deferring update (trigger activated)
intel-microcode: microcode will be updated at next boot
...
```

47번째과 64번째... 업데이트 미루다가 몰아서 한 티가 난다. 이런 시절엔 재깍재깍 해 줘야 하는데 말이다.

커널 패키지 설치 과정에서 `/etc/kernel/preinst.d/intel-microcode`라는 프로그램을 실행하기도 하고, 무엇보다 intel-microcode라는 패키지가 있다. (버전 문자열의 "really20170707"이 거슬리지만 일단 넘어가자.) 그리고 다음 부팅 때 마이크로코드가 업데이트 된다는 친절한 메시지도 있다.

패키지로 들어가기 전에 자잘한 거 하나 확인해 보자. 커널 패키지 설치 때 실행하던 그 프로그램은 뭘까?

```
$ dpkg -S /etc/kernel/preinst.d/intel-microcode
intel-microcode: /etc/kernel/preinst.d/intel-microcode
```

역시나 intel-microcode 패키지가 흑막이다. 근데 뭐 하는 프로그램일까?

`/etc/kernel/preinst.d/intel-microcode`:
```sh
grep -q cpu/cpuid /proc/devices || modprobe -q cpuid || true

:
```

`/proc/devices` 파일에 "cpu/cpuid"라는 행이 없으면 cpuid라는 커널 모듈 적재, 그리고 무조건 웃으며 반환. `true`만으로는 불안했던지 `:`까지 붙어 있다. x86 아키텍처 한정인 cpuid 모듈은 `/dev/cpu/CPUNUM/cpuid`라는 [장치 파일](https://github.com/wariua/manpages-ko/wiki/cpuid%284%29)을 제공한다. 이 파일을 통해 특정 CPU에서 [CPUID 인스트럭션](https://en.wikipedia.org/wiki/CPUID)을 실행해서 [여러 정보](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/cpufeatures.h)를 얻을 수 있다. [모듈 소스 코드](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpuid.c)도 아주 단촐하다. 그럼 장치 파일을 한번 보자.

```
$ ls -l /dev/cpu/
합계 0
crw------- 1 root root 10, 184  2월  7 11:17 microcode
```

이런, 없다. 가능한 원인이 두 가지인데, 시스템 상의 CPU가 0개이거나, cpuid 모듈이 안 올라가 있는 것이다. 정확한 원인을 밝히는 건 기회 될 때 별도 포스트에서 다루도록 하겠다. 그나저나 여기에도 `microcode`라는 이름이 보인다.

`.../preinst.d/intel-microcode`가 cpuid 모듈을 적재하는 건 좋은데 왜 적재하느냐고 하면, 현재 시스템의 CPU 모델을 알아내서 거기 맞는 마이크로코드를 골라내기 위해서이다.

### 역사란 현재와 과거의 끊임없는 대화이다.

이제 슬슬 intel-microcode 패키지를 뒤질 차례인데... 귀찮다.

근데 intel-microcode 패키지 버전 문자열의 그 찜찜한 "really20170707"은 뭐였을까? "really"라는 단어에 담긴 처절한 진정성을 모른 체하고 넘어가는 건 도리가 아니다. 패키지 버전에 무슨 사연이 있었던 걸까.

```
$ apt changelog intel-microcode
intel-microcode (3.20180108.0+really20170707ubuntu17.10.1) artful-security; urgency=medium

  * Revert to 20170707 version of microcode because of regressions on
    certain hardware. (LP: #1742933)

 -- Marc Deslauriers <marc.deslauriers@ubuntu.com>  Mon, 22 Jan 2018 07:16:40 -0500

intel-microcode (3.20180108.0~ubuntu17.10.1) artful-security; urgency=medium

  * New upstream microcode datafile 20180108
    + New Microcodes:
      sig 0x000506c9, pf_mask 0x03, 2017-03-25, rev 0x002c, size 16384
      sig 0x000706a1, pf_mask 0x01, 2017-12-26, rev 0x0022, size 73728
      sig 0x000906ea, pf_mask 0x22, 2018-01-04, rev 0x0080, size 97280
      sig 0x000906eb, pf_mask 0x02, 2018-01-04, rev 0x0080, size 98304
    + Updated Microcodes:
      sig 0x000306c3, pf_mask 0x32, 2017-11-20, rev 0x0023, size 23552
      sig 0x000306d4, pf_mask 0xc0, 2017-11-17, rev 0x0028, size 18432
      sig 0x000306e4, pf_mask 0xed, 2017-12-01, rev 0x042a, size 15360
      sig 0x000306f2, pf_mask 0x6f, 2017-11-17, rev 0x003b, size 33792
      sig 0x000306f4, pf_mask 0x80, 2017-11-17, rev 0x0010, size 17408
      sig 0x00040651, pf_mask 0x72, 2017-11-20, rev 0x0021, size 22528
      sig 0x00040661, pf_mask 0x32, 2017-11-20, rev 0x0018, size 25600
      sig 0x00040671, pf_mask 0x22, 2017-11-17, rev 0x001b, size 13312
      sig 0x000406e3, pf_mask 0xc0, 2017-11-16, rev 0x00c2, size 99328
      sig 0x00050654, pf_mask 0xb7, 2017-12-08, rev 0x200003c, size 27648
      sig 0x00050662, pf_mask 0x10, 2017-12-16, rev 0x0014, size 31744
      sig 0x00050663, pf_mask 0x10, 2017-12-16, rev 0x7000011, size 22528
      sig 0x000506e3, pf_mask 0x36, 2017-11-16, rev 0x00c2, size 99328
      sig 0x000806e9, pf_mask 0xc0, 2018-01-04, rev 0x0080, size 98304
      sig 0x000806ea, pf_mask 0xc0, 2018-01-04, rev 0x0080, size 98304
      sig 0x000906e9, pf_mask 0x2a, 2018-01-04, rev 0x0080, size 98304
   * source: remove unneeded intel-ucode/ directory
   * source: remove superseded upstream data file: 20170707

 -- Marc Deslauriers <marc.deslauriers@ubuntu.com>  Tue, 09 Jan 2018 13:19:16 -0500

...
```

2018년 1월 8일에 새 버전이 나왔다가 문제가 발견돼서 이전 버전으로 되돌렸다. 뉴스 타임라인과 얼추 아귀가 맞는다. 그렇다면 날카로운 육감으로 업데이트 적용을 유예하다 보니 문제 있는 1월 8일 버전 설치를 피할 수 있었던 셈이다. 역시 업데이트는 미뤄야 제맛이다. (참고: 아니다.)

### 패키지를 패키지로 설치하지 않으면 뭐라 불러야 하나

딴짓거리도 떨어졌으니 패키지를 볼 차례다. 뭐가 들어 있을까?

```
$ dpkg -L intel-microcode
/.
/etc
/etc/default
/etc/default/intel-microcode
/etc/kernel
/etc/kernel/preinst.d
/etc/kernel/preinst.d/intel-microcode
/etc/modprobe.d
/etc/modprobe.d/intel-microcode-blacklist.conf
/lib
/lib/firmware
/lib/firmware/intel-ucode
/lib/firmware/intel-ucode/06-0f-02
/lib/firmware/intel-ucode/06-0f-06
/lib/firmware/intel-ucode/06-0f-07
...
/lib/firmware/intel-ucode/0f-06-08
/usr
/usr/share
/usr/share/doc
/usr/share/doc/intel-microcode
/usr/share/doc/intel-microcode/NEWS.Debian.gz
/usr/share/doc/intel-microcode/README.Debian.gz
/usr/share/doc/intel-microcode/changelog.Debian.gz
/usr/share/doc/intel-microcode/copyright
/usr/share/initramfs-tools
/usr/share/initramfs-tools/hooks
/usr/share/initramfs-tools/hooks/intel_microcode
```

`/etc/default/intel-microcode`는 평범한 설정 파일이고 `/etc/kernel/preinst.d/intel-microcode`는 이미 살펴봤다. 블랙리스트 파일은 뭘까?

`/etc/modprobe.d/intel-microcode-blacklist.conf`:
```
# The microcode module attempts to apply a microcode update when
# it autoloads.  This is not always safe, so we block it by default.
blacklist microcode
```

microcode라는 커널 모듈이 있고, 시스템 동작 중 마이크로코드 갱신을 피하기 위해 모듈 적재를 막는다. 이 파일엔 [슬픈 전설](https://bugs.launchpad.net/intel/+bug/1370352)이 있다.

그리고 세 박자 이름의 바이너리 파일들이 잔뜩 이어지다가 문서가 몇 개 나온다. 그리고 마지막에 뜬금없는 경로의 파일이 훅 (pun intended...!) 등장한다.

`/usr/share/initramfs-tools/hooks/intel_microcode`:
```sh
...
# Generates a copy of the Intel microcode (by default tailored to the
# running system), and installs it in the early initramfs.
...

IUCODE_CONFIG=/etc/default/intel-microcode
...
```

아까 그 평범한 설정 파일이 여기 쓰인다. 이어지는 코드가 참 좋은 내용인데, 안 읽고 패스.

요약하면 `/lib/firmware/intel-ucode/`에 마이크로코드 파일들이 옹기종기 모여 있고, [initrd](https://wariua.cafe24.com/wiki/Documentation/admin-guide/initrd.rst) 이미지 만들 때 실행하는 훅에서 뭔가 심오한 작업을 한다.

### 소스 코드의 소스가 없으면... 소스를 끼얹자!

사실 바이너리 패키지는 이분법적인 느낌도 있고 해서 쫌 그렇다. 진정한 프로그래머라면 소스를 봐야 한다.

```
$ apt source intel-microcode
패키지 목록을 읽는 중입니다... 완료
알림: 'intel-microcode' 패키징은 다음 'Git' 버전 컨트롤 시스템에서 관리합니다:
git://git.debian.org/users/hmh/intel-microcode.git
Please use:
git clone git://git.debian.org/users/hmh/intel-microcode.git
to retrieve the latest (possibly unreleased) updates to the package.
...
```

"please"라는데 어찌 무시할 수 있을까. git clone 하고서 로그를 보니, 췟, 22일 다음 변경이 없다. 아직은. 인텔 엔지니어들, 화이팅. (아, AMD 엔지니어들도 화이팅.)

소스 트리에는 뭐가 있을까?

```
$ cd intel-microcode
$ ls *
Makefile            microcode-20080220.dat  microcode-20100914.dat  releasenote
changelog           microcode-20080910.dat  microcode-20110915.dat
cpu-signatures.txt  microcode-20100826.dat  microcode-20171117.dat

debian:
README.Debian  diff-latest-pack.sh       intel-microcode.modprobe-blacklist
README.source  initramfs.hook            intel-microcode.postinst
changelog      intel-microcode.NEWS      intel-microcode.postrm
compat         intel-microcode.default   rules
control        intel-microcode.dirs      source
copyright      intel-microcode.kpreinst  ucode-blacklist.txt
```

딱 봐도 `microcode-YYYYMMDD.dat` 파일이 핵심이다.

`microcode-20171117.dat`:
```
...
/*  Fri Nov 17 01:21:16 CST 2017  */
/*  m02f0a15.inc  */
0x00000001,     0x00000015,     0x08212002,     0x00000f0a,
0x63e49a0c,     0x00000001,     0x00000002,     0x00000000,
0x00000000,     0x00000000,     0x00000000,     0x00000000,
0xffffffe9,     0x80e0e61a,     0x7cb2c607,     0xbe1e2fc9,
0xacaf1526,     0x31514e28,     0x9252d426,     0xb999ebf8,
...
0x78f6c3c5,     0x3d7ca7ef,     0xf3bf6b34,     0x6a42e9df,
/*  m10f252c.inc  */
0x00000001,     0x0000002c,     0x08262004,     0x00000f25,
0x62d062ab,     0x00000001,     0x00000010,     0x00000000,
0x00000000,     0x00000000,     0x00000000,     0x00000000,
0x017464fa,     0xd2fb3494,     0xaf1850af,     0x5af88989,
0xd3a0a9dd,     0x8e8bd3ad,     0x8c288786,     0xf15f02b0,
...
0x68c94a3c,     0xc9efee21,     0x4d432eb7,     0xe3f5a59f,
/*  MU165310.inc  */
0x00000001,     0x00000010,     0x06281999,     0x00000653,
0x4b6dfc5e,     0x00000001,     0x00000001,     0x00000000,
0x00000000,     0x00000000,     0x00000000,     0x00000000,
0xa9f6b598,     0xec73f4eb,     0xffce329e,     0x6c5e06dd,
0xce5188bb,     0xe94f7930,     0x15eedaa4,     0xf18757d4,
...
0x57688086,     0x218e4005,     0xca054e3d,     0xc1a3c3ec,
/*  MU16b11d.inc  */
0x00000001,     0x0000001d,     0x02202001,     0x000006b1,
0x6d9b5661,     0x00000001,     0x00000020,     0x00000000,
0x00000000,     0x00000000,     0x00000000,     0x00000000,
0x965e94ce,     0xea6ba8df,     0xbfae08d4,     0x3ec50299,
0xbf2b31ed,     0xbd80d71d,     0x19bfb06d,     0x95e56995,
...
...
```

총 99,334줄, 주석 빼면 99,136줄, 한 줄에 16바이트니까 1.6MB 정도의 데이터다. 바이너리 패키지가 `/lib/firmware/intel-ucode/`에 설치하는 파일들이 총 1.2MB 정도. 좀 거르거나 압축을 하나 보다.

`debian/README.Debian' (번역):
```
...

마이크로코드 업데이트는 일회성이다. 즉, 프로세서 하드 리셋이나 프로세서 전원 차단 후에는 사라지게 된다. 부팅 때마다, 그리고 시스템이 RAM이나 디스크에서 대기 모드로 있다가 깨어날 때에도 다시 적용시켜야 한다.

...

시스템 시동 중 마이크로코드 업데이트를 건너뛰려면 부트로더(grub, lilo 등)에서 커널로 "dis_ucode_ldr" 매개변수를 (따옴표는 빼고) 전달하게 해야 한다.

...

인텔에서 마이크로코드 번들 새 버전을 직접 내려받을 수 있다. "Linux Microcode"로 검색하면 된다.

> https://downloadcenter.intel.com/search?keyword=linux+microcode

...
```

`debian/README.source`에도 좋은 정보가 있다. (물론 읽지는 않았다.) `debian/intel-microcode.postinst`는 이름 그대로 패키지 설치 후 실행되는 스크립트인데, 여기서 `update-initramfs -u` 명령을 실행해서 initrd 이미지 갱신을 지시하고 그 친절한 메시지("microcode will be updated at next boot")를 찍는다.

`debian/` 안의 `intel-microcode.default`, `intel-microcode.kpreinst`, `intel-microcode.modprobe-blacklist`, `initramfs.hook` 등의 파일은 바이너리 패키지에서 봤던 파일들의 아명이다.

`debian/rules`:
```make
...
PACKAGE := intel-microcode
DEBDIR  := $(CURDIR)/debian
PKGDIR  := $(DEBDIR)/$(PACKAGE)
...
IUCODE_TOOL := iucode_tool
...
ifneq (,$(filter amd64 x32,$(DEB_HOST_ARCH)))
IUCODE_FILE := intel-microcode-64.bin
else
IUCODE_FILE := intel-microcode.bin
endif
...

override_dh_auto_install:
        dh_testdir
        dh_install

        # split microcode pack
        $(IUCODE_TOOL) -q --write-firmware="$(PKGDIR)/lib/firmware/intel-ucode" $(IUCODE_FILE)

        # apply best-effort blacklist
        if [ -r debian/ucode-blacklist.txt ] ; then \
                cat debian/ucode-blacklist.txt | while read -r fn crap ; do \
                        if [ -r "$(PKGDIR)/lib/firmware/intel-ucode/$${fn}" ] ; then \
                                mv "$(PKGDIR)/lib/firmware/intel-ucode/$${fn}" "$(PKGDIR)/lib/firmware/intel-ucode/$${fn}.initramfs" ;\
                                echo "Renaming blacklisted microcode $${fn}" ; \
                        fi ; \
                done ; \
        fi

        mkdir -p "$(PKGDIR)/usr/share/initramfs-tools/hooks"
        install -m 755 "$(DEBDIR)/initramfs.hook" \
                "$(PKGDIR)/usr/share/initramfs-tools/hooks/$(INITRAMFS_NAME)"
        mkdir -p "$(PKGDIR)/etc/kernel/preinst.d"
        install -m 755 "$(DEBDIR)/$(PACKAGE).kpreinst" \
                "$(PKGDIR)/etc/kernel/preinst.d/$(PACKAGE)"

        ...
```

아래쪽을 보면 패키징용 디렉터리에 새 이름으로 설치하는 걸 볼 수 있다. 근데 중요한 건 그게 아니다. 바로 위 블록에선 `debian/ucode-blacklist.txt`에 걸리는 파일들에 `.initramfs`를 붙여서 적용 배제하는데, 이것도 중요한 게 아니다. 이 파일의 정수는 그 위의 줄이다. `$(IUCODE_TOOL)`... 그렇다, [IU](http://ifaveent.co.kr/?p=11)다. [ii](https://tools.suckless.org/ii/)도 있고 [uu](https://linux.die.net/man/1/uuencode)도 있지만 우리는 언제나 마음 한 켠 결핍감을 모른 척 덮어 두고서 하루를, 그리고 별반 다른 것 없는 또 다른 하루를 살아내 왔다. 이제 다가올 시간은 이전과 다를 것이다. 우리에게 내일이 생겼다.

에... 그래서, iu를 많이 보려면 상위 디렉터리의 `Makefile`을 보면 된다. `iucode_tool`을 이용해 `microcode-YYYYMMDD.dat` 파일들을 `intel-microcode-64.bin`/`intel-microcode.bin`으로 '컴파일' 하고, `debian/rules`에서 다시 `iucode_tool`로 그걸 쪼개서 '설치'한다. 그렇게 바이너리 패키지가 만들어진다.

### 처음의 처

intel-microcode 패키지를 설치하면 이런저런 파일들을 복사하고서 좀 있다 initramfs 훅을 실행한다. initramfs 훅에선 뭘 할까?

`/usr/share/initramfs-tools/hooks/intel-microcode`:
```sh
...
IUCODE_FW_DIR=/lib/firmware/intel-ucode
...

# generate early initramfs microcode update mode..."
EFW=$(mktemp "${TMPDIR:-/var/tmp}/mkinitramfs-EFW_XXXXXXXXXX") || {
        echo "E: intel-microcode: cannot create temporary file" >&2
        exit 1
    }
( find /usr/share/misc -maxdepth 1 -type f -name 'intel-microcode*' -print0 ;
  find "${IUCODE_FW_DIR}" -maxdepth 0 -type d -print0 ) 2>/dev/null \
| xargs -0 -r -x ${IUCODE_TOOL} ${IUCODE_TOOL_OPTIONS} \
                --write-earlyfw="${EFW}" --overwrite \
                ${IUCODE_TOOL_EXTRA_OPTIONS} \
&& prepend_earlyinitramfs "${EFW}" && {
        rm "${EFW}"
        exit 0
}

...
```

결국은 `iucode_tool`을 실행하는 것이다. (그렇다, 많이 많이 실행해야 한다.)

```
$ iucode_tool -v --scan-system --write-earlyfw="mkinitramfs-EFW_thisistemp" \
> --overwrite /lib/firmware/intel-ucode
iucode_tool: system has processor(s) with signature 0x000306c3
iucode_tool: assuming all processors have the same type, family and model
iucode_tool: processed 87 valid microcode(s), 87 signature(s), 87 unique signature(s)
iucode_tool: selected 1 microcode(s), 1 signature(s)
iucode_tool: Writing selected microcodes to: mkinitramfs-EFW_thisistemp
iucode_tool: mkinitramfs-EFW_thisistemp: 1 microcode entries written, 23552 bytes
```

`--scan-system` 옵션을 주면 `/dev/cpu/CPUNUM/cpuid` 파일로 프로세서 시그너처 집합을 얻어서 거기 일치하는 마이크로코드들만 모은다.

```
$ file mkinitramfs-EFW_thisistemp
mkinitramfs-EFW_thisistemp: ASCII cpio archive (SVR4 with no CRC)
$ cpio -t -F ./mkinitramfs-EFW_thisistemp
kernel
kernel/x86
kernel/x86/microcode
kernel/x86/microcode/.enuineIntel.align.0123456789abc
kernel/x86/microcode/GenuineIntel.bin
46 blocks
```

이렇게 조그만 cpio 아카이브를 만든 다음 `prepend_earlyinitramfs` 함수를 실행한다. 즉, 마이크로코드 아카이브를 진짜 initramfs 이미지 앞에 덧붙인다.

```
$ cpio -t -F /boot/initrd.img-`uname -r`
kernel
kernel/x86
kernel/x86/microcode
kernel/x86/microcode/.enuineIntel.align.0123456789abc
kernel/x86/microcode/GenuineIntel.bin
46 blocks
```

정말 46개 블럭 뒤에 initrd 이미지가 붙어 있을까?

```
$ dd skip=46 if=/boot/initrd.img-`uname -r` status=none | gunzip | cpio -t --quiet
.
var
var/lib
var/lib/dhcp
var/cache
var/cache/ldconfig
var/cache/ldconfig/aux-cache
var/cache/fontconfig
usr
usr/share
usr/share/fonts
usr/share/fonts/truetype
...
bin/busybox
bin/udevadm
bin/plymouth
bin/cryptroot-unlock
bin/kmod
```

잘 붙어 있나 보다. 하지만 이것만으론 부족하다.

```
$ dd skip=46 if=/boot/initrd.img-`uname -r` status=none | gunzip | \
> cpio -t --quiet | grep iu
lib/modules/4.13.0-32-generic/kernel/drivers/net/ethernet/cavium
lib/modules/4.13.0-32-generic/kernel/drivers/net/ethernet/cavium/thunder
...
lib/modules/4.13.0-32-generic/kernel/drivers/net/ethernet/sun/niu.ko
lib/modules/4.13.0-32-generic/kernel/drivers/net/phy/mdio-cavium.ko
```

아쉽지만 이 정도면 됐다.

### 아... 이번엔 제목 뭐 하지.....

initrd 이미지 앞에 마이크로코드를 왜 덧붙이는 걸까? 커널 문서 [x86/microcode.txt](https://wariua.cafe24.com/wiki/Documentation/x86/microcode.txt)에 답이 있다. 요약하면, 세 가지 방식으로 CPU 마이크로코드를 업데이트 할 수 있다.

1. 부팅 초반: initrd 이미지 앞에 마이크로코드를 담은 cpio 아카이브를 붙여 놓으면 커널이 알아서 적용.
2. 런타임: `/dev/cpu/microcode`나 `/sys/devices/system/cpu/microcode/reload`를 통해 적용. 위험 요소가 있어서 잘 안 씀.
3. 커널 내장: initrd 이미지 대신 커널에 내장.

지금까지 여정은 1번 방식인 셈이다. 그래서 커널은 어떻게 마이크로코드를 적용시킬까?

`linux/arch/x86/kernel/head64.c`:
```c
asmlinkage __visible void __init x86_64_start_kernel(char * real_mode_data)
{
        ...

        /*
         * Load microcode early on BSP.
         */
        load_ucode_bsp();

        /* set init_top_pgt kernel high mapping*/
        init_top_pgt[511] = early_top_pgt[511];

        x86_64_start_reservations(real_mode_data);
}

void __init x86_64_start_reservations(char *real_mode_data)
{
        ...

        start_kernel();
}
```

정말 초반이다. `load_ucode_bsp()`에서 이어지는 나머지 코드는 [linux/arch/x86/kernel/cpu/microcode/](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/cpu/microcode) 안에 있다.

`linux/arch/x86/kernel/cpu/microcode/core.c`:
```c
void __init load_ucode_bsp(void)
{
        ...

	if (check_loader_disabled_bsp())
                return;

        if (intel)
                load_ucode_intel_bsp();
        else
                load_ucode_amd_bsp(cpuid_1_eax);
}

static bool __init check_loader_disabled_bsp(void)
{
        static const char *__dis_opt_str = "dis_ucode_ldr";

        ...
        const char *cmdline = boot_command_line;
        const char *option  = __dis_opt_str;
        bool *res = &dis_ucode_ldr;

        ...

        if (cmdline_find_option_bool(cmdline, option) <= 0)
                *res = false;

        return *res;
}
```

커널 매개변수 "dis_ucode_ldr"가 없어야 적재를 진행한다.

`linux/arch/x86/kernel/cpu/microcode/intel.c`:
```c
static const char ucode_path[] = "kernel/x86/microcode/GenuineIntel.bin";

/* Current microcode patch used in early patching on the APs. */
static struct microcode_intel *intel_ucode_patch;

...

void __init load_ucode_intel_bsp(void)
{
        struct microcode_intel *patch;
        struct ucode_cpu_info uci;

        patch = __load_ucode_intel(&uci);
        if (!patch)
                return;

        uci.mc = patch;

        apply_microcode_early(&uci, true);
}

static struct microcode_intel *__load_ucode_intel(struct ucode_cpu_info *uci)
{
        static const char *path;
        struct cpio_data cp;
        bool use_pa;

        if (IS_ENABLED(CONFIG_X86_32)) {
                path      = (const char *)__pa_nodebug(ucode_path);
                use_pa    = true;
        } else {
                path      = ucode_path;
                use_pa    = false;
        }

        /* try built-in microcode first */
        if (!load_builtin_intel_microcode(&cp))
                cp = find_microcode_in_initrd(path, use_pa);

        if (!(cp.data && cp.size))
                return NULL;

        collect_cpu_info_early(uci);

        return scan_microcode(cp.data, cp.size, uci, false);
}

static int apply_microcode_early(struct ucode_cpu_info *uci, bool early)
{
        struct microcode_intel *mc;
        u32 rev;

        mc = uci->mc;
        if (!mc)
                return 0;

        /* write microcode via MSR 0x79 */
        native_wrmsrl(MSR_IA32_UCODE_WRITE, (unsigned long)mc->bits);

        rev = intel_get_microcode_revision();
        if (rev != mc->hdr.rev)
                return -1;

#ifdef CONFIG_X86_64
        /* Flush global tlb. This is precaution. */
        flush_tlb_early();
#endif
        uci->cpu_sig.rev = rev;

        if (early)
                print_ucode(uci);
        else
                print_ucode_info(uci, mc->hdr.date);

        return 0;
}

static inline void print_ucode(struct ucode_cpu_info *uci)
{
        struct microcode_intel *mc;

        mc = uci->mc;
        if (!mc)
                return;

        print_ucode_info(uci, mc->hdr.date);
}

static void
print_ucode_info(struct ucode_cpu_info *uci, unsigned int date)
{
        pr_info_once("microcode updated early to revision 0x%x, date = %04x-%02x-%02x\n",
                     uci->cpu_sig.rev,
                     date & 0xffff,
                     date >> 24,
                     (date >> 16) & 0xff);
}
```

마이크로코드를 갱신하고 나면 뭔가 찍나 보다.

```
$ dmesg | grep microcode
[    0.000000] microcode: microcode updated early to revision 0x22, date = 2017-01-27
[    0.718671] microcode: sig=0x306c3, pf=0x2, revision=0x22
[    0.718772] microcode: Microcode Update Driver: v2.2.
```

타임스탬프가 무려 0.000000이다. 실제로 `dmesg`의 첫 번째 줄이다.

다시 마이크로코드 적용 코드로 돌아가서, `native_wrmsrl()`을 보면,

`linux/arch/x86/include/asm/microcode.h`:
```c
#define native_wrmsrl(msr, val)                         \
        __wrmsr((msr), (u32)((u64)(val)),
                       (u32)((u64)(val) >> 32))
```

`linux/arch/x86/include/asm/msr.h`:
```c
static inline void notrace __wrmsr(unsigned int msr, u32 low, u32 high)
{
        asm volatile("1: wrmsr\n"
                     "2:\n"
                     _ASM_EXTABLE_HANDLE(1b, 2b, ex_handler_wrmsr_unsafe)
                     : : "c" (msr), "a"(low), "d" (high) : "memory");
}
```

결국 마이크로코드가 저장된 메모리 블록 주소로 <tt>[wrmsr](https://en.wikipedia.org/wiki/Model-specific_register)</tt> 인스트럭션을 실행하는 것이다.

한편으로 `struct microcode_intel`을 보면 마이크로코드 데이터의 기본 형식을 짐작할 수 있다.

`linux/arch/x86/include/asm/microcode_intel.h`:
```c
struct microcode_header_intel {
        unsigned int            hdrver;
        unsigned int            rev;
        unsigned int            date;
        unsigned int            sig;
        unsigned int            cksum;
        unsigned int            ldrver;
        unsigned int            pf;
        unsigned int            datasize;
        unsigned int            totalsize;
        unsigned int            reserved[3];
};

struct microcode_intel {
        struct microcode_header_intel hdr;
        unsigned int            bits[0];
};
```

다음은 `microcode-20171117.dat`의 일부이다.

```
/*  m32306c3_00000022.inc  */
0x00000001,     0x00000022,     0x01272017,     0x000306c3,
0xad518a4e,     0x00000001,     0x00000032,     0x000057d0,
0x00005800,     0x00000000,     0x00000000,     0x00000000,
0x00000000,     0x000000a1,     0x00020001,     0x00000022,
...
```

### 런타임 갱신

한편으로 `core.c` 파일은 런타임 마이크로코드 업데이트 기능을 제공하는 드라이버이기도 하다.

`linux/arch/x86/kernel/cpu/microcode/core.c`:
```c
static struct miscdevice microcode_dev = {
        .minor                  = MICROCODE_MINOR,
        .name                   = "microcode",
        .nodename               = "cpu/microcode",
        .fops                   = &microcode_fops,
};

static int __init microcode_dev_init(void)
{
        int error;

        error = misc_register(&microcode_dev);
        if (error) {
                pr_err("can't misc_register on minor=%d\n", MICROCODE_MINOR);
                return error;
        }

        return 0;
}

...

static DEVICE_ATTR(reload, 0200, NULL, reload_store);
static DEVICE_ATTR(version, 0400, version_show, NULL);
static DEVICE_ATTR(processor_flags, 0400, pf_show, NULL);

static struct attribute *mc_default_attrs[] = {
        &dev_attr_version.attr,
        &dev_attr_processor_flags.attr,
        NULL
};

static const struct attribute_group mc_attr_group = {
        .attrs                  = mc_default_attrs,
        .name                   = "microcode",
};

static int mc_device_add(struct device *dev, struct subsys_interface *sif)
{
        ...

        err = sysfs_create_group(&dev->kobj, &mc_attr_group);
        if (err)
                return err;

        if (microcode_init_cpu(cpu, true) == UCODE_ERROR)
                return -EINVAL;

        return err;
}

static struct subsys_interface mc_cpu_interface = {
        .name                   = "microcode",
        .subsys                 = &cpu_subsys,
        .add_dev                = mc_device_add,
        .remove_dev             = mc_device_remove,
};

...

/**
 * mc_bp_resume - Update boot CPU microcode during resume.
 */
static void mc_bp_resume(void)
{
        int cpu = smp_processor_id();
        struct ucode_cpu_info *uci = ucode_cpu_info + cpu;

        if (uci->valid && uci->mc)
                microcode_ops->apply_microcode(cpu);
        else if (!uci->mc)
                reload_early_microcode();
}

static struct syscore_ops mc_syscore_ops = {
        .resume                 = mc_bp_resume,
};

...

static void mc_bp_resume(void)
{

static struct attribute *cpu_root_microcode_attrs[] = {
        &dev_attr_reload.attr,
        NULL
};

static const struct attribute_group cpu_root_microcode_group = {
        .name  = "microcode",
        .attrs = cpu_root_microcode_attrs,
};

int __init microcode_init(void)
{
        ...

        error = subsys_interface_register(&mc_cpu_interface);
        ...

        error = sysfs_create_group(&cpu_subsys.dev_root->kobj,
                                   &cpu_root_microcode_group);
        ...

        error = microcode_dev_init();
        ...

        register_syscore_ops(&mc_syscore_ops);
        ...
}
```

`/dev/cpu/microcode` 파일의 주인이 드디어 밝혀졌다. 그리고 세 가지 sysfs 파일(`/sys/devices/system/cpu/cpuN/microcode/{processor_flags,version}`, `/sys/devices/system/cpu/microcode/reload`)을 제공하는 것도, CPU 깰 때 마이크로코드 재적용해 주는 것도 이 드라이버다.

### 요약

인텔에서 제공하는 마이크로코드를 initrd 이미지 앞에 붙여 뒀다가 부팅 초반에 그 데이터로 <tt>wrmsr</tt> 인스트럭션을 실행한다. 끝.
