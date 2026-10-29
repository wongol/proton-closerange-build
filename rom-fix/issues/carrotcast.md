**제목:** 커널 5.4 + IPv6 전용 망(SKT)에서 IPv4·핫스팟 DNS 실패: 롬 쪽 원인 2가지와 패치

---

## 요약

커널 5.4 기기에서 IPv6 전용 망(SKT)을 쓰면 폰에 IPv4가 없고 핫스팟 DNS가 응답하지 않습니다. 롬 소스에서 원인 두 곳을 찾았고, 패치를 만들어 두었습니다.

| 증상 | 원인 | 패치 |
|---|---|---|
| clatd·dnsmasq가 exit 127로 죽음 → IPv4 없음, 핫스팟 바로 꺼짐 | bionic: LineageOS 23.2에서 `posix_spawn`의 `close_range` 대체 경로가 빠짐 | 23.0 코드 복원 |
| 핫스팟은 켜지는데 DNS 무응답 ("인터넷 없음") | GSI 패치 `trebledroid-staging/…Connectivity/0006`이 dnsmasq를 끔 | 0006 되돌리기 |

두 패치가 들어가면 **커널 수정과 DNS 우회 모듈이 없어도** 해결될 것으로 봅니다. 아래 "확인한 범위"에 적었듯이 롬 빌드와 실기기 시험은 아직 하지 못했습니다.

- 자세한 분석: https://github.com/wongol/proton-closerange-build/blob/4b5cce0126fbedfe413e02d5ecd47c2b5c446c70/rom-fix/README.md
- 패치 폴더: https://github.com/wongol/proton-closerange-build/tree/4b5cce0126fbedfe413e02d5ecd47c2b5c446c70/rom-fix

## 환경

- 기기: Galaxy S21 SM-G991N (o1s, Exynos 2100), 펌웨어 G991NKSSCHZA5
- 롬: CarrotCast 1.4.5 (`23.2-20261021-GAPPS-EXT4-GSI`)
- 커널: Proton+ v6.1.0 KSU (5.4.302)
- 통신사: SKT, IPv6 전용 (NAT64 `64:ff9b::/96`, APN을 IPv4로 바꾸면 거부)

## 원인 1: bionic `posix_spawn` (clatd·dnsmasq exit 127)

루트와 ftrace로 확인한 결과입니다.

```
close_range(3, ~0U, CLOSE_RANGE_CLOEXEC) = -38 (ENOSYS)
exit_group(127)
```

- `bionic` (`ff9fe01`) `libc/bionic/spawn.cpp` 157줄: `if (close_range(3, ~0U, CLOSE_RANGE_CLOEXEC)) _exit(127);`
- 커널 5.4에는 `close_range`가 없습니다. 5.9에 추가됐고, `CLOSE_RANGE_CLOEXEC`는 5.11에 추가됐습니다.
- **LineageOS 23.0**에는 실패하면 fd를 하나씩 CLOEXEC 처리하는 `cloexec_except_stdioe()`가 있었는데, **23.2에서 빠졌습니다.** 두 판의 `spawn.cpp` 차이는 이 부분뿐입니다.
- 원인이 아닌 것으로 확인한 항목: SELinux, clatd 바이너리, 마운트 옵션, 삼성 커널 보안 기능.
- 커널에 `close_range`를 백포트하자 clatd와 dnsmasq가 정상 실행됐습니다(위 링크의 저장소). 그래서 원인은 이것 하나로 봅니다.

**패치**: `rom-patches/platform_bionic/0001-posix_spawn-fall-back-to-per-fd-FD_CLOEXEC-when-clos.patch`
23.0 코드를 그대로 되살립니다. `close_range`가 되는 커널에서는 동작이 지금과 같습니다.

## 원인 2: GSI 패치 0006 (핫스팟 DNS 무응답)

`MisterZtr/LineageOS_gsi` (`7bf212f`)의 `patches/trebledroid-staging/platform_packages_modules_Connectivity/0006-…`이 `tetherStart(mNetd, false /** usingLegacyDnsProxy */, …)`로 바꿉니다.

1. DHCP는 NetworkStack 자체 서버가 맡아서 `dhcpRanges`가 비어 있습니다.
2. netd `TetherController::startTethering()`은 "DNS 프록시 안 씀 + DHCP 범위 없음"이면 **dnsmasq를 띄우지 않고 '시작됨'만 표시합니다.**
3. 그런데 `IpServer`는 연결 기기에 "DNS 서버 = 폰 주소"를 알려 줍니다.
4. 폰에서 53번 포트를 받는 프로그램이 없으므로 DNS가 응답하지 않습니다.

기기에서 본 것과 일치합니다.
- 커널을 고친 뒤 핫스팟을 켜면 `ndc tether status`는 "started"인데 dnsmasq 프로세스가 없고, ftrace에도 띄우려는 시도가 없었습니다.
- 같은 상태에서 `ndc tether start`로 직접 띄운 dnsmasq는 정상으로 돕니다.

0006 패치 설명은 "GSI에서 dnsmasq가 안 떠서 다른 DNS resolver를 쓴다"입니다. 하지만 dnsmasq가 안 뜨던 이유는 원인 1로 보입니다. 원인 1을 고치면 0006은 필요 없습니다.

**패치**: `rom-patches/platform_packages_modules_Connectivity/0001-Tethering-request-the-dnsmasq-DNS-proxy-again.patch`
0006 위에 얹어 `true`로 되돌립니다. 결과는 LineageOS 원래 코드와 같습니다. GSI 패치 묶음은 건드리지 않습니다.
같은 묶음의 netd `0002`(dnsmasq 통신 실패를 치명 오류로 보지 않음)는 안전장치로 그대로 두면 됩니다.

## 적용 방법

지금 `gsi/apply-patches.py`는 GSI 묶음만 적용합니다. 롬 자체 패치(`gsi/rom-patches/`)를 그 뒤에 같은 규칙으로 적용하도록 넓히는 diff도 함께 만들었습니다.
CarrotCast 저장소 맨 위 폴더에서 실행하면 됩니다.

```bash
B=https://raw.githubusercontent.com/wongol/proton-closerange-build/4b5cce0126fbedfe413e02d5ecd47c2b5c446c70/rom-fix
mkdir -p gsi/rom-patches/platform_bionic gsi/rom-patches/platform_packages_modules_Connectivity
curl -fL -o gsi/rom-patches/platform_bionic/0001-posix_spawn-fall-back-to-per-fd-FD_CLOEXEC-when-clos.patch "$B/rom-patches/platform_bionic/0001-posix_spawn-fall-back-to-per-fd-FD_CLOEXEC-when-clos.patch"
curl -fL -o gsi/rom-patches/platform_packages_modules_Connectivity/0001-Tethering-request-the-dnsmasq-DNS-proxy-again.patch "$B/rom-patches/platform_packages_modules_Connectivity/0001-Tethering-request-the-dnsmasq-DNS-proxy-again.patch"
curl -fL "$B/apply-patches.diff" | git apply
```

- 순서: GSI 묶음(`trebledroid` → `trebledroid-staging` → `personal`) 다음에 `gsi/rom-patches`를 적용합니다.
- 충돌하면 지금처럼 멈추고, 결과는 `goodman-patches.json`에 남습니다. 기존 `patch-overrides/` 동작은 바뀌지 않습니다.

## 확인한 범위

- ○ 고정된 소스(`bionic` `ff9fe01`, `Connectivity` `c9f3e77` + GSI 0006 적용 상태)에 두 패치가 `git apply`로 적용됨
- ○ 바꾼 `apply-patches.py`로 적용 → `applied`, 다시 실행 → `already-present`
- × 롬 전체 빌드와 실기기 부팅은 하지 못했습니다

빌드해 주시면 아래처럼 확인해 보겠습니다. 시험할 때는 `close_range`가 없는 **공식 Proton v6.1.0 커널**을 쓰고, DNS 우회 모듈은 끕니다.
- `logcat`에 `Starting clat version ...`이 나오고, 0.5초마다 재시작하지 않는지
- `ping 8.8.8.8`이 되는지
- 핫스팟을 켰을 때 `ps -A | grep dnsmasq`에 프로세스가 있는지
- 연결한 PC에서 `nslookup`이 되는지

## 함께 고려해 주셨으면 하는 점 (선택)

1. clatd가 계속 실패하면 0.5초마다 다시 시작해서 감사 로그가 넘칩니다. 실패가 반복되면 재시도 간격을 늘리고, `0.0.0.0/0` 주소는 테더링 대역 충돌 계산에서 빼 주시면 좋겠습니다.
2. 수정이 들어가기 전 판을 쓰는 IPv6 전용 통신사 사용자는 `close_range`가 백포트된 커널이 필요하다는 안내를 롬 설명에 넣어 주시면 좋겠습니다.

logcat과 ftrace 전체 기록이 필요하시면 드리겠습니다.
