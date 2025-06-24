# M3u8格式直播地址文件使用php在线转换成txt格式


一些m3u或者m3u8格式的直播地址文件在一些软件中并不支持，在chatgpt的帮助下，使用php实现了在线转换成txt格式，需要能运行php的服务器。

<!--more-->

代码如下，命名为 `m3u-txt.php` ，将其放在能运行php的服务器上，访问  http://xxxxxx.com/m3u-txt.php?url=https://raw.githubusercontent.com/fanmingming/live/main/tv/m3u/ipv6.m3u 可直接转换。

需要注意的是可能一些m3u文件格式不完全一样导致转换不成功，请参考`preg_match_all('/#EXTINF:-1.*?group-title="(.*?)".*?,(.*?)\n(.*?)\n/s', $file, $matches, PREG_SET_ORDER)`修改。

```php
<?php
$file = file_get_contents($_GET['url']);

// 用正则表达式提取group_title和m3u8链接
preg_match_all('/#EXTINF:-1.*?group-title="(.*?)".*?,(.*?)\n(.*?)\n/s', $file, $matches, PREG_SET_ORDER);

$result = array();

foreach ($matches as $match) {
    $group_title = $match[1];
    $url = $match[3];
    $channel_name = $match[2];
    $result[$group_title][] = $channel_name . ',' . $url;
}

$line_f = "\n";

// 输出处理后的结果
if (isset($_SERVER["HTTP_USER_AGENT"]) && strpos($_SERVER["HTTP_USER_AGENT"], "Mozilla") !== false) {
    // 如果是浏览器访问，将 "\n" 替换为 "<br>" 输出
    $line_f = "<br>"; 
}

// 输出结果
foreach ($result as $group_title => $channels) {
    echo $group_title . ',#genre#' . $line_f;
    foreach ($channels as $channel) {
        echo $channel . $line_f;
    }
    echo $line_f;
}
?>
```

效果如下。

https://raw.githubusercontent.com/fanmingming/live/main/tv/m3u/ipv6.m3u  原格式：

 ![image-20230529113558662](https://i2.wp.com/telegra.ph/file/c9f85fc0ce5ae50deee7e.png)

转换后格式：

![image-20230529113721128](https://i2.wp.com/telegra.ph/file/f1143dd072298c19c2191.png)


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/m3u8-convert-to-txt/  

