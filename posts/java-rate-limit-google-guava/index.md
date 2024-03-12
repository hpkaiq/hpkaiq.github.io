# java 频次控制


访问某接口拉取数据，接口需要频次控制，经调研，`com.google.common.util.concurrent.RateLimiter`可轻易实现。

&lt;!--more--&gt;




## 1. Maven
```xml
        &lt;dependency&gt;
            &lt;groupId&gt;com.google.guava&lt;/groupId&gt;
            &lt;artifactId&gt;guava&lt;/artifactId&gt;
            &lt;version&gt;31.1-jre&lt;/version&gt;
        &lt;/dependency&gt;

        &lt;dependency&gt;
            &lt;groupId&gt;cn.hutool&lt;/groupId&gt;
            &lt;artifactId&gt;hutool-all&lt;/artifactId&gt;
            &lt;version&gt;5.8.4&lt;/version&gt;
        &lt;/dependency&gt;
```
## 2. 测试代码
```java
import cn.hutool.http.HttpRequest;
import com.google.common.util.concurrent.RateLimiter;

import java.util.List;
import java.util.Map;
import java.util.concurrent.TimeUnit;

public class RateLimiterTest {
    private final RateLimiter rateLimiter = RateLimiter.create(20,1000, TimeUnit.MILLISECONDS);

    private RateLimiterTest() {
    }

    private static final class InstanceHolder {
        static final RateLimiterTest instance = new RateLimiterTest();
    }

    public static RateLimiterTest getInstance() {
        return InstanceHolder.instance;
    }

    public String runHttpGet(String url, Map&lt;String, List&lt;String&gt;&gt; header) {
        rateLimiter.acquire(1);
        return HttpRequest
                .get(url)
                .header(header)
                .execute()
                .body();
    }
}

```

```java
import java.util.*;
import java.util.concurrent.*;

public class TestRate {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        String url = &#34;https://xxxxx&#34;;
        Map&lt;String, List&lt;String&gt;&gt; header = new HashMap&lt;&gt;();
        header.put(&#34;Content-type&#34;, Collections.singletonList(&#34;application/json&#34;));
        header.put(&#34;Access-Token&#34;, Collections.singletonList(&#34;xxxxx&#34;));

        List&lt;Callable&lt;String&gt;&gt; tasks = new ArrayList&lt;&gt;();

        for (int i = 0; i &lt; 60; i&#43;&#43;) {

            Callable&lt;String&gt; callable = () -&gt; {
                RateLimiterTest instance = RateLimiterTest.getInstance();
                return instance.runHttpGet(url, header);
            };

            tasks.add(callable);

        }

        ExecutorService exec = Executors.newFixedThreadPool(60);
        List&lt;Future&lt;String&gt;&gt; futures = exec.invokeAll(tasks);
        for (Future&lt;String&gt; future : futures) {
            String s = future.get();
            System.out.println(s);
        }
        exec.shutdown();

    }
}

```


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/java-rate-limit-google-guava/  

