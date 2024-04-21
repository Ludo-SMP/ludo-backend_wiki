## Redis를 사용하여 token refresh 구현하기

이 글은 팀 Ludo의 `휴`가 작성했습니다.

### JWT를 사용한 이유
세션 기반 인정 방식은 사용자의 로그인 정보를 서버 측에서 관리하기 때문에, 서버에 부하가 발생할 수 있습니다. 
그리고 저희는 소셜 로그인 REST API를 이용한 CSR 방식의 백엔드 서버를 개발했기 떄문에 무상태성을 유지하기 위해서는 세션 기반 인증 방식은 적절하지 않다고 판단 했습니다.

JWT를 사용하면 무상태성을 유지하면서 인증된 사용자의 자격을 증명할 수 있습니다. 즉, 사용자가 누구인지 기억할 필요 없이 토큰에 있는 정보에 접근 권한이 있는지만 체크하면 되는 것 입니다.
하지만 토큰이 탈취되면 사용자 정보를 그대로 제공하는 꼴이 될 수 있기 때문에 `공격의 파급을 최소화 하기 위해 소셜에서 받아오는 Token이 아닌 별도의 customToken을 생성하여 사용`했고, Auto Increment된 사용자 id와 만료 기간만 토큰에 담도록 했습니다.

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
<img width="1221" alt="스크린샷 2024-04-21 오전 10 09 05" src="https://github.com/Ludo-SMP/ludo-backend_wiki/assets/83931353/da977c1b-ad7c-4572-a3f6-aeb48053dd19">

그러나 사용자 id를 특정하여 예상치 못한 악의적인 공격이 들어올 수 있기에 간접적 방어와 직접적 방어를 통해 위험을 최소화 하고자 했습니다.

<br>

### 간접적 방어
회원 가입시 User entity에서 716L + 현재 id 값을 기본 닉네임에 포함하도록 하여 사용자 id의 외부 노출을 최소화 하고자 했습니다.
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


### 직접적 방어
1. **`UserAgent`헤더와 `ClientIp`를 사용한 accesstoken refresh 로직 추가 (refreshToken 방식 X)**
2. **단순히 refreshToken을 적용하는 방법으로는 보안상의 이점 없이 accessToken을 연장 시키는 것 밖에 되지 않는다고 판단하여 특정 할 수 있는 사용자 식별 정보를 활용하는 방법을 채택했습니다.**
3. **필터 단에서 사용자 권한이 필요한 모든 요청에서 검증 후 accessToken의 만료시간을 연장하도록 구현했습니다.**
    - 별도의 API로 관리하는 방법을 고민했지만, 사용자 경험과 개발 생산성 관점에서 득보다 실이 많은 방법으로 판단했습니다.
    - access 만료 시간인 30분 이내에 호출시 지속적으로 만료기간을 연장합니다.
    - refresh 만료 시간인 7일 초과시 로그아웃 처리 됩니다.
    - accessToken / UserAgent / ClientIp 검증을 진행하며 다른 정보로 접근한 경우 로그아웃 처리 됩니다.

UserAgent와 ClientIp 정보란?
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



(JWT 인증, 인가 과정 이미지)

