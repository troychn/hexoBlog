---
title: Hexo教程(四)-hexo博客被搜索引擎收录 #可以改成中文的，如“新文章”
date: 2016-06-16 22:09:23 #发表日期，一般不改动
updated: 2016-06-23 13:09:23
categories: hexo #文章文类
tags: [hexo,hexo优化] #文章标签，多于一项时用这种格式，只有一项时使用tags: blog
---
# 前言：
现在我们的博客搭起来了，也写了几篇博文，但是各大搜索引擎还是搜索不到你写的文章，如何查看是否被搜索引擎收录，如何设置博客被搜索引擎收录，如何让搜索引擎能及时的搜索到你的文章，这篇就来和大家一起填坑
# 正文
下面准备分别介绍谷歌和百度如何提交搜索引擎，其中有一些共同的地方，这里先说明：
## 确认博客是否被收录
在百度或者谷歌上面输入下面格式来判断，如果能搜索到就说明被收录，否则就没有，用你的域名替代我的
输入：site:troylc.cc
![查看google是否收录](/images/hexo-3/google-troylc.png)
![查看baidu是否收录](/images/hexo-3/baidu-troylc.png)
我这里是已经做完了，在写这篇文章的，所以显示已经收录了
## 验证博客网站
两个搜索引擎入口：
[Google搜索引擎提交入口](https://www.google.com/webmasters/tools/home?hl=zh-CN "Google搜索引擎提交入口")
[百度搜索引擎入口](http://zhanzhang.baidu.com/linksubmit/url "百度搜索引擎入口")
如果没有对应的账号，请对应的注册相应的账号：
![添加百度验证博客网站](/images/hexo-3/baidu-create.png)
![添加google验证博客网站](/images/hexo-3/google-create1.png)
![添加google验证博客网站](/images/hexo-3/google-create2.png)
不管谷歌还是百度都要先添加域名，然后验证网站，这里统一都使用文件验证，就是下载对应的html文件，放到域名根目录下，也就收博客根目录下的source下面
![查看本地hexo验证文件](/images/hexo-3/hexo-verify-file.png)
然后部署到服务器,输入地址：http://www.troylc.cc/googlead0e22632f59a368.html 能访问到就可以点验证按钮。
`注`：这个验证的时候，google有可能会失败，可以等个一两天，再去验证，就好了。

## 生成站点地图sitemap
站点地图是一种文件，您可以通过该文件列出您网站上的网页，从而将您网站内容的组织架构告知Google和其他搜索引擎。Googlebot等搜索引擎网页抓取工具会读取此文件，以便更加智能地抓取您的网站。
先安装一下插件，打开你的hexo博客根目录，分别用下面两个命令来安装针对谷歌和百度的插件
```bash
npm install hexo-generator-sitemap --save
npm install hexo-generator-baidu-sitemap --save
```
编译你的博客
```bash
hexo d -g
```
如果在你的博客根目录的public下面生成了sitemap.xml以及baidusitemap.xml就表示成功了。
这时候sitemap.xml跟baidusitemap.xml里面的内容有点相似.
部署后你分别访问
[http://www.troylc.cc/sitemap.xml](http://www.troylc.cc/sitemap.xml)
[http://www.troylc.cc/baidusitemap.xml](http://www.troylc.cc/baidusitemap.xml)

## 收录hexo博客
### 让谷歌收录我们的博客
谷歌操作比较简单，就是向Google站长工具提交sitemap
登录Google账号，添加了站点验证通过后，选择站点，之后在抓取——站点地图中就能看到添加/测试站点地图，如下图：
![谷歌收录我们的博客](/images/hexo-3/google-sitemap.png)
谷歌提交过了一两天就能搜索到博客了。
### 让百度收录我们的博客
相比google百度操作可能比较麻烦一点：这里是百度介绍的几种主动提交博客文章链接的方式：
如何选择链接提交方式
1. 主动推送：最为快速的提交方式，推荐您将站点当天新产出链接立即通过此方式推送给百度，以保证新链接可以及时被百度收录。
2. 自动推送：最为便捷的提交方式，请将自动推送的JS代码部署在站点的每一个页面源代码中，部署代码的页面在每次被浏览时，链接会被自动推送给百度。可以与主动推送配合使用。
3. sitemap：您可以定期将网站链接放到sitemap中，然后将sitemap提交给百度。百度会周期性的抓取检查您提交的sitemap，对其中的链接进行处理，但收录速度慢于主动推送。
4. 手动提交：一次性提交链接给百度，可以使用此种方式。
从效率上来说：主动推送>自动推送>sitemap
这里介绍自动推送和sitemap提交方式：  

**自动推送：**
![百度自动推送](/images/hexo-3/baidu-zd1.png)
![百度自动推送](/images/hexo-3/baidu-zd2.png)
自动推送是百度站长平台为提高站点新增网页发现速度推出的工具，安装自动推送JS代码的网页，在页面被访问时，页面URL将立即被推送给百度
```javascript
<script>
(function(){
    var bp = document.createElement('script');
    var curProtocol = window.location.protocol.split(':')[0];
    if (curProtocol === 'https') {
        bp.src = 'https://zz.bdstatic.com/linksubmit/push.js';        
    }
    else {
        bp.src = 'http://push.zhanzhang.baidu.com/push.js';
    }
    var s = document.getElementsByTagName("script")[0];
    s.parentNode.insertBefore(bp, s);
})();
</script>
```
我是放在\themes\jacman\layout\_partial\after_footer.ejs中，添加到下面就行。

**sitemap提交:**
直接提交http://www.troylc.cc/baidusitemap.xml 就行，看下图。
![百度自动推送](/images/hexo-3/baidu-sitemaptj.png)

至此我们的博客已经被百度与google这样的搜索引擎收录，如果出现无法收录，可以过几天再登录看看结果

# 参考：
[hexo提交搜索引擎（百度+谷歌）](http://tengj.top/2016/03/14/hexo6seo/)



---
