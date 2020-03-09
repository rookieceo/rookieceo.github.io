---
published: true
title: querydsl에서 DBMS 함수 사용하기
date: "2020-02-19 14:56:00 +0900"
categories: querydsl
last_modified_at: "2020-02-19 17:50:00 +0900"
tags:
  - querydsl
  - jpa
  - spring-data-jpa
  - springboot
  - postgresql
toc_sticky: true
toc: true
---

springboot에 jpa, querydsl을 사용중이다.

Mybatis를 사용해온 내게는 쿼리가 없어지는게 익숙하지 않다.

그렇지만 코드를 짜 놓고 보면 나쁘지 않다.

먼저 쿼리없이 심플하게 인터페이스 선언(JPA Repository)만으로 간단한 CRUD를 만들어 낼 수 있어 생산적이다.

다만 조회의 경우 기존 JPA만으로는 어려워 querydsl을 사용하였는데

코드가 정리된 느낌이고 컴파일 타임에 에러를 잡을 수 있어서 좋았다.

아쉬운 건 자료가 상대적으로 잘 없어서 뻘짓을 해야한다는 것.. ㅜㅠ

실제 개발하다보면 객체 관계로만 다 매핑하기는 아쉬운 로직들이 있다.

예를 들어 보통 DBMS에 내장된 함수를 이용하고 싶을 경우나 쿼리 결과를 단순 가공할 때에 DB 함수를 사용한다.

예전 방식대로라면 바로 만들어서 쿼리 찍어보고 완성된 쿼리를 SQL Mapper framework 쿼리로 넣으면 끝나는 것이지만 querydsl을 그렇게 되지 않는다.

이를 사용하는 방법을 기록해본다.

블로그를 잘 써보지 않아서 역시나 이번에도 글을 쓰고 보면 아쉽다, 서론이 너무 두서없고 길다.

익숙하지 않다. ㅠㅜ

설정을 요약하면 이렇다.

1. pom.xml 추가(의존성 설정)

   > - springboot2.2.x
   > - spring-data-jpa(hibernate)
   > - querydsl
   > - postgresql

2. CustomSQLDialect 생성

   > - DB에 따라 hibernate에서 기본 제공하는 SQLDialect을 extends한 CustomSQLDialect을 생성한다.
   > - 사용할 function을 등록한다.

3. application properties 설정

   > application properties 파일을 이용해 이 class를 지정한다.

4. 실제 사용해보기

해보면 크게 어려운 건 없다.

아래는 코드다.

### 1.pom.xml

```xml
<!-- spring boot -->
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.2.4.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
  </parent>

<!-- (..생략..) -->

<!-- spring-data-jpa -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>

<!-- querydsl -->
  <dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-apt</artifactId>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
  </dependency>

<!-- postgresql -->
  <dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
  </dependency>

<!-- (..생략..) -->
	<build>
		<finalName>ROOT</finalName>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
			<!-- Plugin for query-dsl -->
			<plugin>
				<groupId>com.mysema.maven</groupId>
				<artifactId>apt-maven-plugin</artifactId>
				<version>1.1.3</version>
				<executions>
					<execution>
						<goals>
							<goal>process</goal>
						</goals>
						<configuration>
							<outputDirectory>target/generated-sources/java</outputDirectory>
							<processor>com.querydsl.apt.jpa.JPAAnnotationProcessor</processor>
						</configuration>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
```

### 2. CustomSQLDialect 생성

```java
public class CustomPostgreSQLDialect extends PostgreSQL10Dialect {

	public CustomPostgreSQLDialect() {
		super();
		// postgresql 내장함수 + 사용자 정의 함수 등록

    // DB 내장함수 사용
    // postgresql CONCAT_WS
		this.registerFunction("CONCAT_WS", new StandardSQLFunction("CONCAT_WS", StandardBasicTypes.STRING));

    // 사용자 정의함수의 경우 아래와 같은 방식을 응용하면 된다.
    // 예를 들어 스트링형태의 리턴타입 함수는 StandardBasicTypes.STRING로 선언해준다.
		this.registerFunction("사용할 함수명", new SQLFunctionTemplate(StandardBasicTypes.STRING, "사용할 함수명(?1)"));
	}
}
```

### 3. application.properties

```properties
spring.jpa.database-platform={패키지}.CustomPostgreSQLDialect
```

### 4. 실제 사용

```java
// QuerydslRepositorySupport
// (...생략..)

// 위에 추가한 내장함수(CONCAT_WS) 사용
// 예: 컬럼을 '/'구분자를 가진 데이터로 변환.
Expressions.stringTemplate("CONCAT_WS('/', {0}, {1}, {2}, {3})", param1, param2, param3, param4);

// 위에 추가한 사용자정의함수 사용
// 예 : QMyEntity의 문자열타입의 컬럼으로 가정
Expressions.simpleTemplate(String.class, "사용자정의함수명({0})", QMyEntity.stringPath)

// (...생략..)
```

끝.
