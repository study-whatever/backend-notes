# Pull doesn't scale - or does it?

> 원문: [Prometheus 블로그 (2016-07-23)](https://prometheus.io/blog/2016/07/23/pull-does-not-scale-or-does-it/)
> 참고: [Prometheus FAQ](https://prometheus.io/docs/introduction/faq/#why-do-you-pull-rather-than-push) · [When to use the Pushgateway](https://prometheus.io/docs/practices/pushing/) · [PromCon 2016 발표 영상](https://www.youtube.com/watch?v=likpVWB5Lvo)

**한 줄 요약:** "풀 방식은 확장이 안 된다"는 오해다. 병목은 풀이 아니라 데이터 저장이고, 풀은 오히려 설정·운영 면에서 유리하다.

---

## 1. 왜 풀(Pull) 방식인가 — Prometheus FAQ

- 필요하면 모니터링 인스턴스를 추가로 띄울 수 있다 (예: 개발 중 노트북에서)
- 수집 대상(target)이 다운됐는지 더 쉽고 확실하게 안다
- 웹 브라우저로 대상에 직접 들어가 상태를 볼 수 있다
- 다만 풀이 "약간 나은" 정도이지, 모니터링 시스템 선택의 핵심 기준은 아니다
- 꼭 푸시해야 하면 **Pushgateway** 사용

---

## 2. "풀은 확장이 안 된다"는 오해 6가지

| 오해                                 | 실제                                                                                                                              |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **Nagios처럼 느릴 것이다**                | Nagios는 검사마다 하위 프로세스를 띄워서 중앙 호스트가 과부하된다. Prometheus는 스크립트 실행 없이 HTTP로 메트릭 현재 상태만 가져온다 (고루틴으로 병렬 처리).                            |
| **연결 비용이 크다** (푸시는 UDP로 피할 수 있다)   | 누가 연결을 시작하든 연결 비용은 페이로드·저장 작업에 비해 무시할 수준. TCP라서 전송 실패를 즉시 알 수 있는 장점도 있다.                                                        |
| **이벤트마다 풀해야 한다**                   | Prometheus는 이벤트 기반(StatsD, ELK)이 아니다. 서비스는 메모리에서 카운트만 올리고, Prometheus가 15~30초마다 현재 값만 가져간다 → 트래픽이 적다.                           |
| **모니터링이 모든 인스턴스를 알아야 해서 설정이 늘어난다** | 제대로 된 모니터링이라면 어차피 "있어야 할 인스턴스"를 알아야 한다. 푸시는 여기에 더해 각 서비스가 모니터링 주소까지 알아야 하므로 **오히려 설정이 더 많다**.                                   |
| **모니터링이 과부하로 죽는다**                 | 풀이든 푸시든 감당 이상이면 죽는다. 다만 푸시는 실험용·비정상 작업이 쓰레기 데이터를 밀어 넣을 위험이 더 크다. 풀은 중앙에서 통제·완화할 수 있다.                                           |
| **대규모에서 검증되지 않았다**                 | DigitalOcean은 수백만 대 머신을 Prometheus로 모니터링한다. Google Borgmon(Prometheus의 원형)도 수십 개 데이터센터·수백만 대를 풀 방식으로 모니터링했고, 확장 문제는 풀 때문이 아니었다. |

> **Q. Nagios가 뭐임?**
> 1999년에 나온 오픈소스 인프라 모니터링 도구(원래 이름은 NetSaint). 시계열 메트릭을 쌓는 도구가 아니라, **"살아 있나/정상인가"를 주기적으로 검사하고 경보를 보내는** 도구다.
> - 검사(check) = 플러그인 스크립트 실행. 예: `check_http`, `check_disk`, `check_ping`
> - 결과는 `OK / WARNING / CRITICAL / UNKNOWN` 같은 상태값 → 상태가 바뀌면 경보
> - 원격 서버 검사는 NRPE 같은 에이전트로 그 서버에서 스크립트를 실행시킴
> - 검사마다 프로세스를 띄우므로 대상이 많아지면 중앙 서버가 버거움 → 이 글에서 "풀은 느리다"는 인식의 출처로 언급됨
> - 비슷한 세대 도구: Icinga(Nagios 포크), Zabbix

> **Q. 데이터독(Datadog)은 어떤 방식? StatsD client로 쏘는 경우도 있고, (Kubernetes에서) agent가 알아서 가져가는 경우도 있고
> **둘 다 맞다. 데이터독은 노드마다 Agent를 두는 하이브리드 방식**이고, 구간마다 방향이 다르다.
>
> | 구간 | 방향 | 방법 |
> |---|---|---|
> | 앱 → Agent (커스텀 메트릭) | **푸시** | 앱이 DogStatsD(StatsD 프로토콜 + 태그)로 로컬 Agent에 UDP(기본 8125)나 Unix 소켓으로 전송. Agent가 짧은 간격으로 모아서 집계 |
> | Agent → 앱/인프라 (통합·익스포터) | **풀** | Agent가 체크를 주기적으로 실행해 kubelet, Redis, Prometheus/OpenMetrics 엔드포인트 등을 스크랩 |
> | Agent → 데이터독 SaaS | **푸시** | Agent가 HTTPS로 데이터독 백엔드에 전송 |
>
> - 질문의 두 경우: **StatsD client로 쏜 것 = 앱 → Agent 푸시**, **agent가 알아서 가져간 것 = Agent가 하는 풀**
> - Kubernetes에서는 Agent가 DaemonSet으로 노드마다 하나씩 뜨고, **Autodiscovery**(파드 어노테이션 등)로 수집 대상을 찾아서 가져간다. Prometheus의 서비스 디스커버리와 같은 역할
> - 이 글의 관점으로 보면: 풀/푸시를 노드 안(로컬)에서 끝내고, 중앙으로는 Agent가 한꺼번에 푸시하는 구조. StatsD의 이벤트 단위 푸시도 로컬 Agent가 받아서 집계하므로 중앙 시스템 부담이 줄어든다

### 숫자로 보기 (SoundCloud 운영 데이터)
- Prometheus 서버 1대: 시계열 수백만 개 저장, **초당 샘플 80만 개** 수집 기록
- 스크랩 간격 10초 × 호스트당 시계열 700개 → **서버 1대로 머신 1만 대 이상** 모니터링
- 병목은 항상 풀이 아니라 **메모리 수집 → 디스크/SSD 저장·만료 속도**

### 풀 방식의 운영상 이점
- 운영 모니터링 복사본을 노트북에서 그대로 띄워 실험 가능
- 다른 도구로 메트릭을 가져오거나 엔드포인트를 직접 확인 가능
- HA: 똑같이 설정한 Prometheus 서버 2대를 나란히 실행
- 모니터링 엔드포인트를 옮겨도 메트릭 출처를 다시 설정할 필요 없음
- 서비스 디스커버리 기본 지원: Consul, Marathon, Kubernetes, EC2, DNS 기반 SD, Azure, Zookeeper Serversets 등 (커스텀도 가능)
- 서비스가 백엔드를 찾는 방법과 같은 방법으로 모니터링 대상을 찾음 → 실제 트래픽을 받는 대상을 모니터링, 디스커버리 메커니즘도 하나만 관리

### 그래도 풀이 어려운 경우
- 엔드포인트가 전 세계에 흩어져 있고, 방화벽·복잡한 네트워크 때문에 직접 접근이 안 되며, 구간마다 Prometheus를 둘 수 없는 환경
- 이건 확장성 문제가 아니라 **TCP 연결을 여는 네트워크 운영 문제**
- 우회: Pushgateway 사용 또는 구성 변경

**결론:** "pull doesn't scale"은 실제 걱정거리가 아니다. 논쟁은 더 중요한 부분에 집중하자.

---

## 3. 사례: DigitalOcean — 100만 대 머신 모니터링 (PromCon 2016)

> 발표자: Matthew Campbell (DigitalOcean Time Series 팀)

메트릭 **수집 방식(풀 vs 푸시)** 관점에서만 정리

| 단계 | 수집 방식 | 내용 |
|---|---|---|
| 1단계: 중앙화 | 풀 + 서비스 디스커버리 | 중앙 Prometheus + **Consul 연동**. 서비스가 뜨면 Consul에 자동 등록되고, Prometheus가 그 목록을 보고 스크랩 |
| 2단계: 하이퍼바이저 | 풀 | 11개 DC, 물리 서버 1만~2만 대를 `node_exporter`로 스크랩. 1만 대 이상인 DC(뉴욕)는 서버 1대로 한계 → `node_exporter` 메트릭 필터링, 보존 기간 단축(3일), 더 큰 서버, 튜닝 |
| 3단계: 샤딩 | 풀 (수평 확장) | **레이블 기반 샤딩**으로 스크랩 대상을 여러 Prometheus에 나눔. 한계: 샤드 추가 시 과거 데이터 유실, 짧은 보존, 노드 장애 시 데이터 유실 |
| 4~5단계: Vulcan | **푸시 혼합** | 수백만 고객 VM은 DO가 직접 스크랩할 수 없음(고객 네트워크·멀티 테넌시) → 고객 VM의 **Reverse Node Exporter**가 API 토큰 인증 후 메트릭을 **푸시**. DC 수집기와 Pushgateway도 **Kafka로 푸시** → Cassandra에 저장 |

### 시사점
- 데이터센터 내부 인프라는 끝까지 **풀**(+ 서비스 디스커버리, 샤딩)로 버텼다
- 푸시를 섞은 이유는 확장성보다 **"대상에 직접 접근할 수 없는" 네트워크·보안 조건**(고객 VM) 때문 → 2장 "그래도 풀이 어려운 경우"와 같은 이야기
- **수천 대:** 레이블 기반 샤딩이나 페더레이션이면 충분 / **수백만 대:** 푸시 + Kafka 같은 중앙 파이프라인이 필요

---

## 4. Pushgateway는 언제 쓰나 — Prometheus 문서

Pushgateway: 스크랩할 수 없는 작업이 메트릭을 푸시할 수 있게 해 주는 중개 서비스 ([Pushing metrics](https://prometheus.io/docs/instrumenting/pushing/))

**권장: 일부 제한적인 경우에만 사용**

### 무분별하게 쓰면 생기는 문제
- 하나의 Pushgateway로 여러 인스턴스를 모니터링하면 **단일 장애 지점 + 병목**
- `up` 메트릭(스크랩마다 생성)으로 하는 **인스턴스 상태 자동 감시를 잃음**
- 푸시된 시리즈를 **영원히 기억** → API로 직접 지우지 않으면 계속 노출
  - `instance` 레이블로 구분하면, 인스턴스가 이름이 바뀌거나 사라져도 지표가 남음
  - 원인: Pushgateway(지표 캐시)의 수명 주기 ≠ 푸시하는 프로세스의 수명 주기
  - 풀 방식은 인스턴스가 사라지면 지표도 자동으로 사라지지만, Pushgateway는 직접 삭제하거나 동기화를 자동화해야 함

### 유일하게 적절한 경우: 서비스 수준 배치 작업의 결과 기록
- 서비스 수준 배치 작업 = 특정 머신·인스턴스와 무관한 작업 (예: 서비스 전체에서 사용자 일괄 삭제)
- 지표에 머신·인스턴스 레이블을 넣지 않음 → 수명 주기 분리, 오래된 지표 관리 부담 감소
- 참고: [배치 작업 모니터링 모범 사례](https://prometheus.io/docs/practices/instrumentation/#batch-jobs)

### 대안 전략
| 상황 | 대안 |
|---|---|
| 방화벽·NAT 때문에 풀이 안 됨 | Prometheus 서버를 그 네트워크 안쪽으로 이동 (대상과 같은 네트워크 권장). 어려우면 [PushProx](https://github.com/RobustPerception/PushProx) |
| 특정 머신 관련 배치 작업 (보안 업데이트 크론, 구성 관리 실행 등) | Pushgateway 대신 [Node Exporter](https://github.com/prometheus/node_exporter)의 [textfile collector](https://github.com/prometheus/node_exporter#textfile-collector) |
