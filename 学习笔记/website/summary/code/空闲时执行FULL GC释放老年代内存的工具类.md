## 空闲时执行FULL GC释放老年代内存的工具类

#### 执行GC的任务类

```java
import java.lang.management.ManagementFactory;
import java.lang.management.MemoryPoolMXBean;
import java.lang.management.MemoryUsage;
import java.util.Date;
import java.util.List;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

import org.slf4j.Logger;

import com.vip.osp.core.utils.MemoryUnit;

/**
 * Detect old gen usage of current jvm periodically and trigger a gc if necessary.
 */
public class ProactiveGcTask implements Runnable {
	protected Logger log;
	private static final String OLD = "old";
	private static final String TENURED = "tenured";
	private ScheduledExecutorService scheduler;
	private int oldGenOccupancyFraction;//执行gc的阈值,如50表示老年代超过50%则触发gc
	private MemoryPoolMXBean oldMemoryPool;

	public ProactiveGcTask(ScheduledExecutorService scheduler, Logger log, int oldGenOccupancyFraction) {
		this.scheduler = scheduler;
		this.log = log;
		this.oldGenOccupancyFraction = oldGenOccupancyFraction;
	}

	public void run() {
		log.info("ProactiveGcTask starting at: {}", new Date());
		try {
			oldMemoryPool = getOldMemoryPool();
			long maxOldBytes = getMemoryPoolMaxOrCommitted(oldMemoryPool);
			long oldUsedBytes = oldMemoryPool.getUsage().getUsed();
			log.info(String.format( // NOSONAR
					"max old gen: %.2f MB, used old gen: %.2f MB, available old gen: %.2f MB, "
							+ "target occupancy fraction: %d%%, actual fraction: %.2f%%",
					MemoryUnit.BYTES.toMegaBytes(maxOldBytes), MemoryUnit.BYTES.toMegaBytes(oldUsedBytes),
					MemoryUnit.BYTES.toMegaBytes(maxOldBytes - oldUsedBytes), oldGenOccupancyFraction,
					100.0 * oldUsedBytes / maxOldBytes));
			if (needTriggerGc(maxOldBytes, oldUsedBytes, oldGenOccupancyFraction)) {
				preGc();
				doGc();
				postGc();
			} else {
				log.info("no need to trigger gc");
			}
		} catch (Exception e) {
			log.error(e.getMessage(), e);
		} finally {
             //一天后重新执行此ＧＣ任务
			scheduler.reschedule(this);
		}
	}

	private MemoryPoolMXBean getOldMemoryPool() {
		MemoryPoolMXBean pool = null;
		List<MemoryPoolMXBean> memoryPoolMXBeans = ManagementFactory.getPlatformMXBeans(MemoryPoolMXBean.class);
		for (MemoryPoolMXBean memoryPool : memoryPoolMXBeans) {
			String name = memoryPool.getName().trim();
			String lowerCaseName = name.toLowerCase();
			if (lowerCaseName.contains(OLD) || lowerCaseName.contains(TENURED)) {
				pool = memoryPool;
				break;
			}
		}
		return pool;
	}

	private long getMemoryPoolMaxOrCommitted(MemoryPoolMXBean memoryPool) {
		MemoryUsage usage = memoryPool.getUsage();
		long max = usage.getMax();
		max = max < 0 ? usage.getCommitted() : max;
		return max;
	}

	/**
	 * Determine whether or not to trigger gc.
	 * 
	 * @param capacityBytes
	 *            old gen capacity
	 * @param usedBytes
	 *            used old gen
	 * @param occupancyFraction
	 *            old gen used fraction
	 * @return
	 */
	private boolean needTriggerGc(long capacityBytes, long usedBytes, int occupancyFraction) {
		return (100.0 * usedBytes / capacityBytes) >= occupancyFraction;
	}

	/**
	 * Suggests gc.
	 */
	public void doGc() {
		System.gc(); // NOSONAR
	}

	/**
	 * Stuff before gc.
	 */
	public void preGc() {
		log.warn("old gen is occupied larger than occupancy fraction[{}%], trying to trigger gc...",
				oldGenOccupancyFraction);
	}

	/**
	 * Stuff after gc.
	 */
	public void postGc() {
		long maxOldBytes = getMemoryPoolMaxOrCommitted(oldMemoryPool);
		long oldUsedBytes = oldMemoryPool.getUsage().getUsed();
		log.info(String.format( // NOSONAR
				"max old gen: %.2f MB, used old gen: %.2f MB, available old gen: %.2f MB, actual fraction: %.2f%%, after gc.",
				MemoryUnit.BYTES.toMegaBytes(maxOldBytes), MemoryUnit.BYTES.toMegaBytes(oldUsedBytes),
				MemoryUnit.BYTES.toMegaBytes(maxOldBytes - oldUsedBytes), 100.0 * oldUsedBytes / maxOldBytes));
		oldMemoryPool = null;
	}
}
```

#### 内存转换计算类

```java
/**
 * Representation of basic memory units.<br/>
 * Usage example:<br/>
 * assertTrue(MemoryUnit.BYTES.toMegaBytes(1024 * 1024) == 1.0);<br/>
 * assertTrue(MemoryUnit.GIGABYTES.toBytes(1) == 1024.0 * 1024.0 * 1024.0);
 */
public enum MemoryUnit {
	/** Smallest memory unit. */
	BYTES,
	/** "One thousand" (1024) bytes. */
	KILOBYTES,
	/** "One million" (1024x1024) bytes. */
	MEGABYTES,
	/** "One billion" (1024x1024x1024) bytes. */
	GIGABYTES;
	/** Number of bytes in a kilobyte. */
	private final double BYTES_PER_KILOBYTE = 1024.0;
	/** Number of kilobytes in a megabyte. */
	private final double KILOBYTES_PER_MEGABYTE = 1024.0;
	/** Number of megabytes per gigabyte. */
	private final double MEGABYTES_PER_GIGABYTE = 1024.0;

	/**
	 * Returns the number of bytes corresponding to the provided input for a particular unit of memory.
	 *
	 * @param input Number of units of memory.
	 * @return Number of bytes corresponding to the provided number of particular memory units.
	 */
	public double toBytes(final long input) {
		double bytes;
		switch (this) {
		case BYTES:
			bytes = input;
			break;
		case KILOBYTES:
			bytes = input * BYTES_PER_KILOBYTE;
			break;
		case MEGABYTES:
			bytes = input * BYTES_PER_KILOBYTE * KILOBYTES_PER_MEGABYTE;
			break;
		case GIGABYTES:
			bytes = input * BYTES_PER_KILOBYTE * KILOBYTES_PER_MEGABYTE * MEGABYTES_PER_GIGABYTE;
			break;
		default:
			throw new RuntimeException("No value '" + this + "' recognized for enum MemoryUnit.");
		}
		return bytes;
	}

	/**
	 * Returns the number of kilobytes corresponding to the provided input for a particular unit of memory.
	 *
	 * @param input Number of units of memory.
	 * @return Number of kilobytes corresponding to the provided number of particular memory units.
	 */
	public double toKiloBytes(final long input) {
		double kilobytes;
		switch (this) {
		case BYTES:
			kilobytes = input / BYTES_PER_KILOBYTE;
			break;
		case KILOBYTES:
			kilobytes = input;
			break;
		case MEGABYTES:
			kilobytes = input * KILOBYTES_PER_MEGABYTE;
			break;
		case GIGABYTES:
			kilobytes = input * KILOBYTES_PER_MEGABYTE * MEGABYTES_PER_GIGABYTE;
			break;
		default:
			throw new RuntimeException("No value '" + this + "' recognized for enum MemoryUnit.");
		}
		return kilobytes;
	}

	/**
	 * Returns the number of megabytes corresponding to the provided input for a particular unit of memory.
	 *
	 * @param input Number of units of memory.
	 * @return Number of megabytes corresponding to the provided number of particular memory units.
	 */
	public double toMegaBytes(final long input) {
		double megabytes;
		switch (this) {
		case BYTES:
			megabytes = input / BYTES_PER_KILOBYTE / KILOBYTES_PER_MEGABYTE;
			break;
		case KILOBYTES:
			megabytes = input / KILOBYTES_PER_MEGABYTE;
			break;
		case MEGABYTES:
			megabytes = input;
			break;
		case GIGABYTES:
			megabytes = input * MEGABYTES_PER_GIGABYTE;
			break;
		default:
			throw new RuntimeException("No value '" + this + "' recognized for enum MemoryUnit.");
		}
		return megabytes;
	}

	/**
	 * Returns the number of gigabytes corresponding to the provided input for a particular unit of memory.
	 *
	 * @param input Number of units of memory.
	 * @return Number of gigabytes corresponding to the provided number of particular memory units.
	 */
	public double toGigaBytes(final long input) {
		double gigabytes;
		switch (this) {
		case BYTES:
			gigabytes = input / BYTES_PER_KILOBYTE / KILOBYTES_PER_MEGABYTE / MEGABYTES_PER_GIGABYTE;
			break;
		case KILOBYTES:
			gigabytes = input / KILOBYTES_PER_MEGABYTE / MEGABYTES_PER_GIGABYTE;
			break;
		case MEGABYTES:
			gigabytes = input / MEGABYTES_PER_GIGABYTE;
			break;
		case GIGABYTES:
			gigabytes = input;
			break;
		default:
			throw new RuntimeException("No value '" + this + "' recognized for enum MemoryUnit.");
		}
		return gigabytes;
	}
}
```

#### 定时任务管理类

```java
import com.google.common.util.concurrent.ThreadFactoryBuilder;
import com.vip.osp.core.jmx.ProactiveGcTask;
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.time.DateUtils;
import org.apache.commons.lang3.time.FastDateFormat;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.ArrayList;
import java.util.Calendar;
import java.util.Date;
import java.util.List;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class CleanUpScheduler {

    private static Logger logger = LoggerFactory.getLogger(CleanUpScheduler.class);

    private ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1,
            new ThreadFactoryBuilder().setNameFormat("cleanup" + "-%d").setDaemon(true).build());

    /**
     *
     * @param schedulePlans 传递执行的时间范围,如"03:00-05:00,13:00-14:00"以逗号隔开时间戳,默认"02:00-05:00"
     * @param task 执行gc的任务类
     */
    public void schedule(String schedulePlans, Runnable task) {
        List<Long> delayTimes = getDelayMillsList(schedulePlans);
        for (long delayTime : delayTimes) {
            scheduler.schedule(task, delayTime, TimeUnit.MILLISECONDS);
        }
    }


    /**
     * 时隔一天重新执行此任务
     * @param task
     */
    public void reschedule(Runnable task) {
        if (!scheduler.isShutdown()) {
            try {
                scheduler.schedule(task, 24, TimeUnit.HOURS);
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
            }
        }
    }

    public void shutdown(){
        scheduler.shutdown();
    }

    /**
     * Generate delay millis list by given plans string, separated by comma.<br/>
     * eg, 03:00-05:00,13:00-14:00
     */
    public static List<Long> getDelayMillsList(String schedulePlans) {
        List<Long> result = new ArrayList<>();
        String[] plans = StringUtils.split(schedulePlans, ',');
        for (String plan : plans) {
            result.add(getDelayMillis(plan));
        }
        return result;
    }

    /**
     * Get scheduled delay for proactive gc task，cross-day setting is supported.<br/>
     * 01:30-02:40，some time between 01:30-02:40；<br/>
     * 180000，180 seconds later.
     */
    public static long getDelayMillis(String time) {
        String pattern = "HH:mm";
        Date now = new Date();
        if (StringUtils.contains(time, "-")) {
            String start = time.split("-")[0];
            String end = time.split("-")[1];
            if (StringUtils.contains(start, ":") && StringUtils.contains(end, ":")) {
                Date d1 = getCurrentDateByPlan(start, pattern);
                Date d2 = getCurrentDateByPlan(end, pattern);
                while (d1.before(now)) {
                    d1 = DateUtils.addDays(d1, 1);
                }

                while (d2.before(d1)) {
                    d2 = DateUtils.addDays(d2, 1);
                }

                return RandomUtil.nextLong(d1.getTime() - now.getTime(), d2.getTime() - now.getTime());
            }
        } else if (StringUtils.isNumeric(time)) {
            return Long.parseLong(time);
        }

        // default
        return getDelayMillis("02:00-05:00");
    }

    /**
     * return current date time by specified hour:minute
     *
     * @param plan format: hh:mm
     */
    public static Date getCurrentDateByPlan(String plan, String pattern) {

        try {
            FastDateFormat format = FastDateFormat.getInstance(pattern);
            Date end = format.parse(plan);
            Calendar today = Calendar.getInstance();
            end = DateUtils.setYears(end, (today.get(Calendar.YEAR)));
            end = DateUtils.setMonths(end, today.get(Calendar.MONTH));
            end = DateUtils.setDays(end, today.get(Calendar.DAY_OF_MONTH));
            return end;
        } catch (Exception e) {
            logger.error("发生异常",e);
            throw new RuntimeException("获取当前时间异常");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        CleanUpScheduler scheduler = new CleanUpScheduler();
        //触发执行gc,第一个参数为执行时间范围
        scheduler.schedule("13:41-13:42", new ProactiveGcTask(scheduler.scheduler,logger,50));
        Thread.sleep(500000L);
    }
}
```

