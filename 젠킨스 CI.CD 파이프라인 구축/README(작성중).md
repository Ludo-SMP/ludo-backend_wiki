## 젠킨스 CI/CD 파이프라인 구축
> 이 글은 팀 Ludo의 [아카](https://github.com/june-777)가 작성했습니다.  
> 젠킨스 CI/CD 구축을 담당하게 되어, 진행하며 마주했던 내용들을 공유하고자 합니다. 

### 1️⃣ CI / CD 파이프라인 구축 이유
Ludo 서비스 정식 출시 이후, 사용자 피드백을 21건이나 수집할 수 있었습니다. 해당 **피드백을 서비스에 빠르게 반영하여 고객에게 제공하기 위해선** 기존에 수동으로 진행했던 **`빌드 / 테스트 / 배포`** 과정을 자동화할 필요가 있어 도입을 결정했습니다.

<br><br>

### 2️⃣ Docker 기반 Jenkins 선택 이유
CI / CD DevOps 툴은 **`Github Actions`**, **`Jenkins`** 등 다양하게 존재합니다. **`Github Actions`** 도 충분히 매력적인 선택지였지만, 많은 기업에서 **`Jenkins`** 를 적극 활용한다는 점, **`Jenkins`** **에서 제공하는 플러그인을 활용**할 수 있다는 점, **`Jenkins`** **분산 아키텍처 구조를 활용**할 수 있다는 점을 고려하여 **`Jenkins`** 를 도입했습니다.  

**`Jenkins`** 는 Java 기반 오픈소스입니다. 그에 따라, 다양한 환경 변수를 설정해 주어야 합니다. 해당 과정이 번거롭다 느껴졌고, 데이터베이스와 같이 중요 데이터를 적재하는 용도가 아니기 때문에 **`Docker`** 를 활용했습니다.

<br><br>

### 3️⃣ Docker 기반 Jenkins 컨테이너 구축
도커 컨테이너를 생성하는 방법은 크게 두 가지 있습니다.
1. `CLI에서 docker 명령어`를 입력하는 방법
2. `docker-compose.yml` 파일을 활용하는 방법

두 번째 방법은 첫 번째 방법의 명령어들을 마치 예약어처럼 yml 파일에 등록하여 사용하는 것입니다. 첫 번째 방법은 매번 CLI 입력 명령어를 외워야 하는 번거로움이 있기 때문에, 두 번째 방법을 채택했습니다.

가끔 젠킨스 도커 컴포즈 파일을 수정해서 컨테이너를 삭제하고 다시 띄울 때 일이 있습니다. 이 때 이전에 설치했던 플러그인, 작업 내용들을 유지하기 위해 `/var/jenkins_home` 디렉토리를 볼륨 마운팅합니다.

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

<br>

#### Jenkins 디렉토리 구조
위의 도커 컴포즈 파일의 볼륨을 이해하려면, 젠킨스 디렉토리 구조에 대해 간단히 알고 가면 좋습니다.  
`/var/jenkins_home` 하위에 `젠킨스에 설치한 플러그인`, `파이프라인 작업 공간`,` 파이프라인 빌드 기록 결과` 등의 메타 데이터를 저장하고 있습니다.

```bash
var
└── jenkins_home
  ├── workspace
  │ └── 파이프라인이름(Ludo-CICD)
  │  └── [Ludo-CICD 작업공간] 
  │
  ├── jobs
  │ └── 파이프라인이름(Ludo-CICD) 
  │  └── 빌드번호
  │    └── [Ludo-CICD 빌드기록결과]
  │
  └── plugins
    └── [설치한 플러그인들] 
```


- `jenkins_home/plugins`  
	젠킨스에 설치한 플러그인 정보를 저장하는 디렉토리입니다.


- `jenkins_home/workspace/파이프라인이름`  
	**파이프라인의 실제 작업이 이루어지는 디렉토리**입니다. 즉, 상대 경로 `./` 는 `/var/jenkins_home/workspace/파이프라인이름` 절대 경로와 같습니다.  
  또한 파이프라인을 수행한 결과물은 해당 디렉토리에 위치하게 됩니다.
  
  ```bash
  stage('Test') {
      steps {
          dir('./') { // == /var/jenkins_home/workspace/파이프라인이름
              sh './gradlew clean test'
          }
      }
  }
  ```
	<img src = "https://github.com/Ludo-SMP/ludo-backend_wiki/assets/68291395/8f88be13-d4a4-41c7-9d65-929eaea62ed1" width=50%>


- `jenkins_home/jobs/파이프라인이름/파이프라인작업번호`  
	각각의 파이프라인이 수행한 작업은 번호를 갖게 됩니다. 해당 번호의 이름으로 디렉토리가 생성되고, 로그 기록들을 확인할 수 있습니다.  
	<img src = "https://github.com/Ludo-SMP/ludo-backend_wiki/assets/68291395/8bbbd2f3-7334-4e72-b116-7654097cf7e2" width=50%>
  <img src = "https://github.com/Ludo-SMP/ludo-backend_wiki/assets/68291395/dfe7e914-cf98-4fc2-a798-302d78645073" width=50%>


<br><br>

### 4️⃣ CI / CD 파이프라인 구축

<img src = "https://github.com/Ludo-SMP/ludo-backend_wiki/assets/68291395/311a7598-07d2-47d5-b1c7-e4f8cd8efe7d" width=70%>


파이프라인은 **`Git Clone`**, **`Test`**, **`Build`**, **`Deploy`** 4가지 **`stage`** 으로 구성되어 있습니다.  
각각의 **`stage`** 를 달성하기 위해 구성해야 할 인프라 요소들에 대해 설명하고자 합니다.

```bash
pipeline {
    agent any
    stages {
        stage('Git Clone') {
            steps {
                git branch: 'dev', url: 'https://github.com/june-777/ludo-backend.git'
                withCredentials([GitUsernamePassword(credentialsId: 'submodule_configuration', gitToolName: 'Default')]) {
                    sh 'git submodule init'
                    sh 'git submodule update'
                }
            }
        }
        
        stage('Test') {
            steps {
                dir('./') {
                    sh './gradlew clean test'
                }
            }
        }
        
        stage('Build') {
            steps {
                dir('./') {
                    sh './gradlew bootJar'
                }
            }
        }
        
        stage('Deploy') {
            steps {
                dir('./build/libs/') {
                    sshagent(credentials:['ludo-backend-ssh']) {
                        sh 'scp study-matching-platform-0.0.1-SNAPSHOT.jar archa@35.185.251.89:/home/archa'
                        sh 'ssh -t archa@35.185.251.89 /home/archa/run.sh'
                    }
                }
            }
        }
    }
}
```

<br>

#### stage: Git Clone
```bash
stage('Git Clone') {
    steps {
        git branch: 'main', url: 'https://github.com/Ludo-SMP/ludo-backend.git'
        withCredentials([GitUsernamePassword(credentialsId: 'xxx', gitToolName: 'Default')]) {
            sh 'git submodule init' 
            sh 'git submodule update' 
        }
    }
}
```

<br>

##### 프라이빗 레포지토리 비밀번호 Jenkins Credentials 등록
> withCredentials([GitUsernamePassword(credentialsId: 'xxx', gitToolName: 'Default')])

저희 Ludo 백엔드 팀은 환경 설정 관리를 **`프라이빗 레포지토리`** 로 관리하고 있습니다. 프라이빗이기 때문에 젠킨스 인스턴스에서 접근할 수 있는 인증 수단이 필요합니다.

<img src = "https://github.com/Ludo-SMP/ludo-backend_wiki/assets/68291395/05f87738-dfd0-4321-b48f-5a0c21ec864f" width=50%>

<img src = "https://github.com/Ludo-SMP/ludo-backend_wiki/assets/68291395/4a784005-e6e3-4a7c-9eeb-6336b852922c" width=50%>

<img src = "https://github.com/Ludo-SMP/ludo-backend_wiki/assets/68291395/9f469aae-a0a5-42db-8c63-12fe915b5574" width=50%>

<img src = "https://github.com/Ludo-SMP/ludo-backend_wiki/assets/68291395/76146cf9-f5ba-405a-bf38-44a07c1ef4dd" width=50%>


<br>


##### 서브 모듈 갱신
>sh 'git submodule init'  
>sh 'git submodule update' 

**`환경 설정 프라이빗 레포지토리`** 는 **`서브 모듈`** 로 사용되고 있습니다. `git clone`을 할 경우, 서브 모듈의 내용은 반영되지 않습니다. 이를 제대로 갱신 및 반영하기 위해 필요한 명령어 입니다.

<br>

#### stage: Test
```bash
stage('Test') {
    steps {
        dir('./') {
            sh './gradlew clean test'
        }
    }
}
```

(작성 예정)

<br>

#### stage: Build
```bash
stage('Build') {
    steps {
        dir('./') {
            sh './gradlew bootJar'
        }
    }
}
```

(작성 예정)


<br>

#### stage: Deploy
```bash
stage('Deploy') {
    steps { 
        dir('./build/libs/') {
            sshagent(credentials:[xxx]) {
                sh 'scp xxx.jar USER_NAME@xx.xx.xx.xx:FILE_PATH'
                sh 'ssh -t USER_NAME@xx.xx.xx.xx /home/USER_NAME/run.sh'
            }
        }
    }
}
```

<br>

##### SSH agent 플러그인 설치
해당 플러그인을 설치하면 선언형 스크립트 `sshagent( )` 로 SSH를 편리하게 사용할 수 있습니다.  
설치 방법: Jenkins 대시보드 → Jenkins 관리 → Plugins → Available plugins → `ssh agent` 검색 및 설치

<br>

##### SSH 인증서 생성 및 Jenkins Credentials 등록
SSH 인증서를 생성합니다. 단, **반드시 `pem` 형식으로 인증서를 생성**해야 합니다.
```bash
ssh-keygen -t rsa -f ~/.ssh/KEY_FILENAME -C USERNAME -b 2048 pem
```
<img src = "https://github.com/Ludo-SMP/ludo-backend_wiki/assets/68291395/0a32a58b-dad7-4604-abc5-996142422caa" width=50%>

<br>


생성한 SSH 인증서를 Jenkins Credentials 에 등록합니다.  
SSH 비밀키 값을 등록할 때, **`-----BEGIN RSA PRIVATE KEY-----` 와 `-----END RSA PRIVATE KEY-----` 내용을 모두 포함**해야 합니다.  
<img src = "https://github.com/Ludo-SMP/ludo-backend_wiki/assets/68291395/90284861-a0f4-47dd-8a45-fe4cf666d167" width=50%>
<img src = "https://github.com/Ludo-SMP/ludo-backend_wiki/assets/68291395/ce17e3da-2530-4b7a-9cf3-9e67946cc9b8" width=50%>


<br>

##### 대상 서버를 fingerprint 등록
ssh을 사용하다보면, 최초 접근시에 아래와 같은 화면을 볼 수 있습니다. 이걸 수동으로 직접 해주는 과정이라 보면 됩니다.
<img src = "https://github.com/Ludo-SMP/ludo-backend_wiki/assets/68291395/4edcdc44-aec0-443d-af59-3d1357c8d974" width=50%>


1. 젠킨스 컨테이너에 접속
```bash
# in jenkins host bash
> docker exec -it [컨테이너ID] /bin/bash
```

2. known_hosts 파일에 등록
**`젠킨스 호스트의 ~/.ssh` 가 아닌 `젠킨스 컨테이너의 ~/.ssh` 에 등록**해야 합니다.
```bash
# in jenkins container bash

# 초기엔 .ssh 디렉토리가 없어서, 디렉토리 먼저 생성
> mkdir ~/.ssh/

# 대상 서버 ip 주소 (xx.xx.xx.xx)를 known_hosts 파일에 등록
> ssh-keyscan -t rsa xx.xx.xx.xx >> ~/.ssh/known_hosts
```
