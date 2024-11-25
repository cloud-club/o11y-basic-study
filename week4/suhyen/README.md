# 3. 프로메테우스 오퍼레이터 & 어댑터

### 프로메테우스 오퍼레이터

프로메테우스 오퍼레이터에는 ServiceMonitor와 PodMonitor가 있는데, 그 중 ServiceMonitor 사용을 추천한다. 그 이유는 파드는 상태가 자주 변경되기 때문에, 쿠버네티스 환경에서는 서비스를 사용해서 파드와 통신하는 것을 권장한다.

프로메테우스가 쿠버네티스 내 애플리케이션을 모니터링하려면 엔드포인트가 있어야 한다. 엔드포인트는 기본적으로 IP 주소 목록으로, 쿠버네티스의 서비스는 LabelSelector로 파드를 발견하고 이를 엔드포인트에 추가한다. ServiceMonitor는 차례로 해당 엔드포인트를 검색하고 파드를 모니터링하도록 프로메테우스를 구성한다.

<img width="457" alt="service-monitor" src="https://github.com/user-attachments/assets/4695b7f4-60e0-413c-ab78-ef20d3d24575">

기본적으로 검색된 엔드포인트는 metadata와 labels를 내장하고 있다. 사용자의 요구 사항에 적합하도록 label을 변경하고, 불필요한 label을 제거하거나 label의 개수를 줄이는 등의 작업으로 시계열 데이터 관리를 최적화할 수 있다. 

<img width="374" alt="endpoint" src="https://github.com/user-attachments/assets/9b325c71-55fc-45f2-b23c-53658f0d8bea">

### 프로메테우스 어댑터

프로메테우스 어댑터는 `discovery` 규칙을 통해 노출할 메트릭과 이를 노출하는 방법을 결정한다. 각 규칙은 독립적으로 실행되어야 하기 때문에 규칙이 상호 배타적이어야 한다.

각 규칙은 네 부분으로 나눌 수 있다.

- `discovery` 는 어댑터가 이 규칙에 대한 모든 프로메테우스 메트릭을 찾는 방법을 지정한다.
- `association` 은 어댑터가 특정 메트릭이 연결된 쿠버네티스 리소스를 결정하는 방법을 지정한다.
- `naming` 은 어댑터가 사용자 정의 메트릭 API에서 메트릭을 노출하는 방법을 지정한다.
- `querying` 은 하나 이상의 쿠버네티스 개체의 특정 메트릭에 대한 요청을 프로메테우스의 쿼리로 변환하는 방법을 지정한다.

### KEDA

쿠버네티스 오토스케일링은 프로메테우스 어댑터, 프로메테우스 오퍼레이터, 프로메테우스 익스포터 등과 연계되어 처리된다.

이때, 쿠버네티스에서 지원하는 Pod Autoscaler는 CPU/MEM 사용률 기반으로 Pod의 개수를 동적으로 스케일링한다. 하지만 CPU/MEM 사용률 외에 외부 지표(ex. job queue size, http request rate 등)에 기반해서 Pod 개수를 스케일링하려면 상당히 번거롭고 많은 작업을 해야한다(지표를 prometheus 등의 metrics 서버에 수집하고, HPA에서 외부 metrics 서버 값을 참조 하는 방식 등).

따라서 프로메테우스 어댑터는 프로메테우스 메트릭만을 측정할 수 있으며, 다른 시스템 메트릭과 직접적으로 연계하는 것이 어렵다. 이를 쉽게 하기 위해서 KEDA 프로젝트가 오픈 소스로 시작 되었으며, KEDA는 **K**ubernetes **E**vent-**D**rived **A**utoscaling의 약자로  CNCF재단의 인큐베이팅 프로젝트의 하나이다.

자세한 내용은 아래 링크를 참고한다. 

[공식 문서(아키텍처 참고)](https://keda.sh/docs/2.16/concepts/) / [깃허브](https://github.com/kedacore/keda)

# 4. 프로메테우스 알람

<img width="517" alt="alarm-flow" src="https://github.com/user-attachments/assets/f657b98a-da60-4a83-a86b-cefb51400a9f">

Prometheus 공식 문서에 따르면 Prometheus에서 Alert는 크게 2가지 컴포넌트로 구성되어있다.

- **Alert Rule** : PromQL을 기반으로 정의하는 알람이 발생할 규칙
- **AlertManager** : 수신받은 Alert를 다양한 서비스에 notify를 수행

### Alerting Rule

Prometheus 쿼리문(PromQL)을 바탕으로 정의하는 알람 조건, 조건에 맞아 알람이 발생하면 연결된 외부 서비스로 알람을 전달하게 된다.

Prometheus에서 Alert Rule을 선언하는 방법으로 Prometheus 웹 GUI 상에서 선언해주는 방법과 설정파일을 작성하는 방법이 존재한다.

```yaml
groups:
- name: example
  rules:
  - alert: HighRequestLatency
    expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5 # Alert가 트리거될 쿼리문
    for: 10m # 트리거된 Alert가 얼마만큼 지속되어야 Firing state가 될지
    labels:
      severity: page
    annotations:
      summary: High request latency
```

Alert Rule 설정에서 가장 중요한 부분은 `expr`과 `for` 구문으로 볼 수 있다. `expr`은 알람 조건식으로, 해당 조건식이 참이되는 경우 Alert가 트리거된다. 그리고 `for` 구문은 트리거된 Alarm이 Firing State로 넘어가기 위해 체크할 지속시간이다.

<details> 
<summary> <b>Alarm State</b> </summary> 

Prometheus에서 Alarm은 크게 3가지 상태로 이루어져 있다.

- **Inactive** : Alert Rule에 맞는 alarm이 없는 상태로 **정상 상태**로 본다.
- **Pending** : Alert Rule에 해당하는 **alarm이 발생한 상태**로, 해당 Alert Rule의 `for` 기간 동안 Pending이 유지되면 Firing단계로 넘어가게 된다.
- **Firing** : 설정한 `for`기간 동안 **Pending이 유지되어 전이된 상태**, Firing 상태가 되어야 Prometheus가 AlertManager에게 alarm을 전송하게 된다.

<img width="477" alt="alarm-state" src="https://github.com/user-attachments/assets/64717080-a2f0-496a-aac7-e6fd2c0b7f8e">

</details>

### Alert Manager

Prometheus의 Alert Rule로부터 전달된 Alert를 Notify할 수 있도록 Receiver에 뿌려주는 컴포넌트이다. Prometheus와는 별개의 프로세스로 동작한다.

Prometheus server가 가져온 이벤트(PULL)를 기반으로 Alarm이 발생하는 경우, 이를 다양한 형태의 클라이언트에 뿌려주는(Push alert) 이벤트를 실행한다.

AlertManager도 yaml 설정파일을 작성함으로써 구축할 수 있다. 주된 setting 구간은 아래와 같다.

- **global** : AlertManager의 **global한 설정**구간, 여기에서 이메일을 사용할 경우 SMTP나 notification 전달을 위한 API의 URL 등을 설정할 수 있다.
- **receiver** : AlertManager가 **alert를 전송할 대상**(receiver)을 지정하는 구간, Slack의 경우 수신받을 slack 채널, 메시지 템플릿/내용 등을 지정할 수 있다.
- **route** : 전달된 alert를 **어떤 receiver에 보낼지 Routing하는 설정**을 지정하는 구간, 주로 alert의 라벨의 값에 따라 alert를 분류하고 group_by 설정을 통해 alert들을 그룹핑한다.
- **templates** : recevier에 보낼 alert 메시지의 포맷을 설정할 수 있는 **템플릿 함수가 정의된 파일의 경로**(Go Template 형식을 사용)

<details> 
<summary> 프로메테우스 알람 vs 그라파나 알람 </summary> 

알람을 개발하는 경우에 프로메테우스 알람 규칙과 그라파나 서버는 거의 차이가 없고 동일한 알람 기능을 제공한다. 그라파나 서버는 프로메테우스에서 갈라져서 나와 개발된 오픈소스이기에 프로메테우스를 기준으로 개발된 알람 규칙은 그라파나 서버에서도 동일하게 작동한다. 

(그라파나, 미미르, 로키 알람은 4장에서 살펴보도록 한다.)

</details><br>

# 5. 프로메테우스 운영 아키텍처

[Prometheus를 스케일링 하기 위한 Thanos](https://bcho.tistory.com/1375)

⇒ 책보다 이 글이 내용을 더 깔끔하게 담고 있다.

# 6. 참고 문서

https://product.kyobobook.co.kr/detail/S000211468119
https://prometheus.io/docs/introduction/overview/
https://blog.naver.com/PostView.nhn?blogId=alice_k106&logNo=221829384846
https://devocean.sk.com/blog/techBoardDetail.do?ID=164800
https://blog.omoknooni.me/115
https://bcho.tistory.com/1375
