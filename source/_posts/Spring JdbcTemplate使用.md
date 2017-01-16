---
title: Spring JdbcTemplate使用
date: 2016-11-10 08:43:15
tags: Spring
category: Java
---
Spring JdbcTemplate是对JDBC的简单封装，它可以有效减少我们模板代码的数量，下面是和JDBC的主要区别：
![](/images/10.png)
<!--more-->

## 环境搭建
##### 配置数据源
```java
@Bean
public DataSource dataSource() {
    DruidDataSource dataSource = new DruidDataSource();
    dataSource.setUsername("usr_mesportal");
    dataSource.setPassword("A8mes!");
    dataSource.setUrl("jdbc:postgresql://127.0.0.1:5432/mesportal");
    return dataSource;
}
```
##### 数据源注入到JdbcTemplate中
```java
@Bean
public JdbcTemplate jdbcTemplate(DataSource dataSource){
    return new JdbcTemplate(dataSource);
}
```
##### 使用JdbcTemplate
```java
@Repository
public class TestDao {
    @Autowired
    JdbcTemplate jdbcTemplate;
    public void search(){
        jdbcTemplate.execute("INSERT INTO mes_program_info (name) VALUES ('xx')");
    }
}
```

## 常用方法
##### 查询
```java
Actor actor = this.jdbcTemplate.queryForObject(
        "select first_name, last_name from t_actor where id = ?",
        new Object[]{1212L},
        new RowMapper<Actor>() {
            public Actor mapRow(ResultSet rs, int rowNum) throws SQLException {
                Actor actor = new Actor();
                actor.setFirstName(rs.getString("first_name"));
                actor.setLastName(rs.getString("last_name"));
                return actor;
            }
        });
```
```java
public List<Actor> findAllActors() {
    return this.jdbcTemplate.query( "select first_name, last_name from t_actor", new ActorMapper());
}

private static final class ActorMapper implements RowMapper<Actor> {

    public Actor mapRow(ResultSet rs, int rowNum) throws SQLException {
        Actor actor = new Actor();
        actor.setFirstName(rs.getString("first_name"));
        actor.setLastName(rs.getString("last_name"));
        return actor;
    }
}
```
```java
String lastName = this.jdbcTemplate.queryForObject(
        "select last_name from t_actor where id = ?",
        new Object[]{1212L}, String.class);
```
##### 更新（增/改/删）
```java
this.jdbcTemplate.update(
        "insert into t_actor (first_name, last_name) values (?, ?)",
        "Leonor", "Watling");
```
```java
this.jdbcTemplate.update(
        "update t_actor set last_name = ? where id = ?",
        "Banjo", 5276L);
```
```java
this.jdbcTemplate.update(
        "delete from actor where id = ?",
        Long.valueOf(actorId));
```
## 其他操作
```java
this.jdbcTemplate.execute("create table mytable (id integer, name varchar(100))");
```
```java
this.jdbcTemplate.update(
        "call SUPPORT.REFRESH_ACTORS_SUMMARY(?)",
        Long.valueOf(unionId));
```

## 补充说明
除了jdbcTemplate比较常用外，还有NamedParameterJdbcTemplate也可以完成相同的功能。NamedParameterJdbcTemplate和jdbcTemplate的区别在于对于输入参数的处理不同。
```java
public int countOfActorsByFirstName(String firstName) {

    String sql = "select count(*) from T_ACTOR where first_name = :first_name";

    SqlParameterSource namedParameters = new MapSqlParameterSource("first_name", firstName);

    return this.namedParameterJdbcTemplate.queryForObject(sql, namedParameters, Integer.class);
}
```
在NamedParameterJdbcTemplate中，用变量名代替？，这样在设置参数的时不会因为错位等导致sql语句错误。
关于Spring JDBC还有其它的一些功能，如批处理，如果想进一步了解可以[访问这里](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/jdbc.html)