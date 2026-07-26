# opt-dohun 개인 프로젝트 포트폴리오 (Portfolio)

백엔드 개발자 김도훈(opt-dohun)의 주요 백엔드 및 시스템 엔지니어링 개인 프로젝트 목록입니다. 각 프로젝트는 성능 최적화, 로우레벨 통신, 분산 환경[...]

---

## 🎮 1. HexWar (분산 기반 실시간 게임 서버)
- **GitHub 저장소:** [HexWar (HaxWar)](https://github.com/opt-dohun/HaxWar)
- **기술 스택:** `C# / .NET Core`, `gRPC`, `WebSockets`, `Redis Cluster`, `Redis (SetNX, Pub/Sub)`, `Nginx`, `Docker Compose`

### 💡 기술적 고민 및 해결
* **Redis 클러스터 기반 데이터 샤딩:** 2,000명 동시 접속 타깃 환경에서 단일 Redis 인스턴스의 메모리 병목을 해결하기 위해 Redis Cluster 도입. 게임 상태(맵 정보, 플레이어 위치, 아이템)를 해시 슬롯 기반으로 샤딩하여 수평 확장 가능한 아키텍처 구축. 2-way 레플리카로 고가용성 확보[...]
* **분산 환경 이벤트 브로드캐스팅 및 상태 동기화:** Redis SetNX를 활용한 분산 락으로 게임 상태 ���데이트 시 동시성 제어, Redis Pub/Sub으로 실시간 이벤트 전파. 휘발성 Pub/Sub 문제를 보완하기 위해 서버 로컬 메모리에 고정 크기 원형 큐를 구현하여 재접속 시 이벤트 히스토리 제공[...]
* **힙 할당 및 GC 오버헤드 85.7% 감축:** 초당 수만 건의 패킷 처리 중 발생하는 힙 메모리 파편화를 방지하기 위해 `ArrayPool` 및 `Memory<byte>`, `ReadOnlySpan<byte>` 활용. 객체 재사용 패턴으로 GC 압력 감소[...]

---

## 📊 2. HexWar Exporter (Prometheus 메트릭 수집 및 K8s 인프라 구축)
- **GitHub 저장소:** [hexwar-exporter](https://github.com/opt-dohun/hexwar-exporter)
- **기술 스택:** `Go 1.22`, `Prometheus`, `Kubernetes (k3d)`, `Agones`, `KEDA`, `HPA`, `Karpenter`, `Grafana`, `Loki`, `Promtail`, `LocalStack`, `OpenTelemetry`

### 💡 기술적 고민 및 해결
* **메트릭 수집 사이드카 패턴:** C#/.NET 기반 게임 서버에 관측성 라이브러리 추가로 인한 복잡도 증가를 방지하기 위해 Go 기반 독립 사이드카로 메트릭 수집을 분리. Prometheus 스크랩당 **0.017ms 처리 속도**와 **56KB 메모리 할당**으로 오버헤드 최소화. 서킷 브레이커 패턴으로 게임 서버 장애 시 메트릭 수집 지연으로 인한 상위 시스템 영향 차단[...]
* **K8s 게임 서버 오토스케일링:** LocalStack과 k3d를 연동하여 AWS EC2 Auto Scaling Group API를 모킹하고, Karpenter와 KEDA 연동으로 **Prometheus 메트릭 기반 이벤트 오토스케일링** 구축. 스케일 다운 시 Pod 내부의 활성 게임 세션을 counter로 추적하여 진행 중인 게임이 중단되지 않도록 graceful shutdown 관리[...]
* **통합 로그 수집 파이프라인:** Promtail 데몬셋을 각 노드에 배포하여 쿠버네티스 전체 로그 수집, Loki로 중앙 집중식 저장 및 관리, Grafana와 연동하여 Prometheus 메트릭과 Loki 로그를 통합 모니터링 대시보드 구축[...]

---

## 💬 3. .Net-Socket-ChatServer (로우레벨 TCP 소켓 채팅 서버)
- **GitHub 저장소:** [.Net-Socket-ChatServer](https://github.com/opt-dohun/.Net-Socket-ChatServer)
- **기술 스택:** `C# / .NET`, `System.Net.Sockets`, `Async/Await Task`

### 💡 기술적 고민 및 해결
* **TCP 패킷 단편화 및 병합 대응:** 경계가 없는 TCP 스트림 환경에서 패킷 유실 및 병합 오류를 방지하기 위해 헤더에 페이로드 크기를 명시하는 고유의[...]
* **비동기 기반 통신:** 다중 클라이언트 접속 시 스레드 풀의 고갈을 막기 위해 C#의 비동기 소켓 통신 API를 적용하여 비동기 논블로킹 방식으로 클라[...]

