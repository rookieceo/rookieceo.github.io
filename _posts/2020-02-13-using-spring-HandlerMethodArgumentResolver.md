---
published: true
date: '2020-02-13 18:30:00 +0900'
categories: springboot
title: RestController에서 HandlerMethodArgumentResolver 활용해 컨트롤러 파라미터 변경해보기
tags:
  - springboot
  - ArgumentResolver
  - HandlerMethodArgumentResolver
  - PathVariable
toc_sticky: true
toc: true
toc_label: 요약
last_modified_at: '2020-02-18 12:54:00 +0900'
---
잘 사용하던 Rest Controller에 신규 권한 체크가 필요했다.

Rest API에 @PathVariable로 전달되는 uri값을 이용하여 특정 서비스(DB 조회, 편의상 'A'서비스)를 호출해 현재 접근 가능한 API인지 체크해야 하고 해당 로직이 여러 API에서 작동해야했다.

처음에 AOP를 활용하여 비즈니스 로직에 영향을 주지 않는 범위에서 로직을 넣고자 했다.

커스텀 어노테이션(@Authorised)을 선언하고 이를 포인트 컷으로 지정해 체크하면 되겠다고 생각해서 코드를 넣다가 아쉬운 점을 발견했다.

어떤 API(편의상 '가' API)의 경우 권한 체크에 사용했던 'A'서비스로 리턴되는 DTO(편의상 'A'DTO)를 JSON으로 출력한다.
때문에 '가' API 같은 경우에는 'A' 서비스를 2번이나 호출하게 된다.

맘에 좀 안 든다. 

그래서 RestController에서 전달 받은 PathVariable 값을 기준으로 'A'서비스 호출 권한 체크를 하고 이를 비지니스 로직에 넣어주는 것으로 해보려고 한다.

'가' API인 경우에 'A'DTO를 RestController의 인자 타입으로 변경하여 전달할 것이다.

인자 타입을 변경하려면 HandlerMethodArgumentResolver를 활용해야 한다.

글로 풀어 쓰니 조금 어려운데 요약하면 아래와 같다.

> 비즈니스 로직에 영향을 끼치지 않고 'A' 서비스의 호출결과 비교를 통해 특정 API의 권한을 체크한다.
> 어떤 API에는 'A' 서비스를 재사용하는 비즈니스 로직이 있다.
> HandlerMethodArgumentResolver를 이용해 2번 사용될 'A' 서비스를 한번만 사용한다.
> 테스트 해본다.

구현은 아래와 같이 진행한다.

1. Custom Annotation(@Authorised) 선언
2. Custom ArgumentResolver 정의(AuthorisedArgumentResolver)
3. 설정파일에 Custom ArgumentResolver추가(CustomMVCConfig)
4. Controller에서 사용(AController)
5. 테스트준비 - 테스트 데이터 저장(스프링부트 메인 어플리케이션)
6. 테스트준비 - Spring-Security 설정 (CustomWebSecurityConfig)
7. 테스트 - ControllerTest 코드(AControllerTest)

코드를 보자 

##### 1. Custom Annotation(@Authorised) 선언
```java 
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Authorised {
	String value() default "";
}
```
##### 2. Custom HandlerMethodArgumentResolver 정의
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
		// 1. Get dto at DB by PathVariable keyIndex
		ADTO dto = this.service.getBDTOByKeyIndex(keyIndex);
		if (dto == null) {
			throw new ResponseStatusException(HttpStatus.NOT_FOUND, "ADTO Not Found");
		}
		// 2. Get Login Spring Security User Object
		User loginUser = (User) ((Authentication) webRequest.getUserPrincipal()).getPrincipal();
		// 3. Compare dto, loginUser
		boolean isAuthorized = this.checkIfIsCurrentlyAuthorised(dto, loginUser);

		if (isAuthorized) {
            // if Parameter Type is ADTO, return transformed Object
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
##### 3. CustomMVCConfig(ArgumentResolver 추가)
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

##### 4. Controller에서 사용
```java
@RestController
@RequestMapping("/api")
public class AController {

	@Autowired
	private AService service;

	// Step1 - 원래 API : PathVariable Uri를 활용하여 A DTO 출력
	@GetMapping(value = "/type1/{keyIndex}")
	public ADTO type1API(@PathVariable("keyIndex") Long keyIndex) {
		return this.service.getADTOByKeyIndex(keyIndex);
	}

	// Step2 - 수정 1 권한 체크(ArgumentResolver) 추가 + AService 2번 호출
	@GetMapping(value = "/type2/{keyIndex}")
	public ADTO type2API(@Authorised("keyIndex") Long keyIndex) {
		// AuthorisedArgumentResolver에서 호출된 AService.getADTOByKeyIndex이 다시 호출됨
		return this.service.getADTOByKeyIndex(keyIndex);
	}

	// Step3 - 수정 2 권한 체크(ArgumentResolver) 추가 + AService 1번 호출
	@GetMapping(value = "/type3/{keyIndex}")
	public ADTO type3API(@Authorised("keyIndex") ADTO dto) {
		// AuthorisedArgumentResolver에서 호출된 AService.getADTOByKeyIndex 를 인자로 받아 이를 Json Result로 출력
		return dto;
	}

}
```

##### 5. 테스트준비 - 테스트 데이터 저장(스프링부트 메인 어플리케이션)
```java
@SpringBootApplication
public class CustomArgumentResolverExampleApplication implements CommandLineRunner {

	@Autowired
	private ARepository repository;

	public static void main(String[] args) {
		SpringApplication.run(CustomArgumentResolverExampleApplication.class, args);
	}

	@Override
	public void run(String... args) throws Exception {
		// keyIndex, owner 저장(1, 'user'+1)
		for (int i = 1; i < 30; i++) {
			this.repository.save(new AEntity((long) i, "user" + i));
		}
		this.repository.flush();
	}

}
```

##### 6. 테스트준비 - Spring-Security 설정 (CustomWebSecurityConfig)
```java
@Configuration
@EnableWebSecurity
public class CustomWebSecurityConfig extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.authorizeRequests()
			.antMatchers("/api/type1/**").permitAll() // type1 API는 모든 접근허용
			.anyRequest().authenticated();
	}
}
```

##### 7. 테스트준비 - ControllerTest 코드(AControllerTest)
```java
@SpringBootTest
class AControllerTest {

	private MockMvc mockMvc;

	@Autowired
	private WebApplicationContext ctx;

	@BeforeEach
	void setUp() throws Exception {
		this.mockMvc = MockMvcBuilders.webAppContextSetup(this.ctx)
			.addFilters(new CharacterEncodingFilter(StandardCharsets.UTF_8.toString(), true)) // 필터 추가
			.apply(springSecurity())
			.build();
	}

	@Test
	final void testT1API() throws Exception {
		this.mockMvc.perform(get("/api/type1/1"))
			.andExpect(MockMvcResultMatchers.status().isOk());
	}

	@Test
	@WithMockUser("user2")
	final void testT2API() throws Exception {
		// user2 라는 Mockuser를 통해 해당 API로 접근이 가능한지 확인한다.

		// 1. 실패 응답 확인 : 접근 불가
		this.mockMvc.perform(get("/api/type2/3"))
			.andDo(print())
			.andExpect(MockMvcResultMatchers.status().isForbidden());

		// 2. 정상 응답 확인 및 DTO 출력 : A Service 호출 결과 DTO가 맞는지 확인
		this.mockMvc.perform(get("/api/type2/2"))
			.andDo(print())
			.andExpect(MockMvcResultMatchers.status().isOk())
			.andExpect(jsonPath("$.keyIndex", is(2)))
			.andExpect(jsonPath("$.owner", is("user2")));
	}

	@Test
	@WithMockUser("user3")
	final void testT3API() throws Exception {
		// user3 라는 Mockuser를 통해 해당 API로 접근이 가능한지 확인한다.

		// 1. 실패 응답 확인 : 접근 불가
		this.mockMvc.perform(get("/api/type3/2"))
			.andDo(print())
			.andExpect(MockMvcResultMatchers.status().isForbidden());

		// 2. 정상 응답 확인 및 DTO 출력 : A Service 호출 결과 DTO가 맞는지 확인
		this.mockMvc.perform(get("/api/type3/3"))
			.andDo(print())
			.andExpect(MockMvcResultMatchers.status().isOk())
			.andExpect(jsonPath("$.keyIndex", is(3)))
			.andExpect(jsonPath("$.owner", is("user3")));
	}

}
```

끝.

전체 코드는 여기에 있다.
[https://github.com/rookieceo/CustomArgumentResolverExample](https://github.com/rookieceo/CustomArgumentResolverExample)
