# 리눅스의 이름 해석(Name Resolution) — "자기 자신을 위한 DNS"의 구조와 역사

> 정리 주제: 왜 일반 리눅스의 DNS 기능은 로컬호스트 안에서만 동작하는가, 어떤 순간에 공용 DNS 서버가 되는가,
> 그리고 이 구조가 왜 40년 전 `HOSTS.TXT` 시대의 설계를 그대로 물려받았는가.

---

## 0. 세 줄 요약

1. 기본 상태의 리눅스는 **DNS 서버가 아니라 "스텁 리졸버(stub resolver) + 캐시"** 다. `systemd-resolved`는 루프백 주소 `127.0.0.53:53`에 바인딩되므로, 외부 장비가 이 머신의 IP로 질의해도 **애초에 그 포트가 외부 NIC에 열려 있지 않다.**
2. 리눅스가 남의 질의에 답하는 서버가 되는 순간은 **BIND9 / dnsmasq / Unbound / CoreDNS 같은 DNS 서버 소프트웨어를 올리고 바인딩을 `0.0.0.0`(또는 `any`)로 풀고 53/udp·tcp를 개방할 때**다.
3. "내 컴퓨터의 장부를 내 컴퓨터가 먼저 본다"는 방식은 **DNS 이전 시대(`HOSTS.TXT`)의 원형**이며, 오늘날 `/etc/hosts` 우선 조회와 컨테이너의 `127.0.0.11` 내장 DNS로 같은 패턴이 반복되고 있다.

---

## 1. 용어 정리 — 무엇이 "DNS"인가

혼동의 대부분은 세 가지 역할을 한 단어로 부르는 데서 온다.

| 역할 | 하는 일 | 대표 구현 |
| --- | --- | --- |
| **스텁 리졸버 (stub resolver)** | 애플리케이션의 요청을 받아 상위 리졸버에 물어보고 결과를 돌려줌. 스스로 루트부터 추적하지 않음 | glibc `libresolv`, `systemd-resolved`, 컨테이너의 `127.0.0.11` |
| **재귀 리졸버 (recursive resolver)** | 루트 → TLD → 권한 서버를 직접 추적해 최종 답을 만들고 캐싱 | Unbound, BIND9(recursion yes), `8.8.8.8`, `1.1.1.1` |
| **권한 서버 (authoritative server)** | 특정 존(zone)의 원본 데이터를 보유하고 응답 | BIND9(zone 설정), NSD, Route 53 |

> 일반 리눅스 데스크톱/서버의 기본 구성은 **첫 번째 칸(스텁 리졸버 + 캐시)** 까지다.
> "DNS가 로컬호스트 안에서만 쓰인다"는 관찰은 **정확히 이 지점을 가리킨다.**

---

## 2. 기본 상태 — 리눅스는 오직 자기 자신을 위해 이름을 조회한다

### 2.1 `systemd-resolved`의 스텁 리스너

`systemd-resolved`가 동작하는 환경(Ubuntu 등)에서는 두 개의 루프백 스텁 리스너가 존재한다.

| 주소 | 성격 |
| --- | --- |
| `127.0.0.53:53` | **풀 기능 스텁**. 로컬 캐시, DNSSEC 검증, LLMNR/mDNS 해석까지 제공 |
| `127.0.0.54:53` | **프록시 모드 전용 스텁**. 질의를 거의 그대로 상위 DNS로 흘려보내며, 로컬 처리·DNSSEC 검증·LLMNR/mDNS를 하지 않음 |

두 주소 모두 **루프백**이다. 따라서

- 같은 머신 안의 프로세스(브라우저, 애플리케이션, 도커 데몬 등)만 질의할 수 있고,
- 다른 노트북·스마트폰이 이 머신의 LAN IP(`192.168.x.x`)로 53번을 두드려도 **연결 자체가 성립하지 않는다.**
  이는 방화벽으로 막아서가 아니라, **소켓이 그 주소에 바인딩되어 있지 않기 때문**이다.

```bash
# 무엇이 53번을 듣고 있는지 확인 (루프백만 보이는 것이 정상)
ss -ulnp 'sport = :53'
# 예: UNCONN 0 0 127.0.0.53%lo:53 0.0.0.0:*  users:(("systemd-resolve",...))

resolvectl status              # 링크별 상위 DNS 서버, DNSSEC, 도메인 확인
resolvectl statistics          # 캐시 히트율
dig @127.0.0.53 example.com    # 로컬 스텁에 직접 질의
```

### 2.2 `/etc/resolv.conf`의 세 가지 얼굴

`/etc/resolv.conf`는 "설정 파일"이 아니라 **환경에 따라 의미가 달라지는 심볼릭 링크**인 경우가 많다.

| 실체 | 내용 | 의미 |
| --- | --- | --- |
| `→ /run/systemd/resolve/stub-resolv.conf` | `nameserver 127.0.0.53` | glibc가 로컬 스텁을 향하게 함 (Ubuntu 기본) |
| `→ /run/systemd/resolve/resolv.conf` | 실제 상위 DNS IP 목록 | 스텁을 우회하고 업스트림에 직접 질의 |
| 일반 파일 | 수동 작성한 `nameserver` | Debian 최소 설치, 컨테이너, 구형 구성 |

### 2.3 배포판별 편차 (원문 보완)

"리눅스는 기본적으로 `systemd-resolved`"라는 서술은 **Ubuntu 계열 기준**으로는 맞지만 전부는 아니다.

- **Ubuntu**: `systemd-resolved` 활성 → `127.0.0.53` 구조
- **Debian 최소 설치**: `systemd-resolved` 미사용이 흔함 → glibc가 `/etc/resolv.conf`의 업스트림에 직접 질의
- **RHEL / Rocky / Alma**: 전통적으로 NetworkManager가 `/etc/resolv.conf`를 관리, glibc 직접 질의 (최근 릴리스에서 resolved 선택 가능)
- **Alpine (musl libc)**: NSS 자체가 없다. `/etc/nsswitch.conf`를 보지 않고 `/etc/hosts` → `/etc/resolv.conf` 순으로 자체 처리

> 결론은 동일하다. **어느 쪽이든 "남의 질의에 답하는 서버"는 아니다.**

---

## 3. 조회 순서 — 실제로 어떤 경로를 타는가

### 3.1 전통적 경로 (glibc NSS)

이름 해석의 순서를 정하는 것은 DNS가 아니라 **glibc의 NSS(Name Service Switch)** 설정이다.

```
/etc/nsswitch.conf
  hosts: files dns
          │     └── /etc/resolv.conf 의 nameserver 에 질의
          └──────── /etc/hosts 를 먼저 확인
```

```
[애플리케이션: getaddrinfo("db.internal")]
        │
        ▼
① /etc/hosts        ─ 찾으면 즉시 종료 (DNS 서버에 묻지도 않음)
        │ 없음
        ▼
② /etc/resolv.conf 의 nameserver 로 질의
        │
        ├─ 127.0.0.53  → systemd-resolved → 상위 DNS(공유기/ISP/8.8.8.8)
        └─ 직접 업스트림 → 재귀 리졸버 → 루트 → TLD → 권한 서버
```

### 3.2 `systemd-resolved` 환경에서의 정확한 순서 (중요한 함정)

`resolved`를 쓰는 시스템의 권장 설정은 다음과 같다.

```
hosts: mymachines resolve [!UNAVAIL=return] files myhostname dns
```

여기서 주의할 점이 두 가지 있다.

1. **`files`보다 `resolve`가 앞에 온다.** 그런데도 `/etc/hosts`가 무시되지 않는 이유는,
   `nss-resolve`가 `nss-files`를 대체하며 **`systemd-resolved`가 `/etc/hosts`를 직접 읽어 캐싱하고 최우선순위로 취급**하기 때문이다.
   즉 "`/etc/hosts` 최우선"이라는 결론은 유지되지만, **그 일을 하는 주체가 glibc에서 `resolved`로 옮겨간 것**이다.
2. **`[!UNAVAIL=return]`** 은 "결과가 `UNAVAIL`이 **아니면** 거기서 반환(중단)"이라는 뜻이다.
   → 정상 응답이나 NXDOMAIN이면 뒤의 `files`/`dns`로 내려가지 않고,
   → `systemd-resolved`가 죽어 있을 때(`UNAVAIL`)만 `files` → `dns` 폴백이 동작한다.

### 3.3 디버깅에서 자주 헷갈리는 지점

| 명령 | 경로 | 함의 |
| --- | --- | --- |
| `getent hosts foo` | **NSS 경유** (`/etc/hosts`, resolved 포함) | 애플리케이션이 실제로 보는 결과 |
| `resolvectl query foo` | resolved 경유 | 링크별 라우팅·DNSSEC 상태까지 확인 |
| `dig foo` / `nslookup foo` | **NSS 무시**, DNS 프로토콜로 직행 | `/etc/hosts`를 보지 않으므로 결과가 다를 수 있음 |

> "`dig`는 되는데 앱은 안 된다" 또는 그 반대의 대부분은 이 차이와 `search`/`ndots` 설정에서 온다.

---

## 4. 예외 — 리눅스가 '공용 DNS 서버'로 변신하는 순간

### 4.1 조건은 두 가지뿐

1. **DNS 서버 소프트웨어를 올린다**
2. **바인딩을 루프백에서 풀고, 방화벽에서 53/udp·tcp를 연다**

| 소프트웨어 | 외부 개방 설정 예 | 성격 |
| --- | --- | --- |
| BIND9 | `listen-on { any; };` + `allow-query { ...; };` | 권한 + 재귀, 풀스택 |
| dnsmasq | `listen-address=0.0.0.0` / `interface=eth0` | 경량 포워더 + DHCP, 홈랩·소규모 LAN |
| Unbound | `interface: 0.0.0.0` + `access-control:` | 검증형 재귀 리졸버 |
| CoreDNS | `Corefile`의 `.:53 { forward . ... }` | 쿠버네티스 표준, 플러그인 체인 |
| systemd-resolved | `DNSStubListenerExtra=` | LAN 노출은 가능하지만 본격 서버용은 아님 |

> **53번은 UDP만 열어서는 부족하다.** 512바이트를 넘는 응답, TCP 폴백, DNSSEC, AXFR 때문에 **TCP도 함께** 필요하다.

### 4.2 개방하는 순간 생기는 책임 (원문에 추가할 경고)

외부에 재귀 질의를 무제한 허용한 서버는 **오픈 리졸버(open resolver)** 가 되어 즉시 공격 인프라로 편입된다.

- **DNS 증폭(amplification) DDoS**: 작은 질의로 큰 응답을 유발해 출처를 위조한 피해자에게 트래픽을 몰아준다.
- **캐시 포이즈닝**: 위조 응답이 캐시에 주입되면 그 리졸버를 쓰는 모든 클라이언트가 잘못된 IP로 유도된다.

최소한의 방어선:

- `allow-query` / `access-control`로 **사설 대역만 재귀 허용**
- **권한(authoritative)과 재귀(recursive) 서버를 분리**
- RRL(Response Rate Limiting), DNSSEC 검증, 질의 로깅·모니터링

### 4.3 기본 상태 vs 서버 구축 상태

| 구분 | 기본 설치 상태 | DNS 서버 구축 상태 |
| --- | --- | --- |
| 듣는 주소(Listen) | `127.0.0.53` / `127.0.0.54` (루프백) | `0.0.0.0` 또는 특정 NIC IP |
| 적용 범위 | **해당 머신 내부**로 국한 | 동일 LAN, 설정에 따라 인터넷 전체 |
| 주 역할 | 자기 질의의 캐싱·호스트 이름 변환 | 타 장비의 질의에 응답 |
| 방화벽 | 53번 개방 불필요 | 53/udp + 53/tcp 개방 필요 |
| 대표 위험 | 거의 없음 | 오픈 리졸버, 증폭 DDoS, 캐시 포이즈닝 |
| 대표 소프트웨어 | `systemd-resolved`, glibc | BIND9, Unbound, dnsmasq, CoreDNS |

---

## 5. 역사 — 왜 하필 이런 구조인가

### 5.1 DNS 이전: 중앙 파일 하나를 각자 내려받던 시대

- 1970~80년대 초 ARPANET에는 DNS 서버라는 개념이 없었다.
- **SRI-NIC**(스탠퍼드 연구소 네트워크 정보 센터)이 전체 호스트 이름·IP 목록인 **`HOSTS.TXT` 단일 파일**을 관리했다.
- 각 기관은 이 파일을 FTP로 내려받아 **자기 머신 안에 저장해 자기만 사용**했다. 오늘의 `/etc/hosts`가 바로 그 후손이다.
- 즉 **"이름을 해석하는 주체도 내 컴퓨터, 장부를 보관하는 주체도 내 컴퓨터"** 였다 — 지금 논의의 원형.

### 5.2 한계와 DNS의 탄생 (1983)

호스트 수가 폭증하면서 이 방식은 무너졌다.

- 호스트가 하나 추가될 때마다 **전 세계가 파일을 다시 내려받아야** 했다
- 중앙 배포 서버의 부하, 갱신 지연, 이름 충돌, 단일 실패점

1983년 **폴 모카페트리스(Paul Mockapetris)** 가 계층적·위임형 분산 데이터베이스를 제안한다(**RFC 882 / 883**).
이후 1987년 **RFC 1034 / 1035** 로 개정되어 오늘날 DNS의 기반이 되었다.

핵심 발명은 "서버를 하나 더 두자"가 아니라 **권한의 위임(delegation)** 이다.

```
.  (root)
└── com.            ← TLD 운영자에게 위임
    └── example.com. ← 도메인 소유자에게 위임
        └── www / api / db ...   ← 소유자가 자유롭게 관리
```

### 5.3 현대 리눅스에 남은 흔적

DNS가 전 세계 분산 시스템으로 진화한 뒤에도, **로컬 우선 조회는 1순위 자리에 그대로 남았다.**

- `/etc/hosts`는 `HOSTS.TXT` 시대의 화석이면서, 지금도 가장 빠르고 강한 오버라이드 수단이다
- 개발·운영에서 스테이징 도메인을 임시로 꺾거나, 장애 시 특정 IP로 고정하는 실전 도구로 계속 쓰인다

---

## 6. 같은 패턴의 재현 — 컨테이너/쿠버네티스의 로컬 DNS

"로컬 전용 리졸버 + 상위 위임" 구조는 40년 뒤 컨테이너 환경에서 **그대로 다시 등장한다.**

### 6.1 Docker 내장 DNS `127.0.0.11`

- **사용자 정의 브리지** 네트워크에 붙은 컨테이너는 `/etc/resolv.conf`에 `nameserver 127.0.0.11`을 갖는다
- 이 주소는 **해당 컨테이너의 네트워크 네임스페이스 안에서만 유효한 루프백**이며, 도커 데몬의 내장 DNS로 연결된다 (netns 내부 NAT 규칙 경유)
- 컨테이너 **이름/별칭 → IP** 를 해석해 주고, 모르는 이름은 호스트가 쓰는 상위 DNS로 **위임**한다
- 디폴트 브리지(`docker0`)에서는 이 이름 해석이 동작하지 않아 IP로 통신해야 한다

```bash
docker network create app-net
docker run -d --name db  --network app-net postgres
docker run --rm --network app-net alpine ping -c1 db   # 이름으로 해석됨

docker exec db cat /etc/resolv.conf                    # nameserver 127.0.0.11
```

> 구조적으로 `127.0.0.53`(호스트 전용 스텁)과 `127.0.0.11`(컨테이너 전용 스텁)은 **같은 아이디어의 다른 스코프**다.

### 6.2 쿠버네티스

- 파드의 `/etc/resolv.conf`는 CoreDNS의 **ClusterIP**를 가리키고, `search`에 네임스페이스 도메인이 채워진다
- 기본 `ndots:5` 때문에 `db`처럼 점이 적은 이름은 **search 도메인을 순서대로 붙여 여러 번 질의**한다 → 지연·부하의 흔한 원인
- 외부 도메인을 많이 호출하는 워크로드는 `dnsConfig`로 `ndots`를 낮추거나 FQDN(`example.com.`)을 쓰는 것이 정석

---

## 7. 트러블슈팅 커맨드 모음

```bash
# 어떤 경로로 해석되는지
getent hosts api.internal            # 앱이 실제로 보는 결과 (NSS 경유)
resolvectl query api.internal        # resolved 경유, 링크/DNSSEC 정보 포함
dig +short api.internal              # 순수 DNS (hosts 무시)

# 무엇이 어디에 바인딩되어 있나
ss -ulnp 'sport = :53'
ss -tlnp 'sport = :53'

# resolved 상태 / 캐시
resolvectl status
resolvectl statistics
resolvectl flush-caches

# resolv.conf 의 실체 확인
ls -l /etc/resolv.conf
cat /etc/nsswitch.conf | grep '^hosts'

# 컨테이너 쪽
docker exec <c> cat /etc/resolv.conf
docker exec <c> getent hosts <other-container>
```

---

## 8. 결론

- **기본 리눅스의 DNS 기능은 "내부용 스텁 리졸버 + 캐시"** 이며, 루프백 바인딩 때문에 외부에서 사용할 수 없다 — 원문의 이해가 정확하다.
- **공용 DNS 서버가 되는 것은 별도 소프트웨어 + 바인딩 개방 + 방화벽 개방이라는 명시적 선택**이며, 그 순간부터 오픈 리졸버·증폭 DDoS에 대한 책임이 따라온다.
- **"자기 자신만을 위한 로컬 해석"은 DNS의 출발점**(`HOSTS.TXT`)이었고, 지금도 `/etc/hosts` 1순위 조회와 컨테이너의 `127.0.0.11`로 반복되고 있다. 스코프만 머신 → 네임스페이스로 좁아졌을 뿐, **"로컬에서 먼저 해석하고, 모르면 위임한다"** 는 원리는 40년 동안 바뀌지 않았다.

---

## 참고 자료

- systemd-resolved.service(8) — 스텁 리스너 `127.0.0.53` / `127.0.0.54`의 차이
  <https://man.archlinux.org/man/systemd-resolved.service.8.en>
- nss-resolve(8) — 권장 `hosts:` 순서, `/etc/hosts` 내부 처리, `[!UNAVAIL=return]`
  <https://man.archlinux.org/man/nss-resolve.8.en>
- systemd 공식 매뉴얼
  <https://www.freedesktop.org/software/systemd/man/latest/systemd-resolved.service.html>
- RFC 882 / 883 (1983, Mockapetris) → RFC 1034 / 1035 (1987)
