# 10강. 도커 컴포즈를 이용한 여러 환경 구성

10강을 들어가기 앞서 왜 이게 유용한지 시나리오를 간략하게 설명하겠음. 이식성은 도커의 가장 핵심적인 장점이라고 배웠음. 이미지 전체를 패키징해서 사용하기때문에 환경에 따른 문제들을 모두 없앨 수 있었음. 그런데 서비스를 운영하다보면 같은 애플리케이션도 환경에 따라 동작을 달리해야하는 상황이 있음. 예를 들어 같은 서비스인데 테스트 환경이랑 개발 환경이랑 실제 운영환경에서는 조금씩 환경에 따른 차이가 생길 수 있겠지. 이런 부분을 도커 컴포즈를 활용해서 야무지게 해결하는 방법을 배워볼거임

### 도커 컴포즈로 여러 개의 애플리케이션 배포하기

우선 하나의 docker-compose.yml 파일로 여러개의 애플리케이션을 실행하는 방법을 배워볼거임. 해봤다면 알겠지만 docker-compose up 명령어로 기본.yml 파일을 한번 실행했으면 다시 명령을 내려도 이미 실행중이라면서 달라지는게 없음. 왜냐면 도커가 보기에 해당 프로젝트는 이미 실행중이기때문임. 
특정 도커 리소스가 어떤 프로젝트와 연관이 되는지를 도커가 어떻게 판단하는지가 중요함

도커 컴포즈는 도커 리소스가 어떤 애플리케이션의 일부인지 아닌지를 판정하기 위해 프로젝트 project라는 개념을 사용함. 쉽게 말하면 프로젝트 이름으로 구분을 한다는 소리인데 프로젝트 이름의 기본값은 실행하려는 yml 파일이 포함된 디렉토리의 이름임. 그리고 이제 도커 컴포즈가 도커 리소스를 만들때 이 프로젝트 이름을 접두사로 삼아서 이름을 붙여줌. 예를 들어서 디렉토리 명이 app1이고 컴포즈 파일에 정의된 서비스가 web, 볼륨이 disk라면 실제 애플리케이션이 실행될때는 app1_web_1라는 컨테이너와  app1_disk라는 볼륨이 생성됨. 컨테이너 뒤에는 번호가 붙기때문에 나중에 스케일링해도 쉽게 대응할 수 있음. 

다시 본론으로 돌아가서 같은 yml 파일로 여러 애플리케이션을 실행하려면 도커 컴포즈가 이를 서로 다른 프로젝트로 인식하도록 해주면 됨. 이를 위해서 -p 옵션을 통해 프로젝트 이름을 명시해주면 됨

```docker
# 서비스의 경우 대략 다음과 같은 형식을 따름
디렉토리명_서비스이름_스케일링번호

# docker-compose -p 옵션을 활용한다!
docker-compose -f ./docker-compose-test.yml -p test1 up -d
docker-compose -f ./docker-compose-test.yml -p test2 up -d
docker-compose -f ./docker-compose-test.yml -p test3 up -d
```

### 도커 컴포즈의 오버라이드 파일

하나의 애플리케이션을 여러 설정으로 실행해야 할 필요가 있을때 대부분의 개발자는 컴포즈 파일을 여러개 두는 방법을 사용함. 예를 들면 docker-compose-test.yml, docker-compose-dev.yml, docker-compose-prod.yml 와 같은 방법을 씀. 그러나 이런 방법은 유지 보수 측면에서 정말 나쁜 선택임. 90%이상이 중복이라서 일부 수정사항이 누락된 경우 추적하기 어려운 버그의 시작점이 될 수 있음. 즉 환경 차이 문제를 해결하기 위해서 도커를 쓰는건데 도커를 잘못 써서 환경차이 문제가 발생해버리는 눈물나는 상황에 빠져버림.
이럴때 사용할 수 있는 방법이 오버라이드 파일임. 말그대로 기본이 되는 설정을 가지고 있는 파일을 두고 각 환경에 맞게 수정해야할 사항만 따로 적어놓은 파일로 덮어쓰는거임. 이러면 기본이 수정된 경우 기본 파일만 바꾸면 되고, 각 환경별 수정사항이 생긴경우 각각 그 파일만 수정해주면 됨. 

```docker
# docker-compose.yml -> 기본파일
services:
	todo-web:
		image: dimaol/ch06-todo-list
		ports:
			- 80
		environment:
			- Database:Provider=Sqlite
		networks:
		- app-net

# docker-compose-test.yml -> 오버라이드 파일
services:
	todo-web:
		# test 환경에서는 태그 v2가 붙은 이미지를 사용하도록 덮어써주고 있음
		image: diamol/ch06-todo-list:v2 

# 실제 오버라이드를 활용해서 docker compose 사용하는방법
# config 명령을 사용하면 유효성을 체크후 덮어씌워진 내용을 출력해준다
# 이때 -f 로 넣어주는 순서가 중요한데 먼저 나오는게 베이스가 되는거고 뒤에 나오는게 덮어씌워진다는 의미이다
# 즉, 기본 -> 덮어씌울거 1 -> 덮어씌울거 2 ... 이런 순서로 사용해야한다!
docker-compose -f ./docker-compose.yml -f ./docker-compose-test.yml config

# 실행은 요로코롬하면 된다
docker-compose -f ./docker-compose.yml -f ./docker-compose-test.yml up -d
```

오버라이드 파일 사용할때 중요한점이 몇가지 있다.

1. 오버라이드할 파일에는 덮어쓸 내용을 쓰는거긴 하지만 기본적인 형식은 맞춰줘야한다. 

```docker
version: ~~
services:
	타겟 서비스:
		덮어쓸 항목: 덮어쓸 내용
```

1. -f 로 넣어주는 순서가 중요하다. 위의 코드상에도 적어두었지만 기반 → 그위에꺼1 → 그위에꺼2 .. 
이런 순서로 사용해야 실수가 없다
2. config 명령을 활용하면 전체 덮어씌워진 yml 파일을 확인할 수 있다. 특히 config명령은 유효성 검사만 해주고 실행하는건 아니니까 적극 활용하도록 하자
3. 위의 예처럼 프로젝트 이름을 명시적으로 따로 줘서 docker-compose up 해준경우 down할때도 같은 옵션을 넣어서 down해주어야 한다. 쉽게 말하면 up에 넣어준 옵션을 그대로 down에 넣어줘야 도커 컴포즈가 어떤 프로젝트를 down 시켜야하는지 이해할 수 있다. 

```docker
docker-compose up ⇒ docker-compose down
docker-compose -f ~~.yml -p test1 up ⇒ docker-compose -f ~~.yml -p test1 down
```

### 환경 변수와 비밀값을 이용해 설정 주입하기

이전까지 여러 개의 애플리케이션을 도커 네트워크를 사용해 분리하고 각 환경간의 차이를 도커 오버라이드 파일을 통해 해결하는 방법을 배웠다. 실제로는 환경 간에 애플리케이션 설정을 달리해야하는 경우도 많다. 예를 들면 운영 환경과 테스트 환경의 주소가 다르다든지, 쓰는 디비가 다르다든지 등등.. 대부분의 애플리케이션은 이런 값들을 환경 변수나 설정 파일로부터 읽어오는데 도커는 역시 이런 부분도 충실히 지원해준다. 

```docker
# 기본 베이스 파일 docker-compose.yml
services:
	todo-web:
		image: diamol/ch06-todo-list
	secrets:
	- source: todo-db-connection
		target: /app/config/secrets.json

# 이제 todo-db-connection에 각 환경별로 적절한 값을 채워주면 된다.
# ex. docker-compose-test.yml 파일

services:
	todo-web:
		ports:
		- 8089:80
		environment:
		- Database:Provider=Sqlite
		env_file:
		- ./config/logging.debug.env

secrets:
	todo-db-connection:
		file: ./config/empty.json
```

위에 사용된 3가지 프로퍼티가 대표적으로 비밀값, 환경 변수를 넘겨주는 방법이다

- environment 프로퍼티 → 컨테이너 안에서만 사용되는 환경변수를 추가해줌. 설정값을 전달하는 가장 간단한 방법
- env_file 프로퍼티 → 보통 ~~.env의 확장자를 가지는 환경변수를 정의해둔 텍스트 파일을 넘겨주는 방법
변수이름=변수값 이런식으로 한줄에 하나씩 정의된 형식을 가진 파일이다. 여러 컴포넌트에서 공유해야하는 내용이라면 environment에 매번 같은 내용 적는거보다 이처럼 파일로 넘겨주는게 훨씬 깔끔하다.
- secrets 프로퍼티 → services나 networks처럼 컴포즈 파일의 최상위 프로퍼티. 
예시에서는 베이스 docker-compose.yml 파일에서 사용하고 있는 todo-db-connection에 대한 실제 정의가 이루어지고 있다. 이를 통해 로컬 파일에 있는 비밀키 같은 것을 이미지에만 쏙 넣어줄 수 있다. 실무에서는 로컬 파일에 담아두기보다는 도커 스웜이나 쿠버네티스 클러스터에 저장하는 방법을 사용한다고 한다. 실제 값이 어디에 있든 애플리케이션이 실행될때 컨테이너 속 지정된 경로로 들어간다는 사실을 잘 알아두자~

책에서는 위의 방법 말고도 몇가지 방법을 더 소개하고 있다. 

- 호스트 컴퓨터의 환경 변수값을 활용하는 방법 → 개발이나 테스팅 환경에서나 쓸거같음
    
    ```docker
    todo-web:
    	ports:
    	- "${LOCAL_ASSIGNED_PORT}:80"
    ```
    
- 기본 .env 파일을 활용하는 방법
    
    보통 docker-compose up 이렇게만 많이 사용하는데 이때 해당 디렉토리에 .env 파일이 있으면 거기 있는 내용이 기본적인 환경변수 값으로 사용됨. 그냥 기본에 박아두고 싶은 내용들이 있다면 .env 파일을 활용하는 것도 좋음. 근데 다만 아무 옵션 안넣었을때만 사용되고 변경 추적이 좀 불편하긴함.
    

### 확장 필드로 중복 제거하기

(이건 약간 심화내용일 수 있는데 이렇게 쓰면 실무에서 도커 좀 쓴다고 소리 들을 거 같음. )
계속해서 도커 컴포즈의 강력한 기능들을 배웠음. 근데 이렇게 사용하다보면 기능이 추가되고 추가되고 하다보면 yml 파일이 너무 길어진다는 단점이 발생함. 게다가 보통 서비스별로 반복적으로 활용되는 내용들이 많음. 
예를 들면 로그와 관련된 설정이라든지 등등… 
그래서 이런 반복되는 내용들을 한번 정의해두고 사용처에서는 간단하게 가져다가 쓰는 방법이 바로 확장 필드임. 

다음은 책의 예제임. 확장 필드 정의하는 블록에는 관습적으로 x로 시작하는 이름을 붙임. 
& 다음에 나오는게 지칭하는 이름이라고 생각하면 됨. 
사용하고 싶은 부분에는 <<: * 확장필드이름  의 형식으로 사용해주면 됨~

아래 예시에서 logging 필드는 logging 프로퍼티를 포함하고 있어서 기존 logging 프로퍼티 자리에 그냥 바로 넣어줄 수 있는데 labels 필드의 경우 labels 프로퍼티를 포함하지 않고 있어서 단독으로 못쓰고 labels: 프로퍼티 밑에 들어가있는 모습을 볼 수 있음. 역시나 상황에 맞게 사용해주면 됨.

```docker
x-labels: &logging
  logging:  
    options:
      max-size: '100m'
      max-file: '10'

x-labels: &labels
  app-name: image-gallery

services:
  accesslog:
    <<: *logging
    labels:
      <<: *labels

  iotd:
    ports:
      - 8080:80
    <<: *logging
    labels:
      <<: *labels
      public: api
```

### 도커를 이용한 설정 워크플로우 이해하기

이번 장에서는 도커 컴포즈를 이용해 서로 다른 환경을 정의하는 방법을 다음의 3가지 핵심 영역에서 살펴봤음

- 애플리케이션 구성 요소의 조합
    
    모든 환경에서 전체 스택을 실행할 필요는 없음. 오버라이드 파일을 사용하면 공통된 서비스는 그대로 두고 환경마다 필요한 서비스를 달리하는 설정을 깔끔하게 작성할 수 있었음. 
    
- 컨테이너 설정
    
    각 환경의 상황과 요구사항에 맞춰 설정을 바꿀 필요가 있었음. 대표적으로는 포트가 충돌하지 않게 한다든지 볼륨 경로를 환경에 맞게 바꿔준다든지 등등이 있었음. 오버라이드 파일과 도커 네트워크로 각 애플리케이션을 분리하는 방법을 적절히 활용하면 모든 요구사항을 만족시키면서 단일 서버에 여러 개의 애플리케이션을 실행할 수 있음
    
- 애플리케이션 설정
    
    환경별로 컨테이너 내부 동작이 달라야하는 경우가 있음. 애플리케이션이 생성하는 로그 수준이나, 사용하는 캐시의 크기를 달리할 수도 있고, 아예 특정 기능을 껐다 켰다 할 수도 있음. 오버라이드 파일과 환경 파일, 비밀값을 이용해서 상황에 맞는 설정값을 컨테이너에 주입할 수 있었음. 
    

### 연습문제

연습문제는 이 모든걸 실습해볼 수 있는 좋은 문제였음. 모든 프로젝트마다 세부 사항은 달라지겠지만, 큰 틀에서 책의 예시처럼 활용하는 방법을 꼭 익혀두면 좋을 것 같음!