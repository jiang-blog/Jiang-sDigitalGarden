---
{"dg-publish":true,"permalink":"/Chaos/Picgo/","title":"Picgo 配置","noteIcon":""}
---


# Picgo 配置

> [!cite] 参考
> [配置手册 | PicGo](https://picgo.github.io/PicGo-Doc/zh/guide/config.html#github%E5%9B%BE%E5%BA%8A)

对于markdown的图床(image-host)已经经过了多次的折腾，今后应该也会继续走在折腾的路上，记录一下已经折腾过的几种上传方法和本地备份配置

## 图床配置

至今为止用过*SMMS*、*Github*、*腾讯云*三个图床，配置方法记录如下，同时也列个表格写下这几个和其他粗略了解过的图床的优缺点。

|  图床  |    优点    |             缺点             |
|:------:|:----------:|:----------------------------:|
|  SMMS  |   免费5G   |             国外             |
| Github |  无限免费  |             国外             |
| 腾讯云 | 便宜，国内 |             要钱             |
| 七牛云 |    国内    |         要钱，要域名         |

> [!note]
> 记得把要用的设置成默认图床

### SMMS

**优点**：有5g免费空间，自己写写博客一开始够用了
**缺点**：国外，会被墙

注册并登录 [smms](https://sm.ms/home/apitoken) 后台获取 token 值

![|800](https://image.jiang849725768.asia/2022/202211202144992.png)

填入Picgo，因为源域名被墙了所以现在可选填备用上传域名`smms.app`

### Github

**优点**：可以通过git进行本地备份同步，因此也可以通过本地修改然后push的方法管理云端图片
**缺点**：国外，会被墙，而且图床所有图片完全公开

专门建一个仓库用于存放图片，得是public的，不知道private仓库能否通过特殊设置成为图床，如果可以就能保证图床的隐秘性
然后生成一个 [token](https://github.com/settings/tokens) 提供给Picgo：

> 引用官方配置方法
>  ![|675](https://raw.githubusercontent.com/Molunerfinn/test/master/picgo/generate_new_token.png)
>  把 repo 的勾打上即可。然后翻到页面最底部，点击 `Generate token` 的绿色按钮生成 token。
>  ![|675](https://raw.githubusercontent.com/Molunerfinn/test/master/picgo/20180508210435.png)
>  ![|675](https://raw.githubusercontent.com/Molunerfinn/test/master/picgo/copy_token.png)

> [!warning] **注意**
> token 生成后只会显示一次！以后要用就复制一下存到其他地方，也可以以后忘了把旧的删了重新生成一个。

Picgo配置中`仓库名`为`{github用户名}/{github仓库名}`，`分支`默认是`master`，可自己指定一个便于管理，`存储路径`可指定仓库内某一文件夹，不存在的话会在上传时新建一个（大概）

至于用jsdelivr进行cdn加速，查阅资料官方应该是禁止为图床加速的，由于我只是自己记记笔记实际速度上也没有太大差别，所以没有使用了，
具体使用应该把`自定义域名`设置为`https://cdn.jsdelivr.net/gh/{github用户名}/{github仓库名}

### 腾讯云

**优点**：国内，比较便宜（大概）
**缺点**：收费，配置略有点繁琐

> [!cite] 参考
> [利用腾讯云COS作为图床为Markdown文档提供图片链接 - 腾讯产业互联网学堂 (tencent.com)](https://cloud.tencent.com/edu/learning/course-1825-24635)

腾讯云是这次折腾之后选用的新图床，看重的特点是配置简单，同时仅用作个人图床的话**似乎**收费不贵（具体需要使用一段时间再看，收费规则有点复杂）

#### 存储桶创建

>创建个人账户后访问腾讯云对象存储COS的控制台，进入 [存储桶列表](https://console.cloud.tencent.com/cos5/bucket) 页面
>在页面中创建一个新的存储桶，作为图片自动上传的图床。创建存储桶的操作流程展示如下：
>![|725](https://course-public-resources-1252758970.cos.ap-chengdu.myqcloud.com/%E5%AE%9E%E6%88%98%E8%AF%BE/%E5%88%A9%E7%94%A8%E8%85%BE%E8%AE%AF%E4%BA%91COS%E4%BD%9C%E4%B8%BA%E5%9B%BE%E5%BA%8A%E4%B8%BAMarkdown%E6%96%87%E6%A1%A3%E6%8F%90%E4%BE%9B%E5%9B%BE%E7%89%87%E9%93%BE%E6%8E%A5/20200103144443-133736.png)
>在存储桶中创建文件夹
>获取配置信息
>![|725](https://course-public-resources-1252758970.cos.ap-chengdu.myqcloud.com/%E5%AE%9E%E6%88%98%E8%AF%BE/%E5%88%A9%E7%94%A8%E8%85%BE%E8%AE%AF%E4%BA%91COS%E4%BD%9C%E4%B8%BA%E5%9B%BE%E5%BA%8A%E4%B8%BAMarkdown%E6%96%87%E6%A1%A3%E6%8F%90%E4%BE%9B%E5%9B%BE%E7%89%87%E9%93%BE%E6%8E%A5/20200103144544-431705.png)

文件夹不创建也行，创建方便管理

#### API密钥创建

官方建议是**用子账号创建API密钥**

1. 前往访问管理CAM的 [用户列表](https://console.cloud.tencent.com/cam) 页面
2. `新建用户`-`快速创建`-设置用户名-更改用户权限-x掉标签-`创建用户`
	- 自己使用的话用户权限简单改为`QCloudResourceFullAccess`就行，访问方式默认为`控制台登录`
	- 是否需要重置密码按自己喜好定
	- 创建好后有个自动生成的密码记得保存
3. 然后从`概览`中进入子账号登录界面用刚刚的账号和自动生成的密码进行登录
	![|925](https://image.jiang849725768.asia/2022/202211211106462.png)
4. 登录后`访问密钥`-`API密钥管理`-`新建密钥`，**记录AppId**用于Picgo配置
	- 如果新建子用户时验证了手机号的话这里可以直接**查看`SecretId` 和`SecretKey`**，否则要回主账号看
	![|850](https://image.jiang849725768.asia/2022/202211211128254.png)

5. 重新用主账号登录可以在`用户列表`-`{子用户名称}`-`API密钥`里**查看`SecretId` 和`SecretKey`**

#### Picgo配置

把上面的信息都填入Picgo即可，**版本选v5**

## 本地备份

由于对云端储存始终没法完全信任，因此对每张上传图首先是采用时间戳命名，然后使用Picgo的**autobackup**插件进行本地备份，便于以后统一迁移替换时只需将本地文件夹上传然后进行链接置换即可。

目前[**autobackup**](https://github.com/Redns/picgo-plugin-autobackup)插件的版本是1.4.12，其本地备份功能的设置似乎有点问题，个人测试时在第一张上传时创建了`mark.json`文件但没成功备份，第二张开始就正常了。

## 其他

Picgo插件安装需要先装npm，然后npm不设置镜像或代理时候产生了`Invalid URL`错误，在Picgo上的体现就是插件界面点`安装`，显示`安装中`后重新变为`安装`（Picgo好像没有安装失败的通知）
