---
layout: post
title: "스프링 애플리케이션 EC2에 배포하기" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring, docker]
---

이 글은 "[초보를 위한 도커 안내서](https://subicura.com/2017/02/10/docker-guide-for-beginners-create-image-and-deploy.html), [백기선님 강의 - 스프링부트와 도커](https://www.youtube.com/watch?v=agbpWm2Ho_I)"를 토대로 작성한 글 입니다.

> 만약 저작권 관련 문제가 있다면 "gunjuko92@gmail.com"로 메일을 보내주시면, 바로 삭제하도록 하겠습니다.

# 스프링 애플리케이션 EC2에서 배포하기

이 글은 스프링 애플리케이션을 도커를 이용해서 AWS EC2에서 배포하는 과정을 정리한 글입니다. 저도 도커와 AWS은 처음이기 때문에 이 방법이 최선이라고 말씀드릴순 없습니다. 참고만 하시는게 좋을거 같습니다. 이 글은 아래의 글과 동영상을 보고 정리한 글입니다. 

- [초보를 위한 도커 안내서](https://subicura.com/2017/02/10/docker-guide-for-beginners-create-image-and-deploy.html)
- [백기선님 강의 - 스프링부트와 도커](https://www.youtube.com/watch?v=agbpWm2Ho_I)

시간이 되신다면 위의 글을 읽어보시는걸 강추드립니다. 

### Step 1 Dockerfile 작성

스프링 애플리케이션을 도커라이징 (Dockerizing) 해보겠습니다. 가장 맨처음 해야할 일은 Dockerfile을 작성하는 것입니다. 도커는 이미지를 만들기 위해 Dockerfile이라는 이미지 빌드용 DSL (Domain Specific Language) 파일을 사용합니다. 단순 텍스트 파일로 일반적으로 소스와 함께 관리합니다.

아래는 스프링 애플리케이션용 도커파일입니다. 아래 코드는 백기선님의 유튜브 강의인 "[스프링부트와 도커](https://www.youtube.com/watch?v=agbpWm2Ho_I)"을 참고했습니다. 강의를 보시면 더 자세하고 정확한 설명을 들으실 수 있습니다. 아래 도커파일은 프로젝트의 루트 폴더에 위치 시키면 됩니다.

``` dockerfile
FROM openjdk:8-jre
COPY  build/libs/{appName}-*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

그럼 위에서 사용된 Dockerfile의 기본적인 명령어를 살펴보겠습니다. 명령어에 대해서 더 자세히 알고 싶으시다면 [초보를 위한 도커 안내서](https://subicura.com/2017/02/10/docker-guide-for-beginners-create-image-and-deploy.html) 를 참고하기실 바랍니다.

- FROM \<image>:\<tag> : 베이스 이미지를 지정합니다. 반드시 지정해야 하며 어떤 이미지도 베이스 이미지가 될 수 있습니다. tag는 될 수 있으면 latest(기본값)보다 구체적인 버전을 지정하는게 좋습니다. 이미 만들어진 다양한 베이스 이미지는 Docker hub에서 확인할 수 있습니다.
- MAINTAINER : Dockerfile을 관리하는 사람의 이름 또는 이메일 정보를 적습니다. 빌드에 딱히 영향을 주진 않습니다.
- COPY \<src>... \<dest> : 파일이나 디렉토리를 이미지로 복사합니다. 일반적으로 소스를 복사하는데 사용합니다. target 디렉토리가 없다면 자동으로 생성합니다.
- ENTRYPOINT : docker run 실행 시 실행되는 명령어이다. 

> [공식문서](https://docs.docker.com/engine/reference/builder/)에 가면 모든 명령어를 볼 수 있다. 

### Step 2 Docker Image 생성

도커파일로 도커 이미지를 빌드하는 명령어는 아래와 같습니다.

```
docker build [OPTIONS] PATH | URL | -
```

도커 이미지를 빌드할 때 사용하는 옵션은 아래와 같습니다.

- -t (--tag) : 생성할 이미지의 이름을 지정한다.

Dockerfile이 위치하는 폴더로 이동해서 아래 명령어를 실행하면 도커 이미지가 생성됩니다.

```
docker build -t {image-name} .
```

도커 이미지가 성공적으로 생성되면 아래와 같은 결과가 출력됩니다.

```
Sending build context to Docker daemon  45.82MB
Step 1/3 : FROM openjdk:8-jre
 ---> 823dd2edb8ca
Step 2/3 : COPY  build/libs/java-*.jar app.jar
 ---> faa55f82f375
Step 3/3 : ENTRYPOINT ["java", "-jar", "app.jar"]
 ---> Running in 965623a30ca8
Removing intermediate container 965623a30ca8
 ---> b9d872501dbd
Successfully built b9d872501dbd
Successfully tagged java-qna:latest
```

> 만약에 도커 이미지가 빌드되지 않으면, "{Dockerfile이 위치한 디렉토리}/build/libs" 디렉토리에 jar 파일이 존재하는지 확인해보시길 바랍니다. 만약에 "build/libs" 디렉토리에 jar 파일이 존재하지 않으면 도커 이미지는 생성되지 않습니다.

그럼 도커 이미지가 잘 생성되었는지 아래 명령어로 확인하시면 됩니다.

```
docker images
```

그러면 아래와 같이 출력 결과로 이미지가 생성된 것을 확인하실 수 있습니다.

```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
java-qna            latest              b9d872501dbd        3 minutes ago       477MB
...
```

그럼 생성된 도커이미지를 실행시켜보겠습니다. 아래 명령어를 통해 이미지를 실행시킬 수 있습니다.

```
docker run -d -p 8080:8080 {docker-image-name}
```

### Step 3 Docker Image를 Docker hub에 올리기

![그림](https://subicura.com/assets/article_images/2017-02-10-docker-guide-for-beginners-create-image-and-deploy/docker-registry.png)

도커는 빌드한 이미지를 서버에 배포하기 위해 직접 파일을 복사하는 방법 대신 Docker Registry라는 이미지 저장소를 사용합니다. 도커 명령어를 이용하여 이미지를 Docker Registry에 푸시하고 다른 서버에서 풀 받아 사용하는 구조입니다.

도커 레지스트리 중 가장 대표적인 서비스가 도커 허브입니다.

#### Docker hub

도커 허브는 도커에서 제공하는 기본 이미지 저장소로 ubuntu, centos, debian등의 베이스 이미지와 ruby, golang, java, python 등의 공식 이미지가 저장되어 있습니다.

회원 가입은 [Docker hub](https://hub.docker.com/) 페이지에 가시면 하실 수 있습니다. (회원 가입이 안되시는 분들은 회원 가입부터 하시길 바랍니다)

회원 가입을 완료했다면, 로그인을 하면됩니다. 아래 명령어를 통해 로그인을 할 수 있습니다.

```
docker login
```

아이디와 패스워드를 입력하면 로그인이 되고, `~/.docker/config.json`에 인증정보가 저장되어 로그아웃하기 전까지 로그인 정보가 유지됩니다.

#### 이미지 태그

도커 이미지 이름은 다음과 같은 형태로 구성됩니다.

```
[Registry URL]/[사용자 ID]/[이미지명]:[tag]
```

Registry URL은 기본적으로 도커 허브를 바라보고 있고, 사용자 ID를 지정하지 않으면 기본값(library)을 사용합니다. 따라서 `ubuntu` = `library/ubuntu` = `docker.io/library/ubuntu` 는 모두 동일한 표현입니다.

도커의 `tag`명령어를 이용하여 기존에 만든 이미지에 추가로 이름을 지어줄 수 있습니다.

```
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```

앞에서 만든 이미지에 계정정보와 버전 정보를 추가하려면 아래와 같이 하면됩니다.

```
docker tag {image-name} {your-account}/{new-image-name}:{version}
```

그리고 push 명령을 이용해 도커 허브에 이미지를 전송하면 됩니다.

```
docker push {your-account}/{new-image-name}:{version}
```

이제 어디서든 `{your-account}/{new-image-name}:{version}` 이미지를 사용할 수 있습니다.

#### 배포하기

컨테이너를 사용하면 어떤 언어, 어떤 프레임워크를 쓰든 상관없이 배포 방식이 동일해지고 과정 또한 굉장히 단순해집니다. 그냥 이미지를 다운받고 컨테이너를 실행하면 끝입니다. 그럼 이제 AWS EC2에서 도커 이미지를 pull 받고, 실행시키기만 하면 됩니다.

### Step 4 AWS EC2 시작하기

EC2는 클라우드에서 가상 머신을 만들고 실행하는데 사용하는 Amazon Web Services 입니다. AWS에서는 이러한 가상 머신을 '인스턴스'라고 합니다. AWS 프리티어를 사용하면 AWS 서비스를 무료로 체험해볼 수 있습니다. EC2도 프리터어를 사용해서 무료로 체험해볼 수 있습니다.

[Linux 가상 머신 시작](https://aws.amazon.com/ko/getting-started/tutorials/launch-a-virtual-machine/?trk=gs_card) 에서 단계별로 진행하시면 쉽게 EC2 인스턴스를 실행시킬수 있습니다.

### Step 5 AWS EC2에 도커 설치 및 도커 이미지 가져오기

[Linux 가상 머신 시작](https://aws.amazon.com/ko/getting-started/tutorials/launch-a-virtual-machine/?trk=gs_card) 를 성공적으로 마쳤다면, EC2 인스턴스를 실행하고 AWS Linux 가상머신에 연결을 끝마칠수 있을겁니다. 그러면 AWS Linux 가상머신에 도커를 설치하고, 도커 이미지를 가져오는 작업을 해보도록 하겠습니다.  

#### 도커 설치

AWC EC2에 도커를 설치하기 위해선 인스턴스에 연결을 해야합니다. 이는 Step 4를 참고하시면 됩니다. 인스턴스에 연결된 후에는 아래와 같은 명령어를 차례로 실행하면 됩니다.

1. sudo yum update -y : 인스턴스에 설치한 패키지 및 패키지 캐시를 업데이트 합니다.
2. sudo yum install -y docker : 최신 Docker Community Edition 패키지를 설치합니다.
3. sudo service docker start : 도커 서비스를 시작합니다.
4. sudo usermod -a -G docker ec2-user : ec2-user가 도커 명령을 실행할 수 있도록 docker 그룹에 ec2-user를 추가합니다.
5. 로그아웃하고 다시 로그인해서 새 docker 그룹 권한을 선택합니다. 이를 위해 현재 SSH 터미널 창을 닫고 새 창에서 인스턴스를 다시 연결할 수 있습니다. 새 SSH 세션은 해당되는 docker 그룹 권한을 갖게 됩니다.
6. `ec2-user` 가 도커 명령을 실행할 수 있는지 확인합니다.
   1. docker info

> 참고
>
> - 경우에 따라서는 `ec2-user`가 도커 데몬에 액세스할 수 있는 권한을 제공하기 위해 인스턴스를 재부팅해야 할 수도 있습니다. 아래와 같은 에러가 표시될 경우 인스턴스를 재부팅해 보세요.
>
> ```
> Cannot connect to the Docker daemon. Is the docker daemon running on this host?
> ```

#### 도커 이미지 가져오기

우리는 step3 단계에서 도커 이미지를 도커 허브에 저장했습니다. 그럼 이제 EC2 인스턴스에서 도커 허브에 저장된 이미지를 가져오고 (pull) 해당 이미지를 사용해서 도커 컨테이너를 실행시켜 보겠습니다.

1. docker login : 아이디/패스워드를 입력해서 로그인합니다. 
2. docker pull \<이미지 이름>:\<태그> : 이미지 이름에 gunjuko/{image-name} 처럼 / 앞에 사용자명을 입력하면 Docker hub에서 해당 사용자가 올린 이미지를 받습니다. 공식 이미지는 사용자명이 붙지 않습니다.
3. docker images : 해당 명령어로 도커 허브에서 이미지를 성공적으로 다운받았는지 확인해봅니다.
4. docker run -d -p 8080:8080 {image} : 도커 이미지를 이용해서 컨테이너를 실행합니다.

> docker run 명령어에서 사용한 옵션에 대한 설명은 아래와 같다.
>
> - -d : detached mode 흔히 말하는 백그라운드 모드
> - -p : 호스트와 컨테이너의 포트를 연결 (포워딩)
> - -v : 호스트와 컨테이너의 디렉토리를 연결 (마운트)
> - -e : 컨테이너 내에서 사용할 환경변수 설정
> - -name : 컨테이너 이름 설정
> - -rm : 프로세스 종료시 컨테이너 자동 제거
> - -it : -i와 -t를 동시에 사용한 것으로 터미널 입력을 위한 옵션
> - -link : 컨테이너 연결

위의 과정을 마쳤으면 스프링 애플리케이션이 정상적으로 실행되었는지 확인해보길 바랍니다. 

### Step 6 포트 개방하기

보통 스프링 애플리케이션은 8080 포트를 사용합니다. 따라서 AWS 인스턴스의 8080 포트를 개방시켜줘야 합니다. 이에 대한 자세한 설명은 [AWS 포트 개방하기](https://blog.naver.com/alice_k106/220340515017) 글을 참고하길 바랍니다. (사용하시는 포트를 개방시켜주시면 됩니다)

### 결론

도커를 이용해서 스프링 애플리케이션을 EC2에 배포해봤습니다. 도커와 EC2를 거의 처음해보는 초보자이지만 꽤 쉽게 배포를 성공할 수 있었습니다. docker 이미지를 생성하고, 이미지를 도커 허브에 등록하는 작업을 직접해줘야하고, EC2 인스턴스에서 도커 이미지를 pull하고 실행시키는 작업도 수동으로 해줘야하기 때문에 꽤 귀찮기는 합니다. (이부분을 자동화 할 수 있을지...)

### 출처 및 참고 자료

- [백기선님 강의 - 스프링부트와 도커](https://www.youtube.com/watch?v=agbpWm2Ho_I)

- [초보를 위한 도커 안내서](https://subicura.com/2017/02/10/docker-guide-for-beginners-create-image-and-deploy.html)
- [도커란 무엇인가](https://aws.amazon.com/ko/docker/)
- [Linux 가상 머신 시작](https://aws.amazon.com/ko/getting-started/tutorials/launch-a-virtual-machine/?trk=gs_card)
- [AWS ECS의 도커 기본 사항](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/docker-basics.html)
- [AWS 포트 개방하기](
