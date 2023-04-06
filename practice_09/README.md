# 9강. 컨테이너 모니터링으로 투명성 있는 애플리케이션 만들기

도커를 익힌다는 것은 Dockerfile 스크립트와 도커 컴포즈 스크립트 작성법을 배우는 것만이 아님.
도커의 진짜 매력은 컨테이너를 중심으로 만들어진 생태계와 이들 생태계를 구성하는 도구를 적용하는 패턴임.

이번 시간에는 그 유명한 프로메테우스와 그라파나를 이용해서 컨테이너 모니터링을 하는 방법을 배워볼거임.

### 컨테이너화된 애플리케이션에서 사용되는 모니터링 기술 스택

일단 우리는 도커 엔진으로부터 도커있는 서버주소/metrics를 접속해보면 도커 컨테이너 플랫폼에서 일어나는 일에 대한 정보들을 받아올 수 있음. 일단 이를 위해서는 docker Daemon의 설정을 다음과 같이 바꿔줘야함. 

```docker
"metrics-addr" : "0.0.0.0:9323",
"experimental": true
```

이렇게 수정한후에 [localhost:9323/metrics를](http://localhost/metrics를) 접속해보면 정보가 쫙 나온느걸 알 수 있음. 

```docker
# HELP engine_daemon_container_actions_seconds The number of seconds it takes to process each container action
# TYPE engine_daemon_container_actions_seconds histogram
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.005"} 1
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.01"} 1
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.025"} 1
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.05"} 1
```

위와 같은 형식으로 나오는데 이걸 “프로메테우스 포맷”이라고 부름. 프로메테우스의 역할은 이렇게 도커 엔진이나 혹은 다른 컨테이너로부터 이러한 metric 정보를 계속해서 받아오는 역할임. 프로메테우스의 기본 포트번호는 9090번인데 접속해보면 간단한 쿼리랑 그래프를 그려주는것을 알 수 있음. targets로 이동하면 타겟들과 잘 연결되었는지를 확인해볼 수 있음. 근데 첨에는 로컬 호스트랑 연결이 잘 안되는 문제가 있었음. 이게 생각해보면 난 지금 학교 도서관 와이파이를 쓰고 있는데 여기서 할당받은 ip주소로는 외부에서 내 컴퓨터인지 확인할 방법이 없다는게 문제임. 그래서 로컬호스트라는걸 알려주기 위해서 prometheus.yml 파일을 조금 수정해주었더니 연결에 성공했음. 이런게 공부임

```docker
# 수정전
- job_name: "docker"
    metrics_path: /metrics
    static_configs:
      - targets: ["DOCKER_HOST:9323"]

#수정 후
- job_name: "docker"
    metrics_path: /metrics
    static_configs:
      - targets: ["host.docker.internal:9323"]
```

### 애플리케이션의 측정값 출력하기

잠깐 언급했던것처럼 도커 엔진의 메트릭정보 말고 각 컨테이너의 메트릭 정보도 받아올 수 있음. 근데 이를 위해서 따로 코드를 짤 필요는 없고 각 컨테이너의 런타임 환경에 맞게 프로메테우스 클라이언트 라이브러리를 추가해주면 됨. 이게 무슨말이냐면 Go 라면 promhttp 모듈을, 자바라면 micrometer 모듈을 추가해주면 된다는 말임. 
라이브러리마다 기본 메트릭이 담긴 주소가 다르긴한데 이런건 검색해서 찾으면 되니까 문제가 안됨

여기서 중요한건 라이브러리가 제공하는 기본 측정값만 받아올 수도 있지만, 서비스 비지니스 로직에 맞게 필요한 측정값을 커스텀하게 정의하고 이를 받아올 수 있도록 하는 부분임. 이 부분에대해서는 공부를 좀 해야하는데 아래의 코드를 참고하면 좋음. 각 라이브러리마다 방법은 다르지만 대강 측정하려는 값의 이름과 설명, 그리고 그걸 어떻게 받아올것인지를 정의해주면 됨. 

```docker
const prom = require("prom-client");
const accessCounter = new prom.Counter({
  name: "access_log_total",
  help: "Access Log - total log requests"
});

const clientIpGauge = new prom.Gauge({
  name: "access_client_ip_current",
  help: "Access Log - current unique IP addresses"
});
```

### 측정값 수집을 맡을 프로메테우스 컨테이너 실행하기

프로메테우스는 직접 측정값을 대상 시스템에서 받아다 수집하는 풀링 방식으로 동작함
이렇게 프로메테우스가 측정값을 수집하는 과정을 스크래핑 scraping이라고 부름
프로메테우스를 실행할때는 스크래핑 대상 엔드포인트를 설정해주어야함. 
이러한 설정 파일은 prometheus.yml 이라는 이름을 가지고 프로메테우스 컨테이너의 
/etc/prometheus/prometheus.yml 위치에 저장되어 있어야함. 

```docker
# 10초마다 스크래핑해오겠따는 말임. 
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: "image-gallery"
    metrics_path: /metrics
    static_configs:
      - targets: ["image-gallery"] # 타겟 컨테이너를 적어주고 있음

  - job_name: "iotd-api"
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ["iotd"]

  - job_name: "access-log"
    metrics_path: /metrics
    scrape_interval: 3s
    dns_sd_configs:
      - names:
          - accesslog
        type: A
        port: 80
        
  - job_name: "docker"
    metrics_path: /metrics
    static_configs:
      - targets: ["host.docker.internal:9323"]
```

job 설정은 해당 스크래핑 작업의 이름과 측정값을 수집할 엔드포인트, 대상 컨테이너를 지정하는 정보로 구성됨. 이 설정에는 2가지 유형이 있음. 

1. 하나는 정적 설정인 static_config인데 호스트명으로 단일 컨테이너를 지정함. 
2. 다른 하나는 dns_sd_config로 dns서비스의 디스커버리 기능을 통해 여러 컨테이너를 지정할 수 있고 스케일링 시에도 대상 컨테이너를 자동으로 확대할 수 있음. 

공식 프로메테우스 이미지에다가 내 상황에 맞게 필요한 내용들을 yml 파일에 담아서 추가한 내용을 나만의 이미지로 가지고 있으면 매번 설정할 필요없이 아주 편하게 사용할 수 있음!

### 측정값 시각화를 위한 그라파나 컨테이너 실행하기

이전까지 배운 프로메테우스는 메트릭값을 긁어와주는 역할을 함. 물론 프로메테우스에서도 간단한 쿼리와 그래프를 볼 수 있지만 그렇게 시각화가 뛰어나지는 않음. 그래서 그라파나를 같이 활용해서 대시보드를 만들어줌.

컨테이너 환경상에서의 프로메테우스, 그라파나 활용법을 정리하자면 다음과 같음.

1. 도커 엔진 및 각 애플리케이션 컨테이너에 메트릭을 받아올 수 있는 엔드포인트를 열어준다 
2. 프로메테우스를 활용해서 각 엔드포인트로부터 메트릭을 계속해서 받아온다
3. 그라파나가 프로메테우스로부터 메트릭들을 받아오고 이를 활용해 시각적으로 그려준다

그라파나를 설계할때 중요한 점은 복잡한 PromQL쿼리가 아니라 상황에 맞는 측정값을 골라 필요한 정보를 잘 드러내도록 시각화하는 것이다. 그라파나를 도커 이미지로 돌릴때도 데이터 소스에 대한 정보, 대시보드 설정값등을 넘겨주어야 하는데 이에 대한 형식은 다음과 같다.

```docker
# dashboard-provider.yaml
apiVersion: 1

providers:
- name: 'default'
  orgId: 1
  folder: ''
  type: file
  disableDeletion: true
  updateIntervalSeconds: 0
  options:
    path: /var/lib/grafana/dashboards
```

```docker
# dashboard-prometheus.yml
apiVersion: 1

datasources:
- name: Prometheus
  type: prometheus
  access: proxy
  url: http://prometheus:9090
  basicAuth: false
  version: 1
  editable: true
```

이번장 내용은 엄청 많은 내용을 다뤘고 어렵기도 했는데 그만큼 많이 중요한 내용이란걸 느꼈을거임. 
예시로 들어온 그라파나 이미지가 에러가 나서 잘 작동을 안하는데 그걸 실제로 작동하도록 만들어보는것도 의미가 있을것 같음!

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ab295d12-52cb-4d41-a87f-5b0d9d590689/Untitled.png)

연습으로 도커 허브에서 이미지 받아다가 만들어 그라나다 돌려서 해봤는데 잘 작동하는것 확인했음. 
참 쉽죠잉?