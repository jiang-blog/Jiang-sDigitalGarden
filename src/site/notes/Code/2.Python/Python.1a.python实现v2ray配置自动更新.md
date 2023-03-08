---
{"dg-publish":true,"permalink":"/code/2-python/python-1a-python-v2ray/","title":"python 实现 v2ray 配置自动更新"}
---


# python 实现 v2ray 配置自动更新

一个简单的**python 下载文件**+**win10 计划任务**的组合

本文使用了 Loyalsoldier 的 [v2ray-rules-dat](https://github.com/Loyalsoldier/v2ray-rules-dat) github 仓库中提供的**geoip.dat**以及**geosite.dat**作为自己 v2rayN 软件的配置，每天中午 12：00 通过仓库中包含的自动更新地址下载更新配置文件

## python 下载配置文件

使用了 python3 的 `request` 库进行文件下载，代码如下

```python
"""
放在v2rayN.exe所在文件夹运行
"""

import requests
import os

# 设置使用系统代理进行下载
proxies = {'http': "socks5://127.0.0.1:10808",
           'https': "socks5://127.0.0.1:10808"}

# geoip.dat和geosite.dat的下载地址
geoip_url = 'https://cdn.jsdelivr.net/gh/Loyalsoldier/v2ray-rules-dat@release/geoip.dat'
geosite_url = 'https://cdn.jsdelivr.net/gh/Loyalsoldier/v2ray-rules-dat@release/geosite.dat'

# requests请求头设定
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.132 Safari/537.36 QIHU 360SE'
}

# 下载更新geoip.dat
try:
    geoip = requests.get(geoip_url, headers=headers, proxies=proxies)
    print("geoip.dat下载完成")
    # 确认更新完后删除原文件创建写入新文件
    path = "./geoip.dat"
    if os.path.exists(path):
        os.remove(path)
    geoip_dat = open("geoip.dat", 'wb')
    geoip_dat.write(geoip.content)
    geoip_dat.close()
    print("geoip.dat更新完成")
except Exception as e:
    print("geoip.dat更新失败\n" + str(e))

# 下载更新geosite.dat
try:
    geosite = requests.get(geosite_url, headers=headers, proxies=proxies)
    # geosite = requests.get(geosite_url)
    print("geosite.dat下载完成")
    # 确认更新完后删除原文件创建写入新文件
    path = "./geosite.dat"
    if os.path.exists(path):
        os.remove(path)
    geosite_dat = open("geosite.dat", 'wb')
    geosite_dat.write(geosite.content)
    geosite_dat.close()
    print("geosite.dat更新完成")
except Exception as e:
    print("geosite.dat更新失败\n"+str(e))
```

## win10 计划任务每天更新

通过 win10 提供的计划任务即可对每天进行更新

通过 `win+R` 打开运行界面，然后输入 `taskschd.msc` 打开计划任务窗口

![](https://cdn.jsdelivr.net/gh/jiang849725768/PrivateImgHost/img/20211106101207.png)

操作 ->创建任务

![image-20211106103451214](https://cdn.jsdelivr.net/gh/jiang849725768/PrivateImgHost/img/202111061034754.png)

触发器 ->新建 ->每天 ->设置时间 ->确定

也可以再添加**一次**几分种后的任务来检验自动执行是否成功。

![image-20211106103742021](https://cdn.jsdelivr.net/gh/jiang849725768/PrivateImgHost/img/202111061037825.png)

操作 ->新建 ->设置程序为执行.py 文件的 python.exe 的绝对地址 ->参数为所执行的.py 文件名 ->参数地址为.py 文件的绝对地址 ->确定

![image-20211106130715934](https://cdn.jsdelivr.net/gh/jiang849725768/PrivateImgHost/img/202111061307363.png)

(另有说法如下，如果尝试上面的有问题可以试试如下设置）

>![img](https://cdn.jsdelivr.net/gh/jiang849725768/PrivateImgHost/img/202111061305701.png)
>
>【程序或脚本】文本框中填的是 Python 编译器的名称，一般就是 python.exe，【起始于】文本框中填的是 Python 编译器的目录，上图中假设你的 Python 编译器的完整路径是“C:\Python27\python.exe”，【添加参数】文本框中填的是你的 Python 程序的完整路径，这里假设在 C 盘的 Users 文件夹下面有一个叫做 code.py 的文件。如果你的 Python 程序包含命令行参数，将其添加到 Python 程序的完整路径之后即可。
>
>[win10 系统设置定时自动执行 python 脚本_HeatDeath的博客-CSDN博客](https://blog.csdn.net/HeatDeath/article/details/79533179?ops_request_misc=&request_id=&biz_id=102&utm_term=win10自动运行python脚本&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-1-79533179.nonecase&spm=1018.2226.3001.4187)

然后就实现每天自动更新了，当然在 v2rayN 里面也需要重启文件才会应用到自动更新后的配置，但对个人来说这个没什么必要所以就没有继续研究了。（其实整个更新都没啥必要，写着试试而已）
