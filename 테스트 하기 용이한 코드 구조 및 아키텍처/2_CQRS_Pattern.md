# CQRS Pattern

CQRS는 ludo 팀 개발 초기부터 논의 되었지만, 실제 적용이 되지는 않았습니다.

처음 논의했을 때는 Repository, Service 계층에 CQRS를 모두 적용하는 것이었는데,

기존에 이미 Repository를 `UserRepository`, `UserRepositoryImpl`, `UserJpaRepository` 최소 3가지로 분리했기 때문에

여기에 CQRS를 적용하면 `UserQueryRepository`, `UserQueryRepositoryImpl`, `UserQueryJpaRepository`, `UserCommandRepository`, `UserCommandRepositoryImpl`, `UserCommandJpaRepository`

개발 초기부터 모든 Repository가 6가지로 늘어나는 것이 부담스러운 면이 있었습니다.

그래도 여러 번 논의했던 내용이기 때문에 이에 따른 이점 및 예시 코드를 정리해보겠습니다.

---

# CQRS 적용 시 테스트 이점

<img width="719" alt="image" src="https://github.com/user-attachments/assets/9aa324c5-f8a1-4fab-9e62-a0571eee2898">

`Service`, `Repository`에 모두 CQRS를 적용하면 위와 같은 모양이 됩니다.

JPA는 대부분의 경우 변경 감지를 통해 DB를 업데이트 하므로 `CommandRepository`는 추가하지 않았으며, 우선 Entity를 조회할 필요가 있기에 `CommandService`는 `QueryService`를 의존하게 됩니다.

만약 위 구조에서 `CommandService`를 테스트한다면 `QueryService`를 주입해야 하고, Unit Test화 하려면 `QueryRepository`를 Fake Object로 만들어서 넣어줘야 합니다.

<img width="1030" alt="image" src="https://github.com/user-attachments/assets/ec319717-3397-425b-98d2-944c6ebe6f1f">

앞선 글에서 도입했던 `Service`에서 infrastructure 접근 부분을 `Facade`로 분리하는 방식을 적용하였고, 레이어도 명확하게 표기하였습니다.

이제 `CommandFacade`에서 `QueryService`를 통해 Entity 조회를 담당하고, `CommandService`는 이미 조회한 Entity를 인자로 넘겨 비즈니스 로직 실행 및 변경 감지를 수행합니다.

이렇게 되면 `CommandService`에 대해 별도의 의존성 및 Fake Repository를 주입하지 않고도 Unit Test 작성이 가능하며, 필요한 Entity만 순수 객체로 생성할 수 있게 됩니다.

---

# Master-Slave Replication
또한 CQRS 패턴을 적용하여 DB 이중화 시에도 매우 손쉬운 적용이 가능해졌습니다.

<img width="1376" alt="image" src="https://github.com/user-attachments/assets/3cb8af3f-5bef-451c-aa0a-2c031a66a37b">

`QueryService`는 모든 Query를 Readonly인 Slave DB에  보내고, `CommandFacade`는 모든 Command를 Master DB에 보내게 됩니다.

Service Layer에서부터 목적지 DB가 정해져 있기 때문에 일관적이고 예측 가능합니다.

목적지 DB가 다르기 때문에 Query, Command Service는 각각 Slave, Master DB에 대한 connection을 얻어야 하는데,

Custom Transaction Annotation을 만들어서 이 부분을 해결할 수 있습니다.

`@QueryTransactional`, `@CommandTransactional` 등으로 나누어 각각 class annotation으로 달아주면 이미 Service 자체가 향하는 목적지가 같으므로, 여러 메소드에 별도로 표기할 필요 없이 한 번만 적용해도 되고

Annotation 이름이 명시적이라 혼동할 여지도 줄일 수 있습니다.

Custom Transaction Annotation에 대한 아이디어는 이전에 작성한 PR에 기재되어 있습니다.

해당 PR 내용을 일부 발췌하겠습니다.

---

# PR 내용

원본 링크

https://github.com/Ludo-SMP/ludo-backend/pull/116#issuecomment-2143887596


고생하셨습니다.😊

현재 프로토타입이라 NotificationService이 의존하는 것들이 너무 많아서 테스트하기 어려운 구조이고,

이를 개선하기 위해 CQRS를 적용하여 NotificQueryService, NotificCommandService로 분리하고,

NotificationService을 최상위에 두어 이 둘을 컴포지션으로 가져가는 방식을 말씀해주셨었는데,

이런 방식을 일관적으로 가져가면 depth가 달라서 의존 관계가 꼬일 가능성이 현저히 줄어들고,

Command, Query의 분리도 되면서 재사용성도 늘어날 것 같다는 생각이 들었습니다.

관심사 분리와 테스트가 간편해지는 장점도 있겠구요.👍

이전에 CQRS를 적용했을 때, Query, Command로 분리만 하고 그 위에 하나의 레이어를 더 둘 생각은 못했었는데,

재미있는 구조를 하나 알아가는 것 같네요.😆

이 부분에 대해 흥미를 느껴서 한 번 문제가 없을지 생각을 해봤는데, 현재 구조에서는 전혀 문제가 없을 것 같습니다.

다만, 생각을 해보니 이 구조에서 문제가 생기는 지점이 DB를 Master-Slave 구조로 분리했을 때라는 생각이 들었는데요,
(이 구조여서 생긴다기 보다는 DB 이중화여서 생기는 문제)

그래도 약간의 구조 변경을 통해서 해결할 수 있을 것 같아서 큰 문제는 안 될 것 같아요.

저희가 DB를 분리할지는 모르겠지만, 이번에 안하더라도 언젠가는 마주치게 될 문제라고 생각하여 미리 같이 알아두면 좋을 것 같아서 도표와 코드 예시를 작성해보도록 하겠습니다.

image
현재 구조에서는 XXXService가 Query, Command를 둘 다 하위 Service에 위임하는 구조입니다.

그래서 다음과 같은 코드를 작성하는 것이 가능하겠습니다.

// XXXService의 메소드
var something = xxxQueryService.find();
xxxCommandService.update(something);
그런데 만약 Master-Slave로 DB를 �분리하게 된다면, @Transaction이 내부적으로 EntityManagerFactory로부터 가져오는 Connection을 Master or Slave로 분리해야 합니다.

그렇다면 XXXQueryService-SlaveConnection, XXXCommandService-MasterConnection으로 묶는 것이 아니라,

윗단인 XXXService에서 Connection을 주입해주면 문제를 해결할 수 있겠다고 생각했습니다.

@Service
public class XXXService {
    private final XXXCommandService commandService;
    private final XXXQueryService queryService;


    // 각 Master, Slave에 대한 Connection을 가진 EntityManagerFactory를 Configration으로 수동 등록하여 가져온 것
    @Qualifier("masterEmf")
    private final EntityManagerFactory masterConnection;
    @Qualifier("slaveEmf")
    private final EntityManagerFactory slaveConnection;

    // Command
    public void doSomeCommand() {
        var tx = masterConnection.getTransaction();
        tx.begin();
        queryService.doSomeCommand();
        tx.commit();
    }

    // Query
    public Data doSomeQuery() {
        var tx = masterConnection.getTransaction();
        tx.begin();
        queryService.doSomeQuery();
        tx.commit();
    }
}
pseudo code로 표현하면 위와 같은데, 이렇게 tx.begin, tx.commit, tx.rollback을 사용하는 것은 JPA 방식이 아니기도 하고,

익셉션 헨들링을 수동으로 해줘야 하기 때문에 아래 2가지 방법이 더 좋을 것 같습니다.

@Configuration
@EnableTransactionManagement
@RequiredArgsConstructor
public class TransactionManagerConfig implements TransactionManagementConfigurer {
    // master용, slave용 Entity Manager Factory
    @Bean
    @Primary
    public PlatformTransactionManager masterTransactionManager() {
        return new JpaTransactionManager(masterEntityManagerFactory);
    }

    @Bean
    public PlatformTransactionManager slaveTransactionManager() {
        return new JpaTransactionManager(slaveEntityManagerFactory);
    }

    @Override
    public PlatformTransactionManager annotationDrivenTransactionManager() {
        // HashMap에 두 TxManager를 넣어줌
        Map<String, PlatformTransactionManager> transactionManagers = new HashMap<>();
        transactionManagers.put("master", masterTransactionManager());
        transactionManagers.put("slave", slaveTransactionManager());

        // 기본 트랜잭션 매니저는 Slave
        PlatformTransactionManager defaultTransactionManager = slaveTransactionManager();
        return new DelegatingPlatformTransactionManager(transactionManagers, defaultTransactionManager);
    }
}
설정이 잘 되는지 확인은 안해봤는데 대략 이런식으로 설정하면 다음과 같이 @Transactional에 인자로 master, slave를 전달하여
원하는 DB Connection을 고를 수 있게 되었습니다.

@Service
@RequiredArgsConstructor
public class XXXCommandService {

    private final XXXQueryService masterService;
    private final XXXQueryService queryService;

    @Transactional("master")
    public void doWrite() {
        // query, command를 섞더라도 둘 다 master db 연결 전파
    }

    @Transactional("slave")
    public Data doRead() {
        // slave db 연결
        return result;
    }
}
그러나 이것도 역시 리터럴을 그대로 사용하기 때문에 변경에 취약하고,

오타를 낼 가능성이 있어서 3번째 방법이 더 좋아 보입니다.

// Custom Transactional 어노테이션 작성
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface QueryTransactional {
}

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface CommandTransactional {
}

// Transactional Interceptor 구현(여기까지는 원래 사용하던 Transactional과 동일)
public class QueryTransactionalInterceptor extends TransactionInterceptorSupport {

    @Autowired
    public QueryTransactionalInterceptor(PlatformTransactionManager transactionManager) {
        setTransactionManager(transactionManager);
    }

    @Override
    public Class<? extends Annotation> getTransactionalAnnotationType() {
        return QueryTransactional.class;
    }
}

public class CommandTransactionalInterceptor extends TransactionInterceptorSupport {

    @Autowired
    public CommandTransactionalInterceptor(PlatformTransactionManager transactionManager) {
        setTransactionManager(transactionManager);
    }

    @Override
    public Class<? extends Annotation> getTransactionalAnnotationType() {
        return CommandTransactional.class;
    }
}

// 설정에서 `@QueryTransactional`, `@CommandTransactional`에 별개의 Transaction Manager를 주입하여
// 별개의 Connection을 가지도록 함
@Configuration
public class TransactionConfig extends AbstractTransactionManagementConfiguration {

    // 이전 예제에서 만들었던 TransactionManager와 동일
    @Bean
    public QueryTransactionalInterceptor queryTransactionalInterceptor() {
        return new QueryTransactionalInterceptor(slaveTransactionManager());
    }

    @Bean
    public CommandTransactionalInterceptor commandTransactionalInterceptor() {
        return new CommandTransactionalInterceptor(masterTransactionManager());
    }
}
그러면 실제 Application 레이어 코드는 다음과 같이 변합니다.

@Service
@RequiredArgsConstructor
public class XXXCommandService {

    private final XXXQueryService masterService;
    private final XXXQueryService queryService;

    @CommandTransactional
    public void doWrite() {
        // query, command를 섞더라도 둘 다 master db 연결 전파
    }

    @QueryTransactional
    public Data doRead() {
        // slave db 연결
        return result;
    }
}
Nestjs진영에서 Hibernate와 비슷한 TypeORM서도 @Transactional을 지원하지 않아서 이런 식으로 만들어서 써야하는데,

�Hibernate도 지원을 잘 해줘서 간단하게 Custom Transactional 설정하는 방법을 한 번 살펴볼 수 있었네요.

돌려보지는 않아서 돌아갈진 모르겠네요;; 이런식으로 할 수 있다 정도만 인지하면 좋을 것 같습니다.

마땅한 ORM이 없거나 메타 프로그래밍이 약한 언어들은 db.startTransaction( )에 인자로 콜백 함수를 넣는 원시적인 방법으로 사용하는 경우들도 꽤 있어서

Transactional과 비슷한 걸 구현할 수 없나 생각을 했었던 차라 새삼 Hibernate가 상당히 잘 만들어졌다는 것을 체감하게 되네요.
