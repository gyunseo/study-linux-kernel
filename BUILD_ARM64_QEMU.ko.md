# Linux 4.6.7 ARM64 빌드와 QEMU 실행

이 문서는 이 저장소의 기존 빌드 기록을 바탕으로 작성했다. 호스트는 Ubuntu 24.04 계열 x86_64, 타깃은 ARM64이며, 커널을 호스트에 설치하지 않고 QEMU에서 실행한다.

## 1. 확인된 기존 빌드 방식

`~/.bash_history`에 다음 명령이 남아 있다.

```bash
arm64-kmake defconfig
time arm64-kmake -j$(nproc) Image 2>&1 | sudo tee build.log
```

`arm64-kmake`의 정의는 `~/.bashrc`에 있으며 아래 3절과 같다. 실제 결과물과 로그에서 확인한 정보는 다음과 같다.

| 항목 | 확인한 내용 | 근거 |
| --- | --- | --- |
| 커널 | Linux 4.6.7, ARM64 | `include/config/kernel.release`, `.config` |
| 컴파일러 | aarch64-linux-gnu-gcc 13.3.0 | `include/generated/compile.h`, `init/.main.o.cmd` |
| 빌드 시각 | 2026-09-20 02:07:07 KST | `include/generated/compile.h` |
| 최종 타깃 | `Image` | 히스토리, `build.log` 마지막 줄 |
| 설정 | SMP, PREEMPT, DEBUG_INFO, FRAME_POINTER, XEN, XEN_DOM0 활성화 | 현재 `.config` |

GCC 5.5 툴체인을 다운로드한 이력도 있으나, 현재 남은 산출물은 GCC 13.3.0으로 빌드했다. `compile_commands.json`은 발견되지 않았으며, 개별 컴파일 명령은 Kbuild의 `.*.cmd` 파일에 남아 있다.

현재 `.config`가 `defconfig` 이후 어떤 과정을 거쳐 변경되었는지는 히스토리만으로 확정할 수 없다. 기존 설정 재현에는 현재 `.config`를 보관해서 사용하는 것이 좋다.

## 2. 사전 설치 패키지

Ubuntu 24.04 기준 명령이다.

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu \
  libc6-dev-arm64-cross bc bison flex libncurses-dev libssl-dev libelf-dev \
  qemu-system-arm gdb-multiarch \
  cpio gzip bzip2 curl file
```

| 패키지 | 용도 |
| --- | --- |
| `build-essential` | make, 호스트 GCC 및 빌드 도구 |
| `gcc-aarch64-linux-gnu`, `binutils-aarch64-linux-gnu` | ARM64 크로스 컴파일·링크·objcopy |
| `bc`, `bison`, `flex` | 커널 및 도구 빌드 지원 |
| `libncurses-dev` | `menuconfig` 사용 시 필요 |
| `libssl-dev`, `libelf-dev` | 설정에 따라 사용하는 빌드 의존성 |
| `qemu-system-arm` | Ubuntu의 `qemu-system-aarch64` 제공 패키지 |
| `gdb-multiarch` | ARM64 커널 디버깅 |
| `libc6-dev-arm64-cross` | 선택 사항인 BusyBox 정적 크로스 빌드 |
| `cpio`, `gzip`, `bzip2`, `curl`, `file` | initramfs 생성, 소스 다운로드·압축 해제, 산출물 확인 |

```bash
aarch64-linux-gnu-gcc --version
qemu-system-aarch64 --version
gdb-multiarch --version
```

문서 작성 시 로컬 GCC는 13.3.0, QEMU는 8.2.2였다. 다른 배포판·컴파일러 버전에서는 Linux 4.6.7과의 추가 호환성 조정이 필요할 수 있다.

## 3. ~/.bashrc에 빌드 함수 등록

`~/.bashrc`에 아래 정의를 추가한다. 이미 `arm64-kmake`가 있다면 해당 정의를 수정하고 중복 추가하지 않는다.

```bash
arm64-kmake() {
  make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
    KCFLAGS="-std=gnu89 -fno-pie -gdwarf-4" \
    HOSTCFLAGS="-O2 -std=gnu89 -fcommon" \
    "$@"
}
```

현재 셸에 적용하고 확인한다.

```bash
source ~/.bashrc
type arm64-kmake
```

| 옵션 | 의미 |
| --- | --- |
| `ARCH=arm64` | ARM64 커널 선택 |
| `CROSS_COMPILE=aarch64-linux-gnu-` | ARM64 도구 체인 접두사 |
| `-std=gnu89` | 오래된 소스에 맞는 C 언어 모드 |
| `-fno-pie` | 커널 컴파일 시 PIE 코드 생성 비활성화 |
| `-gdwarf-4` | DWARF 4 디버그 정보 형식 지정 |
| `HOSTCFLAGS=... -fcommon` | 호스트 도구의 tentative 전역 변수 정의를 common 심볼로 처리; 구형 DTC 등의 호환성에 사용 |

`KCFLAGS`는 커널 코드에, `HOSTCFLAGS`는 호스트에서 실행할 빌드 도구에 적용된다. 함수는 현재 디렉터리에서 make를 실행하므로 반드시 저장소 루트에서 호출한다.

## 4. 설정과 빌드

### 기존 설정으로 다시 빌드

```bash
cd ~/kernel-lab/linux-4.6.7
cp .config "/tmp/linux-4.6.7.config.$(date +%Y%m%d-%H%M%S)"
arm64-kmake olddefconfig
```

현재 설정을 유지하려면 `defconfig`나 `mrproper`를 실행하지 않는다. `defconfig`는 기본 설정으로 덮어쓰고, `mrproper`는 `.config`를 포함한 생성 파일을 삭제한다.

### 새 체크아웃에서 설정을 만드는 경우만

```bash
arm64-kmake defconfig
# 디버깅에 필요한 설정을 추가하는 예시
scripts/config --enable DEBUG_INFO --enable FRAME_POINTER
arm64-kmake olddefconfig
# 필요하면 대화형 설정 편집
# arm64-kmake menuconfig
```

이 절차는 새 설정의 예시이며 기존 `.config`와 완전히 같다는 뜻은 아니다. 기존 설정 파일이 있다면 그것을 `.config`로 복사하고 `olddefconfig`를 실행한다.

### 커널 Image 빌드

```bash
(
  set -o pipefail
  time arm64-kmake -j"$(nproc)" Image 2>&1 | tee build-local.log
)
```

`pipefail`은 빌드 실패가 `tee`의 성공 상태에 가려지는 것을 방지한다. 기존 `build.log`는 보존하고 새 로그에 기록한다. 현재 기존 로그 소유자가 `nobody`이므로 같은 파일에 덮어쓰면 권한 문제가 생길 수도 있다. 일반 빌드에는 `sudo`가 필요 없다.

메모리가 부족하면 `-j2` 등으로 병렬 작업 수를 줄인다. 모듈과 보드별 DTB도 필요할 때만 다음을 추가 실행한다.

```bash
arm64-kmake -j"$(nproc)" modules dtbs
```

QEMU `virt`에서 아래처럼 직접 부팅할 때는 QEMU가 DTB를 제공하므로 별도의 보드 DTB를 지정하지 않는다. [QEMU virt 문서](https://www.qemu.org/docs/master/system/arm/virt.html)

## 5. 빌드 산출물과 성공 확인

| 경로 | 용도 | 기존 파일 크기(참고) |
| --- | --- | --- |
| `arch/arm64/boot/Image` | QEMU `-kernel`로 전달하는 커널 이미지 | 약 13 MiB |
| `vmlinux` | 심볼·디버그 정보가 포함된 ELF, GDB에서 사용 | 약 148 MiB |
| `System.map` | 커널 심볼 주소 목록 | 약 2.5 MiB |
| `include/generated/compile.h` | 컴파일러·빌드 시각 정보 | — |
| `build-local.log` | 위 명령으로 저장한 새 빌드 로그 | — |
| 각 디렉터리의 `.*.cmd` | Kbuild가 기록한 실제 명령과 의존성 | — |

```bash
ls -lh arch/arm64/boot/Image vmlinux System.map
file arch/arm64/boot/Image vmlinux
tail -20 build-local.log
cat include/generated/compile.h
```

처음 또는 변경 후 빌드에서는 로그 끝부분에 `LD vmlinux`, `SYSMAP System.map`, `OBJCOPY arch/arm64/boot/Image`가 나타난다. 변경 사항이 없는 증분 빌드에서는 이 단계가 생략될 수 있다. 성공 여부는 명령 종료 상태와 산출물을 함께 확인한다.

`Image` 타깃만으로 `.ko` 모듈, 완성된 루트 파일시스템 또는 사용 가능한 initramfs가 만들어지는 것은 아니다. `usr/initramfs_data.cpio.gz`가 있어도 현재 `CONFIG_INITRAMFS_SOURCE=""`이므로 셸이 포함되었다는 뜻은 아니다.

## 6. QEMU로 커널 실행

### 커널 부팅 로그만 확인

```bash
qemu-system-aarch64 \
  -M virt,gic-version=2 -accel tcg -cpu cortex-a57 \
  -smp 1 -m 2048 -nographic -nic none \
  -kernel arch/arm64/boot/Image \
  -append "console=ttyAMA0"
```

`virt`의 PL011 시리얼 콘솔은 `ttyAMA0`이다. ARM64 CPU를 명시하고, x86_64 호스트에서도 실행할 수 있도록 TCG 에뮬레이션을 사용한다. [QEMU virt 문서](https://www.qemu.org/docs/master/system/arm/virt.html)

이 명령에는 사용자 공간이 없으므로 커널 초기화 이후 `VFS: Unable to mount root fs` 등의 패닉이 예상된다. 셸까지 부팅하려면 아래 initramfs가 필요하다. QEMU 종료는 **Ctrl+a를 누른 뒤 x**이다.

### 선택 사항: BusyBox initramfs로 셸 부팅

다음은 과거 히스토리에서 복원한 절차가 아니라, 사용자 공간을 추가하기 위한 예시다. 호스트의 x86_64 `/bin/sh`를 복사해서는 안 되며 ARM64 실행 파일이 필요하다. 아래는 BusyBox 1.36.1 소스를 정적으로 크로스 컴파일한다. 인터넷 연결이 필요하고, 작업 디렉터리는 임시 디렉터리로 만든다.

```bash
KERNEL_DIR="$PWD"
LAB_DIR="$(mktemp -d /tmp/linux467-userspace.XXXXXX)"
cd "$LAB_DIR"
curl -fL --retry 3 -o busybox.tar.bz2 \
  https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar -xf busybox.tar.bz2
cd busybox-1.36.1
make defconfig
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
# 최신 Linux 사용자 공간 헤더와 구형 BusyBox tc의 호환성 문제 회피
sed -i 's/^CONFIG_TC=y/# CONFIG_TC is not set/' .config
make CROSS_COMPILE=aarch64-linux-gnu- -j"$(nproc)"
make CROSS_COMPILE=aarch64-linux-gnu- CONFIG_PREFIX="$LAB_DIR/rootfs" install
file "$LAB_DIR/rootfs/bin/busybox"
```

`file` 출력이 **ARM aarch64**, **statically linked**인지 확인한다. 이어서 같은 셸에서 실행한다.

```bash
mkdir -p "$LAB_DIR/rootfs"/{dev,proc,sys,tmp}
cat > "$LAB_DIR/rootfs/init" <<'EOF'
#!/bin/sh
mount -t devtmpfs devtmpfs /dev
mount -t proc proc /proc
mount -t sysfs sysfs /sys
echo "Linux 4.6.7 ARM64 initramfs ready"
# 셸이 종료되어도 PID 1은 종료되지 않도록 유지한다.
while true; do
  setsid cttyhack sh
done
EOF
chmod +x "$LAB_DIR/rootfs/init"
(
  set -o pipefail
  cd "$LAB_DIR/rootfs"
  find . -print0 | cpio --null -ov --format=newc --owner=0:0 | gzip -9 \
    > "$LAB_DIR/initramfs.cpio.gz"
)
cd "$KERNEL_DIR"
qemu-system-aarch64 \
  -M virt,gic-version=2 -accel tcg -cpu cortex-a57 \
  -smp 1 -m 2048 -nographic -nic none \
  -kernel arch/arm64/boot/Image \
  -initrd "$LAB_DIR/initramfs.cpio.gz" \
  -append "console=ttyAMA0 rdinit=/init"
```

현재 커널 설정에는 `CONFIG_BLK_DEV_INITRD=y`, `CONFIG_RD_GZIP=y`, `CONFIG_DEVTMPFS=y`, `CONFIG_BINFMT_ELF=y`, `CONFIG_SERIAL_AMBA_PL011_CONSOLE=y`가 있다. 새 설정에서도 이 옵션들이 필요하다.

게스트 셸에서 `uname -a`, `cat /proc/cpuinfo`, `mount`로 확인한다. 이 환경에는 영구 디스크와 네트워크가 없고, 변경 사항은 QEMU 종료 시 사라진다. 임시 initramfs를 계속 쓸 경우 원하는 위치에 보관한다.

## 7. GDB로 부팅 과정 디버깅

위 QEMU 명령의 마지막에 다음 옵션을 추가한다.

```text
-gdb tcp:127.0.0.1:1234 -S
```

`-S`는 CPU를 시작 전에 정지시키므로 콘솔 출력이 없어도 정상이다. 과거 히스토리의 `-s -S`도 같은 용도이며, 여기서는 GDB 접속 주소를 로컬 호스트로 명시했다. [QEMU GDB 문서](https://www.qemu.org/docs/master/system/gdb.html)

다른 터미널에서 저장소 루트로 이동한다.

```bash
cd ~/kernel-lab/linux-4.6.7
gdb-multiarch vmlinux
```

GDB에서 실행한다.

```gdb
target remote 127.0.0.1:1234
hbreak start_kernel
continue
```

커널 진입 전에는 소프트웨어 브레이크포인트를 쓸 주소가 아직 준비되지 않았을 수 있어 하드웨어 브레이크포인트를 사용한다. 현재 설정은 `CONFIG_RANDOMIZE_BASE`가 꺼져 있다. 반드시 QEMU에 전달한 `Image`와 같은 빌드의 `vmlinux`를 사용한다.

## 8. 문제 확인과 검증 범위

| 증상 | 확인할 내용 |
| --- | --- |
| `arm64-kmake: command not found` | Bash에서 `source ~/.bashrc` 실행 여부 |
| `aarch64-linux-gnu-gcc: command not found` | 크로스 컴파일러 설치 여부 |
| `tee: ... Permission denied` | 새 로그 이름 사용 및 디렉터리 쓰기 권한 |
| 호스트 DTC의 `yylloc` 중복 정의 오류 | 함수의 `HOSTCFLAGS`에 `-fcommon`이 있는지 |
| QEMU 화면이 멈춘 것처럼 보임 | `-S` 사용 여부, GDB에서 `continue` 실행 여부, `console=ttyAMA0` |
| 루트 파일시스템 마운트 실패 | initramfs 또는 디스크 사용자 공간을 제공했는지 |
| `/init` 실행 실패 | 실행 권한, ARM64 BusyBox, 정적 링크, `/bin/sh` 링크 확인 |
| `Attempted to kill init` | PID 1이 종료되지 않도록 `/init` 구성 확인 |

기존 로그에는 속성 충돌 및 RWX 세그먼트 경고가 있으나 최종 `Image` 생성까지 완료되어 있다. 새로운 빌드에서는 경고와 별도로 실제 오류 및 종료 상태를 확인한다.

이 문서는 로컬 히스토리·설정·산출물과 QEMU 공식 문서를 확인하여 작성했다. 작성 과정에서 커널 재빌드, BusyBox 다운로드·빌드, QEMU 실제 부팅은 수행하지 않았다. Xen 관련 설정이 켜져 있어도 위 명령은 Linux를 직접 부팅하며, Xen 하이퍼바이저 및 Dom0 부팅 절차는 포함하지 않는다.
