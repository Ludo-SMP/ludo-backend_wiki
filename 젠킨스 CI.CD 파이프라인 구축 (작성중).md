#### CI / CD 파이프라인 구축 이유
Ludo 서비스 정식 출시 이후, 사용자 피드백을 21건이나 수집할 수 있었습니다. 해당 **피드백을 서비스에 빠르게 반영하여 고객에게 제공하기 위해선** 기존에 수동으로 진행했던 **`빌드 / 테스트 / 배포`** 과정을 자동화할 필요가 있어 도입을 결정했습니다.

#### Docker 기반 Jenkins 선택 이유
CI / CD DevOps 툴은 **`Github Actions`**, **`Jenkins`** 등 다양하게 존재합니다. **`Github Actions`** 도 충분히 매력적인 선택지였지만, 많은 기업에서 **`Jenkins`** 를 적극 활용한다는 점, **`Jenkins`** **에서 제공하는 플러그인을 활용**할 수 있다는 점, **`Jenkins`** **분산 아키텍처 구조를 활용**할 수 있다는 점을 고려하여 **`Jenkins`** 를 도입했습니다.  
**`Jenkins`** 는 Java 기반 오픈소스입니다. 그에 따라, 다양한 환경 변수를 설정해 주어야 합니다. 해당 과정이 번거롭다 느껴졌고, 데이터베이스와 같이 중요 데이터를 적재하는 용도가 아니기 때문에 **`Docker`** 를 활용했습니다.

#### Docker 기반 Jenkins 컨테이너 구축
도커 컨테이너를 생성하는 방법은 크게 두 가지 있습니다.
1. `CLI에서 docker 명령어`를 입력하는 방법
2. `docker-compose.yml` 파일을 활용하는 방법
두 번째 방법은 첫 번째 방법의 명령어들을 마치 예약어처럼 yml 파일에 등록하여 사용하는 것입니다. 첫 번째 방법은 매번 CLI 입력 명령어를 외워야 하는 번거로움이 있기 때문에, 두 번째 방법을 채택했습니다.

```yml
version: "3"
services:
    jenkins:
        image: jenkins/jenkins:lts-jdk17
        container_name: ludo-jenkins
        user: root
        volumes:
            - ./jenkins_home:/var/jenkins_home
            - /var/run/docker.sock:/var/run/docker.sock
        ports:
            - 8080:8080
```


#### CI / CD 파이프라인 구축

