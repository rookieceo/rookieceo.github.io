---
published: false
---
## RestController에서 HandlerMethodArgumentResolver 활용해 컨트롤러 파라미터 변경 해보기


어느날, 잘 사용하던 Rest Controller 권한 체크에 이슈가 생겼다.
Rest API에 @PathVariable로 전달되는 key값을 이용하여 특정 서비스(DB 조회, 편의상 'A'서비스)를 호출해 현재 접근 가능한 API인지 체크해야했다.
여러 API에서 해당 로직이 작동해야했다.
처음에 AOP를 활용하여 비즈니스 로직에 영향을 주지 않는 범위에서 로직을 넣고자 했다.
특정 어노테이션(@Authorised)을 선언하고 이를 포인트 컷으로 지정해 체크하면 되겠다고 생각해서 코드를 넣다가 아쉬운 점을 발견했다.
어떤 API(편의상 '가' API)의 경우 AOP 내에서 권한 체크에 사용했던  'A'서비스의 DTO를 JSON으로 출력한다.
때문에 '가' API 같은 경우에는 같은 서비스를 2번이나 호출된다.
맘에 좀 안 든다.
그래서 RestController에서 전달 받은 PathVariable 값을 기준으로 'A'서비스 호출 권한 체크를 하고 이를 비지니스 로직에 넣어주는 것으로 해보려고 한다.
'가' API인 경우에 bDTO를 RestController의 인자 타입으로 변경하여 전달할 것이다.
인자 타입이 없을 경우에는 

요약하면 아래와 같다.
1. Custom Annotation(@Authorised) 선언
2. Custom HandlerMethodArgumentResolver 정의(AuthorisedArgumentResolver)
3. Spring WebMVC에 Custom HandlerMethodArgumentResolver추가
4. RestController에서 사용

#### Custom Annotation(@Authorised) 선언
```java 
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Authorised {
	String value() default "";
}
```
#### Custom HandlerMethodArgumentResolver 정의
```java 
@Component
public class AuthorisedArgumentResolver implements HandlerMethodArgumentResolver {

	@Autowired
	private AService service;

	@Override
	public boolean supportsParameter(MethodParameter methodParameter) {
		return methodParameter.hasParameterAnnotation(Authorised.class);
	}

	@Override
	public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest,
		WebDataBinderFactory binderFactory) throws Exception {

		Authorised authorised = parameter.getParameterAnnotation(Authorised.class);
		String annValue = authorised.value();

		// PathVariable 값을 가져온다.
		Map<?, ?> pathVariableMap = (Map<?, ?>) webRequest.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);

		Long keyIndex = Long.valueOf(pathVariableMap.get(annValue).toString());
		// 1. Get BDTO at DB by PathVariable keyIndex
		BDTO bDTO = this.service.getBDTOByKeyIndex(keyIndex);
		if (bDTO == null) {
			throw new ResponseStatusException(HttpStatus.NOT_FOUND, "BDTO Not Found");
		}
		// 2. Get Login Spring Security User Object
		LoginUser loginUser = (LoginUser) ((Authentication) webRequest.getUserPrincipal()).getPrincipal();
		// 3. Compare bDTO, loginUser
		boolean isAuthorized = this.checkIfIsCurrentlyAuthorised(bDTO, loginUser);

		if (isAuthorized) {

			if (BDTO.class.isAssignableFrom(parameter.getParameterType())) {
				return bDTO;
			} else if (Long.class.isAssignableFrom(parameter.getParameterType()) ||
				long.class.isAssignableFrom(parameter.getParameterType())) {
				return keyIndex;
			} else {
				return pathVariableMap.get(annValue);
			}
		} else {
			throw new ResponseStatusException(HttpStatus.FORBIDDEN, "Access Denied");
		}
	}

	private boolean checkIfIsCurrentlyAuthorised(BDTO dto, LoginUser user) throws Exception {
		// DTO의 값과 로그인 유저의 권한을 체크
		return dto.getUserIndex() == user.getUserIndex();
	}
}

```
#### CustomMVCConfig(ArgumentResolver 추가)
```java 
@Configuration
public class CustomMVCConfig implements WebMvcConfigurer {
	@Autowired
	private AuthorisedArgumentResolver authorisedArgumentResolver;

	// 우리가 만든 ArgumentResolver를 추가한다.
	@Override
	public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
		argumentResolvers.add(this.authorisedArgumentResolver);
	}
}
```

#### Controller에서 사용


<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExNDI4NDk4ODIsMTA5MjgwNTczNCwtNj
IzNzY5NzU4LC0xMDEwNjE5OTcwLC0xODA2NTUxOTMyLC00ODQx
NzQ5MjksLTE5NDQ1NDA5OSwtMTkzODA1MTY5Nl19
-->