---
layout: single
title: "멀티 모듈 CI/CD 적용"
categories: project
tag: [주차의상상은현실이된다, 멀티모듈, CI, CD]
typora-root-url: ../
toc: true
author_profile: false
---

[멀티 모듈 구성 상황과 적용](https://this2sho.github.io/project/multi_module1/)에 이어서 멀티 모듈을 적용하면서 수정한 CI, CD 방법을 기록하려고 한다.

# CI 수정

현재 모듈간 의존성은 다음과 같다.

<img src="/images/2024-04-12/image-20240412201517574.png" alt="image-20240412201517574" style="zoom:33%;" />

기존 깃헙 액션에서 동작하는 CI는 다음과 같다.

```yaml
name: Backend CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

permissions:
  checks: write
  pull-requests: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: 리포지토리를 가져옵니다
        uses: actions/checkout@v4

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

      - name: 테스트 수행
        run: ./gradlew test -i

      - name: 테스트 결과를 PR에 코멘트로 등록합니다
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: github.event_name == 'pull_request'
        with:
          files: '**/build/test-results/test/TEST-*.xml'

      - name: 테스트 실패 시, 실패한 코드 라인에 Check 코멘트를 등록합니다
        uses: mikepenz/action-junit-report@v3
        with:
          report_paths: '**/build/test-results/test/TEST-*.xml'
          token: ${{ github.token }}
        
      - name: build 실패 시 Slack으로 알립니다
        uses: 8398a7/action-slack@v3
        if: failure()
        with:
          author_name: 백엔드 빌드 실패 알림
          status: ${{ job.status }}
          fields: repo,message,commit,author,action,eventName,ref,workflow
          slack_webhook: ${{ secrets.SLACK_WEBHOOK_URL }}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

위 동작에서 도메인 모듈 변경시, 도메인 모듈과 의존하고 있는 모듈들을 테스트 해줘야한다.  
스케줄러나 API 모듈만 변경이 발생했을 때는 해당 모듈만 테스트 해주면된다.

이를 적용하기 위해서, 모듈별로 CI 액션을 생성하는 방법이 있다.  
트리거를 `paths`로 적용하여 `domain, app-api, app-scheduler`로 나누어 생성하는 것이다.

하지만, 이 경우 `테스트 수행 step`을 제외한 step들은 동일하기 때문에 중복이 발생한다.  

그래서 기존 액션에서 변경된 모듈을 체크한는 step을 추가해주고 이에 따라 처리해주기로 결정했다.  
변경된 모듈을 체크하는 방법은 main과 추가된 branch를 비교하여 어떤 디렉토리가 변경되는지 확인하는 것이다.

코드를  보면 다음과 같다.

```yaml
      - name: 변경된 모듈을 체크한다.
        id: check_changes
        run: |
          git fetch origin main
          if git diff --name-only origin/main...HEAD | grep -q "^domain/"; then
              echo "domain 모듈이 변경되었습니다."
              echo "::set-output name=changes::domain_changed"
          elif git diff --name-only origin/main...HEAD | grep -q "^app-api/"; then
              echo "api 모듈이 변경되었습니다."
              echo "::set-output name=changes::api_changed"
          elif git diff --name-only origin/main...HEAD | grep -q "^app-scheduler/"; then
              echo "scheduler 모듈이 변경되었습니다."
              echo "::set-output name=changes::scheduler_changed"
          else
              echo "모듈에 변경사항이 없습니다."
          fi
```

이후, `테스트 수행 step` 도 다음과 같이 수정해주었다.

```yaml
      - name: 변경된 모듈을 테스트한다.
        run: |
          case "${{ steps.check_changes.outputs.changes }}" in
              domain_changed)
                  echo "도메인 모듈 및 하위 모듈을 테스트합니다."
                  ./gradlew test --parallel
                  ;;
              api_changed)
                  echo "api 모듈을 테스트합니다."
                  ./gradlew :app-api:test -i
                  ;;
              scheduler_changed)
                  echo "scheduler 모듈을 테스트합니다."
                  ./gradlew :app-scheduler:test -i
                  ;;
              *)
                  echo "모듈에 변경 사항이 없습니다."
                  ;;
          esac
          echo "테스트 결과를 하나의 디렉토리에 복사합니다."
          ./gradlew collectTestResults
```

- gradle에서는 병렬 작업  `parallel`을 지원하는데 로컬 환경에서 측정했을 때 차이는 다음과 같다.

  - `./gradlew test` 수행시 42초   
    ![image-20240413042851528](/images/2024-04-13-multi_module2/image-20240413042851528.png)

  - `./gradlew test --parallel` 수행시 33초  
    ![image-20240413043001746](/images/2024-04-13-multi_module2/image-20240413043001746.png)


​	9초 정도 차이가 나는데 API 모듈에서 통합 테스트시 테스트 컨테이너를 띄우는데 오래 걸리기 때문에  
​	병렬로 모듈 테스트시 API 모듈 테스트 시간(제일 오래걸리는 모듈 테스트 시간)이 곧 전체 테스트 시간임을 알 수 있다.  
​	(아래는 API 모듈만 테스트했을 때, 걸리는 시간이다)  
​	![image-20240413043858610](/images/2024-04-13-multi_module2/image-20240413043858610.png)

- 특정 모듈을 테스트할 때 다음과 같은 명령어를 사용한다.  
  `./gradlew :app-scheduler(모듈명):test`

- 아래에 `./gradlew collectTestResults` 부분은 테스트 결과를 하나의 디렉토리에 복사하기위해  
  gradle task를 추가한 부분이다.

최상단 루트 디렉토리의 `build.gradle`에 다음과 같이 추가해주면 된다.

```groovy
task collectTestResults(type: Copy) {
    description = 'Collects test results from all subprojects'

    def resultDirs = [
            'domain': "$projectDir/domain/build/test-results/test",
            'app-api': "$projectDir/app-api/build/test-results/test",
            'app-scheduler': "$projectDir/app-scheduler/build/test-results/test"
    ]

    from resultDirs.collect { _, dir ->
        fileTree(dir) {
            include '**/TEST-*.xml'
        }
    }
    into "build/allTestResults"
}

```

- 이렇게 하면 최상단 `build/allTestResults` 경로로 모든 모듈의 테스트 결과를 모을 수 있다.

이후, `테스트 결과를 PR에 코멘트로 등록합니다` step과 `테스트 실패 시, 실패한 코드 라인에 Check 코멘트를 등록합니다` step을 다음과 같이 수정하면 된다.

```yaml
      - name: 테스트 결과를 PR에 코멘트로 등록합니다
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: github.event_name == 'pull_request'
        with:
          files: '**/build/allTestResults/TEST-*.xml'

      - name: 테스트 실패 시, 실패한 코드 라인에 Check 코멘트를 등록합니다
        uses: mikepenz/action-junit-report@v3
        with:
          report_paths: '**/build/allTestResults/TEST-*.xml'
          token: ${{ github.token }}
```

- 경로를 테스트 결과를 복사한 디렉토리로 변경



# CD 수정

CD 부분도 깃헙 액션의 경우 CI 부분과 거의 동일하게 변경해주면 된다.

기존 CD 방식([컨테이너 구성 및 무중단 배포](https://this2sho.github.io/project/container_cd/)) 에서 변경된 부분은 다음과 같다.

```yaml
    - name: 변경된 모듈을 체크한다.
      id: check_changes
      run: |
        git fetch origin main
        if git diff --name-only origin/main...HEAD | grep -q "^domain/"; then
            echo "domain 모듈이 변경되었습니다."
            echo "::set-output name=changes::domain_changed"
        elif git diff --name-only origin/main...HEAD | grep -q "^app-api/"; then
            echo "api 모듈이 변경되었습니다."
            echo "::set-output name=changes::api_changed"
        elif git diff --name-only origin/main...HEAD | grep -q "^app-scheduler/"; then
            echo "scheduler 모듈이 변경되었습니다."
            echo "::set-output name=changes::scheduler_changed"
        else
            echo "모듈에 변경사항이 없습니다."
        fi

    - name: 변경된 모듈을 build한다.
      run: |
        case "${{ steps.check_changes.outputs.changes }}" in
              domain_changed)
                  echo "전체 모듈을 빌드합니다."
                  ./gradlew build --parallel
                  ;;
              api_changed)
                  echo "api 모듈을 빌드합니다."
                  ./gradlew :app-api:build
                  ;;
              scheduler_changed)
                  echo "scheduler 모듈을 빌드합니다."
                  ./gradlew :app-scheduler:build
                  ;;
              *)
                  echo "모듈에 변경 사항이 없습니다."
                  ;;
        esac

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1  

    - name: 도커 로그인
      uses: docker/login-action@v1
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}  
        password: ${{ secrets.DOCKERHUB_TOKEN }}     

    - name: 도커 이미지 build 후 push
      uses: docker/build-push-action@v2
      if: |
        steps.check_changes.outputs.changes == 'domain_changed' || 
        steps.check_changes.outputs.changes == 'api_changed'
      with:
        context: .
        file: app-api/Dockerfile
        push: true  
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/${{ secrets.API_IMAGE }}:${{ github.sha }}
        platforms: linux/amd64

    - name: 도커 이미지 build 후 push
      uses: docker/build-push-action@v2
      if: |
        steps.check_changes.outputs.changes == 'domain_changed' || 
        steps.check_changes.outputs.changes == 'scheduler_changed'
      with:
        context: .
        file: app-scheduler/Dockerfile
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/${{ secrets.SCHEDULER_IMAGE }}:${{ github.sha }}
        platforms: linux/amd64
```

CD의 경우에는 도커 파일을 추가로 실행 모듈 별로 생성해주었다.

API 모듈의 경우 다음과 같이 jar 파일 이름만 수정해서 모듈 하위에 두었다. (스케줄러 모듈도 동일)

```dockerfile
FROM amazoncorretto:17-alpine

COPY build/libs/app-scheduler-0.0.1-SNAPSHOT.jar app.jar

CMD ["java", "-jar", "app.jar"]

```



이후, 서버에 있는 docker-compose와 배포 스크립트도 수정해주었다.

docker-compose의 경우, 스케줄러 이미지를 위한 서비스를 추가해주었다.

```yaml
  {스케줄러 서비스}:
    image: {도커허브계정명/스케줄러 모듈 이미지}:${COMMIT_VERSION}
    environment:
      - PROFILE=prod
      - KAKAO_API_KEY=
      - SEOUL_API_KEY=
      - PUSAN_API_KEY=
      - DB_URL=
      - DB_USERNAME=
      - DB_PASSWORD=
    depends_on:
      - pct-mysql
    deploy:
      resources:
        limits:
          memory: 256M
    restart: on-failure
```

이후, 스케줄러에 필요한 환경 변수들을 추가해주었다.

배포 스크립트의 경우, 도커 pull 명령이 성공하는지 실패하는지에 따라서 해당 모듈이 변경되었는지 판단하도록 구현하였다.  
코드는 다음과 같다.

```sh
echo "배포 시작!"
echo "${COMMIT_VERSION} 커밋의 컨테이너 가져오는중"

api_changed="false"
scheduler_changed="false"

echo "api 이미지가 변경사항이 있으면 가져온다."
docker pull {도커허브계정명/API 모듈 이미지}:${COMMIT_VERSION}

if [ $? -eq 0 ]; then
    echo "api pull 성공"
    api_changed="true"
else
    echo "api pull 실패"
fi

echo "scheduler 이미지가 변경사항이 있으면 가져온다."
docker pull {도커허브계정명/스케줄러 모듈 이미지}:${COMMIT_VERSION}

if [ $? -eq 0 ]; then
    echo "scheduler pull 성공"
    scheduler_changed="true"
else
    echo "scheduler pull 실패"
fi

if [ "$api_changed" = "true" ]; then
    if docker ps | grep -q api-1; then
        echo "컨테이너가 8080 포트 사용중.."
        NEW_CONTAINER={API 모듈 서비스명}-2
        OLD_CONTAINER={API 모듈 서비스명}-1
        NEW_PORT=8081
        OLD_PORT=8080
    elif docker ps | grep -q api-2; then
        echo "컨테이너가 8081 포트 사용중.."
        NEW_CONTAINER={API 모듈 서비스명}-1
        OLD_CONTAINER={API 모듈 서비스명}-2
        NEW_PORT=8080
        OLD_PORT=8081
    else
        echo "컨테이너 새로 시작중.."
        NEW_PORT=8080
        NEW_CONTAINER={API 모듈 서비스명}-1
        docker-compose up -d pct-mysql
        docker-compose up -d pct-redis
        sleep 5
    fi

    echo "${NEW_PORT} 컨테이너 올리는중.."
    docker-compose up -d ${NEW_CONTAINER}

    sleep 5

    if [ -n "${OLD_CONTAINER}" ]; then
      echo "${OLD_PORT} 컨테이너 내리는중.."
      docker-compose down ${OLD_CONTAINER}
    fi
fi

if [ "$scheduler_changed" = "true" ]; then
    echo "스케줄러 컨테이너 재시작 중.."
    docker-compose down {스케줄러 모듈 서비스명}
    docker-compose up -d {스케줄러 모듈 서비스명}
fi
```



---

## 마치며

적용하면서 생각치 못한 에러등 시간을 많이 사용하게 되었지만, 멀티 모듈을 적용하면서 책임을 어디에 둘지 의존성 관리등 생각치 못한 부분에서 공부도 많이 된거 같다.
