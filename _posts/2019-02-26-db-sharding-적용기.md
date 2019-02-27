---
layout: post
title: multi tomcal instance 적용기
tags:
  - NHN
  - 교육
  - DB
---

저는 DB sharding을 SqlSession을 2개 생성하여 특정 문자열을 hashing하여 어떤 DB를 사용할지 결정합니다.
AOP를 사용하여 횡단 관심사를 분리 시키는 것이 더 깔끔하지만 제가 가장 이해되는 코드로 작성해 보았습니다.

DB sharding을 SqlSession을 생성하여 구현하기 위해서는 총 4가지의 작업이 필요합니다. 
1. application properties에 DB설정 입력
1. SqlSession을 각각의 다른 datasource를 사용하여 생성하기
1. DB router에서 해쉬값을 가지고 SqlSession을 선택하기
1. DAO에서 생성한 SqlSession과 Mapper를 연동하기

---
### DB 설정 입력
```
spring.datasource0.url={DB #0 url}
spring.datasource0.username={username}
spring.datasource0.password={password}

spring.datasource1.url={DB #1 url}
spring.datasource1.username={username}
spring.datasource1.password={password}
```
처음으로 각각의 DB에 대한 정보를 application properties에 설정해줍니다.

---

### SqlSession 생성
```
@Autowired
ResourceLoader resourceLoader;

@Value("${db.driver-name}")
private String driverName;

@Bean
@Primary
@ConfigurationProperties(prefix = "spring.datasource0")
public DataSource dataSource0() {
  return DataSourceBuilder.create().driverClassName(driverName).build();
}

@Bean
@ConfigurationProperties(prefix = "spring.datasource1")
public DataSource dataSource1() {
  return DataSourceBuilder.create().driverClassName(driverName).build();
}

@Bean
@Primary
  public SqlSessionFactory sqlSession0Factory(@Autowired @Qualifier("dataSource0") DataSource dataSource) throws Exception {
      return sqlSessionFactory(dataSource);
  }

@Bean(name="sqlSession0")
@Primary
public SqlSession sqlSession0(@Autowired @Qualifier("sqlSession0Factory") SqlSessionFactory factory) {
    return new SqlSessionTemplate(factory);
}

@Bean
public SqlSessionFactory sqlSession1Factory(@Autowired @Qualifier("dataSource1") DataSource dataSource)  throws Exception {
    return sqlSessionFactory(dataSource);
}

@Bean(name="sqlSession1")
public SqlSession sqlSession1(@Autowired @Qualifier("sqlSession1Factory") SqlSessionFactory factory) {
    return new SqlSessionTemplate(factory);
}

public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
  SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
    factoryBean.setDataSource(dataSource);
    factoryBean.setVfs(SpringBootVFS.class);
    factoryBean.setConfigLocation(resourceLoader.getResource(MYBATIS_CONFIG));
    return factoryBean.getObject();
}
```
처음에는 DataSource를 각각의 database 설정에 따라 생성하고
그 datasource를 가지고 sqlSessionFactory를 생성하고 

마지막으로 sqlSession을 생성해 빈에 등록합니다.

---

### DB router에서 어떤 DB를 사용할지 설정.

```
@Autowired
@Qualifier("sqlSession0")
SqlSession sqlSession0;

@Autowired
@Qualifier("sqlSession1")
SqlSession sqlSession1;

public SqlSession getSqlSession(String {해쉬할 값}) {
  switch (HashUtility.getHash({해쉬할 값})) {
    case 0:
      return sqlSession0;
    case 1:
      return sqlSession1;
    default:
      throw new GetSqlSessionException();
  }
}

public <T> T getMapper(String userId, Class<T> type) {
  SqlSession sqlSession = getSqlSession(userId);
  T mapper = sqlSession.getMapper(type);

  return mapper;
}
```

위에서 생성한 sqlSession을 DI받고 hash에 따라 어떤 sqlSession을 사용할지 설정합니다.

그후에 getMapper를 사용하여 필요한 mapper를 반환합니다.

---

### DAO에서 생성한 SqlSession과 Mapper를 연동하기

```
public List<Email> findReceivedEmails(String emailOwner, int limit, int offset) throws SQLException{
  EmailMapper emailMapper = customDataSourceFactory.getMapper({해쉬할 값}, EmailMapper.class);
  return emailMapper.findReceivedEmails(emailOwner, limit, offset);
}
```

이제 DAO에서 이런식으로 원하는 mapper를 선택하여 hash할 값을 같이 보내면 필요한 mapper가 sqlSession이 선택되어 반환됩니다.
