# Cloudflare优选ip并使用dnspodcn Api设置解析


Cloudflare优选ip并使用dnspodcn api设置解析，实现在本地网络环境下nas或者其他服务器上优选IP，并自动解析到自己的dnspodcn域名，实现速度最优。

&lt;!--more--&gt;

## 优选ip

请参考github项目 [XIU2/CloudflareSpeedTest: 🌩「自选优选 IP」测试 Cloudflare CDN 延迟和速度，获取最快 IP (IPv4 / IPv6)！另外也支持其他 CDN / 网站 IP ~ (github.com)](https://github.com/XIU2/CloudflareSpeedTest)

我的本地服务器为centos8系统，下载并解压到 /opt/cloudflareST 目录。

## dnspodcn api 设置解析

参考[rehiy/dnspod-shell: 基于DNSPod用户API实现的纯Shell动态域名客户端 (github.com)](https://github.com/rehiy/dnspod-shell) ，加入线路逻辑修改而来。

进入 /opt/cloudflareST 目录，新建以下脚本。需要修改：

- 修改arToken=&#34;12345,xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx&#34; 为你自己的dnspodcn的token
- 修改arDdnsCheck &#34;xx.yy&#34; &#34;sub1&#34;  为你自己的域名。
- `hostIp=$(awk -F &#39;,&#39; &#39;NR==2 {print $1}&#39; /opt/cloudflareST/result.csv)` 注意此处优选ip结果文件位置，结合你的运行目录修改。

```bash
#!/bin/sh
#

###################################################################
# AnripDdns v6.4.0
#
# Dynamic DNS using DNSPod API
#
# Author: Rehiy, https://github.com/rehiy
#                https://www.rehiy.com/?s=dnspod
#
# Collaborators: https://github.com/rehiy/dnspod-shell/graphs/contributors
#
# Usage: please refer to `ddnspod.sh`
#
################################################################### params ##

export arToken

# The url to be used for querying public ip address.

export arIp4QueryUrl=&#34;http://ipv4.rehi.org/ip&#34;
export arIp6QueryUrl=&#34;http://ipv6.rehi.org/ip&#34;

# The temp file to store the last record ip

# The error code to return when a ddns record is not changed
# By default, report unchanged event as success

export arErrCodeUnchanged=0

################################################################### logger ##

# Output log to stderr

arLog() {

    &gt;&amp;2 echo &#34;$@&#34;

}

################################################################### http client ##

# Use curl or wget open url
# Args: url postdata

arRequest() {

    local url=&#34;$1&#34;
    local data=&#34;$2&#34;

    local params=&#34;&#34;
    local agent=&#34;AnripDdns/6.4.0(wang@rehiy.com)&#34;

    if type curl &gt;/dev/null 2&gt;&amp;1; then
        if echo $url | grep -q https; then
            params=&#34;$params -k&#34;
        fi
        if [ -n &#34;$data&#34; ]; then
            params=&#34;$params -d $data&#34;
        fi
        curl -s -A &#34;$agent&#34; $params $url
        return $?
    fi

    if type wget &gt;/dev/null 2&gt;&amp;1; then
        if echo $url | grep -q https; then
            params=&#34;$params --no-check-certificate&#34;
        fi
        if [ -n &#34;$data&#34; ]; then
            params=&#34;$params --post-data $data&#34;
        fi
        wget -qO- -U &#34;$agent&#34; $params $url
        return $?
    fi

    return 1

}



# Get IPv4 by ip route or network

arWanIp4() {

    local hostIp
    hostIp=$(awk -F &#39;,&#39; &#39;NR==2 {print $1}&#39; /opt/cloudflareST/result.csv)
    echo $hostIp

}


################################################################### dnspod api ##

# Dnspod Bridge
# Args: interface data

arDdnsApi() {

    local dnsapi=&#34;https://dnsapi.cn/${1:?&#39;Info.Version&#39;}&#34;
    local params=&#34;login_token=$arToken&amp;format=json&amp;lang=en&amp;$2&#34;

    arRequest &#34;$dnsapi&#34; &#34;$params&#34;

}

# Fetch Record Id
# Args: domain subdomain recordType

arDdnsLookup() {

    local errMsg
    # Get Record Id
    recordId=$(arDdnsApi &#34;Record.List&#34; &#34;domain=$1&amp;sub_domain=$2&amp;record_type=$3&amp;record_line=$4&#34;)
    recordId=$(echo $recordId | sed &#39;s/.*&#34;id&#34;:&#34;\([0-9]*\)&#34;.*/\1/&#39;)

    if ! [ &#34;$recordId&#34; -gt 0 ] 2&gt;/dev/null ;then
        errMsg=$(echo $recordId | sed &#39;s/.*&#34;message&#34;:&#34;\([^\&#34;]*\)&#34;.*/\1/&#39;)
        arLog &#34;&gt; arDdnsLookup - $errMsg&#34;
        return 1
    fi

    echo $recordId
}

# Update Record Value
# Args: domain subdomain recordId recordType [hostIp]

arDdnsUpdate() {

    local errMsg

    local recordRs
    local recordCd
    local recordIp

    local lastRecordIp

    yys_line=$6

    arLastRecordFile=/run/ardnspod_last_record_&#34;$2&#34;.&#34;$1&#34;.&#34;$yys_line&#34;

    if [ -f $arLastRecordFile ]; then
        lastRecordIp=$(cat $arLastRecordFile)
    fi

    # fetch the last record ip from api
    if [ -z &#34;$lastRecordIp&#34; ]; then
        recordRs=$(arDdnsApi &#34;Record.Info&#34; &#34;domain=$1&amp;record_id=$3&#34;)
        recordCd=$(echo $recordRs | sed &#39;s/.*{&#34;code&#34;:&#34;\([0-9]*\)&#34;.*/\1/&#39;)
        lastRecordIp=$(echo $recordRs | sed &#39;s/.*,&#34;value&#34;:&#34;\([0-9a-fA-F\.\:]*\)&#34;.*/\1/&#39;)
    fi

    if [ &#34;$5&#34; = &#34;$lastRecordIp&#34; ]; then
        arLog &#34;&gt; arDdnsUpdate - unchanged: $recordIp&#34; # unchanged event
        return $arErrCodeUnchanged
    fi
    recordRs=$(arDdnsApi &#34;Record.Ddns&#34; &#34;domain=$1&amp;sub_domain=$2&amp;record_id=$3&amp;record_type=$4&amp;value=$5&amp;record_line=$yys_line&#34;)


    # parse result
    recordCd=$(echo $recordRs | sed &#39;s/.*{&#34;code&#34;:&#34;\([0-9]*\)&#34;.*/\1/&#39;)
    recordIp=$(echo $recordRs | sed &#39;s/.*,&#34;value&#34;:&#34;\([0-9a-fA-F\.\:]*\)&#34;.*/\1/&#39;)

    if [ &#34;$recordCd&#34; != &#34;1&#34; ]; then
        errMsg=$(echo $recordRs | sed &#39;s/.*,&#34;message&#34;:&#34;\([^&#34;]*\)&#34;.*/\1/&#39;)
        arLog &#34;&gt; arDdnsUpdate - error: $errMsg&#34;
        return 1
    elif [ &#34;$recordIp&#34; = &#34;$lastRecordIp&#34; ]; then
        arLog &#34;&gt; arDdnsUpdate - unchanged: $recordIp&#34; # unchanged event
        echo $recordIp
        return $arErrCodeUnchanged
    else
        arLog &#34;&gt; arDdnsUpdate - updated: $recordIp&#34; # updated event
        echo $recordIp &gt; $arLastRecordFile
        echo $recordIp
        return 0
    fi

}

################################################################### task hub ##

# DDNS Check
# Args: domain subdomain [6|4] interface

arDdnsCheck() {

    local errCode

    local hostIp

    local recordType

    arLog &#34;=== Check $2.$1 $(printf $(echo -n &#34;$3&#34; | sed &#39;s/\\/\\\\/g;s/\(%\)\([0-9a-fA-F][0-9a-fA-F]\)/\\x\2/g&#39;)) line===&#34;
    arLog &#34;Fetching Host Ip&#34;


    recordType=A
    hostIp=$(arWanIp4)


    errCode=$?
    if [ $errCode -eq 0 ]; then
        arLog &#34;&gt; Host Ip: $hostIp&#34;
        arLog &#34;&gt; Record Type: $recordType&#34;
    elif [ $errCode -eq 2 ]; then
        arLog &#34;&gt; Host Ip: Auto&#34;
        arLog &#34;&gt; Record Type: $recordType&#34;
    else
        arLog &#34;$hostIp&#34;
        return $errCode
    fi

    arLog &#34;Fetching RecordId&#34;
    recordId=$(arDdnsLookup &#34;$1&#34; &#34;$2&#34; &#34;$recordType&#34; &#34;$3&#34;)

    errCode=$?
    if [ $errCode -eq 0 ]; then
        arLog &#34;&gt; Record Id: $recordId&#34;
    else
        arLog &#34;$recordId&#34;
        return $errCode
    fi

    arLog &#34;Updating Record value&#34;
    arDdnsUpdate &#34;$1&#34; &#34;$2&#34; &#34;$recordId&#34; &#34;$recordType&#34; &#34;$hostIp&#34; &#34;$3&#34;

}

################################################################### end ##

cd /opt/cloudflareST
# https://github.com/XIU2/CloudflareSpeedTest
# 即需要找到 10 个平均延迟低于 300 ms 且下载速度高于 15MB/s 的 IP 才会停止测速
/opt/cloudflareST/CloudflareST -tl 300 -sl 15 -dn 10 -url https://cf.xiu2.xyz/url -o /opt/cloudflareST/result.csv     

# Combine your token ID and token together as follows
arToken=&#34;12345,xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx&#34;

# Web endpoint to be used for querying the public IPv6 address
# Set this to override the default url provided by ardnspod
# arIp4QueryUrl=&#34;http://ipv4.rehi.org/ip&#34;
# arIp6QueryUrl=&#34;http://ipv6.rehi.org/ip&#34;

# Return code when the last record IP is same as current host IP
# Set this to a value other than 0 to distinguish with a successful ddns update
# arErrCodeUnchanged=0

# Place each domain you want to check as follows
# you can have multiple arDdnsCheck blocks

## 联通 %E8%81%94%E9%80%9A
## 移动 %E7%A7%BB%E5%8A%A8
## 电信 %E7%94%B5%E4%BF%A1
yys_lines=(&#34;%E8%81%94%E9%80%9A&#34; &#34;%E7%A7%BB%E5%8A%A8&#34; &#34;%E7%94%B5%E4%BF%A1&#34;)
# 使用循环遍历数组
for yys_line in &#34;${yys_lines[@]}&#34;
do
    arDdnsCheck &#34;xx.yy&#34; &#34;sub1&#34; &#34;$yys_line&#34;
done

```

## 定时运行

我使用crontab，8小时运行一次，请根据需要自行修改。

```bash
0 */8 * * * /opt/cloudflareST/dnspodcn.sh &gt;&gt; /opt/cloudflareST/log.txt
```



---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/cloudflare-betterip-dnspodcn/  

