---
published: true
title: springboot에서 swagger2 적용하기
date: '2020-02-18 14:55:00 +0900'
categories: springboot
last_modified_at: '2020-02-18 14:55:00 +0900'
tags:
  - springboot
  - swagger2
  - springfox
toc_sticky: true
toc: true
---
간단하게 springboot에서 swagger2를 적용하여 API 명세를 출력하는 방법을 기록한다.

현재 Spring Controller가 REST API 형태로 되어있다면 쉽게 적용/테스트 가능하다.

검색하면 많이 나오기는 하지만 그래도 기록의 의미로...

Maven, springboot를 사용하며 아래와 같은 방법으로 진행하면 된다.

1. maven pom.xml에 라이브러리 추가
2. config 파일을 통해 API 명세 정보 입력 및 컨트롤러 자동 적용
3. 적용 확인

#### 1. pom.xml

```xml
<!--..(생략)..-->

<dependency>
  <groupId>io.springfox</groupId>
  <artifactId>springfox-swagger2</artifactId>
  <version>2.9.2</version>
</dependency>

<dependency>
  <groupId>io.springfox</groupId>
  <artifactId>springfox-swagger-ui</artifactId>
  <version>2.9.2</version>
</dependency>

<!--..(생략)..-->

```

#### 2. springboot config

```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
	@Bean
	public Docket api() {
		return new Docket(DocumentationType.SWAGGER_2)
            .ignoredParameterTypes(User.class, ApiIgnore.class) // Security User 나  ApiIgnore 파라미터는 제외한다.
			.apiInfo(this.apiInfo())
			.select()
			.apis(RequestHandlerSelectors.basePackage("Controller가 포함된 패키지명"))
			.paths(PathSelectors.ant("/api/**")).build();
	}

	private ApiInfo apiInfo() {
		return new ApiInfoBuilder()
			.title("REST API Documentation")
			.description("API 리스트")
			.version("API V0.1")
			.termsOfServiceUrl("Terms of service")
            .contact(new Contact("장인기", "", "이메일주소"))
			.license("Apache 2.0")
            .licenseUrl("http://www.apache.org/licenses/LICENSE-2.0")
			.build();
	}
}
```

#### 3. 접속 확인
- http://{접속주소}/swagger-ui.html

끝.
