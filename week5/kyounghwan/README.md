# 4. 오픈소스 관측 가능성, 그라파나

# LGTM 스택 구성 및 설치 가이드

---

## **4.1. 그라파나 관측 가능성**

### **4.1.1 목적과 범위**
- **LGTM 스택**(Loki, Grafana, Tempo, Mimir)은 대중적이고 라이선스에 자유로운 오픈소스 소프트웨어로 구성되어 있으며, 지속적으로 발전 중이다.
- **선택 이유**:
  - 커뮤니티의 활성화와 지속적인 로드맵 지원.
  - 기술적 우수성과 클러스터 구성이 가능.
  - API 활용 범위가 넓어 다양한 데이터 소스와 연동이 가능.
- LGTM 스택은 Elasticsearch와 같은 기존 솔루션과 차별화되며, 오픈소스 기반의 지속적인 개발 환경을 강조한다.

#### **LGTM 구성 요소**
1. **Loki**: 로그 데이터를 관리.
2. **Grafana**: 데이터 시각화를 위한 대시보드 제공.
3. **Tempo**: 분산 추적 관리.
4. **Mimir**: 메트릭 데이터 관리.

LGTM은 Promtail, Fluentbit 등 다양한 데이터 수집기로부터 데이터를 받아 시각화, 추적, 메트릭 관리, 저장까지 지원한다.

---

### **4.1.2 인프라 구성**
- **설치 전 필수 도구**:
  - **Minikube**: 로컬 Kubernetes 클러스터를 구축.
  - **Helm**: Kubernetes용 패키지 관리자.
  - **kubectl**: 클러스터와의 상호작용을 위한 CLI 툴.
- **설치 준비**:
  - Minikube 실행 시 충분한 리소스(메모리 12GB, CPU 4개) 할당 필요.
    ```bash
    minikube start --memory=12000 --cpus=4
    ```

---

### **4.1.3 애플리케이션 구성**
LGTM 스택에 필요한 여러 애플리케이션 설치.

#### **1) Minio**
- **역할**: 오브젝트 스토리지 역할을 하며, S3와 호환된다.
- **활용**: LGTM 스택에서 디스크 I/O 병목현상을 해결하고 메트릭 데이터를 모니터링.
- **설치 방법**:
  ```bash
  curl --progress-bar -O https://dl.min.io/server/minio/release/darwin-arm64/minio
  chmod +x minio
  mkdir minio-dir
  MINIO_ROOT_USER=admin MINIO_ROOT_PASSWORD=password ./minio server ./minio-dir --console-address ":9001"
  ```
- **브라우저로 [http://127.0.0.1:9001](http://127.0.0.1:9001) 접속하여 Minio 콘솔 확인 가능.**

#### **2) Redis**
- **역할**: NoSQL 기반의 인메모리 데이터베이스로, 고성능 캐시 구현에 사용.
- **장점**:
  - 빠른 쿼리 처리.
  - 샤딩을 통해 클러스터 구성 가능.
- **설치 방법**:
  ```bash
  brew install redis
  brew services start redis
  redis-cli
  ```
#### **Redis 관리 툴(`redis-commander`) 설치**
- **Redis 관리 툴 설치 방법**:
  ```bash
  npm install -g redis-commander
  redis-commander
  ```
- Redis Commander 실행 후, 브라우저에서 다음 주소로 접속:

  - [http://127.0.0.1:8081](http://127.0.0.1:8081)

#### **3) Memcached**
- **역할**: Redis와 유사한 NoSQL 인메모리 데이터베이스로, LGTM 스택에서 기본적으로 사용됨.
- **특징 비교 (Redis vs. Memcached)**:
  | 항목              | Memcached         | Redis               |
  |-------------------|-------------------|---------------------|
  | 데이터 분할       | 클라이언트 지원  | 서버 지원           |
  | 데이터 구조 지원  | 단순 (String)     | 다양 (String, Hash) |
  | 스레드 모델       | 멀티스레드        | 싱글스레드          |
  | 데이터 영속성     | 미지원            | 지원                |
  | 복제 및 트랜잭션 | 미지원            | 지원                |

- **설치 방법**:
  1. **macOS에서 Memcached 설치**:
     ```bash
     brew install memcached
     brew services start memcached
     ```
  2. **Memcached 클라이언트 연결 테스트**:
     ```bash
     telnet 127.0.0.1 11211
     set key 0 60 5
     value
     get key
     ```
     - "value"라는 값을 key에 저장한 후, 이를 다시 조회하여 설치 확인 가능.




