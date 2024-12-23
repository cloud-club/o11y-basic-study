# 관측 가능성의 표준, OpenTelemetry. (feat. 실제 활용을 추측해보기)

- 현재 대규모 시스템 운영에 있어서 상황은 기존 텔레메트리 기술과 오픈텔레메트리를 함께 사용하는 추세.

- 오픈텔레메트리의 전면적인 이행은 아직 어려운 듯

- 여전히 스펙과 기능이 완전하지 않기 때문..!



## 오픈텔레메트리 = 오픈트레이싱 + 오픈센서스

...프로젝트 합병

현재 데이터독, 뉴렐릭 등의 전문 벤더사랑, 구글 아마존과 같은 대형 클라우드 사업자 참여중

https://learn.datadoghq.com/courses/take/understanding-opentelemetry/texts/59550760-introduction

Datadog docs에 Understanding OpenTelemetry 과정이 있어서 바로 등록 후 수강 (무료!)

> 강의 간단 설명 : By the end of this course, you’ll be prepared to start using OTel in your applications. You’ll be able to: 
>> - Describe the core components of OTel: SDKs, APIs, and the Collector
>> - Explain each part of an OTel pipeline: receivers, processors, exporters, and connectors
>> - Describe Datadog’s relationship with OTel

![image](https://github.com/user-attachments/assets/c2874d41-214e-42cd-a4eb-d67525375883)

# Otel 3가지 중요한 키워드

## 1. Creating

## 2. Collecting

## 3. Processing


![image](https://github.com/user-attachments/assets/ff8737d7-e6b0-43db-bf52-480f38cf05ef)


- 데이터독은 이렇게 쓰는데, 아마 OTel 활용하는 다른 모니터링 SaaS회사들 다 이렇게 쓸거 같다.

---


# 이어서, OpenTelemetry(OTel)를 쉽게 설명해보자

### 먼저 OpenTelemetry Collector ... 우체국이랑 비슷함

1. **수집 과정의 이해**
- 여러분이 만든 애플리케이션은 편지를 보내는 사람
- OTel Collector는 우체국
- 데이터독(Datadog)같은 분석 도구는 편지를 받는 사람

2. **Collector의 주요 부품들**
프로그램이 어떻게 돌아가는지 추적하는 데이터가 이동하는 과정을 설명하면:

A. **Receivers(수신기)**
- 우체국 창구와 같음
- 여러분의 프로그램이 보낸 데이터를 첫 번째로 받아주는 곳

B. **Processors(처리기)**
- 우체국 직원들이 하는 일과 비슷
- 예를 들면:
  - 중요한 개인정보는 가려주기
  - 비슷한 데이터들을 한 번에 묶어서 처리하기
  - 필요한 정보를 추가로 붙여주기

C. **Exporters(내보내기)**
- 우체부랑 같은 역할
- 처리된 데이터를 최종 목적지(예: Datadog)로 보내주는 역할

D. **Connectors(연결고리)**
- 서로 다른 종류의 데이터를 연결해주는 다리 역할
- 예를 들어, 프로그램의 실행 기록과 성능 측정 데이터를 서로 연결해서 더 자세한 분석을 가능

3. **실제 사용 예시**
간단한 예로 yaml파일 작성한 거
```yaml
# 데이터를 받는 방법 설정
receivers:
  otlp:     # 프로그램에서 보내는 데이터를 받을 준비
    protocols:
      http:  # 인터넷으로 데이터 받기

# 데이터 처리 방법 설정
processors:
  batch:    # 데이터를 모아서 한꺼번에 처리
    timeout: 5s  # 5초마다 한 번씩 처리

# 데이터를 보내는 방법 설정
exporters:
  datadog:  # Datadog으로 보내기
    api:
      key: "여기에_API_키_입력"
```

이렇게 설정하면, 프로그램에서 나오는 정보들이:
1. Receiver로 들어와서
2. Processor에서 필요한 처리를 거친 후
3. Exporter를 통해 Datadog으로 전송된다!

특히 좋은 점은, 레고 처럼 필요한 부품들을 조립해서 데이터 수집 시스템을 커스터마이징 할 수 있다.. 짱짱맨

---

# 결론 : 

> 추가로, Datadog dev 101, 그리고 SRE 강좌를 들으면 좋다고 합니다.. 업무에 데독을 쓰는 날이 있다면 찾아보도록 할것임다..

---




참고자료 (정리가 매우 잘 되어있음. 물론 영어로!)

https://opentelemetry.io/docs/

![image](https://github.com/user-attachments/assets/e12fe12f-c1cf-49df-9aa6-a1fde9b9bd78)

여담으로 위 이미지를 보고나서, 지난 DatadogKRUG 1회 밋업에서 소개되었던 올리브영 대시보드 사례를 떠올리며 아래와 같이 내용을 요약해보았음.(perflexity)

---

올리브영의 온라인 플랫폼을 운영하는 상황을 예시로

## 전체 구조

### 1. 왼쪽 영역 (데이터 발생)

**Microservices:**
- 주문 서비스: 장바구니에서 주문까지의 프로세스 관리
- 결제 서비스: 다양한 결제 방식 처리
- 배송 조회 서비스: 온라인 주문의 배송 상태 추적
- 재고 관리 서비스: 온/오프라인 매장의 재고 연동[3]

**Shared Infra:**
- Kubernetes: 각 마이크로서비스의 컨테이너 오케스트레이션[2]
- L7 Proxy: 고객 요청을 적절한 서비스로 라우팅
- 클라우드 인프라: 확장성과 안정성 확보

### 2. 중앙 (데이터 수집 - OTel Collector)

모든 서비스에서 발생하는 데이터를 수집:
- 주문 처리 시간
- 결제 성공/실패 기록
- 서비스 에러 로그
- 시스템 성능 정보
- 매장별 판매 데이터

### 3. 오른쪽 (데이터 저장 및 분석)

- Time Series DB: 시간대별 주문량, 매장별 판매 추이 기록
- Trace DB: 주문-결제-배송으로 이어지는 전체 고객 여정 추적
- Column Stores: 상품별, 브랜드별 판매 통계 데이터 저장

## 실제 활용 예시

1. 고객이 올리브영 앱에서 마스크팩을 장바구니에 담고 주문 버튼을 클릭합니다[1].
2. OTel이 이 주문이 결제를 거쳐 배송 준비까지 얼마나 걸리는지 추적합니다.
3. 만약 특정 단계에서 지연이 발생하면, 어느 마이크로서비스에서 문제가 있는지 바로 파악할 수 있습니다.
4. 수집된 데이터를 분석하여 인기 상품의 재고를 매장 입구 쪽에 비치하는 등의 최적화를 수행합니다.
5. 국내 및 글로벌 고객을 위한 키오스크 사용 데이터를 분석하여 서비스 개선에 활용합니다.

올리브영 온, 온라인 플랫폼에서 발생하는 모든 상황을 실시간으로 모니터링하고, 문제 발생 시 신속하게 대응. 또한, 수집된 데이터를 바탕으로 고객 경험을 지속적으로 개선하고, 온/오프라인 연계 서비스를 최적화

Citations:

[1] https://retailtalk.co.kr/Issue/?bmode=view&idx=37685260

[2] https://www.redhat.com/ko/topics/microservices/what-are-microservices

[3] https://oliveyoung.tech/introduce/

[4] https://sdy-study.tistory.com/267

[5] https://oliveyoung.tech/2023-10-17/oliveyoung-mall-home-new-architecture/


