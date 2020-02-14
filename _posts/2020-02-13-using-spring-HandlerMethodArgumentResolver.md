---
published: false
---
## RestController에서 HandlerMethodArgumentResolver 활용해 컨트롤러 파라미터 변경 해보기


어느날, 잘 사용하던 Rest Controller의 권한 체크에 이슈가 생겼다.
Rest API에 전달되는 @PathVariable을 이용하여 특정 서비스(DB 조회, 편의상 'A'서비스)를 통해 현재 접근 가능한 API인지 체크해야했다.
여러 API에서 해당 로직이 작동해야했다.
처음에 AOP를 활용하여 비즈니스 로직에 영향을 주지 않는 범위에서 로직을 넣고자 했다.
특정 어노테이션(@Authorised)을 선언하고 이를 포인트 컷으로 지정해 체크하면 되겠다고 생각해서 코드를 넣다가 아쉬운 점을 발겼했다.
어떤 API(편의상 '가' API)의 경우 AOP 내에서 권한 체크에 사용했던  'A'서비스의 DTO를 JSON으로 출력한다.
때문에 '가' API 같은 경우에는 같은 서비스를 2번이나 호출된다.
맘에 좀 안 든다.
차라리 RestController에서 전달 받은 인자로 권한 체크를 하고 이를 비지니스 로직에 넣어주는 것으로 해보려고 한다.

요약하면 아래와 같다.

#### 어노테이션(@Authorised) 선언
```java 
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Authorised {
	String value() default "";
	// Default 로 타입을 Work 타입을 지정한다.(Lock 체크)
	EnumWorkStateType workStateType() default EnumWorkStateType.work;

	EnumWorkableState[] compareTo() default EnumWorkableState.WORKABLE;
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

		Map<?, ?> pathVariableMap = (Map<?, ?>) webRequest.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE,
			RequestAttributes.SCOPE_REQUEST);

		// Work 작업인경우의 권한 체크
		if (EnumWorkStateType.work == authorised.workStateType()) {

			Long KeyIndex = Long.valueOf(pathVariableMap.get(annValue).toString());
			// 1. Get BDTO at DB by PathVariable
			BDTO bDTO = this.service.getBDTOByKeyIndex(KeyIndex);

			if (bDTO == null) {
				throw new ResponseStatusException(HttpStatus.NOT_FOUND, "BDTO Not Found");
			}

			// 2. Get Login User Object
			LoginUser loginUser = (LoginUser) ((Authentication) webRequest.getUserPrincipal()).getPrincipal();

			// 4. Compare Authorised Work
			boolean isAuthorized = false;
			switch (loginUser.getUserRole()) {
			case Manager:
				isAuthorized = true;
				break;
			default:
				// 3. Current BDTO Workable State
				EnumWorkableState state = this.checkIfIsCurrentlyAuthorised(bDTO, loginUser);
				EnumWorkableState[] workableStateTarget = authorised.compareTo();
				isAuthorized = Arrays.stream(workableStateTarget).anyMatch(s -> s == state);
				break;
			}
			
			if (isAuthorized) {
				if (DiagnosisDTO.class.isAssignableFrom(parameter.getParameterType())) {
					return diagnosisDTO;
				} else if (Long.class.isAssignableFrom(parameter.getParameterType()) ||
					long.class.isAssignableFrom(parameter.getParameterType())) {
					return workIndex;
				} else {
					return pathVariableMap.get(annValue);
				}
			} else {
				throw new ResponseStatusException(HttpStatus.FORBIDDEN, "Access Denied");
			}
		}
		return pathVariableMap.get(annValue);
	}

	private EnumWorkableState checkIfIsCurrentlyAuthorised(BDTO dto, LoginUser user) throws Exception {
		EnumWorkableState result;

		log.info("dto.getWorkerUserIndex() : {}", dto.getWorkerUserIndex());
		log.info("dto.getLockedUserIndex() : {}", dto.getLockedUserIndex());
		log.info("user.getUserSystemIndex() : {}", user.getUserSystemIndex());

		if (dto.getWorkerUserIndex().longValue() == -1) {
			// 1) 작업이 미 할당된 상태 => 미 할당 UnAssignment
			result = EnumWorkableState.UN_ASSIGNMENT;
		} else if (dto.getLockedUserIndex().longValue() == -1 && dto.getWorkerUserIndex().longValue() == user.getUserSystemIndex()) {
			// 2) 작업자 할당 && 로그인 유저가 같고 작업시작이 되지 않았다면 => 작업 시작 전 상태 Assignment
			result = EnumWorkableState.ASSIGNMENT;
		} else if (dto.getLockedUserIndex().longValue() == dto.getWorkerUserIndex().longValue()
			&& dto.getWorkerUserIndex().longValue() == user.getUserSystemIndex()) {
			// 3) 작업자 할당 && 로그인 유저 && 작업이 진행 되었다면 => 작업 가능 상태 ==> Working
			result = EnumWorkableState.WORKABLE;
		} else {
			// 4) 해당 없으므로 아무액션 없음 : Lock
			result = EnumWorkableState.LOCKED;
		}
		log.info("checkIfIsCurrentlyAuthorisedWorkByWorkIndex result : {}", result);
		return result;
	}

}
```

#### Controller에서 사용


<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE5MzgwNTE2OTZdfQ==
-->