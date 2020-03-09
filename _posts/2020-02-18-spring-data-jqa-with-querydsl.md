---
published: true
title: springboot+jpa+querydsl 연동
date: "2020-02-18 14:56:00 +0900"
categories: querydsl
last_modified_at: "2020-03-09 15:00:00 +0900"
tags:
  - querydsl
  - jpa
  - spring-data-jpa
  - springboot
toc_sticky: true
toc: true
---

springboot(2.2.X), spring-data-jpa, querydsl 사용...  
간단한 기록...

### 1. 의존성 추가  
> pom.xml - dependency 영역  
```xml

<!-- (..생략..) -->
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-apt</artifactId>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
</dependency>

<!-- (..생략..) -->        

```

### 2. 빌드 시 QType Entity Generate  
> pom.xml - plugin 영역  
```xml
<!-- (..생략..) -->
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
<!-- (..생략..) -->
```

### 3. Repository 구현

```java
@Repository
public interface ARepository extends JpaRepository<AEntity, Long>, CustomARepository {

}

interface CustomARepository {
    // Querydsl을 사용할 커스텀 메소드를 정의
    // ...
}

class CustomARepositoryImpl extends QuerydslRepositorySupport implements CustomARepository {

	public CustomARepositoryImpl() {
		super(AEntity.class);
	}
    // Querydsl을 사용할 커스텀 메소드 구현
    // ...
}
```

### 4. 사용
> 기존 JPARepository DI 받은 후 사용하는 것과 동일

끝.