# Layered Architecture

백엔드 어플리케이션은 일반적으로 **Layered Architecture**를 채택합니다.

어떤 어플리케이션을 구축하더라도 Presentation/Data Access Layer는 그저 IO를 담당할 뿐이며,

각 어플리케이션의 핵심 로직만 Application Layer에 속하는 매우 흔한 구조입니다.

여기서 디렉토리 구조를 Layer/Feature 단위로 나눌 수 있는데, 간단한 어플리케이션은 Layer 단위로 나누는 것으로 충분합니다.

```
presentation/
application/
infrastructure/
```

Layer 단위로 나눌 경우, 위와 같이 각 Layer를 나타내는 3개의 디렉토리 밑에 모든 파일들이 들어가게 됩니다.

협업 시에 기능을 분배하게 되는데 feature A, B, C를 각각 1명씩 맡은 경우

feature 단위로 나누는 것이 보다 Layer 간의 응집도가 높아져서 파일 탐색 시의 indirection이 줄어들고, 필요한 component들이 만들어져 있는지

한 눈에 확인할 수 있다는 장점이 있어서 ludo 어플리케이션 최초 설계 시에 이러한 접근 방식을 택하게 되었습니다.

```
featureA/
  presentation/
  application/
  infrastructure/
featureB/
  presentation/
  application/
  infrastructure/
featureC/
  presentation/
  application/
  infrastructure/
```

고로 위와 같은 디렉토리 구조를 형성하게 되었습니다.

<img width="844" alt="image" src="https://github.com/user-attachments/assets/7a9f561a-ef57-43e0-b0d1-853038a37c13">

feature로 나눴을 뿐이기에 이런 세부 사항을 배제하고 도식화 하면 위와 같은 일반적인 Layered Architecture 형태로 표현할 수 있습니다.

이 때 상단 Layer가 하단 Layer에 의존하게 됩니다.

presentation -> application, application -> infrastructure

application Layer에 비즈니스 로직이 들어가게 될텐데, 핵심 로직이므로 단순 IO인 presentation, infrastructure Layer에 비해 가장 코드량이 많아질 가능성이 높습니다.

오직 Service에 모든 비즈니스 로직을 넣는 것은 Transaction Script Pattern이라 불리지만 규모가 커질수록 유지 보수하기 어려운 코드가 됩니다.

그래서 Service를 Domain으로 쪼개서 객체 간의 협력 및 메시지 전달을 통해 소통하는 Domain Model Pattern을 택하였고,

<img width="821" alt="image" src="https://github.com/user-attachments/assets/a4a39d1c-51ed-4796-b100-ecb887319af6">

도식화 하면 위와 같습니다.

Service가 Domain을 의존하고, Domain은 다른 외부 패키지 없이도 작동하는 독립 모듈로서 동작합니다.

그래서 위와 같은 구조에서는 Domain Layer Test 시에는 외부 infrastructure를 신경쓰지 않고 Unit Test를 하는 것이 가능해졌습니다.

![image](https://github.com/user-attachments/assets/6dd6d386-a66f-4215-9ec2-0b799970ab6b)

Unit Test는 수 천, 수 만 개가 있더라도 금방 처리되기 때문에 빠른 배포에 유리하고, 항상 위와 같이 Test의 비율 중 Unit Test를 가장 높게 가져갈 것이 권장됩니다.

그러나 여전히 Service는 Test 하기 어려운 구조였습니다.

여기까지의 구조에서 Service는 Infrastructure에 의존하고 있기 때문에 H2, Postgres 등의 외부 프로세스와 IPC를 할 필요가 있습니다.

물론 mocking을 통해 이를 Unit Test로 만들 수 있지만, Mocking은 원본 객체의 행위 자체를 일일이 overwrite 해야 하기 때문에 추후 유지 보수에 그리 좋지 않을 것이라,

애초에 Test 하기 좋은 Architecture를 가져가는 것이 좋다고 생각했습니다.

그래서 infrastructure Layer 내의 Repository를 인터페이스/구현체로 분리하고, 인터페이스를 application Layer로 이전 하였습니다.

<img width="822" alt="image" src="https://github.com/user-attachments/assets/7c0fdff2-be46-4e7b-8ee3-ed8991b8e554">

infrastructure Layer 내의 RepositoryImpl이 Repository Interface를 의존하게 되고, DIP로 인해 의존 방향이 역전되었습니다.

이는 추후 멀티 모듈을 도입하게 된다면 application을 완전히 독립적인 모듈로 가져갈 수 있다는 장점도 있습니다.

application module이 infrastructure module을 의존하지 않기 때문입니다.

<img width="997" alt="image" src="https://github.com/user-attachments/assets/81b14213-0037-4ade-9950-073269353855">

좀 더 보기 쉽게 도표를 수정하였습니다.

application Layer가 서비스의 핵심이므로 Service + Domain을 하나의 application Layer에 묶었습니다.

이 때, 의존 방향은 항상 외부에서 application Layer를 향하게 됩니다.

고로 application Layer는 외부에 의존하지 않고 독립적입니다.

<img width="980" alt="image" src="https://github.com/user-attachments/assets/ed11d134-ca0c-4c57-9681-4dae6b656c8c">
<img width="991" alt="image" src="https://github.com/user-attachments/assets/5d545cd9-2345-443c-83ca-aab79b927714">

infrastructure Layer의 RepositoryImpl은 application Layer 내의 Repository Interface에 의존합니다.

그래서 마치 의존 관계 화살표가 Repository Interface를 관통하듯 도표로 묘사했습니다.

Domain Layer 내에는 Entity들이 존재하므로 이것도 추가로 표기했습니다.

![image](https://github.com/user-attachments/assets/6f507fb0-a377-4f46-9e59-ed1abb84d970)

그리고 Hexagonal Architecture를 살펴보면, 온전하진 않더라도 비슷한, 조금 더 가벼운 버전 정도로 보여집니다.

명칭만 좀 더 맞춰주면 더 비슷해지겠습니다.

<img width="1017" alt="image" src="https://github.com/user-attachments/assets/c6e996ae-523c-4710-b85e-311cdea7f214">

presentation Layer에서 Port를 사용하지 않고, Entity가 순수 Pure Java Object가 아니라 Jpa Entity이긴 하지만 큰 문제는 되지 않을 것 같습니다.

오히려 정석 Hexagonal Architecture를 추구하기 위해 그렇게 모든 것을 따라 하는 것이 과투자가 될 것이라고 생각하여 이 정도로 마무리 했습니다.

마찬가지로 Service도 인터페이스/구현체를 따로 두지 않았습니다.

굳이 clean architecture 방식으로 표현해도 얼추 들어 맞습니다.

<img width="837" alt="image" src="https://github.com/user-attachments/assets/71263d32-e548-48dd-9921-50dcec669f7c">

이렇듯 아키텍처가 고도화 될수록 테스트하기 좋은 구조를 띄고, application 모듈은 독립적으로, 의존성은 외부에서 내부로 향하는 것은 동일하다는 것에 초점을 맞춰서

입맛에 맞게 적당히 고도화를 진행했습니다.

infrastructure는 IPC가 필요하므로 Repository만 Interface를 따로 두고 Controller, Service는 굳이 만들지 않았습니다.

이렇게 만들 경우 런타임에 Repository에 대한 구현체를 Fake 객체로 갈아낄 수 있기 때문에 IPC가 필요 없어지고 Unit Test 화가 가능했습니다.

<img width="761" alt="image" src="https://github.com/user-attachments/assets/c890562a-a044-4a98-82ba-e07d866f75db">

그러나 여전히 Service에서 Repository를 접근해야 했기 때문에 단기간에 Fake Repository들을 만들어야 하여 시간 소요가 오래 걸린다는 점과

infrastructure 접근을 제외한 오직 비즈니스 로직 만을 테스트 하기 어렵다는 단점이 존재했습니다.

그래서 Service를 2개로 나누어 하나는 오직 비즈니스 로직만 담당하고, 나머지 하나는 앞선 Service에 비즈니스 로직 위임 및 infrastructure 호출을 담당하도록 하였습니다.

<img width="663" alt="image" src="https://github.com/user-attachments/assets/8726f333-b93c-4e97-a373-195cb6eed0a3">

대략 위와 같은 그림이 되었는데, 초반에는 이미 이러한 구조를 채택하지 않았기 때문에 모든 application Layer를 갈아 엎기보다는 실험적으로 프로젝트 일부에만 적용하였습니다.

Review 기능을 작성할 때 함께 논의하며 도입했던 것이라 `ReviewFacade`, `ReviewService`에만 적용된 형태입니다.

이에 대한 자세한 구현 코드 및 설명은 아래 PR 내용을 옮겨 두었습니다.

`ReviewService`를 infra 접근/비즈니스 로직에 따라 나누지 않을 경우, 수많은 의존성을 주입할 필요가 있어 테스트 코드가 핵심 외의 코드로 덮여서 가독성이 낮아지는 것을 볼 수 있습니다.

https://github.com/Ludo-SMP/ludo-backend/pull/111

---

# 1. 테스트 속도 최적화(실험적)

- 직전까지는 Service Layer 테스트 시, Infrastructure Layer를 의존하고 있기 때문에, `SpringBootTest`를 통해 `ApplicationContext`를 로드하는 시간 + 외부 Node에 접근할 필요가 있었기 때문에 로딩 및 I/O 비용이 크게 소모 되는 구조였습니다.

```java
// ReviewServiceTest.java
@SpringBootTest // ApplicationContext 로드 필요
@Transactional
@ActiveProfiles("test")
public class ReviewServiceTest {
    // 다양한 Infrastructure에 대한 의존성 주입 필요
    @Autowired
    private ReviewService reviewService;

    @Autowired
    private UserRepositoryImpl userRepository;

    @Autowired
    private CategoryRepositoryImpl categoryRepository;

    @Autowired
    private StudyRepositoryImpl studyRepository;

    @Autowired
    private PositionRepositoryImpl positionRepository;

    @Autowired
    private StackRepositoryImpl stackRepository;

    @Autowired
    private StackCategoryRepositoryImpl stackCategoryRepository;

    @Autowired
    private RecruitmentRepositoryImpl recruitmentRepository;

    @Autowired
    private RecruitmentService recruitmentService;

    @Autowired
    private ParticipantRepositoryImpl participantRepository;

    @Autowired
    private StudyApplicantDecisionService studyApplicantDecisionService;

}
```

![integrationTest](https://github.com/Ludo-SMP/ludo-backend/assets/121966058/861d6b2d-c2ab-44e4-a14b-dff2faf6332d)

이 시점에서 `ReviewServiceTest`를 테스트하는 데 약 17초 정도 소요됩니다.

<img width="741" alt="image" src="https://github.com/Ludo-SMP/ludo-backend/assets/121966058/57d7d681-fa31-40b4-8034-1ea2b0b2ddbc">

대략 이러한 형태이기 때문에 unit test가 아닌 integration test로 분류되어야 하며, 속도가 느릴 수밖에 없는 구조입니다.

여기서 mocking을 사용하여 실제 I/O를 제거하면 테스트 시간이 다음과 같이 줄어들었습니다.

![unit_springbootall](https://github.com/Ludo-SMP/ludo-backend/assets/121966058/db367c8f-3097-4ea4-b1e2-e5b2c955c2a6)

Test Results 결과만 보면 I/O 수행 시는 1.318sec,  이를 배제하면 454ms로 거의 3분의 1 수준이 되었지만,
Context load 시간은 그대로이기 때문에 14초에 가까운 시간이 소요되었습니다. (17, 14초는 손으로 타이머를 start, stop 하면서 재서 정확하지 않고 수행 시간만 비교하면 사실상 1초 정도 차이인 것 같습니다.)

여기까지가 어제 이야기를 나눴던 부분 중 ApplicationContext load로 인한 테스트 비용을 측정한 것이며, 아카님이 염려하던 부분이라고 생각됩니다.

실제 의존성을 필요로 하는 mocking의 단점이라고 볼 수도 있겠지만, 이를 위해 fake stub을 만드는 것은 또 다른 시간적 비용이 들기 때문에 둘 다 사용하지 않고 load 시간을 줄이는 방법을 생각해봤습니다.

해결 방법에 대해서 생각하다가 클린 아키텍처에서 영감을 받았는데,
Clean Architecture던, Hexagonal Architecture던 Domain을 외부와 완전히 격리하여 전혀 영향을 받지 않고 독립적인 모듈로서 사용될 수 있다는 부분은 동일합니다.

즉, 이 부분이 비슷하지만 조금은 다른 두 아키텍쳐의 핵심이라고 볼 수 있습니다.

현재 저희 Service는 전부 Infrastructure Layer에 의존하는 형태이며, Service나 Infrastructure를 interface로 만드는 DIP를 적용하지 않았기 때문에 mocking 이외에는 infrastructure를 의존하지 않을 방법이 존재하지 않습니다.

`ReviewService`를 보면 그러한 구조가 여실히 드러납니다.

```java
// ReviewService.java
@Service
@Transactional
@RequiredArgsConstructor
public class ReviewService {

    // infrastructure의 구현체에 직접적으로 의존
    private final ReviewRepositoryImpl reviewRepository;
    private final StudyRepositoryImpl studyRepository;
    private final LocalDateTimePicker localDateTimePicker;

    public ReviewResponse write(WriteReviewRequest request, Long studyId, User reviewer) {
        // infra 접근1
        final Study study = studyRepository.findByIdWithParticipants(studyId)
                .orElseThrow(() -> new IllegalArgumentException("존재하지 않는 스터디입니다. 리뷰를 작성할 수 없습니다."));

        final Participant participantingReviewer = study.findParticipant(reviewer.getId())
                .orElseThrow(() -> new IllegalArgumentException("참여 중인 스터디가 아닙니다. 리뷰를 작성할 수 없습니다."));
        final Participant participantingReviewee = study.findParticipant(request.revieweeId())
                .orElseThrow(() -> new IllegalArgumentException("스터디에 참여 중인 사용자에게만 리뷰를 작성할 수 있습니다."));
        if (participantingReviewer == participantingReviewee) {
            throw new IllegalArgumentException("자기 자신에게는 리뷰를 작성할 수 없습니다.");
        }

        // 명세 - [스터디가 진행 완료 상태로 바뀐 후 3일 후 부터, 14일 간 리뷰 작성 가능]
        // 진행 완료 상태로 바뀐 뒤 날짜를 정확히 추적하기 위해서는 추가적인 state 및 논의 필요할듯
        // -> 일단 EndDateTime 기반으로 작성
        study.ensureReviewPeriodAvailable(localDateTimePicker);

        // infra 접근2
        final boolean isDuplicateReview = reviewRepository.findAllByStudyId(studyId)
                .stream().anyMatch(review -> review.isDuplicateReview(studyId, reviewer.getId(), request.revieweeId()));
        if (isDuplicateReview) {
            throw new IllegalArgumentException("이미 리뷰를 작성 하셨습니다.");
        }

        final Review review = request.toReviewWithStudy(study, participantingReviewer.getUser(), participantingReviewee.getUser());
        // infra 접근3
        return ReviewResponse.from(reviewRepository.save(review));

    }

}
```

이런식으로 infra에 의존하는 코드가 로직 속에 혼재되어 있기 때문에 테스트 시 다음과 같이 수많은 의존성을 필요로 합니다.

```java
// ReviewServiceTest.java
@SpringBootTest
@Transactional
@ActiveProfiles("test")
public class ReviewServiceTest {

    @Autowired
    private ReviewService reviewService;

    @Autowired
    private UserRepositoryImpl userRepository;

    @Autowired
    private CategoryRepositoryImpl categoryRepository;

    @Autowired
    private StudyRepositoryImpl studyRepository;

    @Autowired
    private PositionRepositoryImpl positionRepository;

    @Autowired
    private StackRepositoryImpl stackRepository;

    @Autowired
    private StackCategoryRepositoryImpl stackCategoryRepository;

    @Autowired
    private RecruitmentRepositoryImpl recruitmentRepository;

    @Autowired
    private RecruitmentService recruitmentService;

    @Autowired
    private ParticipantRepositoryImpl participantRepository;

    @Autowired
    private StudyApplicantDecisionService studyApplicantDecisionService;

    @Autowired
    private ParticipantService participantService;

{
  // ... 생략
}
```

그리고  각 테스트 케이스에서도 다음과 같이 JPA 엔티티가 복잡하게 연관된 경우 이들 다수를 적절히 초기화 해야 합니다.

```java
// ReviewServiceTest.java
@DisplayName("[Success] 스터디가 종료된지 3일 ~ 14일 내에 리뷰를 작성할 수 있다.")
    @ParameterizedTest
    @MethodSource("validEndDateTimes")
    void writeReview(final LocalDateTime endDateTime) {

        // given
        final User owner = userRepository.save(
                User.builder()
                        .nickname("owner")
                        .email("a@aa.aa")
                        .social(Social.KAKAO)
                        .build());

        final Category category = categoryRepository.save(
                CategoryFixture.createCategory("category1"));

        final LocalDateTime now = FixedLocalDateTimePicker.DEFAULT_FIXED_LOCAL_DATE_TIME;

        final LocalDateTime startDateTime = now.minusDays(20);

        final Study study = saveStudy(category, owner, startDateTime, endDateTime);
        final StackCategory stackCategory = stackCategoryRepository.save(
                StackCategoryFixture.createStackCategory("stackCategory")
        );

        final Stack stack = stackRepository.save(
                StackFixture.createStack("stack", stackCategory)
        );
        final Position position = positionRepository.save(
                PositionFixture.createPosition("position")
        );

        final User userA = userRepository.save(
                User.builder()
                        .nickname("userA")
                        .email("userA@aa.aa")
                        .social(Social.KAKAO)
                        .build());
        final User userB = userRepository.save(
                User.builder()
                        .nickname("userB")
                        .email("userB@aa.aa")
                        .social(Social.KAKAO)
                        .build());


        participantService.add(study, userA, position, Role.MEMBER);
        participantService.add(study, userB, position, Role.MEMBER);

        final Participant participantB = study.getParticipant(userB);

        final WriteReviewRequest request = WriteReviewRequest.builder()
                .revieweeId(participantB.getUser().getId())
                .activenessScore(5L)
                .professionalismScore(5L)
                .communicationScore(5L)
                .togetherScore(5L)
                .recommendScore(5L)
                .build();


        // when
        final ReviewResponse review = reviewService.write(request, study.getId(), userA);

        // then
        assertThat(review.reviewer().email())
                .isEqualTo(userA.getEmail());

    }
```

덕분에 load 시간 및 I/O 시간이 모두 극한으로 소요되는 형태입니다.

I/O 시간은 mock으로 줄인다고 치고, load 시간을 줄일 방법은 간단합니다.

로드할 모든 의존성을 명시해주면 됩니다.

이러면 필요 없는 의존성들은 가져오지 않을 것입니다.

물론 내부적으로 사용되는 JPA 관련 의존성 등도 알아서 잘 명시해줘야 할 것입니다.

```java
// ReviewServiceTest.java
@SpringBootTest(classes={
  ReviewService.class,
  UserRepositoryImpl.class,
  CategoryRepositoryImpl.class,
  StudyRepositoryImpl.class,
  PositionRepositoryImpl.class,
  StackRepositoryImpl.class,
  StackCategoryRepositoryImpl.class,
  RecruitmentRepositoryImpl.class,
  RecruitmentService.class,
  ParticipantRepositoryImpl.class,
  StudyApplicantDecisionService.class,
  ParticipantService.class
})
```

mock으로 I/O 시간을 잡고, 이 방법으로 로딩 시간까지 잡을 수 있겠지만, 의존성이 많아질수록 사용하기 불편해질 것이라는 생각이 들었습니다.

그래서 앞서 언급했던 clean, hexagonal architecture에서 핵심으로 내세우는 Domain 격리를 시험삼아 적용해보았습니다.

아이디어는 간단합니다.

현재 `ReviewService`에서 infrastructure 접근 로직과 비즈니스 로직을 2개의 클래스로 분리합니다.

비즈니스 로직을 처리하는 클래스를 `ReviewService`로 만들고,
infra 접근 로직을 담당하는 클래스는 실제 로직은 위임한다고 하여 Facade라 명명했습니다.

코드는 다음과 같습니다.

```java
// ReviewFacade.java
@Service
@Transactional
@RequiredArgsConstructor
public class ReviewFacade {

    // infra layer 관련 의존성
    private final ReviewRepositoryImpl reviewRepository;
    private final StudyRepositoryImpl studyRepository;
    private final LocalDateTimePicker localDateTimePicker;
    private final ReviewService reviewService;

    public ReviewResponse write(WriteReviewRequest request, Long studyId, User reviewer) {
        // 모든 infra 관련 로직은 facade에서 처리
        final Study study = studyRepository.findByIdWithParticipants(studyId)
                .orElseThrow(() -> new IllegalArgumentException("존재하지 않는 스터디입니다. 리뷰를 작성할 수 없습니다."));

        if (reviewRepository.existsBy(studyId, reviewer.getId(), request.revieweeId())) {
            throw new IllegalArgumentException("이미 리뷰를 작성 하셨습니다.");
        }

        // 실제 비즈니스 로직은 ReviewService에 위임
        final Review review = reviewService.write(request, study, reviewer);
        return ReviewResponse.from(reviewRepository.save(review));

    }

}

// 실제 비즈니스 로직 담당
// ReviewService.java
@Service
@Profile("test")
@Transactional
@RequiredArgsConstructor
public class ReviewService {

    private final LocalDateTimePicker localDateTimePicker;

    public Review write(WriteReviewRequest request, Study study, User reviewer) {
        final Participant participantingReviewer = study.findParticipant(reviewer.getId())
                .orElseThrow(() -> new IllegalArgumentException("참여 중인 스터디가 아닙니다. 리뷰를 작성할 수 없습니다."));
        final Participant participantingReviewee = study.findParticipant(request.revieweeId())
                .orElseThrow(() -> new IllegalArgumentException("스터디에 참여 중인 사용자에게만 리뷰를 작성할 수 있습니다."));
        if (participantingReviewer == participantingReviewee) {
            throw new IllegalArgumentException("자기 자신에게는 리뷰를 작성할 수 없습니다.");
        }

        // 명세 - [스터디가 진행 완료 상태로 바뀐 후 3일 후 부터, 14일 간 리뷰 작성 가능]
        study.ensureReviewPeriodAvailable(localDateTimePicker);
        return request.toReviewWithStudy(study, participantingReviewer.getUser(), participantingReviewee.getUser());

    }

}
```

이렇게 두 부분으로 나눈 뒤, `ReviewService`의 Test를 다음과 같이 작성할 수 있습니다.

`ReviewService`는 이제 infra와 전혀 관련이 없는 순수 자바코드이기 때문에, 의존성을 대폭 줄일 수 있습니다.

persist조차 하지 않기 때문에 JPA 관련 의존성 및 트랜잭션을 고려할 필요가 없어집니다.

이러한 구성을 시험해보면서 Persistence Context 자체를 코드와 머릿속에서 지워버리니, JPA Entity를 순수한 Domain Entity처럼 다루는 느낌이었습니다.

```java
// 실제로 사용할 의존성만 로드
@SpringBootTest(classes = {ReviewService.class, FixedLocalDateTimePicker.class})
@ActiveProfiles("test")
public final class ReviewServiceTest {
    // 수많은 모든 의존성이 필요 없어짐
    @Autowired
    private ReviewService reviewService;

 @DisplayName("[Success] 스터디가 종료된지 3일 ~ 14일 내에 리뷰를 작성할 수 있다.")
    @ParameterizedTest
    @MethodSource("validEndDateTimes")
    void writeReview(final LocalDateTime endDateTime) {
        // given
        final User reviewer = User.builder()
                .id(1L)
                .nickname("reviewer")
                .email("reviewer@aa.aa")
                .build();
        final User reviewee = User.builder()
                .id(2L)
                .nickname("reviewee")
                .email("reviewee@aa.aa")
                .build();
        final Study study = Study.builder()
                .owner(reviewer)
                .endDateTime(endDateTime)
                .build();
        final Position position = Position.builder()
                .name("position")
                .build();
        study.addParticipant(reviewer, position, Role.OWNER);
        study.addParticipant(reviewee, position, Role.MEMBER);

        final WriteReviewRequest request = WriteReviewRequest.builder()
                .revieweeId(reviewee.getId())
                .activenessScore(5L)
                .professionalismScore(5L)
                .communicationScore(5L)
                .togetherScore(5L)
                .recommendScore(5L)
                .build();


        // when
        final Review review = reviewService.write(request, study, reviewer);

        // then
        assertThat(review.getReviewer()).isEqualTo(reviewer);
        assertThat(review.getReviewee()).isEqualTo(reviewee);

    }

}
```

여기서 약간의 변경으로 Application Context 자체를 지워버릴 수도 있습니다.

이러면 Junit의 `@Test` 등을 제외하면 어떠한 Annotation도 필요하지 않게 됩니다.

test 환경과 그 외 환경에서 구현체가 교체되어야 하는 경우에도 ActiveProfile를 설정할 필요 없이 직접 `FixedLocalDateTimePicker`를 넣어주면 됩니다.

```java
public final class ReviewServiceTest {

    private final ReviewService reviewService = new ReviewService(new FixedLocalDateTimePicker());

}
```

![unit_nodeps](https://github.com/Ludo-SMP/ludo-backend/assets/121966058/4381c677-6ad7-476e-8757-657557208e73)

이를 적용한 결과, 테스트 시간은 1초 이내로 완료 되었습니다.

실험적인 방법이기는 하나 `ReviewTest`에 적용하는 데는 문제가 없었습니다. 이와 비슷한 류의 좀 더 정제된 방법론이 있다면 좀 더 찾아보는 것이 좋을 것 같습니다.

만약 이러한 구조를 적용한다면, Service를 순수 Domain, Infra Access 2가지 역할로 나누고, Infra에 Access하는 Facade에서만 mocking을 적용하면 괜찮지 않을까 싶네요.

<br><br>

---

# 2. IntegrationTest 프로젝트에 적용

어제 이야기를 나눴던 대로 src 폴더 밑에 integrationTest를 두었습니다.

폴더명은 휴님이 올리신 참고 자료를 그대로 차용했습니다.

<img width="264" alt="image" src="https://github.com/Ludo-SMP/ludo-backend/assets/121966058/6d744da6-b900-49fd-84ed-bab078f0e926">

플러그인 및 명령어를 추가했습니다.

```
// build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.1'
    id 'io.spring.dependency-management' version '1.1.4'
    // 플러그인 추가
    id 'org.unbroken-dome.test-sets' version '4.1.0'
}

// ... 생략

// 명령어 추가
tasks.withType(Test) {
    useJUnitPlatform()
}

testSets {
    integrationTest
}
```

---

