## Redis集群执行lua脚本

#### 基本思想

+ xml定义lua脚本
+ map缓存lua脚本映射
+ 依赖脚本键执行对应的脚本

#### lua脚本

```xml
<?xml version="1.0" encoding="UTF-8"?>
<root>
	<redislua luaname="CONTINUOUS_COUNT_TIMES_WRITE">
		--[[
		网联多中心自动熔断与恢复一期的时候使用，现已废弃

		KEYS[1]: 统计 key
		ARGV[1]: 数据发生值
		ARGV[2]: 操作类型：1-累加值，2-设置值
		返回当前累计值
		--]]

		local k = KEYS[1];
		local type = tonumber(ARGV[2]);
		local v1 = redis.call( 'GET', k );
		local v = tonumber(ARGV[1]);
		if(type == 1 and v1) then
		v = v1 + v;
		end
		redis.call( 'SET', k, tostring(v) );
		return v;
	</redislua>

	<redislua luaname="VALID_SUCC_SERVICE">
		--[[
		有效状态下探测请求成功业务处理

		KEYS[1]: 连续失败次数key
		KEYS[2]: 成功记录数
		ARGV[1]: 清除该时间之前的数据（yyyyMMddHHmmss）
		ARGV[2]: 成功记录score
		ARGV[3]: 成功记录member
		ARGV[4]: 过期时间
		--]]
		-- 重置连续失败次数
		redis.call('SET', KEYS[1], '0');

		-- 清除指定时间前的数据
		local sk = KEYS[2];
		local count = redis.call("ZCOUNT",sk,0,'(' .. ARGV[1] );
		if(count > 2) then
		redis.call( 'ZREMRANGEBYRANK', sk, 0, count-2 );
		end

		-- 增加成功请求记录
		redis.call('ZADD', sk, ARGV[2], ARGV[3]);
		-- 设置过期时间
		redis.call( 'EXPIRE', sk, tonumber(ARGV[4]) );
	</redislua>

	<redislua luaname="VALID_FAIL_SERVICE">
		--[[
		有效状态下探测请求失败业务处理

		KEYS[1]: 连续成功次数key
		KEYS[2]: 连续失败次数key
		KEYS[3]: 失败记录
		KEYS[4]: 成功记录
		ARGV[1]: 连续失败次数阈值
		ARGV[2]: 失败率阈值
		ARGV[3]: 清除该时间之前的数据（yyyyMMddHHmmss）
		ARGV[4]: 失败记录score
		ARGV[5]: 失败记录member
		ARGV[6]: 过期时间
		ARGV[7]: 时间窗口
		返回结果：0-默认，1-连续失败次数触发熔断，2-失败率触发熔断
		--]]
		-- 定义返回结果
		local r = 0;

		-- 重置连续成功次数
		redis.call('SET', KEYS[1], '0');

		-- 连续失败次数+1
		local fcv = redis.call('GET', KEYS[2]);
		fcv = (tonumber(fcv) or 0) + 1;
		redis.call('SET', KEYS[2], fcv);

		-- 清除指定时间前的数据
		local fk = KEYS[3];
		local count = redis.call("ZCOUNT",fk,0,'(' .. ARGV[3] );
		if(count > 2) then
		redis.call( 'ZREMRANGEBYRANK', fk, 0, count-2 );
		end

		-- 增加失败请求记录
		redis.call('ZADD', fk, ARGV[4], ARGV[5]);
		-- 设置过期时间
		redis.call( 'EXPIRE', fk, tonumber(ARGV[6]) );

		-- 时间窗口内失败记录数
		local fcount = redis.call("ZCOUNT",fk,ARGV[7], ARGV[4] );
		-- 时间窗口内成功记录数
		local scount = redis.call("ZCOUNT",KEYS[4],ARGV[7], ARGV[4] );

		if(fcv >= tonumber(ARGV[1])) then
			-- 连续失败触发熔断
			r = 1;
		elseif(fcount/(fcount+scount) >= tonumber(ARGV[2])) then
			-- 失败率触发熔断
			r = 2;
		end
		return r;
	</redislua>

	<redislua luaname="BREAK_SUCC_SERVICE">
		--[[
		熔断状态下探测请求成功业务处理

		KEYS[1]: 连续成功次数key
		KEYS[2]: 连续失败次数key
		KEYS[3]: 成功记录
		KEYS[4]: 半开闭总请求数key
		KEYS[5]: 半开闭失败请求数key
		ARGV[1]: 连续成功次数阈值
		ARGV[2]: 探测请求数阈值
		ARGV[3]: 继续熔断失败率阈值
		ARGV[4]: 清除该时间之前的数据（yyyyMMddHHmmss）
		ARGV[5]: 成功记录score
		ARGV[6]: 成功记录member
		ARGV[7]: 过期时间
		返回结果：0-默认，1-连续成功次数触发恢复，2-半开闭失败次数没达到阈值触发恢复，3-继续熔断
		--]]
		-- 定义返回结果
		local r = 0;
		-- 重置连续失败次数
		redis.call('SET',KEYS[2], '0');
		-- 连续成功次数+1
		local scv = redis.call('GET',KEYS[1]);
		scv = (tonumber(scv) or 0) + 1;
		redis.call('SET',KEYS[1], scv);

		-- 清除指定时间前的数据
		local fk = KEYS[3];
		local count = redis.call("ZCOUNT",fk,0,'(' .. ARGV[4] );
		if(count > 2) then
		redis.call( 'ZREMRANGEBYRANK', fk, 0, count-2 );
		end

		-- 增加成功请求记录
		redis.call('ZADD', fk, ARGV[5], ARGV[6]);
		-- 设置过期时间
		redis.call( 'EXPIRE', fk, tonumber(ARGV[7]) );

		-- 半开闭总请求数+1
		local hv = redis.call('GET',KEYS[4]);
		hv = (tonumber(hv) or 0) + 1;
		redis.call('SET',KEYS[4], hv);

		-- 半开闭失败请求数
		local fhv = redis.call('GET', KEYS[5]);
		fhv = tonumber(fhv) or 0;

		if(scv >= tonumber(ARGV[1])) then
			-- 连续成功次数达到恢复阈值
			r = 1;
			return r;
		end

		if(hv >= tonumber(ARGV[2])) then
			-- 失败请求数阈值,四舍五入
			local ftv = math.floor(hv * tonumber(ARGV[3]) * 0.8 + 0.5);
			if(fhv >= ftv) then
				-- 半开闭总请求数达到10次，且半开闭失败请求数达到阈值(继续熔断)
				r = 3;
			else
				-- 半开闭总请求数达到10次，且半开闭失败请求数未达到阈值(触发恢复)
				r = 2;
			end
		end
		return r;
	</redislua>

	<redislua luaname="BREAK_FAIL_SERVICE">
		--[[
		熔断状态下探测请求失败业务处理

		KEYS[1]: 连续成功次数key
		KEYS[2]: 失败记录
		KEYS[3]: 半开闭总请求数key
		KEYS[4]: 半开闭失败请求数key
		ARGV[1]: 清除该时间之前的数据（yyyyMMddHHmmss）
		ARGV[2]: 失败记录score
		ARGV[3]: 失败记录member
		ARGV[4]: 过期时间
		--]]
		-- 重置连续成功次数
		redis.call('SET', KEYS[1], '0');

		-- 清除指定时间前的数据
		local fk = KEYS[2];
		local count = redis.call("ZCOUNT",fk,0,'(' .. ARGV[1] );
		if(count > 2) then
		redis.call( 'ZREMRANGEBYRANK', fk, 0, count-2 );
		end

		-- 增加失败请求记录
		redis.call('ZADD', fk, ARGV[2], ARGV[3]);
		-- 设置过期时间
		redis.call( 'EXPIRE', fk, tonumber(ARGV[4]) );

		-- 半开闭总请求数+1
		local hv = redis.call('GET',KEYS[3]);
		hv = (tonumber(hv) or 0) + 1;

		redis.call('SET',KEYS[3], hv);
		-- 半开闭失败请求数+1
		local fhv = redis.call('GET',KEYS[4]);
		fhv = (tonumber(fhv) or 0) + 1;
		redis.call('SET',KEYS[4], fhv);
	</redislua>

	<redislua luaname="RESET_SERVICE">
		--[[
		重置数据业务处理

		KEYS: 需要重置的key列表
		--]]
		for i=1, #KEYS do
			local k = KEYS[i];
			redis.call( 'SET', k, '0' );
		end
		return 0;
	</redislua>

</root>

```

####  加载XML到缓存

```java
import org.dom4j.Document;
import org.dom4j.DocumentException;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextRefreshedEvent;
import org.springframework.stereotype.Component;

import java.util.Iterator;

@Component
public class LuaCacheInit implements ApplicationListener<ContextRefreshedEvent> {

	public static Logger logger = LoggerFactory.getLogger(LuaCacheInit.class);
	@Override
	public void onApplicationEvent(ContextRefreshedEvent event) {
		// 项目启动降级开关
		Boolean loadStatus = false;
		//加载redis lua
		try {
			loadRedisLua();
		}catch (Exception e){
			loadStatus = true;
			logger.info("Project Run Load redis lua Error!", e);
		}
		
		if (!loadStatus) {
			logger.info("Reconfront Project Run Success!");
		} else {
			logger.error("Reconfront Project Run Fail!");
			throw new RuntimeException();
		}
	}

	private void loadRedisLua() throws DocumentException{
		SAXReader saxReader = new SAXReader();   
		Document document = saxReader.read(LuaCacheInit.class.getClassLoader().getResourceAsStream("prop/redis-lua.xml"));
		Element root=document.getRootElement();   
		for(Iterator<?> i = root.elementIterator(); i.hasNext();){   
			Element element = (Element) i.next();
			String key = element.attributeValue("luaname");
			DataCache.redisLuaCache.put(key, element.getStringValue());
		}
	}
	
}


public class DataCache {
	/** lua脚本缓存 */
	public static Map<String, String> redisLuaCache = new ConcurrentHashMap<String, String>();
}    
```



#### 

#### 脚本执行器

```java
//脚本执行model
@Data
public class BatchCounterModel {

    private String luaId;

    private List<String> keys;

    private List<String> args;
}
```

```java

@Component
public class CounterRepositoryImpl {
    private Logger logger = LoggerFactory.getLogger(CounterRepositoryImpl.class);
    @Autowired
    private JedisCluster jedisCluster;
   
    public int batch(BatchCounterModel batchCounterModel) {
        logger.info("set-count-times begin, batchCounterModel:{}", JSON.toJSONString(batchCounterModel));
        long t1 = System.currentTimeMillis();
        String lua = DataCache.redisLuaCache.get(batchCounterModel.getLuaId());
        int result = 0;
        try {
            Long eval = (Long) jedisCluster.eval(lua, batchCounterModel.getKeys(), batchCounterModel.getArgs());
            if (eval != null) {
                result = eval.intValue();
            }
        }catch (Exception e) {
            logger.error("CounterRepositoryImpl set error!", e);
        }
        logger.info("set-count-times end, spend time:{}ms, return type:{}",System.currentTimeMillis() - t1, result);
        return result;
    }
}
```

#### 执行例子

```java
	//一个执行的例子
    public void afterConnectSuccess() {
        BatchCounterModel batchCounterModel = new BatchCounterModel();
        List<String> keys = new ArrayList<>();
        List<String> args = new ArrayList<>();

        keys.add("{channel:break:upopwxidc:10}:continuous:succ");
        keys.add("{channel:break:upopwxidc:10}:succ");

        DateTime now = DateTime.now();
        DateTime window = now.minusSeconds(60);
        String windowSecounds = window.toString("yyyyMMddHHmmss");

        //时间窗口起始秒数
        args.add(windowSecounds);
        //成功记录score
        args.add(now.toString(SECOUND_TIME_PATTERN));
        //成功记录member
        args.add(StringUtils.replace(UUID.randomUUID().toString(), "-", ""));
        //数据缓存时间，时间窗口*1.5
        args.add(60 * 1.5 + "");

        batchCounterModel.setKeys(keys);
        batchCounterModel.setArgs(args);
        batchCounterModel.setLuaId("VALID_SUCC_SERVICE");
        circuitBreaker.batchService(batchCounterModel);
    }
```





