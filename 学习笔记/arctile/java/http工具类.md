Http工具类

```java
import com.google.common.collect.Maps;
import org.apache.commons.lang.StringUtils;
import org.apache.http.HttpEntity;
import org.apache.http.HttpStatus;
import org.apache.http.NoHttpResponseException;
import org.apache.http.client.config.RequestConfig;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.conn.ConnectTimeoutException;
import org.apache.http.conn.ConnectionPoolTimeoutException;
import org.apache.http.conn.ssl.SSLConnectionSocketFactory;
import org.apache.http.entity.ContentType;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.DefaultHttpRequestRetryHandler;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;
import org.apache.http.util.EntityUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.net.ssl.*;
import java.net.*;
import java.security.cert.X509Certificate;
import java.util.Map;


/**
 * http工具类
 *  HttpClient使用了责任链模式，所有Executor都实现了ClientExecChain接口的execute()方法，
 *  每个Executor都持有下一个要执行的Executor的引用，这样就会形成一个Executor的执行链条，请求在这个链条上传递。
    如 : 
    1.HttpRequestExecutor                           //发送请求报文，并接收响应信息
	2.MainClientExec(requestExec, connManager, ...)   //main Executor，负责连接管理相关
	3.ProtocolExec(execChain, httpprocessor)           //HTTP协议封装
	4.RetryExec(execChain, retryHandler)               //重试策略
	5.RedirectExec(execChain, routePlanner, redirectStrategy)   //重定向
 */
public class HttpClientHelper {

    private final static Logger logger = LoggerFactory.getLogger(HttpClientHelper.class);

    private String url;
    private SSLConnectionSocketFactory sslFactory;
    private volatile CloseableHttpClient keepAliveHttpclient;
    private PoolingHttpClientConnectionManager manager;
    private final SSLHandler simpleVerifier = new SSLHandler();
    private boolean keepAlive = false;
    
    //每个地址的最大链接数
    int DEFAULT_MAX_PER_ROUTE = 20;
    //最大链接数
    int DEFAULT_SOCKET_MAX_TOTAL = 50;
    //隔多久扫描无用的链接
    int VALIDATE_AFTER_INACTIVITY = 500;

    //链接超时
    int CONNECT_TIMEOUT = 2000;
    //读超时
    int SOCKET_TIMEOUT = 3000;
    //从池中获取链接的超时
    int CONNECTION_REQUEST_TIMEOUT = 2000;

    private String encoding = SysConstants.DEFAULT_CHARTSET;
    private String mimeType = "text/plain";


    public HttpClientHelper(String url) {
        this.url = url;
    }

    /**
    * 常用的构造方法,默认为短连接
    */
    public HttpClientHelper(String url, String mimeType) {
        this.url = url;
        this.mimeType = mimeType;
    }

    /**
     *  可实现长链接的构造方法
         keepAlive代表当前创建的对象使用完后不销毁可在下一个请求中接着使用
    */
    public HttpClientHelper(String url, boolean keepAlive, int DEFAULT_MAX_PER_ROUTE, int DEFAULT_SOCKET_MAX_TOTAL, int VALIDATE_AFTER_INACTIVITY, int CONNECT_TIMEOUT, int SOCKET_TIMEOUT, int CONNECTION_REQUEST_TIMEOUT, String encoding, String mimeType) {
        this.url = url;
        this.keepAlive = keepAlive;
        this.DEFAULT_MAX_PER_ROUTE = DEFAULT_MAX_PER_ROUTE;
        this.DEFAULT_SOCKET_MAX_TOTAL = DEFAULT_SOCKET_MAX_TOTAL;
        this.VALIDATE_AFTER_INACTIVITY = VALIDATE_AFTER_INACTIVITY;
        this.CONNECT_TIMEOUT = CONNECT_TIMEOUT;
        this.SOCKET_TIMEOUT = SOCKET_TIMEOUT;
        this.CONNECTION_REQUEST_TIMEOUT = CONNECTION_REQUEST_TIMEOUT;
        this.encoding = encoding;
        this.mimeType = mimeType;
    }

    /**
    * 调用方法 可带http头
    */
    public String submit(String message, Map<String, String> header) {
        CloseableHttpResponse response = null;
        CloseableHttpClient client = null;
        try {
            ContentType contentType = ContentType.create(mimeType, encoding);
            HttpEntity entity = new StringEntity(message, contentType);
            HttpPost httpPost = new HttpPost(url);
            for (Map.Entry<String, String> head : header.entrySet()) {
                httpPost.addHeader(head.getKey(), head.getValue());
            }
            httpPost.setEntity(entity);

            client = getHttpClient();
            response = client.execute(httpPost);

            int statusCode = response.getStatusLine().getStatusCode();
            String result = "";
            if (HttpStatus.SC_OK != statusCode) {
                logger.info("response status code:{},response：{}", statusCode, result);
                throw new ConnectionException("remote server response status is not ok, but " + statusCode);
            }
            result = EntityUtils.toString(response.getEntity(), encoding);
            closeHttpConnection(response, client, false);
            return result;
        } catch (IllegalStateException e) {
            closeHttpConnection(response, client, true);
            throw new ConnectionException("http connection error, incorrect connection status", e, CallRespMessage.connectionTimeout);
        } catch (SSLHandshakeException e) {
            closeHttpConnection(response, client, true);
            throw new ConnectionException("Remote host error", e, CallRespMessage.connectionTimeout);
        } catch (NoHttpResponseException e) {
            closeHttpConnection(response, client, true);
            throw new ConnectionException("remote server response error", e, CallRespMessage.connectionTimeout);
        } catch (ConnectionPoolTimeoutException e) {
            closeHttpConnection(response, client, true);
            throw new ConnectionException("http wait for work thread timeout", e, CallRespMessage.connectionTimeout);
        } catch (ConnectTimeoutException e) {
            closeHttpConnection(response, client, true);
            throw new ConnectionException("http connection timeout", e, CallRespMessage.connectionTimeout);
        } catch (SocketTimeoutException e) {
            closeHttpConnection(response, client, true);
            throw new ConnectionException("http read timeout", e, CallRespMessage.connectionTimeout);
        } catch (Exception e) {
            closeHttpConnection(response, client, true);
            throw new ConnectionException("http request failed", e);
        }
    }

    /**
    * 通过keepAlive属性来决定是否创建长链接
    * HttpClien中使用了连接池来管理持有连接，同一条TCP链路上，连接是可以复用的。
      HttpClient通过连接池的方式进行连接持久化。
    */
    private CloseableHttpClient getHttpClient() throws Exception {
        if (keepAlive) {
            if (keepAliveHttpclient != null) {
                return keepAliveHttpclient;
            }

            synchronized (this) {
                if (keepAliveHttpclient != null) {
                    return keepAliveHttpclient;
                }
                keepAliveHttpclient = createConnection();
                return keepAliveHttpclient;
            }
        }
        return createConnection();
    }

    private CloseableHttpClient createConnection() throws Exception {
        String ip = getRequestIp();
        logger.info("create http client,requestIp:{}", ip);
        HttpClientBuilder builder = HttpClients.custom();
        //默认不开启重试
        builder.setRetryHandler(new DefaultHttpRequestRetryHandler(0, false));

        RequestConfig config = RequestConfig.custom()
                .setConnectionRequestTimeout(CONNECTION_REQUEST_TIMEOUT)
                .setConnectTimeout(CONNECT_TIMEOUT)
                .setSocketTimeout(SOCKET_TIMEOUT).build();

        //创建https请求
        if (url.startsWith("https")) {
            builder.setSSLSocketFactory(getSSLConnSocketFactory());
        }

        //长链接的话使用池化技术
        if (keepAlive) {
            logger.info("create keep alive connection");
            builder.setConnectionManager(getPoolConnectionManager());
        }

        //evictExpiredConnections方法会创建一个线程来清理池中过期的空闲链接
        builder.evictExpiredConnections().setDefaultRequestConfig(config);

        System.setProperty("jsse.enableSNIExtension", "false");
        return builder.build();
    }

    private String getRequestIp() throws MalformedURLException, UnknownHostException {
        if (StringUtils.isBlank(this.url)) {
            return null;
        }
        URL url = new URL(this.url);
        String host = url.getHost();
        String ip = InetAddress.getByName(host).getHostAddress();
        return ip;
    }

    private PoolingHttpClientConnectionManager getPoolConnectionManager() {
        manager = new PoolingHttpClientConnectionManager();
        manager.setMaxTotal(DEFAULT_SOCKET_MAX_TOTAL); //最大链接数
        manager.setDefaultMaxPerRoute(DEFAULT_MAX_PER_ROUTE); //每个路由的最大链接数
        //设置到某个路由的最大连接数，会覆盖defaultMaxPerRoute
        //manager.setMaxPerRoute(new HttpRoute(new HttpHost("${host}", 80)), 150); 
        manager.setValidateAfterInactivity(VALIDATE_AFTER_INACTIVITY);
        logger.info("create http connection pool, maxConn:{}, maxConnPerRoute:{},validateAfterInactivity:{}", DEFAULT_SOCKET_MAX_TOTAL, DEFAULT_MAX_PER_ROUTE, VALIDATE_AFTER_INACTIVITY);
        return manager;
    }

    private void closeHttpConnection(CloseableHttpResponse response, CloseableHttpClient httpClient, boolean closeKeepAliveHttpClient) {
        try {
            if (response != null) {
                EntityUtils.consume(response.getEntity());
            }
            if (Boolean.valueOf(keepAlive)) {
                if (closeKeepAliveHttpClient) {
                    logger.info("close http keep alive connection when exception");
                    if (manager != null) {
                        manager.close();
                        manager = null;
                    }
                    if (keepAliveHttpclient != null) {
                        keepAliveHttpclient.close();
                        keepAliveHttpclient = null;
                    }
                }
            } else {
                if (manager != null) {
                    manager = null;
                }
                if (keepAliveHttpclient != null) {
                    keepAliveHttpclient = null;
                }
                if (httpClient != null) {
                    httpClient.close();
                }
            }
        } catch (Exception e) {
            logger.error("close http connection error:", e);
        }
    }

    public boolean isKeepAlive() {
        return keepAlive;
    }

    public void setKeepAlive(boolean keepAlive) {
        this.keepAlive = keepAlive;
    }

    public int getDEFAULT_MAX_PER_ROUTE() {
        return DEFAULT_MAX_PER_ROUTE;
    }

    public void setDEFAULT_MAX_PER_ROUTE(int DEFAULT_MAX_PER_ROUTE) {
        this.DEFAULT_MAX_PER_ROUTE = DEFAULT_MAX_PER_ROUTE;
    }

    public int getDEFAULT_SOCKET_MAX_TOTAL() {
        return DEFAULT_SOCKET_MAX_TOTAL;
    }

    public void setDEFAULT_SOCKET_MAX_TOTAL(int DEFAULT_SOCKET_MAX_TOTAL) {
        this.DEFAULT_SOCKET_MAX_TOTAL = DEFAULT_SOCKET_MAX_TOTAL;
    }

    public int getVALIDATE_AFTER_INACTIVITY() {
        return VALIDATE_AFTER_INACTIVITY;
    }

    public void setVALIDATE_AFTER_INACTIVITY(int VALIDATE_AFTER_INACTIVITY) {
        this.VALIDATE_AFTER_INACTIVITY = VALIDATE_AFTER_INACTIVITY;
    }

    public int getCONNECT_TIMEOUT() {
        return CONNECT_TIMEOUT;
    }

    public void setCONNECT_TIMEOUT(int CONNECT_TIMEOUT) {
        this.CONNECT_TIMEOUT = CONNECT_TIMEOUT;
    }

    public int getSOCKET_TIMEOUT() {
        return SOCKET_TIMEOUT;
    }

    public void setSOCKET_TIMEOUT(int SOCKET_TIMEOUT) {
        this.SOCKET_TIMEOUT = SOCKET_TIMEOUT;
    }

    public int getCONNECTION_REQUEST_TIMEOUT() {
        return CONNECTION_REQUEST_TIMEOUT;
    }

    public void setCONNECTION_REQUEST_TIMEOUT(int CONNECTION_REQUEST_TIMEOUT) {
        this.CONNECTION_REQUEST_TIMEOUT = CONNECTION_REQUEST_TIMEOUT;
    }

    public String getEncoding() {
        return encoding;
    }

    public void setEncoding(String encoding) {
        this.encoding = encoding;
    }

    public synchronized SSLConnectionSocketFactory getSSLConnSocketFactory() throws Exception {
        if (sslFactory != null) {
            return sslFactory;
        }
        SSLContext sslContext = SSLContext.getInstance("SSL", "SunJSSE");
        sslContext.init(null, new TrustManager[]{simpleVerifier}, null);
        String[] supportedProtocols = {"TLSv1", "TLSv1.2", "TLSv1.1", "SSLv2Hello", "SSLv3"};
        sslFactory = new SSLConnectionSocketFactory(sslContext, supportedProtocols, null, new HostnameVerifier() {

            @Override
            public boolean verify(String arg0, SSLSession arg1) {
                return true;
            }

        });
        return sslFactory;
    }

    private class SSLHandler implements X509TrustManager, HostnameVerifier {
        private SSLHandler() {
        }

        @Override
        public void checkClientTrusted(X509Certificate[] arg0, String arg1) {
        }

        @Override
        public void checkServerTrusted(X509Certificate[] arg0, String arg1) {
        }

        @Override
        public X509Certificate[] getAcceptedIssuers() {
            return null;
        }

        @Override
        public boolean verify(String arg0, SSLSession arg1) {
            return true;
        }
    }

}


 public static void main(String[] args) {
        String url = "https://ip:port/preSvr";
        String mimeType = "application/xml";
        Map<String, String> httpHeader = Maps.newLinkedHashMap();
        httpHeader.put("Content-Type", "application/xml;charset=utf-8");
        httpHeader.put("MsgTp", "${msgType}");
        httpHeader.put("OriIssrId", "#");
        String message = "<?xml version='1.0' encoding='utf-8'?><root><MsgHeader><SndDt>${DATE}</SndDt><MsgTp>epcc.401.001.01</MsgTp><IssrId>${issrid}</IssrId><Drctn>11</Drctn><SignSN>${signsn}</SignSN></MsgHeader><MsgBody><InstgId>#</InstgId></MsgBody></root>{S:#}";
        HttpClientHelper httpClientHelper = new HttpClientHelper(url, mimeType);
        String submit = httpClientHelper.submit(message, httpHeader);
        System.out.println(submit);
    }

```

```java
public enum CallRespMessage {

    success("SUCCESS"),
    uncaught("UNCAUGHT_ERROR"),
    invalidRequest("INVALID_REQUEST"),
    invalidRouteOrg("INVALID_ROUTE_ORG"),
    invalidBizType("INVALID_BIZ_TYPE"),
    verifyError("VERIFY_ERROR"),
    configError("CONFIG_ERROR"),
    connectionError("CONNECTION_ERROR"),
    connectionTimeout("CONNECTION_TIMEOUT"),
    validationError("VALIDATION_ERROR");

    private String message;

    CallRespMessage(String message) {
        this.message = message;
    }

    public String getMessage() {
        return message;
    }

}
```

```java
public class ConnectionException extends RuntimeException {

    private CallRespMessage callRespMessage = CallRespMessage.connectionError;

    public ConnectionException() {
        super();
    }

    public ConnectionException(String message) {
        super(message);
    }

    public ConnectionException(String message, Throwable cause) {
        super(message, cause);
    }

    public ConnectionException(Throwable cause) {
        super(cause);
    }

    public ConnectionException(CallRespMessage callRespMessage) {
        super();
        this.callRespMessage = callRespMessage;
    }

    public ConnectionException(String message, CallRespMessage callRespMessage) {
        super(message);
        this.callRespMessage = callRespMessage;
    }

    public ConnectionException(String message, Throwable cause, CallRespMessage callRespMessage) {
        super(message, cause);
        this.callRespMessage = callRespMessage;
    }

    public CallRespMessage getResponse() {
        return callRespMessage;
    }

}
```

