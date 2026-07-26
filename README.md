# opt-dohun 개인 프로젝트 포트폴리오 (Portfolio)

---

## 🎮 1. HexWar (분산 기반 실시간 게임 서버)
- **GitHub 저장소:** [HexWar (HaxWar)](https://github.com/opt-dohun/HaxWar)
- **기술 스택:** `C# / .NET Core`, `gRPC`, `WebSockets`, `Redis Cluster`, `Redis (SetNX, Pub/Sub)`, `Nginx`, `Docker Compose`

### 💡 기술적 고민 및 해결
* **Redis 클러스터 기반 데이터 샤딩 및 고가용성 확보:**  2,000명 동시 접속 타깃 환경에서 단일 Redis 인스턴스의 메모리 및 처리량 병목을 해결하기 위해 Redis Cluster를 도입했습니다. 게임 상태(맵 정보, 플레이어 위치 등)를 해시 슬롯 기반으로 샤딩하여 수평 확장 가능한 아키텍처를 구축하고, 3-Master/3-Replica 구성으로 자동 페일오버를 통한 고가용성을 확보했습니다.
* **분산 환경 이벤트 브로드캐스팅 및 상태 동기화:**  SetNX 명령과 TTL을 활용한 분산 락으로 게임 상태 변경 시 동시성 제어를 수행하고, Redis Pub/Sub으로 실시간 이벤트를 브로드캐스팅했습니다. Pub/Sub의 메시지 유실 문제를 보완하기 위해 서버 내부에 고정 크기 메모리 저장소 CircularBuffer를 구현하여 일시적인 네트워크 지연 시에도 무손실 복구 경로를 확보했습니다.
* **힙 할당 및 GC 오버헤드 85.7% 감축:**  초당 수만 건의 패킷 처리 중 발생하는 힙 메모리 파편화와 GC로 인한 프리징 현상을 방지하기 위해 ArrayPool 및 Memory<byte>, ReadOnlySpan<byte>를 활용한 고속 처리 경로를 구축했습니다. 객체 재사용 패턴을 적용하여 2세대 GC 발생 빈도를 85.7% 감소시키고, 세션당 메모리 사용량을 56.8% 경량화했습니다.

---

## 📊 2. HexWar Exporter (매트릭 수집 사이트카 패턴 및 k8s 인프라 구축)
- **GitHub 저장소:** [hexwar-exporter](https://github.com/opt-dohun/hexwar-exporter)
- **기술 스택:** `Go 1.22`, `Prometheus`, `Kubernetes (k3d)`, `Agones`, `KEDA`, `HPA`, `Karpenter`, `Grafana`, `Loki`, `Promtail`, `LocalStack`, `OpenTelemetry`

### 💡 기술적 고민 및 해결
* **메트릭 수집 사이드카 패턴을 통한 도메인 분리:**  C#/.NET 기반 게임 서버에 관측성 라이브러리를 추가함으로써 발생하는 비즈니스 로직과 인프라 로직의 결합도 증가를 방지하고자, Go 기반 독립 사이드카로 메트릭 수집을 분리했습니다. 프로메테우스 스크랩 요청당 0.017ms의 처리 속도와 56KB 수준의 메모리 할당을 달성하여 오버헤드를 최소화했습니다. 또한 서킷 브레이커 패턴을 도입하여 게임 서버 장애 시 메트릭 수집 지연이 프로메테우스 등 상위 시스템에 장애를 전파하는 것을 차단했습니다.
* **K8s 게임 서버 오토스케일링 및 세션 생명주기 관리:**  LocalStack과 k3d를 연동하여 AWS EC2 Auto Scaling Group API를 로컬에서 모킹하고, Karpenter와 KEDA를 연동하여 Prometheus 메트릭 기반의 이벤트 오토스케일링을 진행하는 방법과 agones를 활용 스케일 다운 시 Pod 내부의 활성 게임 세션을 counter로 추적하여 진행 중인 게임이 강제 종료되지 않도록 Graceful Shutdown 관리 로직 두가지 패턴으로 구현하여 서버의 성격에 맞추어 스케일링 전략을 선택할 수 있도록 구축하였습니다.
* **통합 로그 수집 파이프라인 구축:**  Promtail 데몬셋을 각 노드에 배포하여 쿠버네티스 환경의 산재된 로그를 수집하고, Loki로 중앙 집중식 저장 및 관리한 뒤, Grafana와 연동하여 Prometheus 메트릭과 Loki 로그를 하나의 대시보드에서 통합 모니터링할 수 있는 환경을 구축했습니다.

---

## 💬 3. .Net-Socket-ChatServer (로우레벨 TCP 소켓 채팅 서버)
- **GitHub 저장소:** [.Net-Socket-ChatServer](https://github.com/opt-dohun/.Net-Socket-ChatServer)
- **기술 스택:** `C# / .NET`, `System.Net.Sockets`, `Async/Await Task`

### 💡 기술적 고민 및 해결
* **TCP 패킷 단편화 및 병합 대응:** 경계가 없는 TCP 스트림 환경에서 패킷 유실 및 병합 오류를 방지하기 위해 헤더에 페이로드 크기를 명시하도록 적용하여 메시지 경계를 명확히 구분했습니다.
* **비동기 기반 통신:** 다중 클라이언트 접속 시 스레드 풀의 고갈을 막기 위해 C#의 비동기 소켓 통신 API를 적용하여 비동기 논블로킹 방식으로 클라이언트의 요청을 안정적이고 효율적으로 처리했습니다.

