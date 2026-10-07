# 김도훈 · 서버 백엔드 개발자

게임 서버의 메모리·GC 최적화, 분산 상태 동기화, 실시간 통신, 그리고 측정 기반 성능 검증에 집중하는 백엔드 개발자입니다.
실제 운영 환경에서 마주친 문제를 측정·비교로 판단하고, 그 결과를 코드와 부하 테스트 결과로 남겨 왔습니다.

---

## 개인 프로젝트

### 1. HaxWar — 분산 기반 실시간 멀티플레이어 게임 서버
`C# / .NET 8` · `WebSocket` · `gRPC` · `Redis` · `Nginx` · 2026.05 – 2026.08 · 1인 프로젝트  
🔗 [서버 코드(HaxWar)](https://github.com/opt-dohun/HaxWar) · [인프라 코드(hexwar-exporter)](https://github.com/opt-dohun/hexwar-exporter)

단일 서버(2core / 4GB)에서 연결당 약 1MB를 점유하는 구조로 **2,000+ 동시 세션**을 목표로 설계한 게임 서버입니다.
분산 환경으로 수평 확장할 때의 시나리오와 메모리 최적화를 중심으로 진행했습니다.

* **Redis Sharding 및 고가용성:** 단일 Redis 인스턴스의 메모리·처리량 병목을 없애기 위해 게임 상태(맵 정보, 플레이어 위치 등)를 해시 슬롯 기반으로 샤딩하고, 3-Master / 3-Replica 구성으로 자동 페일오버를 확보했습니다.
* **분산 노드 간 이벤트 브로드캐스트 및 상태 동기화:** 서로 다른 Pod에 접속한 플레이어가 각자 다른 세션 상태를 보던 문제를 세션 ID 토픽 기반 Redis Pub/Sub(발행/구독) 브로드캐스트로 해결했습니다. 각 메시지에 서버 ID·시퀀스 번호를 담아 자신이 발급한 메시지를 무시하고 순서를 판별했고, Pub/Sub이 재전송을 보장하지 않는 구간은 인스턴스별 **고정 길이 원형 큐(CircularBuffer)** 에 최근 이벤트를 보관해 무손실 재처리 경로를 만들었습니다. 게임 상태 변경의 동시성은 `SETNX` + TTL 분산 락으로 제어했습니다.
* **힙 할당 및 GC 오버헤드 85.7% 감축:** 초당 수만 건의 패킷 처리에서 힙 파편화와 GC 프리징을 막기 위해 `ArrayPool`, `Memory<byte>`, `ReadOnlySpan<byte>` fast-path와 객체 재사용 패턴을 적용했습니다. — **Gen2 GC 7회 → 1회(85.7%↓), GC 힙 232 → 98MB, 세션당 메모리 116 → 50KB**.

### 2. HaxWar Exporter — 메트릭 수집 사이드카 · k8s 인프라
`Go 1.22` · `Prometheus` · `Kubernetes(k3d)` · `Agones` · `KEDA` · `kwok` · `Grafana` · `Vector` · `Kafka` · `Quickwit(MinIO)` · `Loki` · `OpenTelemetry`  
🔗 [hexwar-exporter](https://github.com/opt-dohun/hexwar-exporter)

* **사이드카 패턴으로 도메인 분리:** 게임 서버에 관측성 라이브러리를 넣으면 비즈니스 로직과 인프라 로직의 결합도가 올라가므로, Go 기반 독립 사이드카로 메트릭 수집을 분리했습니다. **스크랩 요청당 0.017ms 처리, 56KB 메모리 할당**으로 오버헤드를 최소화했고, 서킷 브레이커로 게임 서버 장애가 Prometheus 등 상위 시스템으로 전파되는 것을 차단했습니다.
* **이벤트 기반 자동 오토스케일링:** ① LocalStack으로 AWS EC2 Auto Scaling Group API를 로컬에서 모킹하고 Karpenter·KEDA를 연동해 Prometheus 메트릭 기반 이벤트 오토스케일링을 구성했으며, ② Agones로 스케일 다운 시 Pod 내부 활성 세션을 counter로 추적해 진행 중인 게임이 강제 종료되지 않도록 Graceful Shutdown 로직을 구현했습니다. 서버 성격에 맞춰 스케일링 전략을 고를 수 있습니다.
* **통합 로그 수집 파이프라인:** Vector DaemonSet으로 전 Pod의 stdout을 수집해 Kafka 버퍼를 거쳐 Quickwit이 배치 인덱싱한 뒤 객체 저장소에 적재하도록 구성했습니다. — **약 1,020만 건 기준 2.8GB → 577MB**. Grafana에서는 Prometheus 메트릭과 통합 모니터링을 제공합니다.

### 3. 서버 판정 턴제 게임 서버
`C# / .NET 10` · `HTTP` · `SQLite(개발) / MySQL(운영)` · `Unity 6` · `k6` · 2026.09 – 2026.10 · 1인 프로젝트  
🔗 [Unity-Game-Server](https://github.com/opt-dohun/Unity-Game-Server)

턴제는 실시간 연결을 유지할 필요가 없어 같은 자원으로 더 많은 동시 사용자를 받을 수 있다는 가설을 **k6 부하 테스트로 검증**하고,
서버가 PCG32 난수로 내리는 판정이 C 프로토타입·클라이언트와 같은 결과를 내는지 리플레이로 확인한 프로젝트입니다.

* **락 전략을 측정으로 판정:** 전역 락과 version CAS 낙관적 락을 부하 프로필 3종으로 비교했습니다. 원인은 SQLite journal(DELETE) 모드가 쓰기마다 파일을 잠그고 fsync하는 것이었고, **WAL 전환으로 임계 구역 p95 13.67 → 0.23ms(약 59배), 처리량 10.6 → 510.9 전투/초(약 48배), 전체 p95 187.8 → 11.0ms**로 개선했습니다.
* 낙관적 락은 락 대기를 0으로 만들었지만 SQLite는 writer를 하나만 허용해 경합이 DB 쓰기 락으로 옮겨 가며 **p99 170ms · 최대 7.3초**로 악화됐습니다. 결과적으로 **단일 SQLite 파일은 전역 락 + WAL, 행 단위 동시 쓰기가 되는 운영 MySQL은 낙관적 락**으로 정리했습니다.
* **결정론적 전투 판정:** PCG32 난수로 턴 피해를 결정하고 C ↔ C# 구현이 같은 시드에서 같은 값을 내는지 단위 검사로 검증했습니다. 같은 `battleId·turn·action` 재요청은 저장된 응답을 그대로 돌려주고, 이미 처리한 턴에 다른 행동을 보내거나 순서를 건너뛰면 409로 거부합니다.
* **이중 저장소 구성:** `BattleStore:Provider`로 SQLite(WAL)와 MySQL을 전환하고 `DbConnection` 추상화로 같은 코드가 두 저장소에서 동작하게 했으며, version 컬럼 자동 마이그레이션을 SQLite(PRAGMA table_info)·MySQL(INFORMATION_SCHEMA) 양쪽에 구현했습니다.

---

## 💼 경력

| 기간 | 회사 · 팀 | 주요 업무 |
| --- | --- | --- |
| 2025.12 – 2026.03 | (주)보이저 게임즈 · IP 게임 서비스팀 / 사원 | 러브라이브 IP 라이브 서비스 Go 백엔드. GCP(Cloud Run·Spanner·Pub/Sub) 기반 이벤트·방송·운영 기능 개발 |
| 2025.06 – 2025.10 | (주)모토벨로 서비스 · 공유지원팀 / 주임연구원 | 공유 모빌리티 결제 도메인 고도화. 다중 PG사 연동을 전략·팩토리·어댑터 패턴으로 추상화, 결제 오류 정규화 |
| 2022.12 – 2025.01 | (주)비엔씨테크 · 기업부설연구소 / 연구원 | 공유 자전거 IoT 통신 서버. opcode 기반 바이너리 프로토콜, 펌웨어 전송 아키텍처, 서버 운영 자동화 |

**실무에서 해결한 문제**

* **라이브 랭킹 이벤트 부하 분산:** RDB 기반 랭킹은 이벤트마다 인덱스 갱신이 발생해 CPU가 90%까지 치솟았습니다. Redis Sorted Set(Skiplist)으로 정렬·탐색 비용을 줄이고 Pub/Sub + 워커로 후속 작업을 비동기 처리해 **응답 지연 180ms → 5ms, 상용 Spanner 노드 평균 CPU 90% → 50%**로 낮췄고, `SETNX`로 처리 키를 선점해 중복 소비를 막았습니다.
* **1억 건 상용 DB 데이터 정합성 복구:** Spanner 트랜잭션 버퍼의 INSERT가 후속 조회에 보이지 않아 중복 지급 버그가 있었습니다. 집계·추출은 BigQuery(ARRAY_AGG)로 분석계에서 수행하고 Go 스크립트는 UPDATE/DELETE만 하도록 역할을 분리해 부하를 줄였으며, dry-run과 실행 로그 리뷰 후 반영 — **71명 대상, 89건 갱신, 122건 중복 삭제**.
* **저사양 IoT 펌웨어 전송 아키텍처:** MCU의 제한된 RAM과 느린 Flash 쓰기 속도가 네트워크 수신 속도를 못 따라가 파일이 손상됐습니다. 장비가 자신의 가용 메모리와 원하는 딜레이를 요청에 담아 보내면 서버가 그 값으로 전송 루프를 제어하도록 바꿔 **업데이트 실패율 60% → 10% 이하**로 줄였습니다.
* **서버 프로세스 운영 자동화:** 14개 서비스를 단일 Shell Script로 일괄 관리(기동·재시작·중지), 락 파일(`$$` + `kill -0`)과 세션 확인으로 중복 실행을 방지했습니다.

---

## 🧭 그 외 저장소

* [go-rest-example](https://github.com/opt-dohun/go-rest-example) — Go(Gin)·MySQL로 대량 IoT 디바이스 정보와 활성 상태를 관리하는 REST API
* [gen-e2etest](https://github.com/opt-dohun/gen-e2etest) — Go 기반 E2E 테스트
