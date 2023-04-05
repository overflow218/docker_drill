# 7장. 도커 컴포즈로 분산 어플리케이션 실행하기

이전까지 Dockerfile을 통해 이미지를 빌드하고 도커를 활용하는 방법을 배웠음. 이전에 여러 컨테이너를 띄워서 같이 동작하는 어플리케이션을 실행해보기도 했음. 근데 이게 최선일까? 

실제 운영하는 서비스라면 매번 이렇게 수동으로 순서 맞춰서 옵션 넣어주고 하나씩 컨테이너를 실행하는건 아주 위험한 행동임. 여러 컨테이너의 실행 내용을 담아서 하나로 관리해주는 방법이 있으면 참 좋을것 같음. 
→ 그게 바로 도커 컴포즈! 

### 도커 컴포즈 파일의 구조

```docker
version: '3.7'

services:
	todo-web:
		image: dimol/ch06-todo-list
		ports:
			- "8020:80"
		networks:
			- app-net

networks:
	app-net:
		external:
			name: nat
```

뭔가 복잡해보이지만 또 뭔가 알것 같은 느낌의 형식임. 
일단 도커 컴포즈 파일은 기본적으로 docker-compose.yml 이라는 이름과 yml 형식으로 작성된 문서임. 
크게 3가지 중요한 파트가 있는데 version, services, networks 임. 

version → 사용하는 docker compose 파일의 형식을 의미함. 버전마다 사용가능한 명령어가 달라서 꼭 명시해줘야함
services → 실행할 서비스 단위로 내용을 적어줌. 컨테이너 단위가 아니고 서비스 단위인게 핵심임. 왜냐면 같은 이미지로 하나의 서비스를 여러 컨테이너에서 돌릴 수 있기 때문임
서비스 내부의 항목들은 docker container로 실행할 수 있는 명령들에 대해서 [명령 옵션]: [값]의 형식을 띈다고 생각하면 됨. 그래서 어떤 기능 쓸건지 찾아서 검색해보면 어떤 형식으로 넣어야하는지 자세하게 나와있음
network → 각 서비스(컨테이너들)끼리 소통할 수 있도록 도커 네트워크를 지정해주는 부분임. 
app-net 네트워크를 쓰라고 적어주었는데 다시 external 옵션으로 nat를 넣어주고 있음. 이게 무슨말이냐면 app-net 네트워크를 쓰는데 외부에 이미 nat 네트워크가 있기때문에 app-net 네트워크를 따로 만들 필요없이 nat를 쓰라는 의미임. 참고로 nat는 도커 처음 설치될때 자동으로 만들어지는 도커 네트워크인데 윈도위의 경우 저걸 꼭 사용해줘야한다고함. 

services의 각 항목들은 실행되는 컨테이너들의 이름이 되는데, 저 이름으로 도커 내부의 dns에 질의를 하고 서로 통신할 수 있는 주소를 받아올 수 있음. 

### 도커 컴포즈를 사용해 여러 컨테이너로 구성된 애플리케이션 실행하기

```docker
version: '3.7'

services:

  accesslog:
    image: diamol/ch04-access-log
    networks:
      - app-net

  iotd:
    image: diamol/ch04-image-of-the-day
    ports:
      - "80"
    networks:
      - app-net

  image-gallery:
    image: diamol/ch04-image-gallery
    ports:
      - "8010:80" 
    depends_on:
      - accesslog
      - iotd
    networks:
      - app-net

networks:
  app-net:
    external:
      name: nat
```

위의 컴포즈파일이 여러 컨테이너로 구성된 전형적인 예시임.  볼만한 부분은 depends_on을 통해서 특정 컨테이너는 다른 컨테이너들이 실행되고 나서야 실행해야한다는 이런 의존적인 순서도 지정해줄 수 있음.
도커 컴포즈 관련 기본 명령들은 다음과 같음

```docker
# 도커 컴포즈 실행
docker-compose up - d
# 아래와 같이 특정 서비스의 컨테이너만 스케일업 해줄 수도 있음
docker-compose up -d --scale itod=3

# 도커 컴포즈 정지
docker-compose stop

# 도커 컴포즈 다시 실행
docker-compose start

# 도커 컴포즈 종료 (컨테이너 날리기)
docker-compose down

#기타 자세한 옵션과 다른 명령들은 docker-compose 만 입력하면 헬프 나옴
```

나중에 회사같은데서 일할때 좋을지도 모르는거. 도커 컴포즈 안에서의 의존관계를 이미지로 바꿔주는 레포

[https://github.com/pmsipilot/docker-compose-viz](https://github.com/pmsipilot/docker-compose-viz)

여기서 알아두어야할 점이 하나있는게 도커 엔진은 각 컨테이너들이 어떤 관계를 맺고 있는지 전혀 알지 못함. 
그냥 하라는 대로 명령만 실행해줄 뿐임. 컨테이너 간의 관계를 알고 있는건 도커 컴포즈밖에 없음. 도커 컴포즈는 yml 파일에 적혀있는 대로만 실행할 뿐임. 그래서 위에서 —scale 옵션으로 스케일링해줬던거나 따로 docker 명령으로 임의로 수정해주게 되면 컴포즈 파일 내의 정의와 실제 실행되고 있는 컨테이너 환경에서의 상황이 서로 
다른 경우가 발생할 수 있음. 이는 실제 운영에서는 매우 위험한 일이기 때문에 무조건 수정해야할 사항이 있거나 한다면 도커 컴포즈 파일에 기록을 해두도록 하자!

### 도커 컨테이너 간의 통신

이전에도 간단히 말했었지만 컨테이너는 생성되면 가상의 아이피주소를 할당받음. 근데 서로 다른 컨테이너들끼리는 어떻게 통신하는 걸까? 이를 위해서 도커 네트워크 개념을 도입해서 서로 같은 네트워크(서브넷 같은거)로 묶어줬지. 이 안에서 서로 주소를 알아내는건 도커 내에 DNS가 있어서 이걸 통해서 알아낸다고 함. 만약 알아야하는 주소가 외부면 이제 로컬 호스트 운영체제의 도움을 받아서 외부 주소도 알아낸다고 함!

```docker
docker exec -it [컨테이너 이름] sh 
nslookup [알고싶은 도메인 이름] 
```

위와 같이 컨테이너 직접 접속해서 nslookup 명령으로 컨테이너가 특정 도메인의 주소를 가지고 있는지 확인해볼 수 있음

### 도커 컴포즈로 애플리케이션 설정값 지정하기

이건 앞에서의 연장선상인데 이제 docker container run 할때 환경변수를 심어준다든지 여러 정보를 같이넘겨줄 수 있었는데 그걸 컴포즈 파일에서 하는 방법을 배워볼거임

```
version: "3.7"

services:
  todo-db:
    image: diamol/postgres:11.5
    restart: unless-stopped
    environment:
      - PGDATA=/var/lib/postgresql/data
    volumes:
      - type: bind
        source: /data/postgres
        target: /var/lib/postgresql/data
    networks:
      - app-net

  todo-web:
    image: diamol/ch06-todo-list
    restart: unless-stopped
    ports:
      - "8050:80"
    environment:
      - Database:Provider=Postgres
    depends_on:
      - todo-db
    secrets:
      - source: postgres-connection
        target: /app/config/secrets.json

secrets:
  postgres-connection:
    file: postgres-connection.json

networks:
  app-net:
    external:
      name: nat
```

위에서 보이는 것과 같이 environment: 를 통해 환경변수를 심어줄 수도 있고, volumes: 를 통해 바인드 마운트해주고 있는것을 볼 수 있음. 디비 컨테이너를 돌리는 경우라면 저렇게 로컬에 마운트를 해줘야 컨테이너가 내려갔다가 다시 올라와도 데이터가 사라지지 않는다는 것을 지난번에 배웠음. 

여기서 좀 볼만한 부분은 secrets부분임. 실제 서비스를 운영하다보면 디비접속 키라든지 여러 비밀키를 관리해야하는 상황이 발생함. 보통은 쿠버네티스나 도커 스웜같은 컨테이너 플랫폼을 써서 관리해주는데 우리는 지금 그런게 없으니까 그냥 로컬의 파일을 비밀로 넘겨주는 방법을 보여주고 있음.

### 도커 컴포즈도 만능은 아니다

이렇게 보면 도커 컴포즈가 모든걸 다 할 수 있는 거 같은데 아쉽지만 한계가 있음. 
가장 큰 한계는 도커 컴포즈로 실행한 컨테이너들이 지속적으로 정의된 상태를 유지하도록 할 수 없다는 점임. 
도커 컴포즈는 그냥 맨 처음에 실행하는것만 보장해줄 수 있음. 저런 부분에서 도커 스웜이나 쿠버네티스를 활용해야한다고 함! 다만 실제 프로덕션에 올릴때나 그렇고 로컬에서 개발하거나 테스팅할때는 뭐 아주 훌륭한 도구임!

### 실습문제

여러 옵션을 사용해보는 거였음

- 바인드 마운트해서 디비 컨테이너의 정보가 유실되지 않도록 해주기
- 특정 포트 열어주기
- 호스트 컴퓨터가 재부팅되거나 도커 엔진이 다시 시작될때 애플리케이션 컨테이너도 재시작되도록 해주기

맨 마지막에 재시작은 restart 옵션을 통해서 할 수 있음. 위의 코드를 참조하자!