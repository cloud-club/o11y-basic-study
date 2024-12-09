# 4.3 미미르 메트릭 관리

### 1) Mimir

- [docs](https://grafana.com/docs/mimir/latest/)  
    - Grafana Mimir is an open source software project that provides horizontally scalable, highly available, multi-tenant, long-term storage for Prometheus and OpenTelemetry metrics.  
    ![image (4)](https://github.com/user-attachments/assets/3c2c9097-5aef-4bbf-9f81-dc7ea96a21d7)  
    - Grafana Mimir enables users to ingest Prometheus or OpenTelemetry metrics, run queries, create new data through the use of recording rules, and set up alerting rules across multiple tenants to leverage tenant federation. Once deployed, the best-practice dashboards, alerts, and runbooks packaged with Grafana Mimir make it easy to monitor the health of the system.  
- [blog](https://blog.palark.com/prometheus-centralized-storage-mimir/)  
    - the story of a fictional startup engaged in developing software, collecting metrics from it, and struggling to find the best ways to store and process them  
    - Step1: Prometheus  
    ![image (5)](https://github.com/user-attachments/assets/706d4074-e10d-440d-839b-4451df628a5e)  
    - Step2: Prometheus HA (LB or proxy)  
    ![image (6)](https://github.com/user-attachments/assets/bcd47430-8525-403d-bbc8-5da8f164f40a)  
    - Step3: Prometheus Long-term (Mimir)  
    ![image (7)](https://github.com/user-attachments/assets/73c06d65-ca67-4df7-9f65-65c45abc9d35)  

### 2) 메트릭 수집 방식: Pull vs Push  

<img width="439" alt="스크린샷 2024-12-08 오후 5 35 33" src="https://github.com/user-attachments/assets/b8790b20-d335-4ce1-8086-863a023de21c">  

| 구분 | **Pull 방식** | **Push 방식** |
| --- | --- | --- |
| **정의** | 모니터링 시스템이 타겟에서 데이터를 가져옴 | 타겟이 모니터링 시스템에 데이터를 보냄 |
| **구현 복잡성** | 상대적으로 단순 | 전송 로직을 애플리케이션에 추가 필요 |
| **실시간성** | 정기적인 요청(Polling)으로 약간의 지연 있음 | 데이터가 생성되자마자 즉시 전송 가능 |
| **장점** | 안정적이고 지속적인 메트릭 수집에 적합 | 실시간성과 이벤트 중심 작업에 적합 |
| **단점** | 주기적 요청으로 인한 부하 가능, 즉시성 부족 | 네트워크 비용 증가, 전송 실패 위험 |
| **대표 툴** | Prometheus, Zabbix 등 | OpenTelemetry Collector, Prometheus Push Gateway 등 |

- **수집 대상이 Auto-scaling 등으로 가변적일 경우, Push 방식이 Pull 방식보다 유리하다**  
    - 만약 새로운 수집 Host가 추가될 경우, Push방식에서는 Host가 Data-Backend로 수집데이터를 보내주기에, 전송된 데이터를 받기만 하면된다. 하지만 Pull 방식에선 Data-Backend가 수집 Host로 접속하여 데이터를 긁어가야 하기에, 중앙서버에서 pull해갈 Host/Service의 목록들을 관리하여야 한다.
- **수집 대상의 유연성 측면에서는 Pull 방식이 Push 방식보다 유리하다**  
    - Pull 방식에서 중앙서버는 데이터를 수동적으로 긁어가기만 하기에, Pull system에서 Data-Backend는 언제든지 새로운 데이터(unplanned metrics)들을 수용할 수 있으며 모든 메트릭을 요청할 수 있도록 구현되어있다. 하지만 Push 방식에선 메트릭을 중앙에서 정의하고 에이전트로 푸시하는 구조이며, 새로운 메트릭 수용을 위해서는 중앙 서버에서 해당 변경사항을 반영해줘야한다.
- **보안 측면에서는 Push 방식이 Pull 방식보다 유리하다**  
    - Pull방식은 수집 대상 서버에서 중앙 폴러가 접근할 수 있는 포트, IP 등을 수신 대기해야 한다.
- **HA 측면에서 Pull 방식이 Push 방식보다 유리하다**  
    - Data-Backend에서 장애가 발생하더라도, Pull 방식에선 수집 Host에 영향이 없으나 Push 방식에선 데이터 전송 재시도 등 Host에 영향이 생긴다.

[참고링크1](https://velog.io/@zihs0822/Push-vs-Pull-%EB%AA%A8%EB%8B%88%ED%84%B0%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EC%88%98%EC%A7%91-%EB%B0%A9%EC%8B%9D) / [참고링크2](https://velog.io/@hyunshoon/Monitoring-Pull-vs-Push-%EB%8F%99%EC%9E%91-%EB%B0%A9%EC%8B%9D-%EB%B0%8F-%EC%9B%90%EB%A6%AC-%EC%9D%B4%ED%95%B4) / [참고링크3](https://www.alibabacloud.com/blog/pull-or-push-how-to-select-monitoring-systems_599007)

### 3) Memberlist

Memberlist is a [Go](http://www.golang.org/) library that manages cluster membership and member failure detection using a gossip based protocol.  
The use cases for such a library are far-reaching: all distributed systems require membership, and memberlist is a re-usable solution to managing cluster membership and node failure detection.  
memberlist is eventually consistent but converges quickly on average. The speed at which it converges can be heavily tuned via various knobs on the protocol. Node failures are detected and network partitions are partially tolerated by attempting to communicate to potentially dead nodes through multiple routes.

<details>
<summary>gossip protocol</summary>

- 가십 프로토콜 (Gossip Protocol) 이란 분산 환경에서 메시지를 전달하는 커뮤니케이션 방식의 하나이다.
- 가십 커뮤니케이션 방식의 특징은 소문이 전파되어나가듯 Broadcast 해주는 마스터가 없이 각 노드가 주기적으로 TCP/UDP 기반으로 메타데이터를 주고받으면서 데이터를 전송하는 점에 있다.
- 가십 프로토콜은 아주 가볍기 때문에 특히 클라우드와 같은 분산 환경에서 성능 및 안정성을 위해 많이 사용되는데, Peer 간 데이터의 비동기 Sync Up 이 가장 큰 예시가 된다.
- Docker 같은 경우에도 내장된 Control Plane 에서 노드 발견을 위해 Gossip 메커니즘을 사용한다.
- 이를 통해 기존 분산 환경에서의 동기화 알고리즘인 Paxos 나 Raft 의 어려움과 복잡함 대신 다소 가볍게 멀티 클러스터를 관리할 수 있는 근간 알고리즘으로 동작할 수 있게 된다.

![image (8)](https://github.com/user-attachments/assets/b92646b8-f635-4719-bbd6-38996ad6d49f)

</details> <br>

[참고링크1](https://github.com/hashicorp/memberlist?tab=readme-ov-file) / [참고링크2](https://jins-dev.tistory.com/entry/%EA%B0%80%EC%8B%AD-%ED%94%84%EB%A1%9C%ED%86%A0%EC%BD%9C-Gossip-Protocol-%EC%9D%B4%EB%9E%80)

### 4) Memberlist in Mimir

**Grafana Mimir 멤버리스트 및 가십 프로토콜**

[Memberlist](https://github.com/hashicorp/memberlist) 는 가십 기반 프로토콜을 사용하여 클러스터 멤버십, 노드 장애 감지 및 메시지 전달을 관리하는 Go 라이브러리입니다. Memberlist는 최종적으로 일관성이 있으며 네트워크 파티션은 여러 경로를 통해 잠재적으로 죽은 노드와 통신을 시도함으로써 부분적으로 허용됩니다.   
[기본적으로 Grafana Mimir는 멤버 목록을 사용하여 키-값(KV) 저장소를](https://grafana.com/docs/mimir/latest/references/architecture/key-value-store/) 구현하고 인스턴스 간에 [해시 링](https://grafana.com/docs/mimir/latest/references/architecture/hash-ring/) 데이터 구조를 공유합니다.  
멤버리스트 기반 KV 스토어를 사용할 때 각 인스턴스는 해시 링의 사본을 유지합니다. 각 Mimir 인스턴스는 해시 링을 로컬로 업데이트하고 멤버리스트를 사용하여 변경 사항을 다른 인스턴스에 전파합니다. 로컬에서 생성된 업데이트와 다른 인스턴스에서 수신된 업데이트는 병합되어 인스턴스에서 링의 현재 상태를 형성합니다.  
멤버 목록을 구성하려면 [해시 링 구성을](https://grafana.com/docs/mimir/latest/configure/configure-hash-rings/) 참조하세요 .

**멤버리스트가 해시 링 변경 사항을 전파하는 방법**

멤버 목록 기반 KV 스토어를 사용할 때 모든 Grafana Mimir 인스턴스는 다음 기술을 사용하여 해시 링 데이터 구조를 다른 인스턴스로 전파합니다.

1. 최근 변경 사항으로 도입된 차이점만 전파합니다.
2. 전체 해시 링 데이터 구조를 전파합니다.

모든 `-memberlist.gossip-interval`인스턴스는 Grafana Mimir 클러스터 인스턴스의 하위 집합을 무작위로 선택하여 구성된 모든 `-memberlist.gossip-nodes`인스턴스에 최신 변경 사항을 보냅니다. 이 작업은 자주 수행되며 변경 사항을 전파하는 데 사용되는 기본 기술입니다.   
또한 모든 `-memberlist.pullpush-interval`인스턴스는 Grafana Mimir 클러스터에서 다른 인스턴스를 무작위로 선택하고 모든 해시 링을 포함하여 KV 스토어의 전체 콘텐츠를 전송합니다( `-memberlist.pullpush-interval`이 동작을 비활성화하는 0이 아닌 경우). 이 작업이 완료되면 두 인스턴스는 KV 스토어와 동일한 콘텐츠를 갖게 됩니다. 이 작업은 계산적으로 더 비싸고 결과적으로 덜 자주 수행됩니다. 이 작업은 해시 링이 주기적으로 공통 상태로 조정되도록 보장합니다.

**멤버리스트 사용으로 인한 이점**

Grafana Mimir는 **Memberlist**를 활용해 클러스터의 노드 상태 관리, 장애 탐지, 데이터 샤딩 및 리플리케이션과 같은 분산 환경에서의 필수 작업을 효율적으로 수행합니다. 이를 통해 확장성과 안정성을 보장하며, 대규모 환경에서도 복잡성을 최소화할 수 있습니다.

[참고링크1](https://grafana.com/docs/mimir/latest/references/architecture/memberlist-and-the-gossip-protocol/)

# 4.4 템포 추적 관리

### 1) Tempo

**왜 Grafana Tempo가 최근에 많이 떠오르고 있을까?**

- Grafana Mimir(시계열 저장소), Grafana Loki(로그 저장소)와의 통합 및 연동
    - Application Log에 Trace ID를 주입하여 로그와 트레이스 간 상호 연동을 할 수 있다.
    - Examplars를 통해 Latency가 높은 요청의 Trace를 곧바로 조회할 수 있다.
- 장기 객체 스토리지(Minio, GCS, S3) 지원
    - Volume Storage가 아닌 객체 스토리지를 지원함으로써 뛰어난 확장성을 가짐

**read/write data flow**

![image (9)](https://github.com/user-attachments/assets/964c1bff-f8ce-48fb-8cd2-e0f565b3d5b4)

**OpenTelemetry Collector + Grafana Tempo + Grafana 구성 예시**

![image (10)](https://github.com/user-attachments/assets/ea66a727-5285-4844-8100-716d46d00d03)


[참고링크1](https://nyyang.tistory.com/187) /  [참고링크2](https://wlsdn3004.tistory.com/49)

### 2) 부정확한 스팬 결과 vs 정확한 스팬 결과

스팬 사이의 인과관계가 나타나고, 스팬들이 동일한 추적에 속한다는 것을 보여주어야 한다. 이러한 관계는 스팬 레퍼런스로 나타난다.  
스팬이 개별적으로 출력되면 안 되고, 전체 추적 내에서 개별적인 스팬이 연결되는 구조로 생성되어야 한다.
- 부정확한 스팬 결과  
    <img src="https://github.com/user-attachments/assets/3c7499db-1e31-4dbc-9f27-ac904d62b54f" width="300" alt="부정확한 스팬 결과">
- 정확한 스팬 결과  
    <img src="https://github.com/user-attachments/assets/d6310679-67fc-45a3-a868-9142df8eb25f" width="300" alt="정확한 스팬 결과">

# 4.5 예거 추적 관리

### 1) Jaeger

**Jaeger란?**

Jaeger는 Uber에서 개발한 분산서비스 간 트랜잭션을 추적하는 오픈소스 소프트웨어이고 MSA 환경을 모니터링하는 프로젝트이다.
- 기본 구조
  ![image (4)](https://github.com/user-attachments/assets/9c9cfccf-db12-4068-b0d9-d185b17ebbfa)
- kafka 구조
  ![image (5)](https://github.com/user-attachments/assets/9e0170eb-a385-4bbe-9ba2-590f221ecd54)

[참고링크1](https://ohshinyeop.tistory.com/51)

### 2) Jaeger vs Zipkin vs Tempo

![image (12)](https://github.com/user-attachments/assets/f3496e64-5f38-4572-8757-7a13177578dd)

[참고링크1](https://codersociety.com/blog/articles/jaeger-vs-zipkin-vs-tempo)

### 3) HotROD

**HotROD**(Hot Ride On Demand)는 **Jaeger**에서 제공하는 **분산 추적 데모 애플리케이션**입니다. 이 애플리케이션은 마이크로서비스 기반의 아키텍처에서 Jaeger를 활용한 분산 추적 기능을 시연하는 데 사용됩니다. HotROD는 **"택시 호출 서비스"**라는 간단한 시나리오를 구현하며, Jaeger의 기능과 분산 추적의 개념을 쉽게 이해할 수 있도록 설계되었습니다.

[Docs](https://www.jaegertracing.io/docs/1.18/getting-started/#sample-app-hotrod) / [GitHub](https://github.com/jaegertracing/jaeger/blob/main/examples/hotrod/README.md) 

### 4) Jaeger의 데이터베이스

예거의 데이터베이스는 오픈서치와 카산드라를 지원하는데, 성능 결과 리포트에 따르면 오픈서치가 더 높은 성능과 확장성을 제공한다.

### 5) 분산 추적 역사

**1. 초기 모니터링 도구 (1990년대 이전)**

- 초기에는 단일 서버 중심의 시스템에서 로그 파일과 성능 모니터링 도구를 사용.
- **주요 특징**:
    - 단일 노드에서 애플리케이션 로그 분석.
    - 네트워크 분산 환경이 부족했기 때문에 분산 추적의 개념은 등장하지 않음.
- **한계**:
    - 단일 서버 중심이라 분산 환경에서의 성능 병목과 문제를 파악하기 어려웠음.

**2. 분산 시스템의 등장과 문제 인식 (2000년대 초반)**

- **인터넷과 클라우드 기술의 발전**으로 대규모 분산 시스템이 등장.
- 마이크로서비스 아키텍처의 등장으로 인해, 단일 애플리케이션 로그 분석만으로 문제를 진단하기 어려워짐.
- **문제점**:
    - 요청이 여러 서비스에 걸쳐 처리되면서 **호출 체인(call chain)** 분석이 필요.
    - 단일 서버에서 수집한 로그로는 전체 시스템의 병목점을 확인하기 어려움.

**3. Dapper의 탄생 (2010년)**

- **구글(2010)**: 논문 *"Dapper, a Large-Scale Distributed Systems Tracing Infrastructure"* 발표.
- **Dapper의 주요 기여**:
    - 분산 추적의 개념을 정립.
    - 모든 요청에 고유한 **Trace ID**를 부여하여 요청이 여러 서비스에 걸쳐 처리되는 과정을 추적.
    - 이 데이터를 기반으로 성능 병목, 지연 시간 문제를 파악.
- **영향**: Dapper는 이후 여러 오픈소스 프로젝트와 상용 제품의 기반이 됨.

**4. Zipkin과 Jaeger의 등장 (2012-2017)**

- **Zipkin (2012)**:
    - **트위터**에서 Dapper를 오픈소스화한 도구.
    - **주요 기능**: 서비스 간 호출 체인을 시각화하고 성능 지연 문제를 파악.
    - Zipkin은 **HTTP, Kafka** 등 다양한 통신 프로토콜 지원.
- **Jaeger (2017)**:
    - **Uber**에서 개발한 분산 추적 도구.
    - **주요 기능**: 서비스 디스커버리, 트랜잭션 병목 분석, 실시간 모니터링.
    - CNCF(Cloud Native Computing Foundation) 프로젝트로 채택되며 오픈소스 표준화에 기여.

**5. OpenTracing의 등장 (2016)**

- **OpenTracing**:
    - 분산 추적의 표준 API 정의를 위한 프로젝트.
    - 다양한 언어와 프레임워크에서 일관된 방식으로 분산 추적 구현을 지원.
    - 주요 목표: 다양한 분산 추적 도구 간의 상호 운용성 확보.
- **한계**: 고수준 API 제공에 치중, 구현 간의 통합이 어렵다는 피드백을 받음.

**6. OpenTelemetry로의 통합 (2019~현재)**

- **OpenTelemetry**:
    - OpenTracing과 OpenCensus 프로젝트를 통합하여 CNCF가 주도.
    - **특징**:
        - 트레이싱뿐만 아니라 메트릭(Metrics)과 로그(Logs)를 통합하여 표준화.
        - 모든 클라우드 및 프레임워크에서 일관된 모니터링 데이터 수집 가능.
    - **장점**:
        - 현대 마이크로서비스 환경에 적합한 범용 분산 추적 표준.
        - 주요 클라우드 제공업체와 다양한 도구에서 지원.
    - OpenTelemetry는 현재 **분산 추적의 사실상 표준**으로 자리 잡음.

**7. 클라우드 네이티브와 Observability의 통합 (현재와 미래)**

- **클라우드 네이티브 환경의 발전**:
    - Kubernetes와 같은 컨테이너 오케스트레이션 플랫폼의 도입으로 추적 대상이 더욱 분산화.
    - 분산 추적이 **Observability**(관찰 가능성)의 한 축으로 자리 잡음.
- **추세**:
    - **분산 추적 + 메트릭 + 로그**의 통합을 통한 전방위적 모니터링.
    - AI/ML 기반의 **자동화된 문제 진단** 및 성능 최적화 도구 개발.
