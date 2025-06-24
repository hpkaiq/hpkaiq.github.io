# Spring Cloud Java反向代理访问阿里云OSS私有资源


最近用到阿里云oss，有阿里云服务器，通过代理内网访问可以实现免除OSS流量费，查到很多nginx反向代理的教程，但是纯java实现没有找到，感觉可以试一试。

<!--more-->
（一）
首先要解决反向代理的问题，搜到```org.mitre.dsmiley.httpproxy.ProxyServlet```可解决。

```xml
<dependency>
   <groupId>org.mitre.dsmiley.httpproxy</groupId>
   <artifactId>smiley-http-proxy-servlet</artifactId>
   <version>1.11</version>
</dependency>
```

```java
@Configuration
public class MyConfig {
    @Bean
    public ServletRegistrationBean servletRegistrationAliBean01() {

        ServletRegistrationBean oss  = new ServletRegistrationBean(new ProxyServlet(), "/ali/bucket_name/*");//修改为自己的bucket_name
        oss.setName("bucket_name");
        oss.addInitParameter("targetUri", "http://bucket_name.oss-cn-beijing.aliyuncs.com");//本地测试环境用外网地址，线上改成内网访问的地址
        oss.addInitParameter(ProxyServlet.P_LOG, "true");
        return oss;
    }

}
```
（二）
Controller配置如下，转发到自己配置的反向代理
```
@RestController
@RefreshScope
@RequestMapping("/api/es")
public class TestController {
     @RequestMapping("/static/**")
    public void testVideoOne(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException, URISyntaxException {
        String resourcePath = request.getRequestURI().replace("/api/es/static","");
        HeaderUtil.setHeaderOwn(request,resourcePath);
        request.getRequestDispatcher("/ali" + resourcePath).forward(request, response);
    }
}
```
（三）
其中为request设置Header以认证OSS访问的HeaderUtil如下，其中有OSS Authorization计算的java实现，用java的反射在request中添加Header信息。
```

import org.apache.commons.codec.binary.Base64;
import org.apache.tomcat.util.http.MimeHeaders;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import javax.servlet.http.HttpServletRequest;
import java.lang.reflect.Field;
import java.net.URISyntaxException;
import java.nio.charset.Charset;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;
import java.util.TimeZone;

public class HeaderUtil {
    /**
     * 给request加入Header信息，验证OSS访问权限
     * @param request
     * @throws URISyntaxException
     */
    public static void setHeaderOwn(HttpServletRequest request,String resourcePath) throws URISyntaxException {
        SimpleDateFormat sdf = new SimpleDateFormat("EEE, dd MMM yyyy HH:mm:ss 'GMT'", Locale.US);
        sdf.setTimeZone(TimeZone.getTimeZone("GMT"));
        String date_GMT = sdf.format(new Date());
        String data = request.getMethod() + "\n"
                + "\n"
                + "\n"
                + date_GMT + "\n"
                + resourcePath;
        String signature = genHMAC("AccessKeySecret", data);//AccessKeySecret改成自己的
        reflectSetParam(request, "Authorization", "OSS AccessKeyId:" + signature);//AccessKeyId改成自己的
        reflectSetParam(request, "Date", date_GMT);
    }

    /**
     * 反射修改header信息，key-value键值对加入到header中
     *
     * @param request
     * @param key
     * @param value
     */
    private static void reflectSetParam(HttpServletRequest request, String key, String value) {
        Class<? extends HttpServletRequest> requestClass = request.getClass();
        // System.out.println("request实现类="+requestClass.getName());
        try {
            Field request1 = requestClass.getDeclaredField("request");
            request1.setAccessible(true);
            Object o = request1.get(request);
            Field coyoteRequest = o.getClass().getDeclaredField("coyoteRequest");
            coyoteRequest.setAccessible(true);
            Object o1 = coyoteRequest.get(o);
            // System.out.println("coyoteRequest实现类="+o1.getClass().getName());
            Field headers = o1.getClass().getDeclaredField("headers");
            headers.setAccessible(true);
            MimeHeaders o2 = (MimeHeaders) headers.get(o1);
            o2.addValue(key).setString(value);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }


    /**
     * HmacSHA1 Base64加密
     * @param key
     * @param data
     * @return
     */
    public static String genHMAC(String key, String data) {
        byte[] result = null;
        try {
            //根据给定的字节数组构造一个密钥,第二参数指定一个密钥算法的名称
            SecretKeySpec signinKey = new SecretKeySpec(key.getBytes(), "HmacSHA1");
            //生成一个指定 Mac 算法 的 Mac 对象
            Mac mac = Mac.getInstance("HmacSHA1");
            //用给定密钥初始化 Mac 对象
            mac.init(signinKey);
            //完成 Mac 操作
            byte[] rawHmac = mac.doFinal(data.getBytes(Charset.forName("UTF-8")));
            result = Base64.encodeBase64(rawHmac);

        } catch (NoSuchAlgorithmException e) {
            System.err.println(e.getMessage());
        } catch (InvalidKeyException e) {
            System.err.println(e.getMessage());
        }
        if (null != result) {
            return new String(result);
        } else {
            return null;
        }
    }
}

```
最后访问的地址例如：
http://127.0.0.1:8081/api/es/static/bucket_name/object_name
可以加个拦截器做个登陆验证。


---

> 作者: <no value>  
> URL: https://hpk.me/posts/spring-proxy-oss/  

