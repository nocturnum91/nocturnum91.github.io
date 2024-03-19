---
layout: post
title:  "Spring Boot + Spring Batch + Mybatis + Multiple DataSource로 간단한 배치 어플리케이션 만들기"
excerpt: ""
date:   2024-03-20 00:00
category: dev
tags: [Spring Boot, Spring Batch]
---

# History...

이전에 근무지에는 매일 실행되는 70~80여개의 Batch Application이 존재했는데 다양한 문제점이 있었다.
1. Log가 없다
2. 배치가 몇개인지 왜 만들었는지 뭘 하는지 정리된 문서도 없고 아무도 모른다.
3. 배치 실행 이력(성공/실패) 관리가 안된다.
4. 소스코드 형상 관리가 안되고있다.
5. Servlet으로 만들어져있다

<br>
담당자가 따로 있었는데... 그 담당자가 퇴사하고 나에게 넘어오게 되었다.
사실 내가 먼저 퇴사 의사를 밝혔으나 3개월이 넘게 미뤄지고 있는 상황에서
그 담당자는 본인 PC에 있던 소스코드를 커밋하지 않고 인수인계문서도 만들지 않은 상태로 퇴사를 했다.
결국 한달만 더 일을 하기로 했는데 대충 정리하자면 아래와 같은 요구사항을 받았다.

1. 서버에서 실행되는 모든 배치 어플리케이션의 이름, 기능, 실행시간을 정리하는 문서를 만들고
2. 배치 실행 이력 관리가 가능하고 서버 재기동 후 하나씩 url을 입력하지 않아도 되는(ㅠㅠ) 새로운 배치 어플리케이션 프레임워크를 만들고
3. 지속적 배포 환경도 구축하고 (원래 수동배포를 했다)
4. 후임자를 위해 예제라고 생각하며 기존의 배치 서너개를 새 배치로 바꾸고
5. 개발자가 아닌 사람(현업)도 비슷한 유형의 배치면 직접 바꿀 수 있게 가이드 문서를 작성해달라

<br>
...ㅎㅎ;;
아무튼 그래서 만들게 되었다...


나는 Java + Spring + MyBatis 환경에 익숙하고 시간은 촉박했기 때문에
길게 고민하지 않고 Spring Boot + Spring Batch를 사용하기로 결정했다.

[Spring Batch Introduction](https://docs.spring.io/spring-batch/docs/current/reference/html/spring-batch-intro.html#spring-batch-intro)을 대충 읽어보면
- 로깅 및 추적, 트랜잭션 관리, 작업 처리 통계, 작업 재시작, 건너뛰기 및 리소스 관리를 포함한 대량의 레코드를 처리하는 데 필수적인 기능을 제공하며
- 스케줄러를 대체하는 것이 아니라 스케줄러와 함께 작동하도록 설계되었고
- 배치 애플리케이션을 개발할 수 있도록 설계된 가볍고 포괄적인 배치 프레임워크

라고 소개되어 있다.
즉 스케쥴러는 따로 고민을 해야만 했는데 Quartz는 예전에 써봤고, Spring에서 제공하는 @Scheduled도 써봤다.
다만 나는 직접 배치 이력 대시보드를 만들 시간이 없었고 자동 배포도 해야해서 알아보던 중
Jenkins에 스케줄링 기능이 있다는 것을 알게 되었다. 만세.

그렇게 갑자기 폐쇄망 환경에 Jenkins를 설치하게 되었는데 이건 나중에 따로 포스팅을 할 계획이다.


---

# 개발 환경
IntelliJ
JDK 1.8 + Spring Boot 2.7.18 + Spring Batch + MyBatis
MySQL, PostgreSQL
Gradle

Maven만 사용했어서 Gradle은 처음 써봤다...

Sample 프로젝트를 만들어 Github에 올려놓았으니 필요한 분이 있다면 도움이 되었으면 좋겠다.

[Spring Batch Sample Project](https://github.com/nocturnum91/SpringBatch)
<br>

1. Spring Boot 프로젝트를 생성한다.
   ![Alt text](/images/2023-02-07-SpringBootBatch(1)/1.png)
   ![Alt text](/images/2023-02-07-SpringBootBatch(1)/2.png)

2. 의존성 추가

    ```gradle
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.3.2'
    implementation group: 'org.springframework.boot', name: 'spring-boot-starter-web'
    implementation group: 'org.springframework.boot', name: 'spring-boot-starter-jdbc'
    implementation 'org.springframework.boot:spring-boot-starter-log4j2'
    implementation group: 'org.bgee.log4jdbc-log4j2', name: 'log4jdbc-log4j2-jdbc4.1', version: '1.16'
    compileOnly group: 'jakarta.servlet', name: 'jakarta.servlet-api', version: '4.0.4'
    compileOnly 'org.projectlombok:lombok'
    implementation 'com.mysql:mysql-connector-j'
    runtimeOnly 'org.postgresql:postgresql'
    annotationProcessor group: 'org.springframework.boot', name: 'spring-boot-configuration-processor'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.batch:spring-batch-test'
    testCompileOnly 'org.projectlombok:lombok'
    testImplementation "org.mybatis.spring.boot:mybatis-spring-boot-starter-test:2.3.2"
    testAnnotationProcessor 'org.projectlombok:lombok'
    
    // junit5
    testImplementation("org.springframework.boot:spring-boot-starter-test") {
        exclude module : 'junit'
    }
    
    testImplementation("org.junit.jupiter:junit-jupiter-api")
    testCompileOnly("org.junit.jupiter:junit-jupiter-params")
    testRuntimeOnly("org.junit.jupiter:junit-jupiter-engine")
    ```
<br>
3. Log4J 설정
    - log4jdbc.log4j2.properties 파일을 resources 폴더에 추가한다.

    ```properties
    log4jdbc.spylogdelegator.name=net.sf.log4jdbc.log.slf4j.Slf4jSpyLogDelegator
    log4jdbc.dump.sql.maxlinelength=0
    ```
<br>
    - build.gradle 파일의 configurations에 Logback 종속성을 제거한다.

    ```gradle
    configurations {
        compileOnly {
            extendsFrom annotationProcessor
        }
        all {
            exclude group: 'org.springframework.boot', module: 'spring-boot-starter-logging'
        }
    }
    ```
<br>
4. application.yaml 추가
- resources/application.yml 파일을 추가한다.

    ```yaml
    server:
      port: 8081
    
    spring:
    mvc:
    pathmatch:
    matching-strategy: ant_path_matcher
    data:
    rest:
    base-path: /
    task:
    execution:
    pool:
    core-size: 8
    max-size: 8
    batch:
    job:
    enabled: true
    jdbc:
    initialize-schema: never
    ```
<br>
5. DataSource 설정
    - resources/config/datasource.yml 파일을 추가한다.

    ```yaml
    mybatis:
      configuration:
        map-underscore-to-camel-case: true
        jdbc-type-for-null: null
    
    spring:
      datasource:
        db1:
          name: db1DataSource
          driver-class-name: net.sf.log4jdbc.sql.jdbcapi.DriverSpy
          jdbc-url: jdbc:log4jdbc:mysql://localhost:3306/noc?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Seoul
          username: nocturnum
          password: 1234
          type: com.zaxxer.hikari.HikariDataSource
    
        db2:
          name: db2DataSource
          driver-class-name: net.sf.log4jdbc.sql.jdbcapi.DriverSpy
          jdbc-url: jdbc:log4jdbc:postgresql://localhost:5432/
          username: postgres
          password: 1234
          type: com.zaxxer.hikari.HikariDataSource
    ```
<br>
   - Application main 메소드에 SpringApplicationBuilder를 사용하여 실행할 때 사용할 프로퍼티 파일을 지정한다.

   ```java
    public static void main(String[] args) {
        SpringApplicationBuilder builder = new SpringApplicationBuilder(BatchApplication.class)
                .properties("spring.config.additional-location=classpath:/config/datasource.yml");
        System.exit(SpringApplication.exit(builder.run(args)));
    }
   ```
<br>
   - resources/config/datasource.yml 파일을 읽어와서 DataSource를 생성하는 Config 클래스를 만든다.
   
    ```java
   package org.nocturnum.batch.common.config;
   
   
   import com.zaxxer.hikari.HikariDataSource;
   import org.apache.ibatis.session.SqlSessionFactory;
   import org.mybatis.spring.SqlSessionFactoryBean;
   import org.mybatis.spring.SqlSessionTemplate;
   import org.mybatis.spring.annotation.MapperScan;
   import org.springframework.beans.factory.annotation.Qualifier;
   import org.springframework.boot.context.properties.ConfigurationProperties;
   import org.springframework.boot.jdbc.DataSourceBuilder;
   import org.springframework.context.ApplicationContext;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.context.annotation.Primary;
   import org.springframework.jdbc.datasource.DataSourceTransactionManager;
   import org.springframework.transaction.PlatformTransactionManager;
   import org.springframework.transaction.annotation.EnableTransactionManagement;
   
   import javax.sql.DataSource;
   
   @Configuration
   @MapperScan(basePackages = "org.nocturnum.batch.mapper.db1", sqlSessionFactoryRef = "db1SqlSessionFactory") /*멀티DB사용시 mapper클래스파일 스켄용 basePackages를 DB별로 따로설정*/
   @EnableTransactionManagement
   //@PropertySource("file:/C:\\Users\\Nocturnum\\config.properties")
   public class Db1Config {
   
   //    @Autowired
   //    Environment env;
   
       @Primary
       @Bean(name = "db1DataSource")
       @ConfigurationProperties(prefix = "spring.datasource.db1") // appliction.properties 참고.
       public DataSource db1DataSource() {
           return DataSourceBuilder.create()
                   .type(HikariDataSource.class).build();
       }
   
   
       @Primary
       @Bean(name = "db1SqlSessionFactory")
       public SqlSessionFactory db1sqlSessionFactory(@Qualifier("db1DataSource") DataSource db1DataSource,
                                                     ApplicationContext applicationContext) throws Exception {
           final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
           sessionFactory.setDataSource(db1DataSource);
           sessionFactory.setMapperLocations(applicationContext.getResources("classpath:mapper/db1/*.xml"));
           sessionFactory.setTypeAliasesPackage("org.nocturnum.batch.common.utils");
           return sessionFactory.getObject();
       }
   
       @Primary
       @Bean(name = "db1SqlSessionTemplate")
       public SqlSessionTemplate db1sqlSessionTemplate(SqlSessionFactory db1sqlSessionFactory) throws Exception {
           return new SqlSessionTemplate(db1sqlSessionFactory);
       }
   
       @Primary
       @Bean(name = "db1transactionManager")
       public PlatformTransactionManager db1transactionManager(@Qualifier("db1DataSource") DataSource db1DataSource) {
           return new DataSourceTransactionManager(db1DataSource);
       }
   
   }
    ```

    - db2에 대해서도 동일하게 설정한다.
<br>
<br>
6. MyBatis 설정
    - src/main/.../mapper/db1 폴더에 db1에 대한 Mapper.java 파일을 추가한다.
    - src/main/...//mapper/db2 폴더에 db2에 대한 Mapper.java 파일을 추가한다.
    - resources/mapper/db1 폴더에 db1에 대한 Mapper.xml 파일을 추가한다.
    - resources/mapper/db2 폴더에 db2에 대한 Mapper.xml 파일을 추가한다.
<br>
<br>
7. Batch 설정
   - Application 클래스에 @EnableBatchProcessing 어노테이션을 추가한다. 
   - src/main/.../config/JobConfig.java 파일을 추가한다.

    ```java
    /**
   * 예제 시나리오
   * 1. db2의 tb_member 테이블의 데이터를 조회한다
   * 2. 조회된 데이터를 db1의 tb_member 테이블에 저장한다
   * 값을 변경해야 하는 경우 주석 처리 되어있는 processor 참고하여 구현
   */
    @Slf4j
    @Configuration
    @EnableBatchProcessing
    public class JobConfig {

    /**
     * Job Launcher -> Job -> Step -> Tasklet or Chunk(ItemReader / ItemProcessor / ItemWriter)
     * Tasklet: 간단함, 한번에 처리, 대량의 데이터 처리에는 X
     * Tasklet Interface 구현체를 만들거나 MethodInvokingTaskletAdapter를 사용
     * Chunk: 대량의 데이터를 chunkSize 만큼 씩 처리
     * ItemReader: Item을 읽어오는 역할
     * ItemProcessor: ItemReader가 읽어온 데이터를 가공하는 역할 (생략 가능)
     * ItemWriter: ItemReader가 읽어온 데이터나 ItemProcessor가 가공한 데이터를 저장하는 역할
     * Spring Batch의 Chunk 단위로 데이터 처리를 함 / Transaction도 Chunk단위
       */

    private static final int CHUNK_SIZE = 1000;

    public final JobBuilderFactory jobBuilderFactory;
    public final StepBuilderFactory stepBuilderFactory;

    public final SqlSessionFactory db1SqlSessionFactory;
    public final SqlSessionFactory db2SqlSessionFactory;

    public MultiDBSelectInsertBatchConfig(JobBuilderFactory jobBuilderFactory, StepBuilderFactory stepBuilderFactory,
    @Qualifier("db1SqlSessionFactory") SqlSessionFactory db1SqlSessionFactory,
    @Qualifier("db2SqlSessionFactory") SqlSessionFactory db2SqlSessionFactory) {
    this.jobBuilderFactory = jobBuilderFactory;
    this.stepBuilderFactory = stepBuilderFactory;
    this.db1SqlSessionFactory = db1SqlSessionFactory;
    this.db2SqlSessionFactory = db2SqlSessionFactory;
    }

    @Bean
    @Transactional
    public Job selectInsertBatchJob() throws Exception {
    return jobBuilderFactory.get("selectInsertBatchJob").start(selectInsertStep()).incrementer(new RunIdIncrementer()).build();
    }

    @Bean
    @JobScope
    public Step selectInsertStep() throws Exception {
    log.info("############STEP");
    return stepBuilderFactory.get("Step")
    .<ParameterMap, ParameterMap>chunk(CHUNK_SIZE).reader(reader())
    //				.processor(processor(null))
    .writer(writer()).build();
    }

    /**
     * Item을 읽어오는 역할
       */
       @Bean
       @StepScope
       public MyBatisPagingItemReader<ParameterMap> reader() throws Exception {
       log.info("############READER");

       return new MyBatisPagingItemReaderBuilder<ParameterMap>().
       sqlSessionFactory(db2SqlSessionFactory)
       // Mapper안에서도 Paging 처리 시 OrderBy는 필수!
       .queryId("org.nocturnum.batch.mapper.db2.Db2Mapper.selectMemberList")
       //                .parameterValues(parameterMap)
       .pageSize(CHUNK_SIZE).build();
       }

    /**
     * Item을 처리하는 역할
       */
       //    @Bean
       //    @StepScope
       //    public ItemProcessor<ParameterMap, ParameterMap> processor() {
       //
       //        return new ItemProcessor<ParameterMap, ParameterMap>() {
       //            @Override
       //            public ParameterMap process(ParameterMap parameterMap) throws Exception {
       //                // 1000원 추가 적립
       //                parameterMap.put("count", parameterMap.getInt("count") + 1000);
       //                log.info("#################" + parameterMap);
       //                return parameterMap;
       //            }
       //        };
       //    }

    /**
     * 처리된 데이터를 저장하는 역할
       */
       @Bean
       @StepScope
       public MyBatisBatchItemWriter<ParameterMap> writer() {
       log.info("############WRITER");

       return new MyBatisBatchItemWriterBuilder<ParameterMap>()
       .sqlSessionFactory(db1SqlSessionFactory)
       .statementId("org.nocturnum.batch.mapper.db1.Db1Mapper.insertMember")
       .build();
       }


    }

    ```
<br>
8. 배치 실행
   ```bash
   java -jar jarname --spring.batch.job.names=jobname
   ```
