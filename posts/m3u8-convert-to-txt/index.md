# m3u8格式直播地址文件使用php在线转换成txt格式


一些m3u或者m3u8格式的直播地址文件在一些软件中并不支持，在chatgpt的帮助下，使用php实现了在线转换成txt格式，需要能运行php的服务器。

&lt;!--more--&gt;

代码如下，命名为 `m3u-txt.php` ，将其放在能运行php的服务器上，访问  http://xxxxxx.com/m3u-txt.php?url=https://raw.githubusercontent.com/fanmingming/live/main/tv/m3u/ipv6.m3u 可直接转换。

需要注意的是可能一些m3u文件格式不完全一样导致转换不成功，请参考`preg_match_all(&#39;/#EXTINF:-1.*?group-title=&#34;(.*?)&#34;.*?,(.*?)\n(.*?)\n/s&#39;, $file, $matches, PREG_SET_ORDER)`修改。

```php
&lt;?php
$file = file_get_contents($_GET[&#39;url&#39;]);

// 用正则表达式提取group_title和m3u8链接
preg_match_all(&#39;/#EXTINF:-1.*?group-title=&#34;(.*?)&#34;.*?,(.*?)\n(.*?)\n/s&#39;, $file, $matches, PREG_SET_ORDER);

$result = array();

foreach ($matches as $match) {
    $group_title = $match[1];
    $url = $match[3];
    $channel_name = $match[2];
    $result[$group_title][] = $channel_name . &#39;,&#39; . $url;
}

$line_f = &#34;\n&#34;;

// 输出处理后的结果
if (isset($_SERVER[&#34;HTTP_USER_AGENT&#34;]) &amp;&amp; strpos($_SERVER[&#34;HTTP_USER_AGENT&#34;], &#34;Mozilla&#34;) !== false) {
    // 如果是浏览器访问，将 &#34;\n&#34; 替换为 &#34;&lt;br&gt;&#34; 输出
    $line_f = &#34;&lt;br&gt;&#34;; 
}

// 输出结果
foreach ($result as $group_title =&gt; $channels) {
    echo $group_title . &#39;,#genre#&#39; . $line_f;
    foreach ($channels as $channel) {
        echo $channel . $line_f;
    }
    echo $line_f;
}
?&gt;
```

效果如下。

https://raw.githubusercontent.com/fanmingming/live/main/tv/m3u/ipv6.m3u  原格式：

 ![image-20230529113558662](https://i2.wp.com/telegra.ph/file/c9f85fc0ce5ae50deee7e.png)

转换后格式：

![image-20230529113721128](https://i2.wp.com/telegra.ph/file/f1143dd072298c19c2191.png)


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/m3u8-convert-to-txt/  

