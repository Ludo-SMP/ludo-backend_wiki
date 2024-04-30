# Argument Resolver

사용자가 Client에서 fetch API를 통해 API Server와 소통하는 Endpoint는 Backend에서 Controller에 정의합니다.

그 중에서도 모든 사용자가 접근 가능한 Public Handler, Private Handler가 존재하는데,

다음과 같이 Study를 생성하기 위한 Private Handler는 로그인을 하지 않은 사용자의 경우 요청이 거부되어, 401 Unauthorized 응답을 받게 됩니다.

```java
// StudyController.java
public StudyResponse create(
    @RequestBody final WriteStudyRequest request,
    @Parameter(hidden = true) @AuthUser final User user
) {
    return studyCreateService.create(request, user);
}
```

Private, Public Endpoint를 구분하기 위해 Spring Security에서는 `@PreAuthorize` 등의 Annotation를 사용하며, Nestjs는 `@UseGuard`를 사용합니다.

그러나 현재 어플리케이션 상에서는 복잡한 인증 요구 사항이 아닌, 매우 단순하게 로그인 여부만 확인하면 되기 때문에, 따로 Spring Security 프레임워크를 적용하지 않았기 때문에 이러한 로직을 간단하게 작성하는 방향으로 가닥을 잡았습니다.

2번째 인자에 작성된 `@AuthUser` Annotation이 로그인 한 사용자가 아닐 경우 요청을 거부하는 Guard 역할을 하는데,

Nestjs의 경우는 보통 다음과 같이 class 혹은 Handler 상단에 Decorator(Java의 Annotation과 비슷)를 붙여주는 것과 달리, Handler의 인자에 적용하여 위치의 차이가 있습니다.

```typescript
// TYPESCRIPT

// class 단위로 적용
@Controller
@UseGuards(new RolesGuard())
export class Controller {}

// handler 단위로 적용
@Controller
export class Controller {
    
    @UseGuards(new RolesGuard())
    async create() {
        // ...
    }
}
```

가독성 면에서는 이렇게 class나 handler 상단에 Guard를 명시해주는 편이 좋을 것 같다고 생각이 드네요.

`@AuthUser` Annotation을 통한 인증 과정 구현은 매우 간단합니다.

```java
// AuthUser.java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface AuthUser {

}
```

우선 `@AuthUser` Annotation은 따로 값을 갖지 않은 그저 마커일 뿐입니다.
Handler의 인자에 붙여줄 것이기 때문에 Target은 Parameter로 지정하였으며,
Retention은 타입 정보의 지속 시간을 일컫는데 Annotation을 컴파일 타임까지만 지속하고 제거할 것인지, 런타임에도 유지할 것인지를 의미합니다.
어플리케이션 레이어에서의 Annotation의 용도는 런타임에 Reflection을 통한 타입 정보의 추출 및 로직 추가를 목적으로 하는 경우가 거의 전부이기 때문에
Runtime으로 설정하였습니다.

Retention을 SOURCE 혹은 CLASS로 설정하면 컴파일 작업 이후 타입 정보가 제거되기 때문에 `@AuthUser`라는 Annotation이 붙었는지 런타임에 확인이 불가능하게 됩니다.
그래서 주로 어플리케이션 단이 아닌, 정적 분석에 이용되기 때문에 Spring을 통한 어플리케이션 개발 시에는 거의 사용할 일이 없을 것 같습니다.

어쨌든 이제 특정 Parameter에 `@AuthUser` Annotation이 부착 되었는지 여부를 런타임에 확인할 수 있게 되었습니다.

이를 확인하는 방법도 매우 간단합니다.

Spring은 요청을 받고 DispatcherServlet, Controller에 진입하여 Handler를 실행하기 이전에 등록된 `ArgumentResolver`를 실행합니다.
Reflection을 통해 Handler의 다양한 인자 및 인자에 붙은 각종 Annotation을 분석하여 인자를 할당하기 위함입니다.

예를 들어 다음과 같은 매우 간단한 Handler가 있다고 가정하겠습니다.

```java
@GetMapping("/user/{id}")
public Response getUser(
    @RequestParam("id") Long id,
    HttpServletRequest req,
    HttpServletRequest res
) {
    // ...
}
```

인자의 위치나 개수, 종류에 무관하게 Spring은 적당한 값을 '알아서' 잘 할당해줍니다.

이는 Java + Spring의 협업 덕분에 가능한데,

JVM ClassLoader에 의해 Class에 대한 메타데이터가 메모리에 적재되어 `getUser` 메소드의 인자로 어떤 타입의 인자가 주어졌는지, 순서가 어떤지, 어떤 Annotation이 부착 되어 있는지 여부를 모두 데이터로 저장하게 됩니다.

런타임에 `INSTANCE.getClass()` 혹은 `CLASS.class`로 이러한 메타 데이터를 조회 및 조작하는 과정을 Reflection이라고 하는데,

Spring에 등록된 `ArgumentResolver`들이 Reflection을 통해 어떤 Handler가 어떤 타입의 인자를 받았으며, 어떤 전처리 과정이 필요한지를 모두 추적하여 로직을 실행해줍니다.

이제부터 만들 `ArgumentResolver`는 Handler의 인자에 `@AuthUser`가 부착된 경우, 해당 인자에 `User`를 DB로부터 찾아와서 할당해주는 로직을 실행합니다.

만약 `@AuthUser` Annotation이 없다면 로직을 실행하지 않고 스킵합니다.

클래스 선언부는 다음과 같습니다.

```java
// AuthUserResolver.java
@Component
@RequiredArgsConstructor
public final class AuthUserResolver implements HandlerMethodArgumentResolver {
    @Override
	public boolean supportsParameter(final MethodParameter parameter) {
    	// ...
	}

    @Override
	public User resolveArgument(
    	final MethodParameter parameter,
        final ModelAndViewContainer mavContainer,
        final NativeWebRequest webRequest,
		final WebDataBinderFactory binderFactory
	) {
    	// ...
	}
}
```

`HandlerMethodArgumentResolver`를 상속한 뒤, `supportsParameter`, `resolveArgument`를 적절하게 구현하면 됩니다.

`supportsParameter`는 Handler가 호출되었을 때, 입력된 모든 Parameter를 각각 인자로 받아서 실행되는 메소드이며,
`true`를 반환할 경우, `resolveArgument`를 호출하여 추가 로직을 실행합니다.

`resolveArgument`는 원하는 로직을 넣어주면 됩니다.

완성된 코드는 다음과 같습니다.

```java
// AuthUserResolver.java
@Component
@RequiredArgsConstructor
public final class AuthUserResolver implements HandlerMethodArgumentResolver {

	private final UserRepositoryImpl userRepository;

	@Override
	public boolean supportsParameter(final MethodParameter parameter) {
    	// Parameter에 @AuthUser Annotation을 부착한 경우 true
		final boolean hasAuthUserAnnotation = parameter.getParameterAnnotation(AuthUser.class) != null;
		// Parameter의 타입이 User인 경우 true
		final boolean isUser = User.class.isAssignableFrom(parameter.getParameterType());
		// 둘 다 만족해야 true -> resolveArgument 호출
		return hasAuthUserAnnotation && isUser;
	}

	@Override
	public User resolveArgument(
    	final MethodParameter parameter,
        final ModelAndViewContainer mavContainer,
		final NativeWebRequest webRequest,
        final WebDataBinderFactory binderFactory
    ) {
        // 현재 요청에 대한 webRequest를 HttpServletRequest로 변환
		final HttpServletRequest request = (HttpServletRequest)webRequest.getNativeRequest();
		// 로그인이 수행된 경우, 요청으로 받은 JWT를 Filter에서 디코딩하여 request의 Attribute에 Object 타입으로 저장해둠
		// 이를 가져와서 AuthUserPayload 타입으로 변환한 뒤, DB에서 해당 id를 가진 사용자를 찾아와서 User 타입으로 반환
		final AuthUserPayload payload = (AuthUserPayload)request.getAttribute(
				JwtAuthenticationFilter.AUTH_USER_PAYLOAD);

		return userRepository.getById(payload.getId());
	}
}
```

한 줄로 요약하면 Handler에 `@AuthUser` Annotation이 부착된 `User` 타입의 Argument가 존재하는 경우, 이를 검사하고 JWT에 등록된 id와 매칭되는 `User`를 DB에서 찾아서 해당 인자의 값으로 넣어줍니다.

Filter 로직은 뒤에서 이야기하도록 하겠습니다.
지금은 로그인 한 사용자가 요청을 보낼 때마다 Filter에서 Cookie에 포함된 JWT를 통해 해당 사용자의 `id`를 담은 `AuthUserPayload` 인스턴스를 만들어서 Request의 Attribute에 넣어둔다 까지만 인지하고 위 코드 및 주석을 읽어보면 도움이 될 것 입니다.

```java
// WebConfig.java
@Component
@RequiredArgsConstructor
public class WebConfig implements WebMvcConfigurer {

	private final AuthUserResolver authUserResolver;

	@Override
	public void addArgumentResolvers(final List<HandlerMethodArgumentResolver> resolvers) {
		resolvers.add(authUserResolver);
	}

    // ...
}
```

이제 `WebMvcConfigurer`에 `AuthUserResolver`를 등록해주면 설정이 완료됩니다.

---
