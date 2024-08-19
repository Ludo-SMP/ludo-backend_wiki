# Jenkins CI/CD Pipleline -> Github Actions로 이전

## Jenkins -> Github Actions 이전 이유

이전에 구축한 Jenkins CI/CD Pipeline을 이용하여 배포를 진행하면서 문제를 겪게 되었습니다.

가장 큰 문제는 Jenkins나 Pipeline 자체의 문제가 아닌 Computing Resource의 부족이 컸습니다.

상용 어플리케이션이 아니기에 현재 GCP의 Free-tier를 이용하여 Jenkins를 설치하여 쓰고 있는데,

AWS의 `ap-northeast-2` Region에서 제공되는 Free-tier instance는 `t2.micro`로, Credit-based system으로 돌아가는 Burstable Instance입니다.

각 AWS HA 내의 데이터 센터에 존재하는 수 많은 Rack 내의 Rack Computer에서 물리적인 CPU를 물리적/시간적 분할을 통해(하이퍼쓰레드) 공유하는 vCPU로 추상화 한 개념이기 때문에

하나의 CPU를 N개로 쪼개서 0.X CPU 단위로 여러 사람들이 사용하게 되고, 한 명의 사용자가 너무 많은 리소스를 소모하면 쓰로틀링 등이 발생하여 다른 사용자도 느려지거나 CPU 수명에 영향을 줄 수 있기 때문에

이러한 급격한 사용량 증가는 Credit을 감소하는 방식으로 페널티를 발생 시킵니다.

그래서 build 및 unit, integration, e2e test, test container를 띄워서 여러 테스트를 진행하는 과정에서 Credit이 급격히 소모되어 CPU 성능이 10% 미만으로 줄어들어 너무 오랜 시간이 소요됩니다.

Pipeline 고도화 과정에서 Infra Repository와의 통신을 하며 Provisioning 자동화를 추가할 예정인데 상당히 많은 실험과 변경이 일어날 것이 예상되어(현재 Github Actions으로 전환한 뒤 실험적으로 하루만에 100번 정도의 pipeline 변경 및 재실행이 발생)

이러한 시간 소요는 시간이 촉박한 가운데 개발 시간을 매우 늦출 것이라 생각 되었습니다.

또한 메모리도 AWS의 `t2.micro`, GCP의 `e2.micro` 모두 1GiB로 부족하여 스왑 메모리를 설정한 뒤 디스크 영역에 read/write가 발생하기 때문에 속도를 더욱 늦추는 원인이 될 것입니다.

CPU가 `t2.micro`의 약 4배 성능에 달하고 메모리가 4GiB인 `t3.medium` 정도를 사용하면 사용량에 따라 5만원 ~ 10만원 정도의 요금이 부과되는데 이 정도 스펙이면 잘 돌아갈 것으로 보이나,

요금 문제로 Instance를 pipeline 수행 시마다 키고 끄는 작업이 들어가야 하여 Github Action을 통해 프로비저닝을 트리거 해줘야 할텐데 역시 실험 단계에서는 VM 구조상 Instance Provisioning이 1분 정도 걸리기 때문에

100회면 100분이 추가로 소요되는 문제가 발생합니다.

또한 초기 세팅도 별도로 필요하고 설정이 잘못되어 꺼지지 않는 경우에 요금이 부과될 수도 있다는 문제도 있어서 배제하였습니다.

자료가 많은 AWS 위주로 글을 작성하고 있으나 GCP도 거의 AWS와 비슷할 것이라고 생각됩니다. 디테일한 명칭이나 UI가 꽤 다르긴 했지만, Spanner, Big Table 등의 특화된 서비스를 제외하면 서로 제공하는 서비스, 머신 스펙이나 리전 종류도 크게 다르지 않았습니다.

Github Actions의 장점은 Serverless이기 때문에 소규모에서는 별도의 머신에 환경을 구축할 필요가 없다는 점입니다.

또한 무료 플랜도 월 2,000분의 빌드 타임을 제공하며, 머신 스펙이 현재 4-vCPU, 16GiB로 업그레이드 되어 AWS, GCP 등의 Free-Tier 머신보다 훨씬 더 쾌적한 성능을 발휘 하였습니다.

Github Repository의 통합과 모든 개발자가 익숙한 `yaml` 문법 및 다양한 action plugin을 제공하여 marketplace에서 원하는 action을 다운로드 하여 pipeline을 확장할 수 있다는 점,

설정이 복잡해지면 간단하게 action을 분리하여 모듈화 할 수 있다는 점도 괜찮다고 느껴졌습니다.

또한 현재 LUDO Team Frontend에서도 Github Actions을 사용하고 있는데, 이 부분도 협업 시에 소통에 용이할 것으로 생각됩니다.

이러한 연유로 현재 프로젝트에서는 기존 Jenkins에서 Github Actions로 이전하는 것이 좋겠다고 생각하였습니다.

---

# CI/CD Pipeline Workflow 

전체 파이프라인이 어떤 과정을 거쳐서 동작하는지를 간략하게 말씀 드리겠습니다.

크게 본다면 이름 그대로 CI, CD 두 부분으로 나눌 수 있을 것 같습니다.

적절한 관심사 분리를 통해 유지보수를 편하게 하기 위해 Application, Infrastructure 2개의 Repository로 나누었습니다. 

먼저 main branch에 PR이 작성될 때 build 및 test가 진행됩니다.

이미 build나 test에 실패한 commit은 리뷰 이전에 수정이 이루어져야 하기 때문에 이를 즉시 인지하기 위함입니다.

test는 unit, integration, e2e로 이루어져 있으며, 이는 build 의존적입니다.

build가 실패하면 애초에 test의 의미가 없어지고, build 이후 caching 된 것을 바탕으로 test를 진행합니다.

e2e test의 경우는 test를 위한 infra provisioning이 이루어진 뒤에 진행되고, e2e test가 끝나면 다시 destroy 됩니다.

test 성공 이후 Container Image를 Build하여 Dockerhub에 push합니다.

artifact 명은 `<APP_NAME>-<YYYYMMDD>-<SHA>.jar` 형태로 현재 날짜 및 commit의 sha를 기반으로 자동으로 정해집니다.

만약 정상적으로 build 및 test에 성공 하였고, approve가 완료 되었다면 CD Pipeline으로 넘기게 됩니다.

<img width="898" alt="image" src="https://github.com/user-attachments/assets/517a0faf-acd0-48fe-ac95-05661413b391">

현재까지 임시로 구축한 CI Pipeline입니다.

[작성중]
