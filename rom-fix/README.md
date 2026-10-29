# CarrotCast 롬 근본 해결 제안

대상: [go1sk1/CarrotCast](https://github.com/go1sk1/CarrotCast) (LineageOS 23.2 GSI)
확인한 소스: CarrotCast `gsi/source-lock.xml`에 고정된 커밋과 GSI 패치 묶음 `MisterZtr/LineageOS_gsi` `7bf212f` (2026-10-29 기준 `lineage-23.2` 최신)

## 요약

| 증상 | 원인 위치 | 고칠 곳 | 이 폴더의 패치 |
|---|---|---|---|
| 커널 5.4에서 `clatd`·`dnsmasq`가 exit 127 → IPv4 없음, 핫스팟 바로 꺼짐 | bionic `posix_spawn` | `close_range` 실패 시 대체 경로 복원 | `rom-patches/platform_bionic/0001-…` |
| 핫스팟 DNS 응답 없음 ("인터넷 없음") | GSI 패치 `trebledroid-staging/…Connectivity/0006` | 테더링이 다시 dnsmasq DNS 프록시를 요청 | `rom-patches/platform_packages_modules_Connectivity/0001-…` |

두 가지가 다 들어가면 **커널 수정과 DNS 우회 모듈이 둘 다 필요 없습니다.**
`close_range`가 있는 커널(이 저장소의 빌드)을 그대로 써도 문제는 없습니다.

---

## 1. bionic: `posix_spawn` 대체 경로 복원

### 원인

롬의 bionic(`LineageOS/android_bionic` `ff9fe01`) `libc/bionic/spawn.cpp` 157번째 줄:

```cpp
if ((flags & POSIX_SPAWN_CLOEXEC_DEFAULT) != 0) {
  // mark all open fds except stdin/out/err as close-on-exec
  if (close_range(3, ~0U, CLOSE_RANGE_CLOEXEC)) _exit(127);
}
```

`close_range`가 실패하면 대체 경로 없이 바로 `_exit(127)` 합니다.
커널 5.4에는 `close_range`가 없어서(5.9 추가, `CLOSE_RANGE_CLOEXEC`는 5.11) 항상 실패합니다.

### 언제 빠졌나

- **LineageOS 23.0** (`lineage-23.0`)의 같은 파일에는 대체 경로 `cloexec_except_stdioe()`가 있습니다. `close_range`를 먼저 해 보고, 실패하면 fd를 하나씩 `fcntl(F_SETFD, FD_CLOEXEC)` 처리합니다.
- **LineageOS 23.2**에서 이 대체 경로가 빠졌습니다. 두 판의 `spawn.cpp` 차이는 이 부분뿐입니다.
- AOSP 16 QPR2는 5.11 이상(또는 ACK 5.10-T) 커널만 지원한다고 보고 지운 것으로 보입니다. 하지만 GSI는 그보다 오래된 제조사 커널 위에서 돌기 때문에 문제가 됩니다.
- GSI 패치 묶음의 `trebledroid-staging/platform_bionic`에도 오래된 커널용 대체 경로 패치가 이미 있습니다(`epoll_pwait2` → `epoll_pwait`, `MADV_WIPEONFORK` 실패 허용). 이번 패치도 같은 성격입니다.

### 수정

[`rom-patches/platform_bionic/0001-posix_spawn-fall-back-to-per-fd-FD_CLOEXEC-when-clos.patch`](rom-patches/platform_bionic/0001-posix_spawn-fall-back-to-per-fd-FD_CLOEXEC-when-clos.patch)

- LineageOS 23.0 / AOSP에 있던 코드를 **한 글자도 바꾸지 않고** 되살립니다.
- `close_range`가 되는 커널에서는 동작이 지금과 같습니다. 대체 경로는 실패할 때만 탑니다.

---

## 2. 테더링 DNS: GSI 패치 0006 되돌리기

### 원인

GSI 패치 묶음의 `patches/trebledroid-staging/platform_packages_modules_Connectivity/0006-Tethering-fall-back-to-modern-DNS-resolver-when-dns.patch`가 아래처럼 바꿉니다.

```java
NetdUtils.tetherStart(mNetd, false /** usingLegacyDnsProxy */, dhcpRanges);  // 원래 true
```

이 값이 `false`일 때 실제로 일어나는 일:

1. 요즘 테더링은 DHCP를 NetworkStack의 자체 서버가 맡아서 `dhcpRanges`가 비어 있습니다.
2. netd `TetherController::startTethering()`은 "DNS 프록시 안 씀 + DHCP 범위 없음"이면 **dnsmasq를 띄우지 않고 '시작됨'만 표시하고 끝냅니다.**
   ```cpp
   if (!usingLegacyDnsProxy && num_addrs == 0) {
       // Both DHCP and DnsProxy are disabled, we don't need to start dnsmasq
       configureForTethering(true);
       mIsTetheringStarted = true;
       return 0;
   }
   ```
3. 그런데 `IpServer`는 DHCP로 연결 기기에 **"DNS 서버 = 핫스팟 주소(폰)"** 를 그대로 알려 줍니다(`makeServingParams(addr, addr /* dnsServer */, …)`).
4. 폰에서 53번 포트를 받는 프로그램이 없으니 DNS가 응답하지 않고, 연결한 기기에는 "인터넷 없음"이 뜹니다.

실기기 관찰과 정확히 맞습니다. `ndc tether status`는 "started"인데 dnsmasq 프로세스가 없고, ftrace에 띄우려는 시도도 없었습니다.

### 이 패치가 들어간 이유와 문제점

- 패치 설명은 "GSI에서 dnsmasq가 안 뜰 수 있으니, dnsmasq 대신 NetworkStackService의 DNS resolver(`tetherDnsProxy`)를 쓴다"입니다.
- 하지만 확인한 소스(`Tethering.java`, `IpServer.java`, netd `TetherController.cpp`)에는 dnsmasq 대신 핫스팟 DNS를 받아 주는 경로가 없습니다. `false`는 다른 DNS로 바꾸는 값이 아니라 **DNS를 끄는 값**입니다.
- "GSI에서 dnsmasq가 안 뜬다"는 현상 자체는 **1번(bionic) 문제**였을 가능성이 큽니다. dnsmasq도 `posix_spawn(POSIX_SPAWN_CLOEXEC_DEFAULT)`로 뜨기 때문에 커널 5.4에서는 exit 127로 죽습니다.
- 1번을 고치면 dnsmasq가 정상으로 뜨므로 0006은 필요 없습니다.

### 수정

[`rom-patches/platform_packages_modules_Connectivity/0001-Tethering-request-the-dnsmasq-DNS-proxy-again.patch`](rom-patches/platform_packages_modules_Connectivity/0001-Tethering-request-the-dnsmasq-DNS-proxy-again.patch)

- 0006이 적용된 뒤에 얹어서 `true`로 되돌립니다. 결과는 LineageOS/AOSP 원래 코드와 같습니다(시험으로 확인).
- GSI 패치 묶음 자체는 그대로 두므로, 고정 커밋과 기록(`goodman-patches.json`)이 깨지지 않습니다.
- 같은 묶음의 netd 패치 `trebledroid-staging/platform_system_netd/0002`(dnsmasq 통신 실패를 치명 오류로 보지 않음)는 그대로 둬도 됩니다. dnsmasq가 혹시 죽어도 핫스팟이 꺼지지 않게 막아 주는 안전장치입니다.

---

## CarrotCast에 적용하는 방법

지금 `gsi/apply-patches.py`는 GSI 패치 묶음만 적용합니다. 롬 자체 패치를 그 뒤에 적용하도록 조금 넓히고, 패치 두 개를 넣습니다.

1. 이 폴더의 `rom-patches/`를 CarrotCast 저장소의 `gsi/rom-patches/`로 복사합니다.
2. CarrotCast 저장소 맨 위 폴더에서 [`apply-patches.diff`](apply-patches.diff)를 적용합니다.

```bash
git apply apply-patches.diff
```

바뀌는 동작:
- GSI 묶음(`trebledroid` → `trebledroid-staging` → `personal`)을 적용한 **다음에** `gsi/rom-patches/<대상>/*.patch`를 같은 규칙으로 적용합니다.
- 폴더 이름 규칙은 GSI 묶음과 같습니다(`platform_bionic` → `bionic`, `platform_packages_modules_Connectivity` → `packages/modules/Connectivity`).
- 충돌하면 지금처럼 빌드를 멈추고, 적용 결과는 `goodman-patches.json`에 `gsi/rom-patches/...` 경로로 남습니다.
- GSI 패치의 경로·오버라이드(`patch-overrides/`) 동작은 바뀌지 않습니다.

### 여기서 시험한 것 / 못 한 것

- ○ 고정된 소스(`bionic` `ff9fe01`, `Connectivity` `c9f3e77` + GSI 0006 적용 상태)에 두 패치가 `git apply`로 적용됨
- ○ 바꾼 `apply-patches.py`로 적용 → `applied`, 다시 실행 → `already-present`
- ○ GSI 패치의 오버라이드 경로·기록 경로는 바꾸기 전과 같음
- × 롬 전체 빌드와 실기기 부팅은 못 했습니다 (빌드 환경 없음)

## 기기에서 확인하는 방법

bionic 수정의 효과를 보려면 **`close_range`가 없는 공식 Proton v6.1.0 커널**로 시험해야 합니다. 이 저장소의 커널에서는 원래부터 잘 되기 때문입니다.
DNS 우회 모듈은 끄고 시험합니다.

| 확인 | 명령·방법 | 기대 결과 |
|---|---|---|
| clatd | `logcat -d \| grep -i clat` | `Starting clat version ...`, 0.5초 재시작 반복 없음 |
| 폰 IPv4 | `ping -c 3 8.8.8.8` | 응답 있음 |
| 핫스팟 유지 | 핫스팟 켜기 | 바로 꺼지지 않음 |
| dnsmasq | 핫스팟 켠 상태에서 `ps -A \| grep dnsmasq` | 프로세스 있음 |
| 연결 기기 DNS | PC에서 `nslookup google.com` | 주소가 나옴 |

## 함께 하면 좋은 일 (선택)

1. **GSI 패치 묶음 쪽에 알리기**: 0006은 `MisterZtr/LineageOS_gsi`의 `lineage-23.2`에 아직 그대로 있습니다. 위 분석(값 `false`는 DNS를 끔, 원래 원인은 bionic)을 그쪽에 알리면 다른 GSI 사용자도 같이 해결됩니다.
2. **clatd 재시도 간격**: clatd가 계속 실패하면 0.5초마다 다시 시작해서 감사 로그가 넘치고, 다른 SELinux 로그가 묻힙니다. 실패가 반복되면 간격을 늘리고, `0.0.0.0/0` 주소는 테더링 대역 충돌 계산에서 빼면 실패해도 핫스팟이 막히지 않습니다.
3. **롬 설명에 안내 추가**: IPv6 전용 통신사(SKT 등)에서는 1번 수정이 필요하다는 내용입니다. 수정 전 판을 쓰는 사람은 `close_range`가 들어간 커널이 필요합니다.
