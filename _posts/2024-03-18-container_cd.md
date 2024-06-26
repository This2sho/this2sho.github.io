---
layout: single
title: "컨테이너 구성 및 무중단 배포"
categories: project
tag: [주차의상상은현실이된다, 도커, 컨테이너, 무중단배포]
typora-root-url: ../
toc: true
author_profile: false
---

“주차의 상상은 현실이된다” 프로젝트를 진행하면서 컨테이너를 구성하고 공부한 내용을 기록하고자 한다.

## 상황

2024년 3월 9일 기준 기존에 AWS EC2 프리티어에서 사용할 수 있던 t4g.small이 유료로 변경되고 t2.micro을 사용하게 됨으로서 메모리가 2gib에서 1gib로 줄어들었다.

- 추가 변경사항(IPv4)

  무료로 제공되던 IPv4 주소에 새로운 요금이 도입되었다. 그래서 프리티어도 유료로 사용해야하나? 아니면 IPv6를 사용해야하나 궁금해서 문의한 결과 다행히 IPv4는 무료로 사용할 수 있다는 답변을 받았다.

  `EC2용 AWS 프리 티어에는 2024년 2월 1일부터 처음 12개월 동안 매월 750시간의 퍼블릭 IPv4 주소 사용이 포함됩니다. 다만, EC2를 제외한 서비스에서는 IPv4에 대한 요금이 프리 티어 기간이라도 청구됩니다.`

메모리가 반으로 줄어들면서 메모리 초과 및 특정 프로세스의 메모리 사용량이 다른 프로세스에 영향을 줄 것이라 판단했고  
트래픽이 증가해 EC2를 하나 더 사용해야 한다면 컨테이너를 사용하는게 메모리 제한이나 새로운 환경에 배포하기 편할거라 판단하였다.

------

## 컨테이너 구성

이미지를 만들기 전 계획한 흐름은 다음과 같다.

1. main 브랜치에 소스코드가 push 된다.
2. 배포용 Github Action이 실행된다.
   1. 해당 소스코드를 build 한다.
   2. 만들어진 jar파일을 가지고 이미지를 생성한다.
   3. 생성된 이미지를 도커 허브에 전송한다.
   4. 서버 EC2에 접속하여 이미지를 pull 받는다.
   5. 해당 이미지로 컨테이너를 실행한다.

### **DB**

**Dokcerfile**

```docker
FROM mysql:latest

COPY init.sql /docker-entrypoint-initdb.d/
COPY my-config.cnf /etc/mysql/conf.d/my-config.cnf

ENV MYSQL_USER={유저}
ENV MYSQL_PASSWORD={비밀번호}
ENV MYSQL_ROOT_PASSWORD={루트_비밀번호}
```

- 베이스 이미지는 공간 좌표 기능을 사용하기 위해 현재 mysql 최신 버전을 사용하였다.
- 위의 init.sql에는 초기 데이터베이스를 생성하는 쿼리가 작성되어 있다.
- my-config.cnf는 아래 파일을 사용하면 된다.
- docker build -t {도커허브계정명/DB} .(현재 폴더에 도커파일이 있다는 뜻) 명령어로 이미지를 생성한다
- docker image push {도커허브계정명/DB} 으로 본인 또는 팀의 도커 허브에 push 한다.



**my-config.cnf**

```bash
[mysqld]
character_set_server=utf8mb4
collation_server=utf8mb4_unicode_ci

[client]
default-character-set=utf8mb4
```

- mysql에서 한글을 사용하기 위한 세팅이다.
- /etc/mysql/conf.d 복사하면 mysql이 시작되면서 설정 파일을 읽고 실행한다.



### **Spring**

```docker
FROM amazoncorretto:17-alpine

COPY build/libs/parking-0.0.1-SNAPSHOT.jar app.jar

CMD ["java", "-jar", "app.jar"]
```

- 베이스 이미지를 개발에 사용하는 자바 17 amazoncorretto 벤더를 사용했고 alpine은 경량화 os를 의미한다
- gradle을 사용하지 않는 이유는 깃헙 액션에서 빌드하고 jar 파일만 가지고 이미지를 생성하기 위해서이다.
- 이 파일은 프로젝트 소스 코드 최상단에 두면 된다. (깃헙 액션에서 사용할 예정)

추가로 Redis 컨테이너를 사용하는데 이는 redis:7.2.4-alpine 이미지를 사용하였다.



**docker-compose.yml** (나눠서 설명하겠음)

```yaml
version: '3.8'

x-environment:  &common_environment
  environment:
    - PROFILE=prod
    - MAIL_HOST=smtp.gmail.com
    - MAIL_PORT=587
    - MAIL_USERNAME=
    - MAIL_PASSWORD=

    - REDIS_HOST=pct-redis
    - REDIS_PORT=6379

    - DB_URL=jdbc:mysql://pct-mysql/parkingcomestrue
    - DB_USERNAME=
    - DB_PASSWORD=

    - ORIGIN=
		
    - KAKAO_API_KEY=
    - SEOUL_API_KEY=
    - PUSAN_API_KEY=
```

- 맨 위는 버전을 나타내고 도커 컴포즈에서 환경 변수를 지정하기 위해서 3.8버전을 사용했다.
- 아래에 x-environment는 공통적으로 사용되는 것들을 환경변수로 지정해놓았다. (이 값들은 Spring boot 설정 파일에 들어간다)
- DB URL 같은 경우는 도커 컴포즈를 사용하여 컨테이너를 사용하는 경우 도커 네트워크로 연결되어 서비스 명이 도메인이 된다.

``` yaml
services:
  pct-mysql:
    image: {도커허브계정명/DB}
    volumes:
      - mysql-volume:/var/lib/mysql
    deploy:
      resources:
        limits:
          memory: 512M
    restart: on-failure
```

- 여기서 부터는 서비스들 즉, 실행될 컨테이너에 대한 설정이다.
- 이 부분은 아까 도커파일로 생성한 이미지를 도커허브 계정에 push 한후 가져오는 것이다.
- 컨테이너는 상태를 저장하지 않기 때문에 데이터베이스같은 영속성을 가지는 친구를 컨테이너로 사용하려면 volume을 지정해줘야한다.
- mysql-volume:/var/lib/mysql 이 부분은 mysql-volume을 해당 컨테이너의 /var/lib/mysql 에 선언하는 것인데 볼륨은 해당 도커 컴포즈 제일 아래에서 해두었다.
- 메모리 제한을 처음에는 256M으로 지정했는데 이렇게 하면 OOM이 발생하는 문제가 있다. 그래서 현재 512M로 지정해두었다.
- `restart: on-failure`를 사용하면 컨테이너가 예기치 못하게 종료되는 경우 재시작하게된다.

```yaml
  pct-redis:
    image: redis:7.2.4-alpine
    deploy:
      resources:
        limits:
          memory: 256M
    restart: on-failure
```

- 레디스의 경우 아까 말했듯이 redis:7.2.4-alpine 이미지를 사용했다.

```yaml
  pct-backend-1:
    image: {도커허브계정명/Spring}
    << : *common_environment

    depends_on:
      - pct-redis
      - pct-mysql
    ports:
      - "8080:8080"
    deploy:
      resources:
        limits:
          memory: 512M
    restart: on-failure
 
   pct-backend-2:
    image: {도커허브계정명/Spring}
    << : *common_environment

    depends_on:
      - pct-redis
      - pct-mysql
    ports:
      - "8081:8080"
    deploy:
      resources:
        limits:
          memory: 512M
    restart: on-failure

volumes:
  mysql-volume:
  
```

- 이 부분은 위에서 도커 파일로 생성한 Spring 이미지를 도커허브 계정에 push 한후 가져오는 것이다.
- 1과 2로 나눈 것은 무중단 배포를 실행하기 위해서 포트만 다른 두개의 서비스를 두었다. (실제 실행은 하나로 할 예정)
- spring의 경우 Redis와 Mysql이 선행되야한다.
- 포트는 호스트의 포트가 앞에 컨테이너 내부의 포트가 뒤에 설정한다.(컨테이너의 8080포트 즉, 톰캣을 의미)
- mysql, redis, spring의 메모리 최대 사용량을 다합치면 1G가 넘는데 이는 이후 스왑메모리를 설정해야한다.

위와 같이 설정 하고 `docker-compose up` 명령으로 잘 실행되는지 확인하면된다.

------

## CD(무중단 배포)

CD의 경우 Github Action과 Jenkins를 후보로 두었는데 Jenkins의 경우 배포를 위해 서버가 필요함으로 EC2를 하나 더 만들지 않고 **Github Action**을 사용하기로 결정했다.

Github Action을 정의 하기전에 우선 EC2에 다음과 같은 작업을 해줘야한다.

**스왑 메모리 설정**

메모리 1gib로 spring, db, redis를 다 돌리면 메모리 부족이 발생한다.

AWS의 권장 스왑 공간을 확인해보면 메모리가 1gib인 t2.micro는 2gib로 스왑 메모리를 설정하는게 권장된다. (너무 많이 설정하면 디스크 I/O 가 증가한다)

![4-2](/images/2024-03-18-4/4-2.png)

```bash
# swap 메모리 할당 (한번에 읽거나 쓰는 데이터의 크기를 128M로 16개 즉, 2gib)
$ sudo dd if=/dev/zero of=/swapfile bs=128M count=16

# 스왑 파일에 대한 읽기 및 쓰기 권한 설정
$ sudo chmod 600 /swapfile

# 스왑영역 설정
$ sudo mkswap /swapfile

# 스왑 공간에 스왑 파일을 추가하여 사용할 수 있도록
$ sudo swapon /swapfile

# 성공했는지 확인
$ sudo swapon -s

# 편집기에 파일을 열어서 아래 코드를 파일 끝에 복사해야한다.
$ sudo vi /etc/fstab

/swapfile swap swap defaults 0 0

# 이후 free -h 명령어로 스왑 메모리가 잘 설정되었는지 확인.
```

**`service-url.inc` 파일 생성**

```bash
set $service_url <http://127.0.0.1:8080>;
```

- 스프링 컨테이너 주소
- 배포시마다 위 포트를 변경

**`default.conf` 파일 생성**

```bash
server {
        listen 80;
        server_name {서버 주소};
        include /etc/nginx/conf.d/service-url.inc;

        location / {
                proxy_pass $service_url;
        }
}
```

- nginx를 통해 서버 주소로 들어오는 부분을 `$service_url` 로 변환해주는 역할을 한다.

**`deploy.sh` 파일 생성**

```bash
echo "배포 시작!"
echo "최신 컨테이너 가져오는중"
docker pull {도커허브계정명/WAS}

if docker ps | grep -q backend-1; then
    echo "컨테이너가 8080 포트 사용중.."
    NEW_CONTAINER=pct-backend-2
    OLD_CONTAINER=pct-backend-1
    NEW_PORT=8081
    OLD_PORT=8080
elif docker ps | grep -q backend-2; then
    echo "컨테이너가 8081 포트 사용중.."
    NEW_CONTAINER=pct-backend-1
    OLD_CONTAINER=pct-backend-2
    NEW_PORT=8080
    OLD_PORT=8081
else
    echo "컨테이너 새로 시작중.."
    NEW_PORT=8080
    NEW_CONTAINER=pct-backend-1
    docker-compose up -d pct-mysql
    docker-compose up -d pct-redis
    sleep 5
fi

echo "${NEW_PORT} 컨테이너 올리는중.."
docker-compose up -d ${NEW_CONTAINER}

echo "NGINX PORT ${NEW_PORT}로 전환중.."
echo "set \\$service_url <http://127.0.0.1>:${NEW_PORT};" | sudo tee /etc/nginx/conf.d/service-url.inc

echo "NGINX 재시작 중.."
sudo service nginx reload
sleep 5

if [ -n "${OLD_CONTAINER}" ]; then
  echo "${OLD_PORT} 컨테이너 내리는중.."
  docker-compose down ${OLD_CONTAINER}
fi

echo "배포 완료!"
```

- 위에서 명시적으로 `docker pull {도커허브계정명/WAS}` 을 사용하는 이유는 latest 태그를 사용하여 서버에 존재하는 이미지를 사용할 수 있기 때문이다

서버에 파일을 다 작성했으면 이제 **Github Action을 작성**하면 된다. (나눠서 설명하겠음)

```yaml
name: Backend CD

on:
  pull_request:
    types: [closed]

jobs:
  build-and-push:
    if: |
      github.event.pull_request.merged == true &&
      github.event.pull_request.base.ref == 'main' &&
      contains(github.event.pull_request.labels.*.name, '🎅🏼deploy')

    runs-on: ubuntu-latest  

    steps:
    - name: Checkout Repository
      uses: actions/checkout@v2  

    - name: JDK 17을 설치합니다
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'corretto'

    - name: Install redis
      run: sudo apt-get install -y redis-tools redis-server

    - name: Verify that redis is up
      run: redis-cli ping

    - name: Gradle 명령 실행을 위한 권한을 부여합니다
      run: chmod +x gradlew

    - name: Gradle build를 수행합니다
      run: ./gradlew build

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1  

    - name: 도커 로그인
      uses: docker/login-action@v1
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}  
        password: ${{ secrets.DOCKERHUB_TOKEN }}     

    - name: 도커 이미지 build 후 push
      uses: docker/build-push-action@v2
      with:
        context: .
        file: Dockerfile  
        push: true  
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/pct-backend:${{ github.sha }}
        platforms: linux/amd64
       
```

- PR이 닫혔을 때 수행되고 아래 if 문을 보면  🎅🏼deploy 태그가 달린 PR이 main에 머지됐을 때 동작하는걸 알 수 있다.
- 컨테이너 환경은 ubuntu-latest로 동작한다.
- 이후 동작은 현재 PR의 소스 코드를 가져와서 필요한 환경을 설치하고 소스코드를 build한다.
- build 후, Spring 이미지를 만드는 도커파일을 소스코드 최상단에 두어 빌드하고 본인 또는 팀의 도커허브에 push 한다.

```yaml
 	deploy-to-ec2:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
    - name: Install SSH
      run: sudo apt update && sudo apt install -y openssh-client

    - name: EC2 접속 및 배포 진행
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.EC2_HOST }}
        username: ${{ secrets.EC2_USERNAME }}
        key: ${{ secrets.EC2_PRIVATE_KEY }}
        script: |
          ./deploy.sh
```

- 이 부분은 서버 EC2에 접속하여 위에서 생성했던 deploy.sh 를 실행하는 작업이다.

  

```yaml
	notify-slack:
    runs-on: ubuntu-latest
    needs: deploy-to-ec2
    
    steps:
      - name: 배포 성공시 Slack으로 알립니다
        uses: 8398a7/action-slack@v3
        with:
          author_name: 배포 성공
          status: ${{ job.status }}
          fields: repo,message,commit,author,action,eventName,ref,workflow 
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }} 
        if: always()
          
```

- 이 작업은 배포 성공시 슬랙으로 알림을 보내주기 위해 사용했고 굳이 없어도 된다.

위와 같이 설정하고 깃헙 액션을 동작하면 된다.

----

### 추가) 커밋별로 태그달기

위에서 처럼해도 잘 동작하지만 만약, 커밋 기록별로 도커 허브에 태그를 달려면 아래 처럼 수정하면된다.

**docker-compse.yml 수정**

```yaml
  pct-backend-1:
    image: {도커허브계정명/Spring}:${COMMIT_VERSION}
    << : *common_environment

    depends_on:
      - pct-redis
      - pct-mysql
    ports:
      - "8080:8080"
    deploy:
      resources:
        limits:
          memory: 512M
    restart: on-failure

  pct-backend-2:
		image: {도커허브계정명/Spring}:${COMMIT_VERSION}
```

- 이미지 이름 뒤에 :${COMMIT_VERSION} 태그를 설정

**깃헙 액션 수정**

```yaml
...

- name: 도커 이미지 build 후 push
      uses: docker/build-push-action@v2
      with:
        context: .
        file: Dockerfile  
        push: true  
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/pct-backend:${{ github.sha }}
        platforms: linux/amd64
        
...

deploy-to-ec2:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
    - name: Install SSH
      run: sudo apt update && sudo apt install -y openssh-client

    - name: EC2 접속 및 배포 진행
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.EC2_HOST }}
        username: ${{ secrets.EC2_USERNAME }}
        key: ${{ secrets.EC2_PRIVATE_KEY }}
        script: |
          export COMMIT_VERSION=${{ github.sha }} &&
          ./deploy.sh
```

- 도커 이미지 push 할 때, 뒤에 **:${{ github.sha }}** commit id를 추가 해준다.
- EC2 접속 후 명령어 에서도 **export COMMIT_VERSION=${{ github.sha }}** 로 변수를 지정해주면된다.

이후 깃헙 액션 수행후 도커 허브를 보면 다음과 같이 커밋 별로 태그가 설정된 것을 확인할 수 있다.

![image-20240319065358731](/images/2024-03-18-4/image-20240319065358731.png)

이렇게 하면 커밋 별로 태그를 할 수 있다는 장점이 있지만 서버에 이미지가 쌓인다는 단점도 존재한다.

---

## 결론

도커 컨테이너를 활용하면 인프라 구성을 유연하게 할 수 있다.

하지만, 도커만으로는 컨테이너 자원을 유동적으로 활당하기 어려운거 같다.

------

### 참고자료

https://repost.aws/ko/knowledge-center/ec2-memory-partition-hard-drive
