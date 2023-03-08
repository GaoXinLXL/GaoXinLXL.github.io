---
title: AbstractRoutingDataSource+Mybatis拦截器实现动态切换数据源与切库

date: '2023-02-14'
categories:
    - 笔记
tags:
    - Java
---


> 场景：线上4个MySQL实例分属4个机器，每个实例8个库。根据某一业务id，能够唯一确定数据存放在某个机器的某个库下。
要求：执行sql的时候，根据业务id切换数据源
工具：`AbstractRoutingDataSource` 切换数据源+`Mybatis`拦截器切换库
> 

# 1.数据源配置

yml配置文件参考如下：

```yaml
dlcms:
  dataSource:
    dlcms:
      url: jdbc:mysql://${MYSQL_URL_1}:${MYSQL_PORT}/dlcms?useUnicode=true&characterEncoding=UTF-8&useSSL=false&allowLoadLocalInfile=true&serverTimezone=Asia/Shanghai
      driver-class-name: com.mysql.cj.jdbc.Driver
      username: ${DB_USER}
      password: ${DB_PASSWORD}
    dlcms_01:
      url: jdbc:mysql://${MYSQL_URL_2}:${MYSQL_PORT}/dlcms_01?useUnicode=true&characterEncoding=UTF-8&useSSL=false&allowLoadLocalInfile=true&serverTimezone=Asia/Shanghai
      driver-class-name: com.mysql.cj.jdbc.Driver
      username: ${DB_USER}
      password: ${DB_PASSWORD}
......
```

MYSQL_URL_1、MYSQL_URL_2是不同的ip，表示MySQL分属不同机器。

# 2.读取数据源配置

```java
@Component
@ConfigurationProperties(prefix = "dlcms")
public class DlcmsDataSourceProfile {
    private Map<String, DataSourceProperties> datasource;

    public Map<String, DataSourceProperties> getDatasource() {
        return datasource;
    }

    public void setDatasource(Map<String, DataSourceProperties> datasource) {
        this.datasource = datasource;
    }
}
```

SpringBoot启动时会加载配置，将以`dlcms` 开头的配置存放进  `datasource` 同名Map<String, DataSourceProperties>集合

![https://i.imgtg.com/2023/02/15/dzA9S.png](https://i.imgtg.com/2023/02/15/dzA9S.png)

datasource集合的key就是配置里的dlcms、dlcms_01等，value就是数据源信息，包括url、账号密码等。

# 3.继承`AbstractRoutingDataSource` 重写`determineCurrentLookupKey`

```java
@Slf4j
@Component
public class DlcmsRoutingDataSource extends AbstractRoutingDataSource {
    public DlcmsRoutingDataSource(DlcmsDataSourceProfile profile) {
        Map<String, DataSourceProperties> datasource = profile.getDatasource();
        if (ObjectUtils.isEmpty(datasource)) {
            throw new BizException("数据源加载为空");
        }
        Map<Object, Object> map = datasource.entrySet().stream()
                .collect(Collectors.toMap(
                        Map.Entry::getKey,
                        e -> e.getValue().initializeDataSourceBuilder().type(DruidDataSource.class).build()
                ));
        setTargetDataSources(map);
    }

    @Override
    protected Object determineCurrentLookupKey() {
        HospitalDbMap current = DbRouteUtil.getCurrent();
        if (ObjectUtils.isEmpty(current)) {
            throw new BizException("当前数据映射为空");
        }
        return current.sourceName();
    }
}
```

`DlcmsRoutingDataSource` 构造函数获取2步骤得到的数据源配置，进行格式转换，调用`setTargetDataSources(map)` 指定数据源映射。

![https://i.imgtg.com/2023/02/15/dzZUN.png](https://i.imgtg.com/2023/02/15/dzZUN.png)

可以理解为将配置文件中的数据源映射交给`AbstractRoutingDataSource` 管理。

切换数据源的关键在于重写的`determineCurrentLookupKey()` 方法。

![https://i.imgtg.com/2023/02/15/dzvvC.png](https://i.imgtg.com/2023/02/15/dzvvC.png)

该方法确定了下一次调用的sql的数据源是哪个，在每次执行sql前触发该方法，`determineCurrentLookupKey()` 方法返回值就是构造方法中设置的数据源的key，以此获取当前数据源。

# 4.数据源路由信息应当保存到`ThreadLocal`

由第3步知，要路由到哪个数据源，只需要告诉`determineCurrentLookupKey()` 方法当前的数据源key就行了。这个key一般叫做上下文（*Context*）。

> 上下文应当由`ThreadLocal` 容器存放。
> 

![https://i.imgtg.com/2023/02/15/dzPIx.png](https://i.imgtg.com/2023/02/15/dzPIx.png)

`ThreadLocal` 保证数据只属于当前线程，防止并发时数据被错乱读取。

如下，构建工具类：

```java
@Slf4j
@Component
public class DbRouteUtil {

    private static final ThreadLocal<HospitalDbMap> _alias = new ThreadLocal<>();

......
```

其中HospitalDbMap为：

```java
public record HospitalDbMap(
        String hospitalCode,
        DataSouceEnum dataSouceEnum,
        String sourceName,
        String ylDatabase,
        String plgDatabase
) {

}
```

存放着一些与数据源相关的信息，其中sourceName可确定当前数据源。

# 5.`Mybatis`拦截器进行切库

`AbstractRoutingDataSource` 可以根据自定义条件动态路由某数据源，由于业务背景，数据分库存储，所以还需要进行切库操作。

在书写sql的时候，每次指明库名很麻烦，所以需要能够自动切换库。又因为业务数据的库可以根据某业务id确定，所以可以利用`Mybatis`拦截器，在执行sql前根据该业务id进行切库，sql就不再指明库名。

实现`Interceptor` 接口，重写`intercept` 方法：

```java
@Component
@Intercepts({
        @Signature(
                type = StatementHandler.class,
                method = "prepare",
                args = {Connection.class, Integer.class}
        )
})
@Slf4j
public class MybatisInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object target = invocation.getTarget();
        if (target instanceof StatementHandler) {
            Connection connection = (Connection) invocation.getArgs()[0];
            HospitalDbMap current = DbRouteUtil.getCurrent();
            DataSouceEnum dataSouceEnum = current.dataSouceEnum();
            if (dataSouceEnum == null) {
                log.error("切换数据源，必须指定数据源类型");
                throw new BizException("切换数据源未指定数据源类型");
            }
            switch (dataSouceEnum) {
                case yueli -> connection.setCatalog(current.ylDatabase());
                case plg -> connection.setCatalog(current.plgDatabase());
                case dlcms -> connection.setCatalog(DataSouceEnum.dlcms.name());
                default -> log.debug("拦截到其他类型，dataSouceEnum={}", dataSouceEnum);
            }
        }
        return invocation.proceed();
    }
}
```

进入`@Intercepts`查看：

![https://i.imgtg.com/2023/02/15/dzydL.png](https://i.imgtg.com/2023/02/15/dzydL.png)

- `@Intercepts` 用于指明要在哪拦截的目标方法。
- `invocation.proceed()` 就是被拦截的sql执行，在该方法前后可自定义书写内容，比如我这里修改当前数据源连接的库名。

`@Signature` 下包含了3个标签：

```java
@Signature(
                type = StatementHandler.class,
                method = "prepare",
                args = {Connection.class, Integer.class}
        )
```

- type：就是指定拦截器类型（ParameterHandler ，StatementHandler，ResultSetHandler）
- method：拦截器类型中的方法，不是自己写的方法
- args：是method中方法的入参

点进`StatementHandler` 查看：

![https://i.imgtg.com/2023/02/15/dzpNX.png](https://i.imgtg.com/2023/02/15/dzpNX.png)

在这里面选取method及设置对应args。

# 6.使用

![https://i.imgtg.com/2023/02/15/dz4Rp.png](https://i.imgtg.com/2023/02/15/dz4Rp.png)

利用 `AbstractRoutingDataSource` 动态路由数据源，需要设置上下文（可以理解为某个特定数据源的标志，以此决定用哪个数据源）

我这里提取了一个工具类，上下文信息保存在`ThreadLocal`中，确保线程之间隔离。
在每次执行不同数据源的sql之前，需要手动执行切换数据源，在调用sql前，mybatis拦截器自动切换库名。

如：

![https://i.imgtg.com/2023/02/15/dzJEt.png](https://i.imgtg.com/2023/02/15/dzJEt.png)

提取工具类，手动进行数据源的切换。

# 7.总结

- `AbstractRoutingDataSource` ：切换数据源
- `ThreadLocal` ：保存上下文（数据源标记信息）
- `Mybatis`拦截器：根据上下文自动切换库
- 数据源路由工具类：方便手动执行切换数据源

**参考资料**

[https://www.baeldung.com/spring-abstract-routing-data-source](https://www.baeldung.com/spring-abstract-routing-data-source)