## JVM状态辅助类

#### 适用场景

在团队初期或团队无有效的监控支撑工具项目时,可临时辅助来帮忙判断记录JVM趋势.

#### 代码

```java
import java.util.Timer;
import java.util.TimerTask;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

public class SystemStatusTask extends TimerTask {
	private static final Log logger = LogFactory.getLog(SystemStatusTask.class);
	public static String RUNTIME_LOGGER_FLAG = "_runtime_logger_flag_";
	private long repeatInterval = 30000;

	public void setRepeatInterval(long repeatInterval) {
		this.repeatInterval = repeatInterval;
	}

	public void init() {
		synchronized (Object.class) {
			String flag = System.getProperty(RUNTIME_LOGGER_FLAG);
			if (flag == null) {
				System.setProperty(RUNTIME_LOGGER_FLAG, "true");
				Timer timer = new Timer();
				timer.schedule(this, 0, repeatInterval);
			} else {
				logger.info("runtime logger already running............");
			}
		}
	}

	public void run() {
		long tid = Thread.currentThread().getId();
		logger.info(tid
				+ "---------------------jvm status info : nowTreadCount["
				+ Thread.activeCount() + "]");
		logger.info(tid + "---------------------jvm status info : maxMemory["
				+ Runtime.getRuntime().maxMemory()/1024/1024 + "M]");
		logger.info(tid + "---------------------jvm status info : totalMemory["
				+ Runtime.getRuntime().totalMemory()/1024/1024 + "M]");
		logger.info(tid + "---------------------jvm status info : freeMemory["
				+ Runtime.getRuntime().freeMemory()/1024/1024 + "M]");
	}

	public static void main(String[] args) {
		System.out.println(Runtime.getRuntime().totalMemory()/1024/1024+"M");
	}
}
```

