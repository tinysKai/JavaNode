#### Logback日志配置

#### 参考配置

```groovy
import ch.qos.logback.classic.AsyncAppender
import ch.qos.logback.classic.encoder.PatternLayoutEncoder
import ch.qos.logback.classic.filter.ThresholdFilter
import java.nio.charset.Charset

statusListener(OnConsoleStatusListener)

def props = new Properties()
props.load(this.getClass().getClassLoader().getResourceAsStream("properties/application.properties"))

def config = new ConfigSlurper().parse(props)

def env =  System.properties['spring.profiles.active'] ?: 'production'
def appenderList = []
def logLevel = INFO

def appName = config.app.name
def instanceName =  System.properties['app.instance.name'] ?: appName
def LOG_RECEIVER_DIR = '/tmp/logs/log_receiver/'+ instanceName
def LOG_MERCURY_DIR = '/tmp/logs/trace/logs/' + instanceName

jmxConfigurator()

if (env == 'production') {
    //配置日志异步化
    appenderList.add("ROLLING-ASYNC")
    logLevel = INFO
} else if(env == 'integratetest') {
    appenderList.add("ROLLING-ASYNC")
    logLevel = INFO
} else if(env == 'development') {
    appenderList.add("CONSOLE")
    appenderList.add("ROLLING-ASYNC")
    logLevel = INFO
}

if(env=='production' || env=='integratetest'  || env=='development') {
	appender("ROLLING", RollingFileAppender) {
	    file = "${LOG_RECEIVER_DIR}/application.log"
		encoder(PatternLayoutEncoder) {
		    pattern = "[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%level] [traceId=%X{traceId}] [%thread] [%logger{50}] >>> %msg%n"
		}
		rollingPolicy(TimeBasedRollingPolicy) {
			fileNamePattern = "${LOG_RECEIVER_DIR}/application.%d{yyyy-MM-dd}.log"
		}
	}
	appender("ROLLING-ASYNC", AsyncAppender) {
        // discardingThreshold，当队列的剩余容量小于这个阈值并且当前日志level TRACE, DEBUG or INFO，
        //则丢弃这些日志。默认为queueSize大小的20%
		discardingThreshold = 0
		queueSize = 1024
         //永不阻塞
		neverBlock=true	
         //指向上面定义的`ROLLING`appender
		appenderRef("ROLLING")
    }
}


//特别地独立一个单独的flow日志
appender("FLOW", RollingFileAppender) {
	file = "${LOG_RECEIVER_DIR}/flow.log"
	rollingPolicy(TimeBasedRollingPolicy) {
		fileNamePattern = "${LOG_RECEIVER_DIR}/flow.%d{yyyy-MM-dd}.log"
	}
	encoder(PatternLayoutEncoder) {
		charset = Charset.forName("UTF-8")
		pattern = "%date{HH:mm:ss.SSS} [traceId=%X{traceId}] %-5level %logger{36} - %msg%n"
	}
}

//AsyncAppender并不处理日志，只是将日志缓冲到一个BlockingQueue里面去，并在内部创建一个工作线程从队列头部获取日志，
//之后将获取的日志循环记录到附加的其他appender上去，从而达到不阻塞主线程的效果。
//因此AsyncAppender仅仅充当的是事件转发器，必须引用另外一个appender来做事
appender("FLOW-ASYNC", AsyncAppender) {
	discardingThreshold = 0
	queueSize = 1024
	neverBlock=true
    //异步指向上面的flow日志,appender-ref表示AsyncAppender使用哪个具体的`appender`进行日志输出
	appenderRef("FLOW")
}
//定义一个logger
logger("flow", INFO, ["FLOW-ASYNC"], false)

root(logLevel, appenderList)
```

#### 使用后的日志展示

```
[2019-12-13 09:26:53.722] [INFO] [traceId=8342249153762600319] [线程] [类] >>> 日志内容
```

#### TRACEID使用方式

使用MDC来put使用

```java
//简单的说就是从上游或者某个中转的地方获取到一个链路标志,然后使用ThreadLocal来传递

import org.slf4j.MDC;
public class TraceLogBeforeFilter extends ####Filter
{
	static final String TRACE_ID = "traceId";


    @Override
    public <I> void doFilter(FilterContext filterContext) throws OspException {
        if (####Switch()) {
            //从Mercury获取TraceId，设置到MDC
            String traceId = getTraceId();//假设这个方法能拿到一个上下文链路标记
            if (StringUtils.isNotEmpty(traceId)) {
                MDC.put(TRACE_ID, traceId);
            }
        }

    }

    @Override
    public void onException(FilterContext filterContext, Throwable e) {
        if (####Switch()) {
            MDC.remove(TRACE_ID);
        }

    }
}
```

#### 单独的flow文件的使用方式

```java
public class ServiceImpl {

    //业务独立的日志,就是平常使用的日志logger
    private static final Logger logger = LoggerFactory.getLogger(ServiceImpl.class);
    private static final String FLOW_LOG = "flow";
    //单独的flow日志
    private static final Logger flowLog = LoggerFactory.getLogger(FLOW_LOG);
    
    public void execute(RequestModel requestModel)  {
         //写到独立的flow日志
	     flowLog.info("{},{}","测试","日志");
    }
}    
```

#### logback属性解释

**`<logger>`用来设置某一个包或者具体某一个类的日志打印级别、以及指定`<appender>`**。`<logger>`可以包含零个或者多个`<appender-ref>`元素，标识这个appender将会添加到这个logger。`<logger>`仅有一个name属性、一个可选的level属性和一个可选的additivity属性：

- name：用来指定受此logger约束的某一个包或者具体的某一个类
- level：用来设置打印级别，五个常用打印级别从低至高依次为TRACE、DEBUG、INFO、WARN、ERROR，如果未设置此级别，那么当前logger会继承上级的级别
- additivity：是否向上级logger传递打印信息，默认为true

`<root>`也是`<logger>`元素，但是**它是根logger，只有一个level属性，因为它的name就是ROOT**

