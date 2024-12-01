프로메테우스 서버를 포함한 프로메테우스 생태계(시계열 DB, 블록 처리, 알람, 오토스케일링)는 클라우드 네이티브를 구현하는 데 중요하다. 또한 프로메테우스를 기반으로 그라파나 관측 가능성 솔루션을 개발했기 때문에 내부 처리 방식과 운영 방식이 유사하다. 그렇기에 프로메테우스의 내부 원리를 이해하고 있으면 문제 발생 시 해결책을 근본적인 해결책을 찾는데 도움이 될 것이다.

 
# 프로메테우스 바이너리 구성
## 1️⃣ 프로메테우스의 기능
프로메테우스 기능은 운영자를 위한 메트릭 모니터링, 개발자를 위한 exporter, 오토스케일링 설정, 시계열 DB, 서비스 모니터를 사용한 서비스 디스커버리, 알람과 업무 규칙 등이 있다. 프로메테우스와 쿠버네티스는 긴밀하게 연결된다. 쿠버네티스는 런타임 플랫폼이고, 나머지 운영과 자동화는 프로메테우스를 통해 구현한다. 아래 그림은 프로메테우스의 기능과 쿠버네티스와의 관계를 나타낸 것이다.


<img width="805" alt="image" src="https://github.com/user-attachments/assets/85bbb7ac-59f2-43bb-b0b8-7007ed5bda8d"><br/>
프로메테우스 기능

<br/>

#### 프로메테우스 오퍼레이터 (Prometheus Operator)
- 역할: 쿠버네티스에서 프로메테우스를 구성할 때 자원과 프로비저닝을 수행한다.
- 설치: 헬름이나 오퍼레이터로 프로메테우스를 설치할 수 있다. (그라파나는 헬름 차트를 사용한 설치만 가능하고, 추후에 오퍼레이터 방식도 지원할 예정이다.)
- 동작 방식: 오퍼레이터가 서비스와 파드의 증감을 모니터링하고, 증감 발생 시 프로메테우스 구성 파일 업데이트 (쿠버네티스 내에 동적으로 증가하는 서비스와 파드를 발견 > 오퍼레이터 내 ServiceMonitor와 PodMonitor가 서비스 디스커버리를 제공)
 
#### 프로메테우스 익스포터 (Prometheus Exporter)
- 역할: 특정 메트릭을 수집해 엔드포인트에 노출시킨다. (여러 시스템에 대한 익스포터, 기존 모니터링에 사용되는 에이전트들과 통합하는 익스포터도 존재)
- 특징: 기본 제공 익스포터 외에 도메인에 적합한 마이크로서비스 메트릭 측정을 위해 커스텀 익스포터는 직접 개발해야 함 (프로메테우스가 API, SDK 제공)
 
#### 프로메테우스 어댑터 (Prometheus Adapter)
- 역할: 복잡한 커스텀 메트릭을 측정하고, HPA와 연계해 신속하게 오토스케일링을 구현한다.
- 특징: 기존 메트릭 서버는 간단한 기본 메트릭만 측정할 수 있지만, 어댑터는 복잡한 커스텀 메트릭 지원이 가능하다.
- 동작 방식: 익스포터를 통해 제공된 커스텀 메트릭이 서버에 의해 스크래핑되고, 메트릭 측정과 시스템 부하 대응을 위해 HPA를 통한 파드 오토스케일링을 수행한다.
 
> 💡 쿠버네티스의 역할이 애플리케이션을 포함한 컨테이너 운영 및 스케줄링이라면, 프로메테우스는 쿠버네티스 기반의 애플리케이션이 원활히 돌아가도록 운영과 자동화를 담당한다.
 
 
#### 프로메테우스의 내부 동작 원리
1. 메트릭 수집과 시계열로 메트릭을 저장한다.
2. 메트릭 측정과 리소스를 오토스케일링 처리한다.
3. 변경된 리소스를 자동으로 찾는다.
4. HPA와 연계해 증가한 리소스로 유저 트래픽을 분배한다.

아래 그림은 노드 익스포터를 통해 프로메테우스로 전달되는 메트릭의 이동을 보여준다.

<img width="350" alt="image" src="https://github.com/user-attachments/assets/ea191637-bd1f-4b81-9ab3-09938d704a95"><br/>

 

## 2️⃣ 예제 - 익스포터
노드 익스포터가 OS 시스템 메트릭을 수집하고 프로메테우스 DB에 저장하는 과정을 살펴보자.
> 💡 여기서 잠깐!Node Exporter는 프로메테우스 익스포터 중 하나로, 하드웨어의 상태와 커널 관련 메트릭을 수집하는 메트릭 수집기이다. 노드 익스포터로부터 메트릭을 수집하면 프로메테우스 내의 TSDB에 저장되고, PromQL로 메트릭을 쿼리해 서버 상태를 모니터링할 수 있다.
 
 
#### 1. 프로메테우스 설치

Installation | Prometheus
An open-source monitoring system with a dimensional data model, flexible query language, efficient time series database and modern alerting approach.
prometheus.io


```
#wget 명령어로 압축 파일 다운로드
$ wget https://github.com/prometheus/prometheus/releases/download/v2.37.0/prometheus-2.37.0.linux-amd64.tar.gz
```

```
# 다운받은 압축파일의 압축 해제
$ tar xzvf prometheus-2.37.0.linux-amd64.tar.gz
```
 
#### 2. yaml 파일 설정
노드 익스포터의 metrics HTTP endpoint에 접근하는 yaml을 구성한다.
```
# yaml 파일 설정
$ cd prometheus-2.37.0.linux-amd64
$ vi prometheus.yml
```

```
#prometheus.yaml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
    - targets: ["localhost:9090"]
    
  - job_name: "node"
    static_configs:
    - targets: ["localhost:9100"] # 노드 익스포터의 default port:9100
```
 
> 각 세션이 하는 역할
- global: 전역 설정
- scrape_interval: 메트릭 수집할 주기 설정 (default: 1m)
- evaluation_interval: 규칙 평가할 주기 설정 (default: 1m)
- scrape_configs: 메트릭을 수집할 엔드포인트 설정
- job_name: 이 설정에서 수집한 타임시리즈에 대해 job=<job_name>으로 라벨 추가
- static_configs: 타겟과 타겟에 대한 라벨 지정
- targets: 메트릭을 수집할 엔드포인트 지정
 
 
#### 3. Service의 User를 프로메테우스로 지정하고 리눅스 시스템에 등록하는 파일 작성

```
#prometheus.service

[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart= /usr/local/bin/prometheus\
  --config.file /etc/prometheus/prometheus.yaml \
  --storage.tsdb.path /var/lib/prometheus/\
  --web.console.templates=/etc/prometheus/consoles\
  --web.console.libraries=/etc/prometheus/console_libraries\
  --storage.tsdb.max-block-duration=1m\ #default: 2h
  --storage.tsdb.min-block-duration=1m\ #default: 2h
  --web.enable-lifecycle \
  --web.enable-admin-api \
  --log.level=info
  
[Install]
WantedBy=multi-user.target
```
 
#### 4. 프로메테우스 시작
프로메테우스를 시작하는 명령어를 통해 위의 파일을 시작한다.

```
$ systemctl start prometheus
$ systemctl status prometheus
```
 
#### 5. 노드 익스포터 다운

```
$ tar xvfz node_exporter-1.3.1.linux.amd64.tar.gz
```
 
#### 6. node_exporter.service 파일 작성
시스템 등록을 위해 root로 사용자 파일 작성

```
#node_exporter.service

[Unit]
Description=node-exporter
After=network-online.target

[Service]
Type=simple
User=root
Group=root
SyslogIcentifier=node-exporter
WorkingDirectory=/root/node_exporter-1.3.1.linux-amd64
Restart=always
RestartSec=0s
ExecStart=/root/node_exporter-1.3.1.linux-amd64/node_exporter

[Install]
WantedBy=multi-user.target
```
 
#### 7. 노드 익스포터 실행 및 상태 확인

```
$ systemctl start node-exporter
$ systemctl status node-exporter
```
 
 

출처: https://youjava.tistory.com/46 [Software Engineer가 될래요:티스토리]
