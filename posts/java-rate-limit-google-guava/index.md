# java 频次控制


访问某接口拉取数据，接口需要频次控制，经调研，`com.google.common.util.concurrent.RateLimiter`可轻易实现。

<!--more-->




## 1. Maven
```xml
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>31.1-jre</version>
        </dependency>

        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>5.8.4</version>
        </dependency>
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

    public String runHttpGet(String url, Map<String, List<String>> header) {
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
        String url = "https://xxxxx";
        Map<String, List<String>> header = new HashMap<>();
        header.put("Content-type", Collections.singletonList("application/json"));
        header.put("Access-Token", Collections.singletonList("xxxxx"));

        List<Callable<String>> tasks = new ArrayList<>();

        for (int i = 0; i < 60; i++) {

            Callable<String> callable = () -> {
                RateLimiterTest instance = RateLimiterTest.getInstance();
                return instance.runHttpGet(url, header);
            };

            tasks.add(callable);

        }

        ExecutorService exec = Executors.newFixedThreadPool(60);
        List<Future<String>> futures = exec.invokeAll(tasks);
        for (Future<String> future : futures) {
            String s = future.get();
            System.out.println(s);
        }
        exec.shutdown();

    }
}

```


---

> 作者: [hpkaiq](https://hpk.me)  
> URL: https://hpkaiq.github.io/posts/java-rate-limit-google-guava/  

