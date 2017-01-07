---
title: Hexo教程(六)-hexo-jacman主题文章页优化 #可以改成中文的，如“新文章”
toc: true   # 在文章侧边增加文章目录
date: 6/28/2016 4:23:43 PM  #发表日期，一般不改动
updated: 2017-01-01 23:47:29 Sunday
categories: hexo #文章文类
tags: [hexo,hexo优化,jacman] #文章标签，多于一项时用这种格式，只有一项时使用tags: blog
---


### 前言：
上一篇对hexo博客基于jacman主题的首页页面的各项优化，本次为针对文章页的页面优化。
jacman主题优化之文章页优化,文章页的头尾及侧边栏和首页的一样，这里就不做说明了。主要说一下文章页中的文章内容部分的优化，顺序从上到下，从左到右进行优化

### 正文

![文章内容页](/images/hexo-5/article-context.png)
![文章内容页](/images/hexo-5/article-context-3.png)
#### 文章访问量、评论数
这里我添加到标题右下方，正文上面的地方。在themes\jacman\layout_partial\post\header.ejs中，找到
```html
<p class="article-time">
<time datetime="<%= date_xml(item.date) %>" itemprop="datePublished"> <%= __('datepublished') %> <%= item.date.format(config.date_format) %></time>
</p>
```
增加如下内容：
```html 
  <p class="article-time"> 
	<% if (theme.duoshuo_shortname && page.comments){ %>
	   <span class="head-plus">
	     阅读次数<i class="fa fa-user"></i><span id="busuanzi_value_page_pv"><i class="fa fa-spinner fa-spin"></i></span>次,
	   </span>
	   <span class="head-plus">
	   <i class="fa fa-comments"></i><span class="ds-thread-count" data-thread-key="<%- page.path %>"><i class="fa fa-spinner fa-spin"></i></span>
	   </span>
     <% } %>
    <time datetime="<%= date_xml(item.date) %>" itemprop="datePublished"> <%= __('datepublished') %> <%= item.date.format(config.date_format) %></time>
  </p>
</header>
```
以上判断了是否开启多说评论，在[前一篇文章](http://www.troylc.cc/hexo/2016/06/27/Hexo-5.html "Hexo教程(五)-hexo博客jacman主题首页优化")中我们已经有说过，多说评论数和网站的访问量，这里涉及到了文章的访问量，都是静态数据，不会对IP进行限制，一刷新就多一次访问量，因为在[前一篇文章](http://www.troylc.cc/hexo/2016/06/27/Hexo-5.html "Hexo教程(五)-hexo博客jacman主题首页优化")中已经加载了记数的JS，所以这里只要放入显示位置就可以了
#### 修改文章页内分享
用jiathis的站内分享：首页进入网站注册：[http://www.jiathis.com](http://www.jiathis.com "jiathis的站内分享")然后配置修改\themes\jacman\_config.yml文件中的jiathis分享属性：
```bash
jiathis:
  enable: true ## if you use jiathis as your share tool,the built-in share tool won't be display.
  id: 2103875   ## e.g. 1889330 your jiathis ID. 
  tsina: 2705524937 ## e.g. 2176287895 Your weibo id,It will be used in share button.
```
默认jacman好像已经集成了这个分享，只是没有开启，如果你的主题里没有集成，看看在\hexo\themes\jacman\layout\_partial\post目录是否有jiathis.ejs这个文件，如果有说明已经集成了，没有的话，可以自己添加集成上，在此目录下创建一个jiathis.ejs文件，内容为在jiathis分享上获取的代码：
```html
<% if (theme.jiathis.enable){ %>
<div class="jiathis_style_24x24">
	<a class="jiathis_button_tsina"></a>
	<a class="jiathis_button_weixin"></a>
	<a class="jiathis_button_renren"></a>
	<a class="jiathis_button_qzone"></a>
	<a class="jiathis_button_googleplus"></a>
	<a class="jiathis_button_douban"></a>
	<a href="http://www.jiathis.com/share" class="jiathis jiathis_txt jtico jtico_jiathis" target="_blank"></a>
	<a class="jiathis_counter_style"></a>
</div>
<script type="text/javascript" >
    var jiathis_config={
    data_track_clickback:true,
    sm:"copy,renren,cqq",
    pic:"<%- item.photos %>",
    summary:"",
    <% if (theme.jiathis.tsina){ %> ralateuid:{"tsina":"<%= theme.jiathis.tsina %>"},hideMore:false}
    <% } %>
  </script> 
<script type="text/javascript" src="//v3.jiathis.com/code/jia.js?uid=
<% if (theme.jiathis.id){ %><%= theme.jiathis.id %><% } %>" charset="utf-8"></script>      
<% } %>
```
在\hexo\themes\jacman\layout\_partial\post目录下找到footer.ejs文件，在此文件添加判断：
```html
<% if (!index){ %>
	<div class="article-share" id="share">
	<% if(theme.jiathis.enable){ %>
	<div class="share-jiathis">
	  <%- partial('jiathis') %>
	 </div>
	<% } else { %>
	  <div data-url="<%- item.permalink %>" data-title="<% if (item.title){ %><%= item.title %> | <% } %><%= config.title %>" data-tsina="<%= theme.author.tsina %>" class="share clearfix">
	  </div>
	<% } %>
	</div>
<% } %>
```
设置完成，可以到文章页内容下面看到分享的按钮。
#### 增加文章页的评论
修改\themes\jacman下_config.yml中的duoshuo_shortname属性
```bash
#### Comment
duoshuo_shortname: troylc   ## e.g. troylc   your duoshuo short name.
disqus_shortname:     ## e.g. wuchong   your disqus short name.
```
关于获取shoutname，shoutname不是登陆的用户昵称，而是多说首页点击我要安装，注册你的多说二级域名。去掉 .duoshuo.com 部分 就是你的shoutname，下图中troylc就是我的shoutname。
![多说安装](/images/hexo-5/article-context-2.png)
![多说安装](/images/hexo-5/article-context-1.png)

### 参考：
[hexo的Jacman主题优化](http://tengj.top/2016/03/06/hexo干货系列：（三）hexo的Jacman主题优化/ "hexo的Jacman主题优化")
[Hexo博客Jacman主题的一些优化](http://www.tuicool.com/articles/FRrQvi3 "Hexo博客Jacman主题的一些优化")


---
