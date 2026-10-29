**제목:** 핫스팟 DNS 무응답 ("인터넷 없음"): GSI 패치 0006이 dnsmasq를 끄는 문제

---

1.6.7에서 구형 커널의 `close_range` / `posix_spawn` 실패(CLAT)를 고쳐 주셔서 감사합니다.
같은 증상 묶음에서 **핫스팟 DNS 문제**가 하나 더 남아 있어서 원인과 수정 방향을 정리했습니다.

## 증상

- 핫스팟은 켜지고 연결도 되는데, 연결한 기기에 **"인터넷 없음"** 이 뜹니다.
- 연결한 PC에서 숫자 주소(`1.1.1.1`)로는 접속되는데, 도메인 이름은 풀리지 않습니다.
- 폰에서 확인하면 `ndc tether status`는 "started"인데 **dnsmasq 프로세스가 없습니다.** ftrace에도 dnsmasq를 띄우려는 시도가 없습니다.
- 같은 상태에서 `ndc tether start`로 직접 띄운 dnsmasq는 정상으로 돕니다. dnsmasq 자체에는 문제가 없습니다.

확인 환경: SM-G991N, CarrotCast 1.4.5, `close_range`를 백포트한 커널(clatd·IPv4 정상), SKT IPv6 전용 망

## 원인

`gsi/source-lock.xml`에 고정된 GSI 패치 묶음 `MisterZtr/LineageOS_gsi` (`7bf212f`)의
`patches/trebledroid-staging/platform_packages_modules_Connectivity/0006-Tethering-fall-back-to-modern-DNS-resolver-when-dns.patch`가
`Tethering.java`의 `turnOnMainTetherSettings()`에서 테더링 시작 값을 `usingLegacyDnsProxy = false`로 바꿉니다.

1. DHCP는 NetworkStack 자체 서버가 맡아서 `dhcpRanges`가 비어 있습니다.
2. netd `TetherController::startTethering()`은 "DNS 프록시 안 씀 + DHCP 범위 없음"이면 **dnsmasq를 띄우지 않고 '시작됨'만 표시합니다** (`// Both DHCP and DnsProxy are disabled, we don't need to start dnsmasq`).
3. 그런데 `IpServer`는 DHCP로 연결 기기에 "DNS 서버 = 폰 주소"를 알려 줍니다 (`makeServingParams(addr, addr /* dnsServer */, ...)`).
4. 폰에서 53번 포트를 받는 프로그램이 없으므로 DNS가 응답하지 않습니다.

0006 패치 설명은 "GSI에서 dnsmasq가 안 떠서 다른 DNS resolver를 쓴다"입니다. 하지만 `false`는 다른 DNS로 바꾸는 값이 아니라 **DNS를 끄는 값**입니다.
dnsmasq가 안 뜨던 이유는 1.6.7에서 고치신 `posix_spawn` 문제로 보입니다. netd도 dnsmasq를 `posix_spawn(POSIX_SPAWN_CLOEXEC_DEFAULT)`로 띄우기 때문입니다. 그래서 이제는 0006이 필요 없습니다.

## 수정 방향

**핵심: 테더링이 다시 dnsmasq DNS 프록시를 요청하게 하기** (`usingLegacyDnsProxy = true`, LineageOS·AOSP 원래 동작)

방법은 둘 중 하나입니다. 결과는 같습니다.
1. **0006을 적용하지 않기.** 지금 `gsi/apply-patches.py`에는 특정 패치를 빼는 기능이 없으니 제외 목록 같은 기능이 필요합니다.
2. **0006 적용 뒤 되돌리기.** `Tethering.java`의 `tetherStart(...)` 두 곳(첫 호출과 재시도)을 `true`로 바꾸는 롬 자체 패치를 GSI 묶음 다음에 적용합니다.
   ```java
   NetdUtils.tetherStart(mNetd, true /** usingLegacyDnsProxy */, dhcpRanges);
   ```

**함께 확인할 점**
- **1.6.7 수정이 netd 경로에도 적용되는지:** netd는 dnsmasq를 `POSIX_SPAWN_USEVFORK | POSIX_SPAWN_CLOEXEC_DEFAULT`로 띄웁니다. 1.6.7의 대체 경로가 bionic `posix_spawn` 자체에 들어갔다면 dnsmasq에도 그대로 적용됩니다. vfork 자식에서 도는 코드라 메모리 할당 없이 동작해야 합니다.
- **netd 패치는 유지:** 같은 묶음의 `trebledroid-staging/platform_system_netd/0002`(dnsmasq 통신 실패를 치명 오류로 보지 않음)는 그대로 두면 됩니다. dnsmasq가 혹시 죽어도 핫스팟이 꺼지지 않게 막아 주는 안전장치입니다.
- **GSI 패치 묶음 쪽:** 0006은 `MisterZtr/LineageOS_gsi`의 `lineage-23.2`에 아직 그대로 있습니다. 그쪽에도 알리면 다른 GSI 사용자도 같이 해결됩니다.

## 확인 방법

DNS 우회 모듈 없이 핫스팟을 켠 뒤 확인합니다.
- `ps -A | grep dnsmasq`에 프로세스가 있는지
- 연결한 PC에서 `nslookup google.com`이 되는지

1.6.7 릴리스 노트에는 DNS 관련 내용이 없어서 남아 있는 것으로 보고 올립니다. 이미 고치셨다면 알려 주세요.
