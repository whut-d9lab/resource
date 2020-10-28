## springboot数据库读写分离

### 1 pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.cjs.example</groupId>
    <artifactId>cjs-datasource-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>cjs-datasource-demo</name>
    <description></description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.2.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.8</version>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>

        </plugins>
    </build>
</project>
```

### 2 application.yml

```yaml
server:
  port: 8101 #运行端口号
spring:
  application:
    name: user-service #服务名称
  datasource:
    master:
      jdbc-url: jdbc:mysql://kdsql.d9lab.net:3306/change_buy?useSSL=false  #主数据库地址
      username: whatie
      password: D9Lab@171829
      driver-class-name: com.mysql.jdbc.Driver
      type: com.alibaba.druid.pool.DruidDataSource
    slave:
      jdbc-url: jdbc:mysql://kdsql.d9lab.net:3406/change_buy?useSSL=false #从数据库地址
      username: whatie
      password: D9Lab@171829
      driver-class-name: com.mysql.jdbc.Driver
      type: com.alibaba.druid.pool.DruidDataSource
```

### 3 java代码

- DataSourceConfig

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;
import java.util.HashMap;
import java.util.Map;


/**
 * @Author shuxibing
 * @Date 2020/9/12 15:06
 * @Uint d9lab_2019
 * @Description:
 */
@Configuration
@Slf4j
public class DataSourceConfig {


    @Primary
    @Bean
    @ConfigurationProperties("spring.datasource.master")
    public DataSource masterDataSource()
    {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }


    //配置数据库信息
    @Bean
    public DataSource myRoutingDataSource(@Qualifier("masterDataSource") DataSource masterDataSource,
                                          @Qualifier("slaveDataSource") DataSource slaveDataSource) {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("master", masterDataSource);
        targetDataSources.put("slave", slaveDataSource);
        MyRoutingDataSource myRoutingDataSource = new MyRoutingDataSource();
        myRoutingDataSource.setDefaultTargetDataSource(masterDataSource);
        myRoutingDataSource.setTargetDataSources(targetDataSources);
        return myRoutingDataSource;
    }

}
```

- MyRoutingDataSource (数据源配置)

```java
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

/**
 * @Author shuxibing
 * @Date 2020/9/12 16:33
 * @Uint d9lab_2019
 * @Description:
 */
public class MyRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceSwitcher.getDataSource();
    }

}
```

- MyBatisConfig（mybatis配置）

```java
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionFactoryBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.annotation.EnableTransactionManagement;

import javax.annotation.Resource;
import javax.sql.DataSource;

/**
 * @Author shuxibing
 * @Date 2020/9/12 16:47
 * @Uint d9lab_2019
 * @Description:  mapper数据源切换
 */
@EnableTransactionManagement
@Configuration
public class MyBatisConfig {

    @Resource(name = "myRoutingDataSource")
    private DataSource myRoutingDataSource;

    @Bean
    public SqlSessionFactory sqlSessionFactory() throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(myRoutingDataSource);
        sqlSessionFactoryBean.setMapperLocations(new 	        		          		         PathMatchingResourcePatternResolver().getResources("classpath*:mapper/*.xml"));
        return sqlSessionFactoryBean.getObject();
    }

    @Bean
    public PlatformTransactionManager platformTransactionManager() {
        return new DataSourceTransactionManager(myRoutingDataSource);
    }
}
```

- DataSourceSwitcher（数据源切换工具类）

```java
/**
 *
 * @Author shuxibing
 * @Date 2020/9/14 11:57
 * @Uint d9lab
 * @Description:
 * 
 */
public class DataSourceSwitcher {
    private static final ThreadLocal contextHolder = new ThreadLocal();
    public static void setDataSource(String dataSource){
        contextHolder.set(dataSource);
    }
    public static void setMaster(){
        clearDataSource();
    }
    public static void setSlave(){
        setDataSource("slave");
    }
    public static String getDataSource(){
        return (String) contextHolder.get();
    }
    public static void clearDataSource(){
        contextHolder.remove();
    }
}
```



- DataSourceAop （AOP拦截实现数据源切换）

```java
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

/**
 * @Author shuxibing
 * @Date 2020/9/12 17:04
 * @Uint d9lab_2019
 * @Description:
 */
@Component
@Aspect
@Slf4j
public class DataSourceAop {

    @Pointcut(
            "(execution(* edu.whut.pocket.user.*.service..*.select*(..)) " +
            "|| execution(* edu.whut.pocket.user.*.service..*.get*(..)))")
    public void readPointcut() {

    }

    @Pointcut("execution(* edu.whut.pocket.user.*.service..*.save*(..)) " +
            "|| execution(* edu.whut.pocket.user.*.service..*.insert*(..)) " +
            "|| execution(* edu.whut.pocket.user.*.service..*.add*(..)) " +
            "|| execution(* edu.whut.pocket.user.*.service..*.update*(..)) " +
            "|| execution(* edu.whut.pocket.user.*.service..*.edit*(..)) " +
            "|| execution(* edu.whut.pocket.user.*.service..*.delete*(..)) " +
            "|| execution(* edu.whut.pocket.user.*.service..*.remove*(..))")
    public void writePointcut() {

    }

    @Before("readPointcut()")
    public void read() {
        log.info("从服务器");
        DataSourceSwitcher.setSlave();
    }

    @Before("writePointcut()")
    public void write() {
       log.info("主服务器");
        DataSourceSwitcher.setMaster();
    }
}

```



