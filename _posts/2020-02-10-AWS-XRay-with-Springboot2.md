---
title: AWS X-Ray 적용하기(using Spring AOP)
date: 2020-02-10 18:28:00 +0900
categories: aws x-ray xray springboot
---

### AWS X-Ray 적용하기(using Spring AOP)

Spring Boot Application에 AWS X-Ray를 적용해 보았다. 
아쉽지만 sql 트레이싱의 경우에는 아직 HikariCP 로는 적용할 수 있는 방법이 없어 라이브러리만 추가해두었다. X-Ray 버전이 업그레이드 되면 boot 2.x의 기본 DataSource인 Hikari도 지원되면 좋겠다. 

현재 적용한 내역은 아래와 같다. 

> 1. Front / Backend API가 분리되어 있으므로 /api/* 로 시작되는 영역만을 트레이싱한다.  
> 2. 필요없는 특정 API는 수집하지 않는다. 
> 3. X-Ray User 개념을 도입하여 사용자 ID를 기록한다. (로그인 한 유저의 호출 API 추적) 
> 4. 추후 문제 추적을 위해 X-Ray Trace ID를 어플리케이션 로그 파일에 함께 넣어준다.  

- 참고 자료 
  - 개념 : https://docs.aws.amazon.com/ko_kr/xray/latest/devguide/aws-xray.html
  - X-Ray SDK For Java 구성 : https://docs.aws.amazon.com/ko_kr/xray/latest/devguide/xray-sdk-java-configuration.html
  - AWS X-Ray 샘플 애플리케이션 : https://github.com/aws-samples/eb-java-scorekeep
  - 어느(?) 개발자 님(sykim) 블로그 (Spring Boot 프로젝트에 AWS X-Ray 적용하기 - 1 ~ 5)
    -  https://ccii83.blogspot.com/2019/04/spring-boot-aws-x-ray-1.html

- Enviroment
  - Springboot 2.2.4(with Spring Security)
  - Elastic Beanstalk(환경 > 구성 > 소프트웨어 > X-Ray데몬 활성화 체크하여야 함)
  - S3(AWS SDK For Java v2)

- 요약
  - XRay 관련 라이브러리 추가
  - AWSXRayServletFilter 추가
  - 샘플링 규칙 추가(xray-custom-sampling-rules.json)
  - Spring Security SuccessHandler에 User 정보 추가(로그인 성공 시)
  - X-Ray 활성화(AbstractXRayInterceptor 상속) - AOP Pointcut 지정
  - S3 연동(AWS SDK For Java v2)
  - Application 로그(Logback)에 트레이스 ID 주입

#### 1. 라이브러리 추가(maven : pom.xml)

```xml
...
<dependencyManagement>
    <dependencies>
        <!-- AWS SDK for Java 2.0 -->
        <dependency> 
            <groupId>software.amazon.awssdk</groupId>
            <artifactId>bom</artifactId> 
            <version>2.10.57</version> 
            <type>pom</type> 
            <scope>import</scope>
        </dependency>
        <!-- AWS XRAY SDK for Java -->
        <dependency>
            <groupId>com.amazonaws</groupId>
            <artifactId>aws-xray-recorder-sdk-bom</artifactId>
            <version>2.4.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
...

<!-- dependencies 추가 -->
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-xray-recorder-sdk-aws-sdk-v2-instrumentor</artifactId>
</dependency>
<dependency> 
    <groupId>com.amazonaws</groupId> 
    <artifactId>aws-xray-recorder-sdk-spring</artifactId>
</dependency>		
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-xray-recorder-sdk-slf4j</artifactId>
</dependency>
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-xray-recorder-sdk-sql-postgres</artifactId>
</dependency>
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-xray-recorder-sdk-sql</artifactId>
</dependency>

...
```

#### 2. AWSXRayServletFilter 추가(AWSXRayTracingFilterConfig)

```java
@Profile("prd")
@Configuration
public class AWSXRayTracingFilterConfig {

	@Bean
	public FilterRegistrationBean<AWSXRayServletFilter> TracingFilter() {
		String from = null;
		try {
			from = InetAddress.getLocalHost().getHostName();
		} catch (UnknownHostException e) {
			from = "serverName";
		}

		FilterRegistrationBean<AWSXRayServletFilter> registration = new FilterRegistrationBean<>();
		registration.setOrder(Ordered.HIGHEST_PRECEDENCE);
		registration.setFilter(
			new AWSXRayServletFilter(
				new DynamicSegmentNamingStrategy(from, "*.도메인명")));
		registration.addUrlPatterns("/api/*", CustomWebSecurityConfig.LOGIN_API_ENTRY_POINT);

		return registration;
	}

	static {
		AWSXRayRecorderBuilder builder = AWSXRayRecorderBuilder.standard()
			.withPlugin(new EC2Plugin())
			.withPlugin(new ElasticBeanstalkPlugin())
			.withSegmentListener(new SLF4JSegmentListener());

		URL ruleFile;
		try {
			ruleFile = ResourceUtils.getURL("classpath:xray-custom-sampling-rules.json");
			builder.withSamplingStrategy(new LocalizedSamplingStrategy(ruleFile));
		} catch (FileNotFoundException e) {}

		AWSXRay.setGlobalRecorder(builder.build());
	}
}
```

#### 3. 샘플링 규칙 항목 추가(src/main/resources/xray-custom-sampling-rules.json)

```json
{
	"version": 1,
	"default": {
		"fixed_target": 1,
		"rate": 1
	},
	"rules": [
		{
			"description": "Notification",
			"host ": "*",
			"service_name": "*",
			"http_method": "GET",
			"url_path": "/api/notifications",
			"fixed_target": 0,
			"rate": 0.0
		},
		{
			"description": "Notification All",
			"host ": "*",
			"service_name": "*",			
			"http_method": "GET",
			"url_path": "/api/notifications/all",
			"fixed_target": 0,
			"rate": 0.0
		},
		{
			"description": "User Auth Status",
			"host ": "*",
			"service_name": "*",			
			"http_method": "GET",
			"url_path": "/api/auth/status",
			"fixed_target": 1,
			"rate": 0.01
		},
		{
			"description": "Menu Item",
			"host ": "*",
			"service_name": "*",			
			"http_method": "GET",
			"url_path": "/api/menu-items",
			"fixed_target": 1,
			"rate": 0.01
		},		
		{
			"description": "Code Controller",
			"host ": "*",
			"service_name": "*",			
			"http_method": "GET",
			"url_path": "/api/user-codes",
			"fixed_target": 1,
			"rate": 0.01
		}		
	]
}
```

#### 4. Spring Security SuccessHandler에 User 정보 추가

```java
@Component
public class AjaxAwareAuthenticationSuccessHandler implements AuthenticationSuccessHandler {

	private static Logger logger = LoggerFactory.getLogger(AjaxAwareAuthenticationSuccessHandler.class);

	private final ObjectMapper mapper;

	@Autowired
	public AjaxAwareAuthenticationSuccessHandler(final ObjectMapper mapper) {
		this.mapper = mapper;
	}

	@Override
	public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication)
		throws IOException, ServletException {

		CustomUser customUser = (CustomUser) authentication.getPrincipal();

		response.setStatus(HttpStatus.OK.value());
		response.setContentType(MediaType.APPLICATION_JSON_VALUE);

		Map<String, Object> resultMap = new HashMap<String, Object>();

		resultMap.put("isLogin", true);
		resultMap.put("userMessage", "환영합니다.");
		resultMap.put("userRole", customUser.getUserRole());
		// Xray User 추가
		AWSXRay.getCurrentSegmentOptional()
		.ifPresent(e -> e.setUser(customUser.getUserId()));

		this.mapper.writeValue(response.getWriter(), resultMap);
	}
}
```

#### 5. X-Ray 활성화(AbstractXRayInterceptor 상속) - AOP Pointcut 지정

```java
@Profile("prd")
@Aspect
@Component
public class XRayInspector extends AbstractXRayInterceptor {
	@Override
	protected Map<String, Map<String, Object>> generateMetadata(ProceedingJoinPoint proceedingJoinPoint, Subsegment subsegment) {

		// User를 체크하여 존재하면 해당 User를 기준으로 Tracing 한다.
		Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
			.filter(Authentication::isAuthenticated)
			.map(Authentication::getPrincipal)
			.filter(CustomUser.class::isInstance)
			.map(CustomUser.class::cast)
			.ifPresent(user -> {
				subsegment.getParentSegment().setUser(user.getUserId());
			});

		return super.generateMetadata(proceedingJoinPoint, subsegment);
	}

	@Override
	@Pointcut("execution(* 패키지명..*(..)) && @within(org.springframework.web.bind.annotation.RestController)")
	public void xrayEnabledClasses() {
	}
}

```

#### 6. S3 연동(AWS SDK For Java v2)

```java
@Slf4j
public class S3FileService implements FileService {

	....

	public S3FileService(String bucketName) {
		this.bucketName = bucketName;
		// S3 Client에 TracingInterceptor 추가
        // ClientOverrideConfiguration.builder().addExecutionInterceptor(new TracingInterceptor()).build()
		Region region = Region.AP_NORTHEAST_2;
		AwsCredentialsProvider provider = EnvironmentVariableCredentialsProvider.create();

		this.s3 = S3Client.builder()
			.credentialsProvider(provider)
			.region(region)
.overrideConfiguration(ClientOverrideConfiguration.builder().addExecutionInterceptor(new TracingInterceptor()).build())
			.build();
	}
    ....
}
```

#### 7. Application 로그(Logback)에 AWS-TRACE-ID 주입 - 운영계 properties 파일

```properties
logging.file.path=/var/log/appLog
logging.file.name=${logging.file.path}/${spring.application.name}.log
logging.pattern.file=%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}} ${LOG_LEVEL_PATTERN:-%5p} ${PID:- } --- [%t] %-40.40logger{39} %X{AWS-XRAY-TRACE-ID} : %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}

```



