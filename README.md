# Proton v6.1.0 + close_range (KernelSU)

Galaxy S21 (SM-G991N, Exynos 2100)에서 쓰는 **Proton+ v6.1.0 KSU 커널(5.4)** 에 `close_range()` 시스템 콜을 백포트해서 GitHub Actions로 빌드하는 저장소입니다.
빌드 설정은 [`.github/workflows/build.yml`](.github/workflows/build.yml) 하나뿐입니다.

## 왜 만들었나

| 항목 | 내용 |
|---|---|
| 기기 | Galaxy S21 SM-G991N (o1s, Exynos 2100), 펌웨어 G991NKSSCHZA5 |
| 롬 | [CarrotCast](https://github.com/go1sk1/CarrotCast) 1.4.5 (LineageOS 23.2 기반 GSI) |
| 커널 | Proton+ v6.1.0 KSU (리눅스 5.4.302) |
| 통신사 | SKT, IPv6 전용 망 (NAT64 `64:ff9b::/96`, APN을 IPv4로 바꾸면 거부됨) |

**증상**
1. 모바일 데이터에 IPv6만 붙고, `clatd`가 0.5초마다 재시작되며 실패합니다(`status=32512`, 즉 exit 127). 그래서 폰에 IPv4가 없습니다.
2. 이 때문에 테더링이 `No available ipv4 address`로 실패하고, **핫스팟이 켜자마자 꺼집니다.**
3. `skip_464xlat=1`로 핫스팟을 억지로 켜도 **DNS가 응답하지 않아** 연결한 기기에 "인터넷 없음"이 뜹니다.

**원인 (루트 + ftrace로 확인)**

```
close_range(3, ~0U, CLOSE_RANGE_CLOEXEC) = -38 (ENOSYS)
exit_group(127)
```

- 안드로이드 C 라이브러리(bionic)의 `posix_spawn(POSIX_SPAWN_CLOEXEC_DEFAULT)`는 자식 프로세스를 띄울 때 `close_range`를 호출합니다.
- 그런데 **커널 5.4에는 이 시스템 콜이 없습니다.** `close_range`는 5.9, `CLOSE_RANGE_CLOEXEC`는 5.11에 추가됐습니다.
- 이 롬의 bionic은 실패하면 대체 경로 없이 바로 `_exit(127)` 합니다. 그래서 `clatd`(system_server가 띄움)와 `dnsmasq`(netd가 띄움)가 둘 다 시작하자마자 죽습니다.
- 원인이 아닌 것으로 확인한 항목: SELinux, clatd 바이너리, 마운트 옵션, 삼성 커널 보안 기능.

## 무엇을 바꿨나

### 1. 기준 커밋: `48bd510c`

Proton 저장소의 `v6.1.0` 태그는 2025년 12월 커밋을 가리킵니다. 그런데 **공식 v6.1.0 배포판은 그보다 나중 커밋 `48bd510c`** ("init: Implement a way to toggle the uname spoofing", 2026-01-17)로 빌드됐습니다.
공식판과 같은 바탕에 `close_range`만 더하려고 이 커밋을 기준으로 잡았습니다.

### 2. `close_range` 백포트 (커밋 10개)

Proton의 `proton-ksu-2026` 브랜치에 이미 들어 있는 업스트림 커밋을 순서대로 가져왔습니다(`git cherry-pick -x`).

| 순서 | 커밋 | 내용 | 방식 |
|---|---|---|---|
| 1 | `ddfdede` | UPSTREAM: open: add close_range() | 그대로 |
| 2 | `82b901a` | UPSTREAM: close_range: add CLOSE_RANGE_UNSHARE | 그대로 |
| 3 | `5714efb` | BACKPORT: arch: wire-up close_range() | **직접 수정** (아래) |
| 4 | `f5c8660` | UPSTREAM: fs, close_range: add flag CLOSE_RANGE_CLOEXEC | 그대로 |
| 5 | `cebb8f6` | UPSTREAM: close_range: unshare all fds for CLOSE_RANGE_UNSHARE \| CLOSE_RANGE_CLOEXEC | 그대로 |
| 6 | `65453d0` | UPSTREAM: file: fix close_range() for unshare+cloexec | 그대로 |
| 7 | `6365313` | UPSTREAM: file: simplify logic in __close_range() | 그대로 |
| 8 | `0466cde` | UPSTREAM: fs: fd tables have to be multiples of BITS_PER_LONG | 그대로 |
| 9 | `e628a10` | UPSTREAM: fs: fix fd table size alignment properly | 그대로 |
| 10 | `14d8a61` | UPSTREAM: close_range(): fix the logics in descriptor table trimming | 그대로 |

**3번을 직접 수정한 이유**: 원래 커밋은 `process_madvise` 시스템 콜이 있다는 전제로 만들어져서, 그게 없는 이 기준 커밋에서는 충돌합니다. 그래서 arm64에 필요한 부분만 손으로 넣었습니다.

| 파일 | 바꾼 내용 |
|---|---|
| `include/uapi/asm-generic/unistd.h` | `__NR_close_range 436` 추가, `__NR_syscalls` 436 → 437 |
| `arch/arm64/include/asm/unistd32.h` | 32비트 앱용 `__NR_close_range 436` 추가 |
| `arch/arm64/include/asm/unistd.h` | `__NR_compat_syscalls` 436 → 437 |

빌드 중에 `grep`으로 위 값들이 실제로 들어갔는지 확인하고, 하나라도 없으면 빌드를 멈춥니다.

### 3. 컴파일러 받는 주소 수정

Proton 빌드 스크립트(`build/ckbuild.sh`)가 받는 clang-r563880이 원래 브랜치(`mirror-goog-main-llvm-toolchain-source`)에서 빠졌습니다.
그래서 같은 컴파일러가 있는 `refs/heads/android16-qpr2-release`로 주소를 바꿉니다.

### 4. 빌드 방식

공식 KSU 배포판과 같은 명령(`bash build/ckbuild.sh -ckR`)으로 빌드하고, 나온 zip을 Actions 결과물로 올립니다.

### 빌드 기록

| 회차 | 날짜 | 바꾼 것 | 결과 |
|---|---|---|---|
| 1 | 2026-10-23 | 처음 만듦 | × 실패 (컴파일러 주소 없음) |
| 2 | 2026-10-23 | 컴파일러 주소 수정 | ○ 성공 |
| 3 | 2026-10-23 | 기준을 `v6.1.0` 태그에서 `48bd510c`로 | ○ 성공 (지금 쓰는 판) |

## 결과 (실기기에서 확인)

- `clatd` 정상 실행(`Starting clat version 1.8 ... v4=192.0.0.4`), 폰 IPv4 정상(`ping 8.8.8.8` 성공)
- `clatd`가 켜진 상태에서도 핫스팟 유지 (`skip_464xlat`은 원래 값 -1 그대로)
- 연결한 PC에서 숫자 IPv4 주소(1.1.1.1)와 IPv4 전용 사이트까지 접속 (NAT44 → clat)
- `ndc tether start`로 직접 띄운 `dnsmasq`도 정상 실행 (`close_range = 0`)

## 남은 문제: 핫스팟 DNS → KernelSU 모듈로 우회 중

커널을 고친 뒤에도 **핫스팟을 켤 때 `dnsmasq`가 뜨지 않습니다.**
`ndc tether status`는 "started"인데 `dnsmasq` 프로세스가 없고, ftrace에도 띄우려는 시도 자체가 없습니다.

이건 커널 문제가 아니라 **롬이 dnsmasq를 요청하지 않는 문제**라서 커널로는 고칠 수 없습니다.
원인은 롬 소스에서 찾았습니다. GSI 패치 묶음의 `trebledroid-staging/platform_packages_modules_Connectivity/0006`이 테더링에 "dnsmasq DNS 프록시 쓰지 않음"(`usingLegacyDnsProxy=false`)을 넣고 있습니다. 자세한 내용은 [`rom-fix/README.md`](rom-fix/README.md)에 있습니다.

지금은 테더링 인터페이스가 올라오면 `/system/bin/dnsmasq`를 상위 DNS와 함께 직접 띄우는 KernelSU 모듈로 우회하고 있습니다. 이 모듈로 핫스팟 DNS가 정상 동작합니다.

## 롬에서 근본 해결하는 방법

롬에 아래 두 가지가 들어가면 **커널 수정과 DNS 모듈 없이도** 해결됩니다. 제안서와 바로 쓸 수 있는 패치는 [`rom-fix/`](rom-fix/)에 있습니다.

1. **bionic `posix_spawn` 대체 경로 복원**: `close_range`가 실패하면 fd를 하나씩 CLOEXEC 처리합니다. LineageOS 23.0까지 있던 코드가 23.2에서 빠졌습니다. → **CarrotCast 1.6.7에 반영됨** (릴리스 노트 기준)
2. **GSI 패치 0006 되돌리기**: 테더링이 다시 dnsmasq DNS 프록시를 요청하게 합니다.

## 쓰는 방법

**내려받기**
1. 이 저장소의 **Actions** 탭 → 가장 최근 성공한(○) 빌드를 엽니다.
2. 아래쪽 **Artifacts**에서 `ProtonPlus-v6.1.0-closerange-KSU`를 내려받습니다. 보관 기간은 빌드 후 90일입니다(지금 판은 2027-01-21까지).
3. 내려받은 파일을 풀면 나오는 zip을 공식 Proton 배포판과 같은 방법으로 설치합니다.

**다시 빌드하기**
- **Actions** 탭 → 왼쪽 `Proton v6.1.0 (48bd510c) + close_range (KSU)` → **Run workflow**를 누르면 됩니다. 30분~1시간 반쯤 걸립니다.
- 이 저장소에 push해도 자동으로 빌드됩니다. 문서만 고칠 때는 커밋 메시지에 `[skip ci]`를 넣으면 빌드가 돌지 않습니다.

## 참고

- 실제로 써서 확인한 기기는 SM-G991N 하나입니다.
- Proton의 `proton-ksu-2026` 브랜치에는 같은 백포트가 이미 들어 있습니다. 그 브랜치로 공식 배포판이 나오면 이 저장소는 필요 없습니다.
