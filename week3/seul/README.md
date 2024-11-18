# 프로메테우스 바이너리 구성

![KakaoTalk_20241116_223117109](https://github.com/user-attachments/assets/8c03e796-0b49-44a8-a203-db713dc278fa)

### 프로메테우스 오퍼레이터

- 쿠버네티스에서는 헬름이나 오퍼레이터로 설치할 수 있음
- ServiceMonitor, PodMonitor가 서비스 디스커버리 기능을 제공해서 서비스와 파드의 증감이 발생할 경우 프로메테우스 구성 파일을 업데이트 함

### 프로메테우스 익스포터

- 특정 메트릭을 수집해서 엔드포인트에 노출시킴
- 마이크로서비스의 메트릭을 측정하기 위해서는 커스텀 익스포터를 개발해야 하는데 프로메테우스는 이에 필요한 API, SDK를 제공함

### 프로메네우스 어댑터

- 익스포터로 수집한 메트릭을 프로메테우스 서버가 스크래핑하여 파드를 오토스케일링함
- 복잡한 커스텀 메트릭은 프로메테우스 어댑터로 측정하여 오토스케일링할 수 있음

### 프로메테우스 라이프 사이클

- 메트릭을 수집하고 시계열로 저장
- 메트릭을 측정하고 리소스를 오토스케일링
- 변경된 리소스를 자동으로 디스커버리
- HPA와 연계해 증가한 리소스로 유저 트래픽 분배

</br>

# 프로메테우스 시계열 데이터베이스

### 데이터 형식

> http_requests_total {status = “200”, method = “GET”} @14331756013 94355
> 
- 키: http_requests_total {status = “200”, method = “GET”} @14331756013
- 값: 94355
- 메트릭 이름: http_requests_total
- 레이블: {status = “200”, method = “GET”}
- 타임스탬프: @14331756013

### 카디널리티

- 프로메테우스 서버가 스크래핑을 결정하는 기준
- 메트릭 이름과 연관된 레이블의 이름/값의 조합에 따라 생성되는 고유 시계열 수
- 위 예시에서 status 레이블은 세 종류(200, 400, 500)이고 method 레이블은 두 종류(GET, POST)이므로 레이블은 총 3X2=6개의 시계열 데이터를 생성
- 한계치가 명확하지 않은 레이블 값이 무한히 증가하거나 수백 개 이상의 예상치 못한 개수를 초과하는 경우를 ‘카디널리티 폭발’이라고 함

### 데이터 관리

- 스크래핑된 시계열 데이터는 프로메테우스 TSDB(Time Series Database)에 저장됨
- 샘플(시계열에서 사용되며 수집 데이터를 의미)을 수집하고 블록 형태로 만들어 디스크에 저장
- 프로메테우스에서는 ‘데이터(샘플)’ 대신 ‘청크’라는 용어를 사용함

![KakaoTalk_20241117_040830223](https://github.com/user-attachments/assets/d8a9e99f-cbc6-49ca-9010-efeb34094cbc)

### 블록 관리

- 샘플은 수집된 데이터 포인트로 시계열 수치를 나타냄
- 프로메테우스의 데이터 포인트는 일정한 시간 간격으로 수집됨
- 크기가 작은 파일이 많을 경우 모든 파일에 대한 인덱스를 만들고 검색해야 해서 조회 속도가 느려지는 반면, 파일 크기가 크면 작은 데이터를 조회하기 위해 불필요한 크기의 파일을 로딩해야 하므로 파일의 개수와 크기를 적절하게 유지해야 함 → 블록 병합
    - `--storage.tsdb.min-block-duration`: 하나의 블록에 저장된 데이터의 시간
    - `--storage.tsdb.max-block-duration`: 하나의 블록에 최대로 저장할 수 있는 시간
    - `min-block-duration`이 2h이고 `max-block-duration`이 12h이라고 가정할 때, 한 번에 병합되는 블록의 개수는 3개로 정해져 있기 때문에 블록은 최대 2X3=6h 블록까지 병합될 수 있음

### 데이터 저장 단계

- 가장 최근의 데이터는 두 시간 동안 **메모리**에 보관되며 쿼리 속도가 빠름
- 메모리에서 데이터가 손실되는 것을 방지하기 위해 디스크의 **로그 선행 기입(write-ahead logging, WAL)**으로 메모리 내 데이터 상태를 유지하며 프로메테우스가 충돌 혹은 재시작할 때 이를 재생시켜 복구함
- 두 시간이 지나면 청크가 디스크에 기록되며 청크는 불변의 형태. 데이터를 삭제할 경우 삭제 표시 파일이 생성됨