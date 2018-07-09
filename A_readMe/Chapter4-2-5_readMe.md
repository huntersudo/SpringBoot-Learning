#chapter4-2-5：使用log4j记录日志到MongoDB

之前在《使用AOP统一处理Web请求日志》一文中介绍了如何使用AOP统一记录web请求日志。
基本思路是通过aop去切web层的controller实现，获取每个http的内容并通过log4j将日志内容写到应用服务器的文件系统中。

但是当我们在集群中部署应用之后，应用请求的日志被分散记录在了不同应用服务器的文件系统上，这样分散的存储并不利于我们对日志内容的检索。
解决日志分散问题的方案多种多样，本文思路以在《使用AOP统一处理Web请求日志》一文的基础之上，扩展log4j实现将日志写入MongoDB。

### 准备工作

可以先拿Chapter4-2-4工程为基础，进行后续的实验改造。该工程实现了一个简单的REST接口，一个对web层的切面，并在web层切面前后记录http请求的日志内容。

### 通过自定义appender实现

思路：log4j提供的输出器实现自Appender接口，要自定义appender输出到MongoDB，只需要继承AppenderSkeleton类，并实现几个方法即可完成。

引入mongodb的驱动
在pom.xml中引入下面依赖
```
<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongodb-driver</artifactId>
    <version>3.2.2</version>
</dependency>
```

### 实现MongoAppender

编写MongoAppender类继承AppenderSkeleton，实现如下：
```
public class MongoAppender  extends AppenderSkeleton {

    private MongoClient mongoClient;
    private MongoDatabase mongoDatabase;
    private MongoCollection<BasicDBObject> logsCollection;

    private String connectionUrl;
    private String databaseName;
    private String collectionName;

    @Override
    protected void append(LoggingEvent loggingEvent) {

        if(mongoDatabase == null) {
            MongoClientURI connectionString = new MongoClientURI(connectionUrl);
            mongoClient = new MongoClient(connectionString);
            mongoDatabase = mongoClient.getDatabase(databaseName);
            logsCollection = mongoDatabase.getCollection(collectionName, BasicDBObject.class);
        }
        logsCollection.insertOne((BasicDBObject) loggingEvent.getMessage());

    }

    @Override
    public void close() {
        if(mongoClient != null) {
            mongoClient.close();
        }
    }

    @Override
    public boolean requiresLayout() {
        return false;
    }

    // 省略getter和setter

}
```

- 定义MongoDB的配置参数，可通过log4j.properties配置：

  - connectionUrl：连接mongodb的串
  - databaseName：数据库名
  - collectionName：集合名

- 定义MongoDB的连接和操作对象，根据log4j.properties配置的参数初始化：

  - mongoClient：mongodb的连接客户端
  - mongoDatabase：记录日志的数据库
  - logsCollection：记录日志的集合
- 重写append函数：

  - 根据log4j.properties中的配置创建mongodb连接
  - LoggingEvent提供getMessage()函数来获取日志消息
  - 往配置的记录日志的collection中插入日志消息
- 重写close函数：关闭mongodb的

### 配置log4j.properties

设置名为mongodb的logger：

记录INFO级别日志
- appender实现为com.didispace.log.MongoAppende
- mongodb连接地址：mongodb://localhost:27017
- mongodb数据库名：logs
- mongodb集合名：logs_request

```
log4j.logger.mongodb=INFO, mongodb
# mongodb输出
log4j.appender.mongodb=com.didispace.log.MongoAppender
log4j.appender.mongodb.connectionUrl=mongodb://localhost:27017
log4j.appender.mongodb.databaseName=logs
log4j.appender.mongodb.collectionName=logs_request
```

切面中使用mongodb logger
修改后的代码如下，主要做了以下几点修改：

- logger取名为mongodb的
- 通过getBasicDBObject函数从HttpServletRequest和JoinPoint对象中获取请求信息，并组装成BasicDBObject
- getHeadersInfo函数从HttpServletRequest中获取header信息
- 通过logger.info()，输出BasicDBObject对象的信息到mongodb
```
@Aspect
@Order(1)
@Component
public class WebLogAspect {

    private Logger logger = Logger.getLogger("mongodb");

    @Pointcut("execution(public * com.didispace.web..*.*(..))")
    public void webLog(){}

    @Before("webLog()")
    public void doBefore(JoinPoint joinPoint) throws Throwable {
        // 获取HttpServletRequest
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        // 获取要记录的日志内容
        BasicDBObject logInfo = getBasicDBObject(request, joinPoint);
        logger.info(logInfo);
    }


    private BasicDBObject getBasicDBObject(HttpServletRequest request, JoinPoint joinPoint) {
        // 基本信息
        BasicDBObject r = new BasicDBObject();
        r.append("requestURL", request.getRequestURL().toString());
        r.append("requestURI", request.getRequestURI());
        r.append("queryString", request.getQueryString());
        r.append("remoteAddr", request.getRemoteAddr());
        r.append("remoteHost", request.getRemoteHost());
        r.append("remotePort", request.getRemotePort());
        r.append("localAddr", request.getLocalAddr());
        r.append("localName", request.getLocalName());
        r.append("method", request.getMethod());
        r.append("headers", getHeadersInfo(request));
        r.append("parameters", request.getParameterMap());
        r.append("classMethod", joinPoint.getSignature().getDeclaringTypeName() + "." + joinPoint.getSignature().getName());
        r.append("args", Arrays.toString(joinPoint.getArgs()));
        return r;
    }

    private Map<String, String> getHeadersInfo(HttpServletRequest request) {
        Map<String, String> map = new HashMap<>();
        Enumeration headerNames = request.getHeaderNames();
        while (headerNames.hasMoreElements()) {
            String key = (String) headerNames.nextElement();
            String value = request.getHeader(key);
            map.put(key, value);
        }
        return map;
    }

}
```
完整示例：Chapter4-2-5

### 其他补充

上述内容主要提供一个思路去实现自定义日志的输出和管理。
我们可以通过jdbc实现日志记录到mongodb，也可以通过spring-data-mongo来记录到mongodb，
当然我们也可以输出到其他数据库，或者输出到消息队列等待其他后续处理等。

对于日志记录到mongodb，也可以直接使用log4mongo实现更为方便快捷。https://github.com/log4mongo/log4mongo-java（too old）


