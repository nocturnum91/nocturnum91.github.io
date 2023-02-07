---
layout: post
title:  "Spring Boot + Spring Batch + Multiple DataSource 구성에서 @MybatisTest 사용하기"
excerpt: "Spring Boot + Multiple DataSource + @MybatisTest"
date:   2023-02-07 00:00
category: dev
tags: [Spring Boot]
---

제목처럼 Spring Batch에 Multiple DataSource를 설정하고 @MybatisTest로 Mapper 단위 테스트를 하고 싶었다.

@SpringBootTest는 전체 어플리케이션을 로드하여 테스트를 진행하기 때문에 어플리케이션이 무거워질수록 테스트 시간이 길어지므로
쿼리 하나, 새로 추가하거나 수정한 기능 하나를 테스트하기에는 비효율적이라는 생각이 들었기 때문이다.
특히 Spring Batch는 배치를 통째로 실행해버린다는 점에서 더... 귀찮았다. (spring.batch.job.enabled=false 설정을 하면 되긴 하지만)


그렇게 가벼운 마음으로 새로 만든 테스트 클래스에 @MybatisTest를 추가하고 만난 에러는 다음과 같다

> java.lang.IllegalStateException: Failed to load ApplicationContext

> ...Error creating bean with name 'org.springframework.batch.core.configuration.annotation.SimpleBatchConfiguration': Unsatisfied dependency expressed through field 'dataSource'; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dataSource': Invocation of init method failed; nested exception is java.lang.IllegalStateException: Failed to replace DataSource with an embedded database for tests. If you want an embedded database please put a supported one on the classpath or tune the replace attribute of @AutoConfigureTestDatabase.

> ...Error creating bean with name 'dataSource': Invocation of init method failed; nested exception is java.lang.IllegalStateException: Failed to replace DataSource with an embedded database for tests. If you want an embedded database please put a supported one on the classpath or tune the replace attribute of @AutoConfigureTestDatabase.

> ...Failed to replace DataSource with an embedded database for tests. If you want an embedded database please put a supported one on the classpath or tune the replace attribute of @AutoConfigureTestDatabase.

<br>
대충 이해하기로는 기본적으로 인 메모리 DB를 사용하려고 하는데 나는 설정해놓은 인 메모리 DB가 없으니 dataSource 생성에 실패했고 @AutoConfigureTestDatabase를 사용해서 기본 설정값을 바꿀 수 있다는 것 같았다.
실제로 @MybatisTest 주석에도 인 메모리 DB를 사용하고, @AutoConfigureTestDatabase로 재정의가 가능하다고 설명되어 있었다.

그래서 아래처럼 AutoConfigureTestDatabase 어노테이션 설정을 추가했다.

```java
@MybatisTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Slf4j
public class MapperTests { ...
```
<br>
사실 이미 @SpringBootTest를 사용한 상태에서 테스트를 성공하고 미래에 추가될 테스트를 위해 @MybatisTest를 적용하려고 했던 상황이라
다시 에러를 만나게 될 줄은 상상도 못했다.

> Description:
> Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.
> Reason: Failed to determine a suitable driver class

> Action:
> Consider the following:
>   If you want an embedded database (H2, HSQL or Derby), please put it on the classpath.
>   If you have database settings to be loaded from a particular profile you may need to activate it (the profiles local are currently active).

> Failed to instantiate [com.zaxxer.hikari.HikariDataSource]: Factory method 'dataSource' threw exception; nested exception is org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Failed to determine a suitable driver class
> Failed to determine a suitable driver class

<br>
역시 대충 이해하자면 내가 DataSource url을 설정하지 않아서 적절한 driver class를 찾지 못했다는 말이다.
나는 썼는데.

위 에러를 긁어서 검색해 보면 application.properties에 DB 접속 정보 설정을 하라거나
SpringApplication 클래스에 @SpringBootApplication(exclude={DataSourceAutoConfiguration.class}) 설정을 하라거나
외부 DB 사용을 안 하면 spring-boot-starter-jdbc 종속성을 제거하라고 한다.

@MybatisTest이 mybatis 테스트와 관련된 구성만 가져와서 사용하기 때문에
@Component나 @ConfigurationProperties는 물론이고 @Configuration 빈도 가져오지 않는듯했다...

나는 아래 코드처럼 DBConfig 클래스에 @Configuration를 붙이고
DataSource를 설정할 때 @ConfigurationProperties(prefix = "spring.datasource.db1")를 추가해서
application.properties에 있는 값을 가져오고 있었는데
DataSourceBuilder.create().url("...")처럼 Config 클래스에서 url 값을 직접 입력해도 같은 에러가 출력되었기 때문이다.
(그냥 이러면 되나 해보고 싶었음)

```java
@Configuration
@MapperScan(basePackages = "org.nocturnum.batch.mapper.db1", sqlSessionFactoryRef = "db1SqlSessionFactory") /*멀티DB사용시 mapper클래스파일 스켄용 basePackages를 DB별로 따로설정*/
@EnableTransactionManagement
public class Db1Config {

    @Primary
    @Bean(name = "db1DataSource")
    @ConfigurationProperties(prefix = "spring.datasource.db1") // appliction.properties 참고.
    public DataSource db1DataSource() {
        return DataSourceBuilder.create()
                .type(HikariDataSource.class).build();
    }
```
<br>

그래서 @ContextConfiguration를 사용해서 내가 추가할 Config 클래스들을 입력해 주었다.
이러면 Cofig 클래스들이 ApplicationContext에 로딩된다 (드디어)


```java
@MybatisTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@ContextConfiguration(classes = {Db1Config.class, Db2Config.class})
//@Rollback(value = false)
@Slf4j
public class MapperTests { ...
```
<br>

@MybatisTest를 이용하여 CRUD Mapper 테스트를 진행하면 테스트 완료 후 Rollback이 된다.
취향에 맞게 @Rollback(value = false)를 선언해서 사용하면 되겠다.



