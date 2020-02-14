---
published: false
---
## RestController에서 HandlerMethodArgumentResolver 활용해 컨트롤러 파라미터 변경 해보기

어느날, 잘 사용하던 Rest Controller 권한 체크에 이슈가 생겼다.
Rest API에 @PathVariable로 전달되는 key값을 이용하여 특정 서비스(DB 조회, 편의상 'A'서비스)를 호출해 현재 접근 가능한 API인지 체크해야 하고 해당 로직이 여러 API에서 작동해야했다.
처음에 AOP를 활용하여 비즈니스 로직에 영향을 주지 않는 범위에서 로직을 넣고자 했다.
커스텀 어노테이션(@Authorised)을 선언하고 이를 포인트 컷으로 지정해 체크하면 되겠다고 생각해서 코드를 넣다가 아쉬운 점을 발견했다.
어떤 API(편의상 '가' API)의 경우 권한 체크에 사용했던  'A'서비스로 리턴되는 DTO(편의상 'A'DTO)를 JSON으로 출력한다.
때문에 '가' API 같은 경우에는 'A' 서비스를 2번이나 호출하게 된다.
맘에 좀 안 든다. 
그래서 RestController에서 전달 받은 PathVariable 값을 기준으로 'A'서비스 호출 권한 체크를 하고 이를 비지니스 로직에 넣어주는 것으로 해보려고 한다.
'가' API인 경우에 bDTO를 RestController의 인자 타입으로 변경하여 전달할 것이다.
인자 타입을 변경하려면 HandlerMethodArgumentResolver를 활용해야 한다.

요약하면 아래와 같다.

1. spring-security 설정
2. Custom Annotation(@Authorised) 선언
3. Custom ArgumentResolver 정의(AuthorisedArgumentResolver.java)
4. 설정파일에 Custom ArgumentResolver추가(CustomMVCConfig.java)
5. RestController에서 사용(AController.java)
6. 테스트 결과 확인(AControllerTest.java)

하나씩 

#### spring-security 설정
```java
@Configuration
@EnableWebSecurity
public class CustomWebSecurityConfig extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.authorizeRequests()
			.antMatchers("/api/type1/**").permitAll() // type1 접근허용
			.anyRequest().authenticated();
	}
}
```

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
@Slf4j
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
		ADTO dto = this.service.getBDTOByKeyIndex(keyIndex);
		if (dto == null) {
			throw new ResponseStatusException(HttpStatus.NOT_FOUND, "ADTO Not Found");
		}
		// 2. Get Login Spring Security User Object
		User loginUser = (User) ((Authentication) webRequest.getUserPrincipal()).getPrincipal();
		// 3. Compare bDTO, loginUser
		boolean isAuthorized = this.checkIfIsCurrentlyAuthorised(dto, loginUser);

		if (isAuthorized) {

			if (ADTO.class.isAssignableFrom(parameter.getParameterType())) {
				return dto;
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

	private boolean checkIfIsCurrentlyAuthorised(ADTO dto, User user) throws Exception {
		log.debug("dto.getKeyIndex() : {}", dto.getKeyIndex());
		log.debug("dto.getOwner() : {}", dto.getOwner());
		log.debug("loginUser.getUsername() : {}", user.getUsername());

		// DTO의 Owner값과 로그인 유저의 ID로 권한을 체크, 편의상 getUsername으로 비교
		return dto.getOwner().equals(user.getUsername());
	}

}

```
#### CustomMVCConfig(ArgumentResolver 추가)
```java 
@Configuration
public class CustomMVCConfig implements WebMvcConfigurer {
	@Autowired
	private AuthorisedArgumentResolver authorisedArgumentResolver;

	// 앞서 만든 ArgumentResolver를 추가한다.
	@Override
	public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
		argumentResolvers.add(this.authorisedArgumentResolver);
	}
}
```

#### Controller에서 사용
```java
@RestController
@RequestMapping("/api")
public class AController {

	@Autowired
	private AService service;

	// Step1 - 일반 PathVariable 활용 로직
	@GetMapping(value = "/type1/{keyIndex}")
	public ADTO type1API(@PathVariable("keyIndex") Long keyIndex) {
		return this.service.getADTOByKeyIndex(keyIndex);
	}

	// Step2 - AuthorisedArgumentResolver를 활용하여 비교 로직 -1
	@GetMapping(value = "/type2/{keyIndex}")
	public ADTO type2API(@Authorised("keyIndex") Long keyIndex) {
		// AuthorisedArgumentResolver에서 호출된 AService.getADTOByKeyIndex이 다시 호출
		return this.service.getADTOByKeyIndex(keyIndex);
	}

	// Step3 - AuthorisedArgumentResolver를 활용하여 비교 로직 -2
	@GetMapping(value = "/type3/{keyIndex}")
	public ADTO type3API(@Authorised("keyIndex") ADTO dto) {
		// AuthorisedArgumentResolver에서 호출된 AService.getADTOByKeyIndex 1번만 호출
		return dto;
	}

}

```

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExMzI0OTI5MzQsLTIwNTI3MjYwMzEsND
EyNzE1NTc3LDExODUzMzExOTcsLTE1NTc1NDcyMzEsMTA5Mjgw
NTczNCwtNjIzNzY5NzU4LC0xMDEwNjE5OTcwLC0xODA2NTUxOT
MyLC00ODQxNzQ5MjksLTE5NDQ1NDA5OSwtMTkzODA1MTY5Nl19

-->