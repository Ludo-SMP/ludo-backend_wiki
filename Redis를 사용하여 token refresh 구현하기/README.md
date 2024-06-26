## Redis를 사용하여 token refresh 구현하기

> 이 글은 팀 Ludo의 `휴`가 작성했습니다.

### JWT를 사용한 이유
세션 기반 인정 방식은 사용자의 로그인 정보를 서버 측에서 관리하기 때문에, 서버에 부하가 발생할 수 있습니다. 
그리고 저희는 소셜 로그인 REST API를 이용한 CSR 방식의 백엔드 서버를 개발했기 떄문에 무상태성을 유지하기 위해서는 세션 기반 인증 방식은 적절하지 않다고 판단 했습니다.

JWT를 사용하면 무상태성을 유지하면서 인증된 사용자의 자격을 증명할 수 있습니다. 즉, 사용자가 누구인지 기억할 필요 없이 토큰에 있는 정보에 접근 권한이 있는지만 체크하면 되는 것 입니다.
하지만 토큰이 탈취되면 사용자 정보를 그대로 제공하는 꼴이 될 수 있기 때문에 간접적 방어와 직접적 방어를 통해 위험을 최소화 하고자 했습니다.

---

### 간접적 방어
1. **공격의 파급을 최소화 하기 위해 소셜에서 받아오는 Token이 아닌 별도의 customToken을 생성하여 사용**
```java
@Component
public class JwtTokenProvider {

	@Value("${jwt.token.secret-key}")
	private String secretKey;

	@Getter
	@Value("${jwt.token.access-token-expiresin}")
	private String accessTokenExpiresIn;

	public String createAccessToken(final AuthUserPayload payload) {
		final Claims claims = createClaims(payload);
		return createToken(accessTokenExpiresIn, claims);
	}

	private String createToken(final String tokenValidityInSeconds, final Claims claims) {
		final Long tokenValidity = Long.parseLong(tokenValidityInSeconds);
		final Date expiresIn = createExpiresIn(tokenValidity);
		final Key signingKey = createSigningKey();
		return Jwts.builder()
				.setClaims(claims)
				.setExpiration(expiresIn)
				.signWith(signingKey)
				.compact();
	}
```

<br><br>

2. **회원 가입시 User entity에서 716L + 현재 id 값을 기본 닉네임에 포함하도록 하여 사용자 id의 외부 노출을 최소화**
```java
@Entity
@Table(name = "`user`")
@Getter
@SuperBuilder
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
@Slf4j
public class User extends BaseEntity {

	public static final String DEFAULT_NICKNAME_PREFIX = "스따-디 %d";
	public static final Long DEFAULT_NICKNAME_ID = 716L;
  
        public void setInitialDefaultNickname() {
		validateInitDefaultNickname();
		this.nickname = String.format(DEFAULT_NICKNAME_PREFIX, DEFAULT_NICKNAME_ID + id);
	}
```
<img width="1353" alt="스크린샷 2024-04-21 오전 10 25 02" src="https://github.com/Ludo-SMP/ludo-backend_wiki/assets/83931353/067648bb-fe6a-4d0a-90d1-5045158af05f">

<br>

### 직접적 방어
1. **`UserAgent`헤더와 `ClientIp`를 사용한 accesstoken refresh 로직 추가 (refreshToken 방식 X)**
    - 단순히 refreshToken을 적용하는 방법으로는 보안상의 이점 없이 accessToken을 연장 시키는 것 밖에 되지 않는다고 판단하여 특정 할 수 있는 사용자 식별 정보를 활용하는 방법을 채택했습니다.
- **UserAgent와 ClientIp 정보란?**
    - 클라이언트가 HTTP를 통해 어떤 요청을 보내면 HTTP header에 사용자 IP주소와 기기정보(Agent)가 담기게 됩니다.
    - ClientIp 주소는, 다양한 종류의 proxy를 고려하여 각 header를 전부 확인하는 작업이 필요합니다.
    - IPv6형식으로 받아오도록 구현했지만, 만약 IPv4형식으로만 IP주소를 얻길 원한다면 [Run]-[Configuration] Arguments VM에 설정을 걸어줄 수 있습니다.
```java
public class WebUtils {

	public static String getUserAgent(HttpServletRequest request) {
		return request.getHeader("User-Agent");
	}

	public static String getClientIp(HttpServletRequest request) {

		String ip = request.getHeader("X-Forwarded-For");

		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getHeader("x-real-ip");
		}
		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getHeader("x-original-forwarded-for");
		}
		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getHeader("Proxy-Client-IP");
		}
		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getHeader("WL-Proxy-Client-IP");
		}
		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getHeader("HTTP_X_FORWARDED_FOR");
		}
		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getHeader("HTTP_X_FORWARDED");
		}
		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getHeader("HTTP_X_CLUSTER_CLIENT_IP");
		}
		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getHeader("HTTP_CLIENT_IP");
		}
		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getHeader("HTTP_FORWARDED_FOR");
		}
		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getHeader("HTTP_FORWARDED");
		}
		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getHeader("HTTP_VIA");
		}
		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getHeader("REMOTE_ADDR");
		}
		if (ip == null || ip.length() == 0 || ip.equalsIgnoreCase("unknown")) {
			ip = request.getRemoteAddr();
		}

		return ip;
	}

}
```

<br>

2. **필터 단에서 사용자 권한이 필요한 모든 요청에서 검증 후 accessToken의 만료시간을 연장하도록 구현했습니다.**
    - 별도의 API로 관리하는 방법을 고민했지만, 사용자 경험과 개발 생산성 관점에서 득보다 실이 많은 방법으로 판단했습니다.
    - access 만료 시간인 30분 이내에 호출시 지속적으로 만료기간을 연장합니다.
    - refresh 만료 시간인 7일 초과시 로그아웃 처리 됩니다.
    - accessToken / UserAgent / ClientIp 검증을 진행하며 다른 정보로 접근한 경우 쿠키를 제거하여 로그아웃 처리 됩니다.

```java
@Override
protected void doFilterInternal(final HttpServletRequest request, final HttpServletResponse response,
                                final FilterChain filterChain) throws ServletException, IOException {
    if (!request.getMethod().equals("OPTIONS")) {
      final Optional<String> authToken = cookieProvider.getAuthToken(request);
      // 토큰이 null 이면 예외 response 를 반환한다.
      if (authToken.isEmpty()) {
        jwtExceptionHandler(response, HttpStatus.UNAUTHORIZED, "Authorization 쿠키가 없습니다.");
        return;
      }
      try {
        Claims claims = jwtTokenProvider.verifyAuthTokenOrThrow(authToken.get());
        final AuthUserPayload payload = AuthUserPayload.from(claims);
        // 사용자 agent / ip 검증
        userDetailsService.verifyUserDetails(payload.getId(), request);
        request.setAttribute(AUTH_USER_PAYLOAD, payload);
        // 토큰 갱신
        accessTokenRefresh(payload.getId(), response);
        // NotFoundException - 만료된 사용자 정보 검증 추가
      } catch (final AuthenticationException | NotFoundException e) {
        // 만료되거나 잘못된 토큰일 경우 예외 response 를 반환한다.
        cookieProvider.clearAuthCookie(response);
        jwtExceptionHandler(response, HttpStatus.UNAUTHORIZED, e.getMessage());
        return;
      }
    }
    filterChain.doFilter(request, response);
}
```

<br>

### 고려한 보안 사항
  - **중간자 공격(MITM Attack)**
    - **네트워크 통신을 조작하여 통신 내용을 훔쳐 보거나 조작하는 공격 유형**
    - 기존에 적용된 Secure 속성 사용으로 해결했습니다. HTTPS를 사용하면 모든 통신 내용은 암호화되기 때문에 공격자의 중간자 공격을 통한 토큰 탈취를 방지할 수 있습니다.

  - **XSS(Cross-Site Scripting)**
    - **대상 웹사이트에 악성 스크립트를 주입하여 비정상적인 동작을 실행시키는 공격 유형.**
    - HttpOnly 속성 사용으로 해결했습니다. HttpOnly 속성은 웹브라우저가 스크립트를 통한 Dom document.cookie 객체 접근을 허용하지 않도록 합니다. 즉 공격자의 XSS를 통한 토큰 탈취를 방지할 수 있습니다.
   
  - **세션 하이재킹(Session Hijacking)**
    - **TCP 의 고유한 취약점을 이용해 정상적인 접속을 빼앗는 공격 유형**
    - SameSite옵션을 Strict로 설정했습니다. Strict는 가장 보수적인 보안 정책으로 같은 도메인 사이에서만 쿠키 전달을 가능하게 합니다. 그리고 accessToken / userAgent 헤더 / clientIp 를 통한 유효한 사용자 검증을 진행하여 쿠키가 탈취되어 악용될 가능성을 최소화 했습니다.

  - **CSRF(Cross Site Request Forgery)**
    - **사용자가 자신의 의지와는 무관하게 공격자가 의도한 행위를 특정 웹사이트에 요청하도록 만드는 공격 유형**
    - 가장 많은 고민을 녹인 공격 유형입니다. 해결하기 위해 몇 가지 보안 요소를 추가했지만 위험을 최소화 했을 뿐, 완벽한 방어는 아니라고 생각됩니다.
       1. CORS 허용 도메인 지정
       2. 만료 시간: access 30분 / refresh 7일
       3. 로그인시 사용자 리프레시 데이터 갱신 (새로운 환경에서 로그인할 경우 대비)

---

### UserAgent헤더와 ClientIp의 저장소 선정
기존에 RDB에 저장하여 갱신하도록 구현했습니다. 그러나 사용자 10,000명을 가정하고 테스트를 진행한 결과 로컬 환경임에도 불구하고 15초의 시간이 소요되는 이슈가 발생했고, Redis도입을 고민하게 되었습니다. 

<br>

### 왜 Redis?
레디스는 key-value 쌍으로 데이터를 관리하는 데이터 스토리지입니다. in-memory로 데이터릴 관리하므로, 저장된 데이터가 영속적이지 않기때문에 데이터베이스틑 아니라고 생각합니다.

데이터가 HDD나 SSD가 아니라 RAM에 저장되므로 영구적으로 저장할 수 없는 대신, 굉장히 빠른 엑세스 속도를 보장받을 수 있습니다. 빠른 엑세스 속도로 리프레시 로직 수행시 병목이 되지 않습니다.

또한 리프레시를 위한 데이터는 일정 기간 이후 만료되어야 합니다. RDB 저장시 스케줄러등을 사용하여 주기적으로 만료된 토큰을 제거해주는 기능을 구현해야 하지만 레디스는 기본적으로 데이터의 유효기간(Time To Live)을 지정할 수 있습니다. 즉, 해당 공수를 추가 기능 구현으로 돌릴 수 있습니다.

<br>

### Redis의 단점
최악의 경우 레디스 서버가 내려가 사용자의 리프레시 데이터가 소실되어 모든 회원들이 로그아웃처리 될 수 있습니다. 그러나 추후 구현된 기능의 캐시 적용 이라는 확장성까지 고민했을때 단점보다 장점이 크다고 판단하여 도입을 결정하게 되었습니다.

<br>

### build.gradle
```java
dependencies {
    // redis
    // Client 구현체는 크게 Lettuce, Jedis가 있는데, Lettuce로 구현 했습니다.
    // 이유는 별도의 의존성 설정 없이 사용할 수 있기 때문이고,
    // Jedis에 비해 더 높은 성능과 상세한 문서를 제공하기 때문입니다.
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
}
```

### RedisConfig
```java
@Configuration
public class RedisConfig {

	@Value("${spring.data.redis.host}")
	private String host;

	@Value("${spring.data.redis.port}")
	private Integer port;

	@Value("${spring.data.redis.password}")
	private String password;

	@Bean
	public RedisConnectionFactory redisConnectionFactory() {
		RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
		config.setHostName(host);
		config.setPort(port);
		config.setPassword(password);
		return new LettuceConnectionFactory(config);
	}

	@Bean
	public RedisTemplate<String, String> redisTemplate() {
		// redisTemplate 을 받아와서 set, get, delete 를 사용
		RedisTemplate<String, String> redisTemplate = new RedisTemplate<>();
		// setKeySerializer, setValueSerializer 설정
		// redis-cli 을 통해 직접 데이터 조회 시 알아볼 수 없는 형태로 출력되는 것을 방지
		redisTemplate.setKeySerializer(new StringRedisSerializer());
		redisTemplate.setValueSerializer(new StringRedisSerializer());
		redisTemplate.setConnectionFactory(redisConnectionFactory());
		return redisTemplate;
	}
}
```

### application.yml 설정
```java
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT}
      password: ${REDIS_PASSWORD}
```

### Domain
```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
@RedisHash(value = "user_details")
public class UserDetails {

	@Id
	private Long userId;

	private String userAgent;

	private String clientIp;

	@TimeToLive
	private Long ttl;

	public void update(final Long ttl) {
		this.ttl = ttl;
	}

}
```

### Repository
```java
public interface UserDetailsRepository extends CrudRepository<UserDetails, Long> {

}
```

---

### 서버구축
### Swap 메모리 설정
사용중인 GCP 프리티어의 경우 메모리가 1GB로 많은 사용자의 동시 접속 혹은 캐시 저장소로 활용시 매모리 부족으로 멈추거나 데이터가 소실될 수 있습니다. 이를 방지하기위해 스왑 메모리를 설정했습니다.
```java
// 메모리 상태 확인
$free -h
```

<img width="618" alt="스크린샷 2024-04-23 오전 2 39 01" src="https://github.com/Ludo-SMP/ludo-backend_wiki/assets/83931353/d2ad5efc-661b-44b2-9435-1da7cbe8a1a9">
<br>

현재 메모리 상태가 메모리만 1GB이고 스왑 메모리는 0B인 것을 확인할 수 있습니다. 아래의 명령어를 통해 스왑 메모리를 설정할 수 있습니다.

```java
// swap 파일을 생성해준다. (메모리 상태 확인 시 swap이 있었지만 디렉토리 파일은 만들어줘야한다.)
$ sudo mkdir /var/spool/swap
$ sudo touch /var/spool/swap/swapfile
$ sudo dd if=/dev/zero of=/var/spool/swap/swapfile count=2048000 bs=1024
// swap 파일을 설정한다.
$ sudo chmod 600 /var/spool/swap/swapfile
$ sudo mkswap /var/spool/swap/swapfile
$ sudo swapon /var/spool/swap/swapfile
// swap 파일을 등록한다.
$ sudo vim /etc/fstab
파일이 열리면 해당 파일 아래쪽에 하단 내용 입력 후 저장
- 입력 할 수 있도록 하는 명령어 -> if
- 파일 수정 후 저장하는 명령어-> esc키 누른 후 :wq 입력 후 엔터
/var/spool/swap/swapfile    none    swap    defaults    0 0

// 메모리 상태 확인
$free -h
```

<img width="607" alt="스크린샷 2024-04-23 오전 2 52 44" src="https://github.com/Ludo-SMP/ludo-backend_wiki/assets/83931353/52d1962f-868b-42d3-84a3-c8bd57570773">
<br>

이렇게 스왑 메모리 설정이 끝났습니다.

### Swap 메모리의 단점
레디스가 physical memory 이상을 사용시 swap 메모리를 사용하게 되며, page 접근 시 마다 읽고 쓰기 때문에 성능 하락이 발생할 수 있습니다. 그러나 현재 시나리오에서 동시 사용자들의 실시간 로그아웃 이슈를 막기 위한 최선의 방법이라고 생각합니다.

<br>

### Redis 설치
먼저 apt-get을 업데이트 합니다.
```java
$ sudo apt-get update
$ sudo apt-get upgrade
```

아래 명령어로 설치합니다.
```java
$ sudo apt-get install redis-server
```

설치가 완료되면 버전을 확인할 수 있습니다.
```java
$ redis-server --version
```

<img width="662" alt="스크린샷 2024-04-23 오전 3 14 07" src="https://github.com/Ludo-SMP/ludo-backend_wiki/assets/83931353/28801090-070d-4bff-bf5a-c120cda70e12">
<br>

<br><br>

### Redis 설정
redis.conf 파일을 열어서 Redis가 사용할 수 있는 최대 사용 메모리양을 정하고 최대 사용 메모리를 초과하게 될때 데이터를 어떻게 상제할지 정의하겠습니다.
```java
$ sudo nano /etc/redis/redis.conf
```

설정 파일에서 maxmemory와 maxmemory-policy를 찾아서 다음과 같이 바꿔줍니다. 최대 사용 메모리양은 2GB로 정하고, 최대 사용 메모리를 초과할 시 가장 오래된 데이터를 지워서 메모리를 확보하며 가장 최근에 저장된 데이터를 사용하는 것으로 설정했습니다.

<img width="497" alt="스크린샷 2024-04-23 오전 2 57 55" src="https://github.com/Ludo-SMP/ludo-backend_wiki/assets/83931353/b05b9155-2ed7-455d-9a36-4bda4ef829d5">
<br>

설정이 적용되도록 Redis를 재시작 합니다.
```java
$ sudo systemctl restart redis-server.service
```

Redis의 기본 포트는 6379 입니다. 6379 포트를 사용중인지 확인합니다.

<img width="636" alt="스크린샷 2024-04-23 오전 3 11 38" src="https://github.com/Ludo-SMP/ludo-backend_wiki/assets/83931353/0f92e44c-98d1-44e3-ae14-c722ba2d71c0">
<br>

---

### 저장 형식

![image](https://github.com/Ludo-SMP/ludo-backend_wiki/assets/83931353/5e2bd830-69e1-4987-a58d-421d8c69c402)
<br>
<br>

### RDB vs Redis 성능 차이 비교 테스트
사용자 10,000명을 기준으로 모든 사용자가 한 번씩 갱신 요쳥을 진행했을 때 비슷한 성능을 내지만, 1명의 사용자가 10,000번 갱신 요청힌 경우 Cache-hit로 약 80%의 성능 향상을 확인할 수 있습니다.

짧은 시간동안 여러번 발생할 수 있는 리프레시 요청 특성상 Redis도입으로 큰 이점을 얻게 되었습니다.

![image](https://github.com/Ludo-SMP/ludo-backend_wiki/assets/83931353/4a81cd32-ac69-4bb3-8c7d-dc25ffe7d748)
<br>


