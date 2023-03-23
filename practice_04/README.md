# 4장. 애플리케이션 소스 코드에서 도커 이미지까지

이번장에서는 정말 엄청난 기술을 배웠음. 바로 도커를 활용해서 빌드서버 없이 최종 버전을 빌드하는 방법임. 예를 들어 빌드서버를 두고 자바 애플리케이션을 빌드한다고 해보자. 일단 자바같은 컴파일 언어는 먼저 컴파일 해줘야겠죠? maven, jdk 이런거 다 필요해. 그래서 빌드해서 패키지 만들고 다시 패키지를 실행하는 식으로 가는거잖아. 근데 빌드 서버랑 일반 개발하는 팀원들 사이에 버전이 바뀐다든지 그런 일이 생긴다면? 아니면 신입이 들어왔는데 필요한 모든 파일들을 버전을 맞춰서 다운로드해줘야하는 상황이라면? 아주 귀찮고 복잡한 일이 될거임.

우리는 이런 상황에서 도커를 활용함으로써 이러한 빌드 툴체인을 한번에 패키징해서 어떤 환경에서든 도커만 실행할 수 있으면 이미지를 빌드하고 빌드한 이미지를 받아서 컨테이너를 실행할 수 있도록 해볼거임.

그걸 위해서 여기서는 multi stage Dockerfile 스크립트를 작성하는 방법을 배울거임. 지난시간에 Dockerfile 스크립트 작성하는 걸 배웠음. 그때는 하나의 스테이지만 있었다면 이번엔 스테이지가 여러개임. 스테이지를 나누는건 이런 느낌임. 사용자는 결국에 실행 프로그램만 가지고 실행할 수 있으면 됨. 근데 이 실행프로그램을 만들기 위해 필요한 온갖 패키지 관련 코드들, 라이브러리 관련 코드 등등을 다 다운받아야 한다면 무겁잖아. 실행 프로그램만 간단하게 실행하고 싶은거임. 이런 느낌으로 스테이지를 나눠서 예를 들면 첫번째 스테이지에서는 테스트를 진행하고 두번째 스테이지에서는 필요한 패키지들을 다운받아서 파일을 컴파일하고(컴파일이 필요한 경우) 세번째 스테이지에서는 최종 결과만 복사해서 각종 환경변수나 필요한 세팅과 함께 빌드해주면 우리는 마지막 이미지만 받아오면 되니까 아주 편할거임. 그리고 각 단계가 순차적으로 이루어져야 하기때문에 앞단계에서 실패하면 (예를 들어 테스트에 걸린다든지, 빌드가 실패한다든지) 자동으로 깨지기 때문에 오류 발생을 확인하는것도 쉬움.

요약하자면 이거 왜 안씀? 이 되는거임.

- 더 알아가면 좋은점은 multistage Dockerfile을 잘 활용하면 이미지 파일의 크기를 획기적으로 줄일 수 있음. 이게 용량을 적게 먹는다는 것도 있지만, 그만큼 코드가 간소해지는거라서 의존성 관련 보안 공격을 원천차단하는 효과도 있다고 함.

이번 시간에는 크게 3가지 종류의 애플리케이션을 빌드하는 연습을 해봤음.

1.  자바 → 자바, .NET Core, Erlnag 등 컴파일이 필요한 언어라면 모두 적용가능한 패턴
2.  Node js → Python, PHP, 루비 등 여타 스크립트 언어에 그대로 적용가능한 패턴
3.  Go → Rust, Swif 등 네이티브 바이너리로 컴파일되는 언어에 적용가능한 패턴

### Docker network

들어가기에 앞서 docker network 명령을 간단히 배워볼거임. 이제 실습할 내용은 노드로 로그 서버를 돌리고, 자바로 이미지 서버를 돌리고, 고로 웹 서버를 돌리는 방식임. 이렇게 여러 컨테이너를 돌릴건데 컨테이너 간에 통신을 할 수 있어야함. 이럴때 사용하는게 docker network 명령임. 이걸로 네트워크를 만들어두고 docker run 할때 네트워크를 지정해주면 됨. 언제나 당부하고 싶은건 무슨 기능이 있는지 궁금하거나 새로운 명령어를 배울때는 docker -help를 십분 활용하자!

```docker
# 도커 네트워크 만들기
docker network create [네트워크 이름]

# 컨테이너 실행시 네트워크를 지정해준다
docker container run --network [네트워크 이름] [컨테이너 이름]

# 이미 실행중인 컨테이너면 이 방법도 가능
docker network connect [options] [네트워크 이름] [컨테이너 이름]
```

### 자바 소스코드 빌드

```docker
# Dockerfile
# AS 명령어를 통해 특정 stage에 이름을 붙여줄 수 있음
FROM hj/maven AS builder

# maven 빌드에 필요한 pom.xml 파일을 호스트에서 복사해오고 실행해주고 있음
# RUN 명령어는 빌드 중 컨테이너 안에서 명령을 실행하고 그 결과를 이미지 레이어에 저장하는 역할을 함
WORKDIR /usr/src/iotd
COPY pom.xml .
RUN mvn -B dependency:go-offline

# app
FROM hj/openjdk

WORKDIR /app
# 위의 builder stage에서 빌드한 jar 파일만 가져와서
#이를 실행하라고 ENTRYPOINT에서 명령하고 있음
COPY --from=builder /usr/src/iotd/target/iotd-service-0.1.0.jar .

# EXPOSE 명령어는 사실 기능이 있는거 아니고 80번 포트를 열거라고 개발자들 사이에 소통하는거임
# 진짜 포트를 열려면 docker container run -p 명령을 통해 포트 바인딩을 해주어야함.
EXPOSE 80
ENTRYPOINT ["java", "-jar", "/app/iotd-service-0.1.0.jar"]
```

이렇게 하면 최종 마지막 이미지에는 빌드에 필요한 maven 관련 내용이 들어가지 않기 때문에 훨씬 컴팩트해진 다는 장점이 있음. 그리고 이전 단계의 내용은 기본적으로 복사되지 않기때문에 필요한 내용이 있다면 COPY 명령어를 통해 명시적으로 복사해와야함. 각 빌드 단계는 격리되어 있다는걸 이해하는게 아주 중요함. 이걸 이해해야 필요에 맞게 여러 스테이지로 분리하고, 최대한 이미지 레이어 캐싱을 활용할 수 있도록 Dockerfile을 작성할 수 있음.

### Node.js 소스코드 빌드

```docker
FROM hj/node AS builder

WORKDIR /src
COPY src/package.json .

RUN npm install

# app
FROM hj/node

EXPOSE 80
CMD ["node", "server.js"]

WORKDIR /app
COPY --from=builder /src/node_modules/ /app/node_modules/
COPY src/.
```

node도 마찬가지다. node는 스크립트 언어라 컴파일이 필요없지만 npm을 통해 필요한 패키지를 다운받아줘야한다. 위에서는 그 과정을 두개로 분리해서 아래 최종 이미지에서는 위에서 받은 패키지를 node_modules에 넣어줘서 실행하고 있다. 최종 이미지에는 npm 관련 코드가 들어갈 필요가 없다.

### Go 소스코드 빌드

```docker
FROM hj/golang AS builder

COPY main.go .
RUN go build -o /server

#app
FROM diamol/base

ENV IMAGE_API_URL="!!!" \
		ACCESS_API_URL="@@@"
CMD ["/web/server"]

WORKDIR web
COPY index.html
COPY --from-builder /server .
RUN chmod +x server
```

Go 는 네이티브 바이너리로 컴파일된다. 그래서 컴파일된 프로그램을 실행만 하면 된다
위에선ㄴ 컴파일 해주고 아래에서는 실행하기 위해 권한을 수정해주는걸 볼 수 있다. 그리고 눈치챘을수도 있는데 최대한 이미지 레이어 캐싱을 활용하기 위해서 수정이 빈번하지 않는 instruction들은 최대한 위에 배치하고 자주 바뀌는 내용들을 아래로 빼둔것을 확인할 수 있다.

### RUN vs. CMD vs. ENTRYPOINT

사실 run의 경우는 목적이 빌드중에 컨테이너에서 실행하는 거라 명확하다고 생각했다.

그런데 CMD와 ENTRYPOINT는 정확한 차이를 모르겠어서 검색해봤다.
검색해보니 ENTRYPOINT는 컨테이너가 실행될때 작동할 명령인데 컨테이너를 띄울때 항상 같이 실행되어야 하는 커맨드를 넘겨줄때 사용한다. 해당 커맨드가 죽으면 도커 컨테이너도 같이 죽는다

반대로 CMD는 컨테이너 명령시 실행할 명령어를 넘겨주는거긴 하지만 위와 같은 성질은 없다.
보통 활용하는게 ENTRY POINT로 실행할꺼 띄워놓고 CMD로 인자를 넘겨주는 식으로 활용한다고 하는데 그럴꺼면 그냥 ENTRY POINT에 한번에 넣어주면 되는거 아닌가 싶긴하다 암튼 차이가 있으니 알아두자

### 마치며

이 장은 마지막 연습문제가 진또배기다 하나의 스테이지로 작성된 Dockerfile을 상황에 맞게 여러 스테이지로 나눔으로써 전체 이미지 용량을 800mb → 20mb로 줄이는 방법을 보여준다. 참고하고 연습하도록 하자~