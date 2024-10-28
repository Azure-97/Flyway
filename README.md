# SpringBoot集成Flyway

## Flyway 简介

官方解释：Flyway 将 DevOps 扩展到您的数据库，以**加速软件交付**并**确保代码质量**。从版本控制到持续交付，Flyway 以**应用程序交付流程为基础**，实现数据库部署**自动化**。

通俗来说，Flyway可以作为数据库迁移工具服务到我们的应用程序升级发布流程中，减少人为处理sql脚本带来的繁琐和易出错问题。

## 入门实践

参考版本信息

springboot 3

flyway 10

mysql 8

### pom

```pom
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.5</version>
        <relativePath/> <!-- lookup parent from repository -->
</parent>

<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-mysql</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.33</version>
            <scope>runtime</scope>
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
```

- 如果使用flyway版本在 8.2.0及之前，可以直接使用flyway-core（groupId=org.flywaydb;artifaceId=flyway-core;version=8.2.0）的依赖
- flyway在8.2.1版本移除了mysql的默认支持，并创建了一个新的artifactId，即 flyway-mysql

### application.yaml

```yaml
server:
  port: 9009
logging:
  level:
    com.ramble: debug
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/caht_lease_uat?allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8&useSSL=false
    username: root
    password: root
  jpa:
    show-sql: true
  flyway:
    baseline-on-migrate: true
    baseline-version: 0
```

- baseline-on-migrate：是否自动创建迁移记录表，默认为false，需要设置为true，否则启动报错。flyway的原理就是依靠一个迁移记录表来管理各个版本的迁移脚本执行情况
- baseline-version：迁移版本初始版本号，默认为1。

### 迁移sql脚本

#### sql 目录中存放脚本文件，脚本名称命名方式

> - 版本化迁移 ：执行一遍，版本号唯一，有重复会报错：格式：V+版本号 +双下划线+描述+结束符
> - 重复的迁移 ，不需要版本号，脚本发生变化启动就会执行：格式：R+双下划线+描述+结束符
> - 撤消迁移 ：格式：U+版本号 +双下划线+描述+结束符

在resources目录下创建db文件夹，然后在db文件夹下创建migration文件夹，最后创建V1.0.1__migration.sql的迁移脚本文件。

```sql
create table PERSON (
    ID int not null,
    NAME varchar(100) not null
);
 
insert into PERSON (ID, NAME) values (1, 'Axel');
insert into PERSON (ID, NAME) values (2, 'Mr. Foo');
insert into PERSON (ID, NAME) values (3, 'Ms. Bar');
```

aclasspath:/db/migration 目录为flyway默认的迁移脚本存放目录，可以通过flyway.locations进行配置
迁移脚本文件名称：flyway的实现就是根据判断迁移记录表和迁移脚本文件名称来实现的。对于版本升级需要执行的sql，文件名称必须以“V”开头，后面跟版本号，再跟两个下划线，再跟自定义名称，最后跟“.sql”
V1.0.1__migration.sql：sql的内容就是在版本1.0.1中新增了一条数据

### 启动程序

启动程序之后库里将新增一个表 flyway_schema_history，此表是flyway自动生成的，同时会向user表中新增 一条数据。

### 常见问题

#### Unsupported Database

程序启动报如下错误

```
Caused by: org.flywaydb.core.api.FlywayException: Unsupported Database: MySQL 8.0
```

当pom中 引入flyway-core并且版本高于8.2.0将引发此异常。
原因是从8.2.1开始，flyway对mysql的支持拆分出去了，使用flyway-mysql支持。

解决方案有两个：

降低版本到8.2.0
替换依赖为flyway-mysql

#### No migration necessary

一起配置正常，程序启动不报错，但是脚本就是没有执行，日志中一直提示没有需要迁移的必要（No migration necessary）。

可能的原因：迁移脚本没有放在classpath:db/migration 下，并且也没有配置 flyway.locations

解决办法：创建db文件夹，在db下创建migration文件夹；或者配置 flyway.locations属性

#### Found non-empty schema(s)

问题原因：第一执行的时候没有找到schema history table
这张表其实就是application.properties文件中spring.flyway.table属性配置的表
因此要么使用命令创建一个或者在application.properties文件中设置 spring.flyway.baseline-on-migrate=true，


## flyway配置详解
```
flyway.baseline-description对执行迁移时基准版本的描述.
flyway.baseline-on-migrate当迁移时发现目标schema非空，而且带有没有元数据的表时，是否自动执行基准迁移，默认false.
flyway.baseline-version开始执行基准迁移时对现有的schema的版本打标签，默认值为1.
flyway.check-location检查迁移脚本的位置是否存在，默认false.
flyway.clean-on-validation-error当发现校验错误时是否自动调用clean，默认false.
flyway.enabled是否开启flywary，默认true.
flyway.encoding设置迁移时的编码，默认UTF-8.
flyway.ignore-failed-future-migration当读取元数据表时是否忽略错误的迁移，默认false.
flyway.init-sqls当初始化好连接时要执行的SQL.
flyway.locations迁移脚本的位置，默认db/migration.
flyway.out-of-order是否允许无序的迁移，默认false.
flyway.password目标数据库的密码.
flyway.placeholder-prefix设置每个placeholder的前缀，默认${.
flyway.placeholder-replacementplaceholders是否要被替换，默认true.
flyway.placeholder-suffix设置每个placeholder的后缀，默认}.
flyway.placeholders.[placeholder name]设置placeholder的value
flyway.schemas设定需要flywary迁移的schema，大小写敏感，默认为连接默认的schema.
flyway.sql-migration-prefix迁移文件的前缀，默认为V.
flyway.sql-migration-separator迁移脚本的文件名分隔符，默认__
flyway.sql-migration-suffix迁移脚本的后缀，默认为.sql
flyway.tableflyway使用的元数据表名，默认为schema_version
flyway.target迁移时使用的目标版本，默认为latest version
flyway.url迁移时使用的JDBC URL，如果没有指定的话，将使用配置的主数据源
flyway.user迁移数据库的用户名
flyway.validate-on-migrate迁移时是否校验，默认为true.
```