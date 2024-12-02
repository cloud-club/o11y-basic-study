# Like Prometheus, but for logs.

> 지난번에 학습했던 프로메테우스에서 영감을 받아서 발전한 오픈소스라 한다. 실제로 프로젝트 구성이 조금 비슷하다.

<img width="659" alt="image" src="https://github.com/user-attachments/assets/7e7805bb-4e72-4837-b990-d7fe8d9dbc77">



## perflexity를 사용해서 loki의 특징을 물어보았다.

Loki의 주요 구성 요소는 다음과 같습니다:
Distributor: 로그 데이터를 수신하고 메타데이터 및 해시 알고리즘에 따라 여러 Ingester로 분배하는 첫 번째 단계입니다1.
Ingester: 로그 데이터를 압축하고 chunk 블록을 구성하며, 일정 조건에 도달하면 데이터베이스에 저장합니다1.
Querier: 지정된 시간 범위와 레이블 선택기에 따라 데이터베이스에서 로그를 검색하고 결과를 표시합니다1.
Query frontend: 선택적 구성 요소로, 쿼리 성능을 향상시키고 캐싱 기능을 제공합니다1.
Promtail: Loki로 로그를 수집하여 전송하는 에이전트로, 각 노드에 배포되어 로그를 전처리하고 라벨을 추가합니다1.
이러한 구성 요소들이 함께 작동하여 Loki는 효율적인 로그 수집, 저장, 처리 및 쿼리 기능을 제공합니다


교재에서는 Loki에 대한 배경이나 특징은 짤막하게 넘어가고 바로 동작방식과 내부 설계에 대해서 설명하는데 궁금해서 기능 검색을 해보았음.

그 중에서도 마지막단에 동작하는 Promtail 소스를 찾아보기로 (지난 프로메테우스때는 수집을 뜯어보았기 떄문..!)

loki 메인 저장소: [https://github.com/grafana/loki/tree/main]

그 중에서도 기능구현이 활발한 /pkg에서 promtail을 찾을 수 있음.

<img width="1138" alt="image" src="https://github.com/user-attachments/assets/55adaa44-e70e-4daa-8737-c4744e9e1b92">

그런데, promtail의 핵심 로직이 어디에 구현되어있는지 찾아보다가 포기하고(...)

loki에 기여한 최근 커밋이나 좀 훑어보자는 생각에.. 2주내에 /pkg에서 commit meesage feat으로 merge된 히스토리 중 하나를 선택

---

경로 : loki/pkg/pattern/drain
/drain.go

우선 소스가 꽤 긴데, 메서드들 이름을 보면 뭐하는건지 유추가 가능하다.

주석에도 써놓았듯 로그 클러스터링할때 패턴매칭을 수행하는 구조체랑 함수를 구현해둔 곳.

그 중에서 아래 고수는 200~300번대사이에 이르는 함수 구현을 더 수정하고, 

drain.go초반의 여러 default값을 알맞게 수정했다.

<img width="1015" alt="image" src="https://github.com/user-attachments/assets/9a8be5d6-72f1-44b7-a83c-d7a5e0c66763">


이것이 고수들의 PR..

<img width="1108" alt="image" src="https://github.com/user-attachments/assets/cbfaf7bf-cba2-4b11-9e58-ce8aa1f37b93">
