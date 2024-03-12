# spring cloud java反向代理访问阿里云OSS私有资源


最近用到阿里云oss，有阿里云服务器，通过代理内网访问可以实现免除OSS流量费，查到很多nginx反向代理的教程，但是纯java实现没有找到，感觉可以试一试。

&lt;!--more--&gt;
（一）
首先要解决反向代理的问题，搜到```org.mitre.dsmiley.httpproxy.ProxyServlet```可解决。

```xml
&lt;dependency&gt;
   &lt;groupId&gt;org.mitre.dsmiley.httpproxy&lt;/groupId&gt;
   &lt;artifactId&gt;smiley-http-proxy-servlet&lt;/artifactId&gt;
   &lt;version&gt;1.11&lt;/version&gt;
&lt;/dependency&gt;
```

```java
@Configuration
public class MyConfig {
    @Bean
    public ServletRegistrationBean servletRegistrationAliBean01() {

        ServletRegistrationBean oss  = new ServletRegistrationBean(new ProxyServlet(), &#34;/ali/bucket_name/*&#34;);//修改为自己的bucket_name
        oss.setName(&#34;bucket_name&#34;);
        oss.addInitParameter(&#34;targetUri&#34;, &#34;http://bucket_name.oss-cn-beijing.aliyuncs.com&#34;);//本地测试环境用外网地址，线上改成内网访问的地址
        oss.addInitParameter(ProxyServlet.P_LOG, &#34;true&#34;);
        return oss;
    }

}
```
（二）
Controller配置如下，转发到自己配置的反向代理
```
@RestController
@RefreshScope
@RequestMapping(&#34;/api/es&#34;)
public class TestController {
     @RequestMapping(&#34;/static/**&#34;)
    public void testVideoOne(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException, URISyntaxException {
        String resourcePath = request.getRequestURI().replace(&#34;/api/es/static&#34;,&#34;&#34;);
        HeaderUtil.setHeaderOwn(request,resourcePath);
        request.getRequestDispatcher(&#34;/ali&#34; &#43; resourcePath).forward(request, response);
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
        SimpleDateFormat sdf = new SimpleDateFormat(&#34;EEE, dd MMM yyyy HH:mm:ss &#39;GMT&#39;&#34;, Locale.US);
        sdf.setTimeZone(TimeZone.getTimeZone(&#34;GMT&#34;));
        String date_GMT = sdf.format(new Date());
        String data = request.getMethod() &#43; &#34;\n&#34;
                &#43; &#34;\n&#34;
                &#43; &#34;\n&#34;
                &#43; date_GMT &#43; &#34;\n&#34;
                &#43; resourcePath;
        String signature = genHMAC(&#34;AccessKeySecret&#34;, data);//AccessKeySecret改成自己的
        reflectSetParam(request, &#34;Authorization&#34;, &#34;OSS AccessKeyId:&#34; &#43; signature);//AccessKeyId改成自己的
        reflectSetParam(request, &#34;Date&#34;, date_GMT);
    }

    /**
     * 反射修改header信息，key-value键值对加入到header中
     *
     * @param request
     * @param key
     * @param value
     */
    private static void reflectSetParam(HttpServletRequest request, String key, String value) {
        Class&lt;? extends HttpServletRequest&gt; requestClass = request.getClass();
        // System.out.println(&#34;request实现类=&#34;&#43;requestClass.getName());
        try {
            Field request1 = requestClass.getDeclaredField(&#34;request&#34;);
            request1.setAccessible(true);
            Object o = request1.get(request);
            Field coyoteRequest = o.getClass().getDeclaredField(&#34;coyoteRequest&#34;);
            coyoteRequest.setAccessible(true);
            Object o1 = coyoteRequest.get(o);
            // System.out.println(&#34;coyoteRequest实现类=&#34;&#43;o1.getClass().getName());
            Field headers = o1.getClass().getDeclaredField(&#34;headers&#34;);
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
            SecretKeySpec signinKey = new SecretKeySpec(key.getBytes(), &#34;HmacSHA1&#34;);
            //生成一个指定 Mac 算法 的 Mac 对象
            Mac mac = Mac.getInstance(&#34;HmacSHA1&#34;);
            //用给定密钥初始化 Mac 对象
            mac.init(signinKey);
            //完成 Mac 操作
            byte[] rawHmac = mac.doFinal(data.getBytes(Charset.forName(&#34;UTF-8&#34;)));
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

> 作者:   
> URL: https://hpk.me/posts/spring-proxy-oss/  

