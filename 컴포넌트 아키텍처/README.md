## 컴포넌트 아키텍처
안녕하세요. 팀 Ludo 백엔드 Hugh, Archa, Back입니다. 저희가 고민했던 컴포넌트 아키텍처 구조를 소개하려고 합니다.

<br><br>

### 초기 구상

도메인을 중심으로 전반적인 아키텍처 구조를 고민하는 방향으로 접근했습니다.
도메인 객체에도 결국 자바 객체이기 때문에 상태와 행위를 가지고 있어야 한다고 판단했고, 도메인 객체에 대한 비즈니스 로직이 있다고 하면 서비스 계층에 작성하는 것 보다, 해당 도메인에 작성하는 방향으로 아키텍처를 구성했습니다.
그리고 서비스 계층은 단순히 요청을 받아 도메인 객체에 이를 위임해주는 것 입니다. 대신 여러 종류의 도메인 객체를 조합하거나 도메인  객체와 결합도가 떨어지는 비즈니스 로직 등은 서비스 계층에서 작성 했습니다.


- 비즈니스 로직을 도메일 객체에 작성함으로써 얻는 장점은 아래와 같습니다.
  - 객체지향스러운 개발을 할 수 있다.
  - 도메인 객체의 응집도를 높일 수 있다.
  - DI를 줄일 수 있다.
  - 서비스 계층층의 코드가 간결하다.


### 핵심 도메인 산출

핵심 비즈니스 로직은 도메인 이 담당해야 하고, 이를 위해 도메인 을 도출할 필요가 있습니다. 1차적으로 기획 단계에서 나왔던 용어들을 모두 정리하여, 이를 기반으로 도메인 을 구상했고, 연관성이 있는 도메인 들을 밀집시키는 방향으로 설계를 진행했습니다.

(디렉토리 구조 txt)

<br><br>


### 서비스 레이어의 역할

저희 코드의 서비스 계층은 유즈 케이스 역할을 담당하고 있습니다. 어떤 고민을 통해 이런 결론을 도출하게 되었는지 소개하고자 합니다.



협업을 시작하며 고민이 되었던 부분은 코드 충돌을 최소화하는 것이었습니다. 엔티티 도메인 은 특성상 코드 충돌을 감수해야했지만, 서비스 계층은 코드 충돌을 최소화할 수 있을 것으로 생각했습니다.



클린 아키텍처의 청사진을 참고해보면, 컨트롤러 계층과 엔티티 계층 사이에 유즈 케이스 계층이 존재하고, 서비스 계층 과 유사하다는 생각이 들었습니다.





또한 저희의 업무 프로세스는 아래와 같이 유저 스토리 티켓 산출 후 업무를 할당해 기능 구현을 진행했습니다. 각자가 맡은 유즈 스토리 즉, 유즈 케이스 를 서비스 계층 내에서 분리하고 비즈니스 로직은 엔티티 도메인  에게 위임하는 방식으로 진행하면 서비스 계층의 코드 충돌을 최소화할 수 있을 것으로 판단했습니다. 오브젝트 서적 중, 하나의 협력을 달성하기 위한 객체들의 의사소통이라는 얘기가 많이 나오는데 유즈 케이스 가 곧 협력 이 된다는 생각도 함께 들었습니다.





결론적으로 저희 서비스 계층의 코드는 아래와 같이 유즈 케이스를 맡고 있는 형태가 되었습니다.


```java
public class UseCaseBlahBlahService {
    
    public Repository repository;
    
    public void service() {
        Domain domain repository.something();
        domain.bizLogicStart();
    }

}
```


<br><br>


### repository 상속 구조

하나의 Repository를 통해 JpaRepository의 기능과 QueryDSL의 기능을 모두 사용할 수 있도록 repository, repositoryImpl로 나눠 구성했습니다.



(구조 이미지)



해당 구조는 크게 JpaRepository를 extends하는 Repository 인터페이스에 QueryDSL의 기능을 사용하는 RepositoryImpl 클래스를 구현체로 가지고 있습니다.



즉 Repository 하나로 기본적인 JpaRepository의 기능과 QuertDSL을 통한 디테일한 기능을 모두 사용할 수 있기 때문에 Repository를 사용하는 Service 단에서의 코드가 깔끔해진다는 장점을 가지게 됩니다.



고민했던 구조

```java
//Repository
public interface ExampleRepository extends JpaRepository<Example, Long>, ExampleRepositoryCustom {
    ...
}

//RepositoryCustom
public interface ExampleRepositoryCustom {
    ...
}

//RepositoryImpl
public class ExampleRepositoryImpl implements ExampleRepositoryCustom {
    ...
}
```

현재 프로젝트 규모와 앞으로의 확장성을 고려했을 때 구조적인 간결함을 가져가는것이 옳은 판단이라고 생각했습니다.



적용한 구조

```java
//Repository
public interface ExampleRepository extends JpaRepository<Example, Long> {
    ...
}

//RepositoryImpl
public class ExampleRepositoryImpl implements ExampleRepositoryCustom {
    ...
}
```

<br><br>

### 현재 컴포넌트 아키텍처 구조

(디렉토리 구조 txt)
