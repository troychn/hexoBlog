---
title: Hexo教程(五)-hexo-jacman主题首页优化 #可以改成中文的，如“新文章”
date: 2016-06-27 22:09:23 #发表日期，一般不改动
updated: 2016-06-28 13:09:23
categories: hexo #文章文类
tags: [hexo,hexo优化,jacman] #文章标签，多于一项时用这种格式，只有一项时使用tags: blog
---
# 前言：
上一篇说到了hexo的博客被百度和google收录的方法，虽然搜索引擎是收录了，但是我在百度上搜索我文章的标题，还是没有搜索结果，结果只有首页一个。这个不知道是什么原因，有可能是百度索引量的问题，后续再观察，再折腾吧。下面来分享一下优化jacman主题，包括首页和文章页等，
# 正文
jacman主题优化之首页优化,顺序从上到下，从左到右进行优化
## 首页头部优化
![头部header](/images/hexo-5/header.png)

1.修改网站文字描述，修改hexo(F:\Blog\hexo\_config.yml)主配置文件_config.yml中的
```bash
# Site 网站
title: troylc博客                   #网站标题
subtitle: 爱生活爱编程，一起来进步吧！  #网站副标题
description:  hello,every body!         #网站描述
author: troylc                          #您的名字 
language: zh-CN                         #网站使用的语言
timezone:                               #网站时区。Hexo 默认使用您电脑的时区
```
2.修改文章URL结构
默认文章链结是以: http://xxx.com/2016/06/06/your-title/ 的格式，末尾没有.html结尾有点动态页面的感觉，对搜索引擎能否收录也是个问题，于是，我改成了 http://xxx.com/hexo/2016/03/18/hello-world.html 这样的格式，具体方法是在 根目录(F:\Blog\hexo\_config.yml)下的_config.yml文件里:
```bash
#permalink: :year/:month/:day/:title/
permalink: :category/:year/:month/:day/:title.html
```

3.开启URL目录映射
如果你的分类是中文的，在url中也会显示相应的中文，为了在URL尽量少出现中文，做以下修改，方法是在_config.yml 下:
```
# Category & Tag  分类和标签的设置
default_category: uncategorized   #默认分类         
category_map:                     #分类别名 category_map 是为了让url中尽量少出现中文，做的映射。如下配置
    心得: experience
#    生活: life
#    其他: other
tag_map:                          #标签别名
```
其中, category_map，tag_map 是为了让url中尽量少出现中文，做的映射。
例如:
在文章开头，标柱目录为:
```
 ---
xxx: xxx
categories: 心得
 ---
```
则在url中， 会变成: http://xxx.com/experience/year/month/day/xxx.html

4.修改网站小图标、博客logo,在jacman(F:\Blog\hexo\themes\jacman\)主题目录下的配置文件_config.yml,找到以下配置,指定favicon，imglogo的图片位置，或者把原本对应位置下的图片名称不变，直接换成自己喜欢的图片
```bash
#### Image
imglogo:
  enable: true             ## display image logo true/false. 是否显示logo
  src: img/logo.png        ## `.svg` and `.png` are recommended,please put image into the theme folder `/jacman/source/img`.
favicon: img/favicon.ico   ## size:32px*32px,`.ico` is recommended,please put image into the theme folder `/jacman/source/img`.     
apple_icon: img/jacman.jpg ## size:114px*114px,please put image into the theme folder `/jacman/source/img`.
author_img: img/author.jpg ## size:220px*220px.display author avatar picture.if don't want to display,please don't set this.
banner_img: #img/banner.jpg ## size:1920px*200px+. Banner Picture
### Theme Color 
theme_color:          ##主题颜色
    theme: '#2ca6cb'    ##the defaut theme color is blue
```

5.首页头部的菜单menu 默认没有启用 /tags 和 /categories 页面，如果需要启用请在博客目录下的(F:\Blog\hexo\source\)source文件夹中分别建立tags和categories文件夹每个文件夹中分别包含一个index.md文件。内容为：
```bash
 ---
layout: tags (或categories)
title: tags (或categories)
 ---
```
在jacman(F:\Blog\hexo\themes\jacman\_config.yml)主题目录下的配置如下：
```bash
##### Menu
menu:
  home: /
  categories: /categories
  tags: /tags
  archives: /archives
##  life: /life
##  about: /about
## you can create `tags` and `categories` folders in `../source`.
## And create a `index.md` file in each of them.
## set `front-matter`as
## layout: tags (or categories)
## title: tags (or categories)
## ---
```

6.增加百度站内搜索-[极速体验版本](http://zn.baidu.com/cse/wiki/index?category_id=25 "百度站内搜索-极速体验版本")
在百度站内搜索-帮助中心-极速体验版中填空站内搜索代码，获取搜索展示代码，放在(F:\Blog\hexo\themes\jacman\layout\_partial\header.ejs):  
```jsp
<% } else if(theme.baidu_search.enable){ %>
   <form class="search" action="<%- theme.baidu_search.site %>" target="_blank">
    <label>Search</label>
    <input name="cc" type="hidden" value= <%= theme.baidu_search.id %> ><input type="text" name="q" size="30" placeholder="<%= __('search') %>"><br>
  </form>
<% } else if(theme.swift_search.enable){ %>
```
再修改F:\Blog\hexo\themes\jacman\_config.yml文件中的百度搜索属性：
```bash
baidu_search:     ## http://zn.baidu.com/  极速体验版本http://zn.baidu.com/cse/wiki/index?category_id=25
  enable: true
  id: troylc.cc  ## e.g. "troylc.cc"  for your baidu search webname
  site: http://zhannei.baidu.com/cse/site  ## your can change to your site instead of the default site
```
百度搜索极速检验版，需要百度搜索引擎收录你的网站，具体参考[《Hexo教程(四)-hexo博客被搜索引擎收录》](http://www.troylc.cc/hexo/2016/06/16/Hexo-4.html "搜索引擎收录网站")

## 首页内容部分优化
![内容部分优化](/images/hexo-5/index-context.png)
### 首页文章列表的优化
1.文章列表的展示方式，默认是全部展开，感觉展示文章全部内容比较没有吸引力，我关闭掉了，只展示少量摘要。修改\themes\jacman下面_config.yml中的expand改成false即可
```bash
index:
  expand: false    ## default is unexpanding,so you can only see the short description of each post.
  excerpt_link: Read More
```
2.文章列表的展示形式，修改成列表模式，并在每一格列表的头显示标题，中间显示文章的摘要，尾显示发布时间，分类，标签，及评论数。修改F:\java\githubPages\Blog\hexo\themes\jacman\layout\_partial\article.ejs文件中的以下部分：
```html
<section class="post" itemscope itemprop="blogitem">
  <% if (item.link) { %> 
    <a href="<%- item.link %>" target="_blank"> 
  <% } else{ %>
    <a href="<%- config.root %><%- item.path %>" title="<%= item.title %>" itemprop="url">
  <% } %>
    <h1 itemprop="name"><%= item.title %></h1>
    <% if (desc){ %>
     <% if(item.description){ %>
      <p itemprop="description" ><%- item.description %></p>
      <% } else if(item.excerpt){ %>
       <p itemprop="description" ><%= strip_html(item.excerpt).replace(/^\s*/, '').replace(/\s*$/, '').substring(0, 140) %></p>
      <% } else {%>
           <p itemprop="description" ><%= strip_html(item.content).replace(/^\s*/, '').replace(/\s*$/, '').substring(0, 140) %></p>
        <% } %>
    <% } %>
    <time datetime="<%= date_xml(item.date) %>" itemprop="datePublished"><%= item.date.format(config.date_format) %></time>    
  </a>            
</section>
```
修改为：
```html
<article class="post-expand <%= item.layout %>"" itemprop="articleBody">
<%- partial('post/header-index') %>
    <div style="padding: 0.5em 0.5em 0.5em 0.8em">
    <% if (item.link) { %> 
    <a href="<%- item.link %>" target="_blank"> 
      <% } else{ %>
        <a href="<%- config.root %><%- item.path %>" title="<%= item.title %>" itemprop="url">
      <% } %>
        <% if (desc){ %>
         <% if(item.description){ %>
          <p itemprop="description" ><%- item.description %></p>
          <% } else if(item.excerpt){ %>
           <p itemprop="description" ><%= strip_html(item.excerpt).replace(/^\s*/, '').replace(/\s*$/, '').substring(0, 240) %></p>
          <% } else {%>
               <p itemprop="description" ><%= strip_html(item.content).replace(/^\s*/, '').replace(/\s*$/, '').substring(0, 240) %></p>
            <% } %>
        <% } %>
        </a>
   </div>    
<%- partial('post/footer-index', {index: true}) %>
</article>
```
在当前目录下的post目录里增加三个文件catetags-index.ejs,footer-index.ejs,header-index.ejs
catetags-index.ejs 内容:
```html
<div class="article-catetags">
<% if (item.categories && item.categories.length){ %>
<div style="float: left;padding: 0.5em 0;margin-right: 3em;margin-top: 0.3em;">
    <time datetime="<%= date_xml(item.date) %>" itemprop="datePublished"> <%= __('datepublished') %> <%= item.date.format(config.date_format) %></time>
    By<% if(theme.author.google_plus){ %>
        <a href="https://plus.google.com/<%=theme.author.google_plus %>?rel=author" title="<%= config.author %>" target="_blank" itemprop="author"><%= config.author %></a>
        <% }else{ %>
        <a href="<%= config.root %>about" title="<%= config.author %>" target="_blank" itemprop="author"><%= config.author %></a>
        <% } %>
</div>
<div class="article-categories">
  <span></span>
  <%- list_categories(item.categories, {
      show_count: false,
      class: 'article-category',
      style: 'none',
      separator: '►'
  }) %>
</div>
<% } %>
<% if (item.tags && item.tags.length){ %>
  <div class="article-tags">
  <% var tags = [];
    item.tags.forEach(function(tag){
      tags.push('<a href="' + config.root + tag.path + '">' + tag.name + '</a>');
    }); %>
  <span></span> <%- tags.join('') %>
  </div>
<% } %>
</div>
```
footer-index.ejs 内容:
```html
<footer class="article-footer clearfix">
<%- partial('catetags-index') %>
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
<% if (index){ %>
<div class="comments-count">
    <% if((config.disqus_shortname || theme.disqus_shortname) && !theme.duoshuo_shortname) { %>
          <span></span>
        <a href="<%- config.root %><%- item.path %>#disqus_thread" class="comments-count-link">Comments</a>
    <% } else if(theme.duoshuo_shortname) { %>
          <span></span>
        <a href="<%- config.root %><%- item.path %>#comments" class="ds-thread-count comments-count-link" data-thread-key="<%- item.path %>" data-count-type="comments">&nbsp;</a>
    <% } %>
</div>
<% } %>
</footer>
```
header-index.ejs 内容:
```html
<header class="article-info clearfix">
  <h1 itemprop="name">
    <% if(item.link) { %>
      <a href="<%- item.link %>" target="_blank" title="<%= item.title %>"><%= item.title %></a><% } else { %>
      <a href="<%- config.root %><%- item.path %>" title="<%= item.title %>" itemprop="url"><%= item.title %></a><% } %>
  </h1>
</header>
```
调整对应的样式：
具体:
    1.\jacman\source\css\_partial\index.styl
    2.\jacman\source\css\_partial\footer.styl
    3.\jacman\source\css\_partial\article.styl
    4.\jacman\source\css\_partial\helper.styl
我主要调整了以上四个文件中的样式，具体各位自己找着改吧。太多了，我也是通过firefox的firebug工具慢慢找得。


3.文章列表分页条数限制（含首页文章列表页，分类列表页，标签列表页）修改hexo(F:\Blog\hexo\_config.yml)主配置文件_config.yml中的：
```bash
# Others
index_generator:
  per_page: 5 ##首页默认10篇文章标题 如果值为0不分页
archive_generator:
    per_page: 0 ##归档页面默认10篇文章标题
    yearly: true  ##生成年视图
    monthly: true ##生成月视图
tag_generator:
    per_page: 0 ##标签分类页面默认10篇文章
category_generator: 
    per_page: 0 ###分类页面默认10篇文章
feed:
    type: atom ##feed类型 atom或者rss2
    path: atom.xml ##feed路径
    limit: 20  ##feed文章最小数量  
```


### 首页内容侧边栏优化

侧边栏顺序配置：
```bash
#### Widgets  首页侧边栏
widgets: 
- github-card
- category
- tag
- hot
- tagcloud
- links
#- douban
- weibo
- rss
  ## provide eight widgets:github-card,category,tag,rss,archive,tagcloud,links,weibo
```
1.github名片，友情链接,RSS订阅，在[Hexo教程(一)-hexo环境搭建](http://www.troylc.cc/hexo/2016/05/31/Hexo-1.html#%E5%AE%89%E8%A3%85Hexo%E6%8F%92%E4%BB%B6)中的安装插件时，安装RSS插件. 在jacman(F:\Blog\hexo\themes\jacman\)主题目录下的配置文件_config.yml,找到以下配置,并修改如下：
```bash
author:
  intro_line1:  "Hello ,I'm troylc Page in github."    ## your introduction on the bottom of the page
  intro_line2:  "This is my blog,believe it or not."  ## the 2nd line
  ...
  github: xxxx     ## e.g. 配置你对应的github用户名
  ...
##----------------------------------------------------------------------------
#### RSS  RSS订阅 
rss: /atom.xml ## RSS address.
##----------------------------------------------------------------------------
#### Links
links:
  攻城狮: http://www.troylc.cc/, 一个面向程序员交流分享的博客
  troylc's Blog: http://www.troylc.cc/```

2.分类，标签，标签云，这几个只有在创建文章中写上分类和标签，就会在此处自动显示分类，标签，标签云，以分类和标签的文章个数。

3.热评文章、我的微博
**热评文章：**
由于hexo没有内置诸如“热评文章”，“最新评论”等的widget，那么只能自定义widget,仿照其他的widget，在themes的_config.yml文件中的widgets下添加自定义widget的名称，如上面我添加了一个hot，然后在F:\Blog\hexo\themes\jacman\layout\_widget目录下新建一个hot.ejs，在[多说](http://duoshuo.com/ "多说网站")(如果没有多说账号就注册一个，后续文章页的文章评论也会用到)->后台管理->工具->热评文章中获取代码，写到hot.ejs文件中，然后在其上面写上名字“热评文章”，如下：
```html
<div>
<!-- 国际化文件(F:\Blog\hexo\themes\jacman\languages\zh-CN.yml)定义hot -->
<p class="asidetitle"><%= __('hot') %></p>
<!-- 多说热评文章 start -->
<div class="ds-top-threads" data-range="daily" data-num-items="5"></div>
<!-- 多说热评文章 end -->
<!-- 多说公共JS代码 start (一个网页只需插入一次) -->
<script type="text/javascript">
var duoshuoQuery = {short_name:"troylc"};
	(function() {
		var ds = document.createElement('script');
		ds.type = 'text/javascript';ds.async = true;
		ds.src = (document.location.protocol == 'https:' ? 'https:' : 'http:') + '//static.duoshuo.com/embed.js';
		ds.charset = 'UTF-8';
		(document.getElementsByTagName('head')[0] 
		 || document.getElementsByTagName('body')[0]).appendChild(ds);
	})();
</script>
<!-- 多说公共JS代码 end -->
</div>
```
**我的微博-微博秀**
需要注意的是，如果要启用微博秀，您必须填上author属性下tsina和weibo_verifier的值，前者是您微博ID，后者是您微博秀的验证码，登录新浪微博后，访问 [http://app.weibo.com/tool/weiboshow](http://app.weibo.com/tool/weiboshow "微博秀") 在如下图位置，可以获得您的 verifier，如：我的是0f3h06c8。
![内容侧边栏-微博秀](/images/hexo-5/index-context-2.png)
在jacman(F:\Blog\hexo\themes\jacman\)主题目录下的配置文件_config.yml,找到以下配置,并修改如下：
```bash
author:
  ...
  weibo_verifier: 0f3h06c8    ## e.g. 0f81d6c8 Your weibo-show widget verifier ,if you use weibo-show it is needed.
  tsina: 2705522637      ## e.g. 2705522637  Your weibo ID,It will be used in share button.
  weibo: 2705522637     ## e.g. troylc or 2705522637 for http://weibo.com/2705522637
  ...
```
第一次不知道是什么原因，要么无法显示，要么显示出现问题，耐心的多刷新，或者等待几天再去刷新。

## 首页footer优化
![尾部footer](/images/hexo-5/footer.png)
1.修改网站的主页的头和尾的英文字体：\hexo\themes\jacman\source\css\_base\variable.styl中的font-custom-family = "covered_by_your_graceregular"修改为inherit如下:
```html
//Font
font-default = "Helvetica Neue", "Helvetica","Microsoft YaHei", "WenQuanYi Micro Hei",Arial, sans-serif
font-serif = "Georgia", serif
font-mono = Monaco, Menlo, Consolas, Courier New, monospace
font-custom-family = inherit  //英文字体inherit
font-custom-filename = coveredbyyourgrace-webfont
font-icon-family = "FontAwesome"
font-icon-filename = fontawesome-webfont
font-icon-version = "4.0.3"
font-icon-diao = "fontdiao"
font-icon-diao-filename = "fontdiao"
font-icon-diao-version = "0.0.8"
ShowCustomFont = hexo-config("ShowCustomFont")
font-size = 100%
line-height = 1.5
//image
author-img = hexo-config("author_img")
```
2.尾部的作者图片，在上面头部优化中，有提到，在jacman(F:\Blog\hexo\themes\jacman\)主题目录下的配置文件_config.yml,找到以下配置,指定author_img的图片位置，或者把原本对应位置下的图片名称不变，直接换成自己喜欢的图片
```bash
#### Image
imglogo:
  enable: true             ## display image logo true/false. 是否显示logo
  src: img/logo.png        ## `.svg` and `.png` are recommended,please put image into the theme folder `/jacman/source/img`.
favicon: img/favicon.ico   ## size:32px*32px,`.ico` is recommended,please put image into the theme folder `/jacman/source/img`.     
apple_icon: img/jacman.jpg ## size:114px*114px,please put image into the theme folder `/jacman/source/img`.
author_img: img/author.jpg ## size:220px*220px.display author avatar picture.if don't want to display,please don't set this.
banner_img: #img/banner.jpg ## size:1920px*200px+. Banner Picture
### Theme Color 
theme_color:          ##主题颜色
    theme: '#2ca6cb'    ##the defaut theme color is blue
```
3.添加网站的总pv计数和总uv计数
在F:\Blog\hexo\themes\jacman\layout\_partial\footer.ejs中最后面添加脚本和总pv计数和总uv计数：
```html
<script async src="https://dn-lbstatics.qbox.me/busuanzi/2.3/busuanzi.pure.mini.js"></script>
<span id="busuanzi_container_site_pv">
      Total visits: <span id="busuanzi_value_site_pv"></span>
</span>
<span id="busuanzi_container_site_uv">
    You are Visiter No.<span id="busuanzi_value_site_uv"></span>
</span>
```

# 参考：
[如何使用 Jacman 主题](http://wuchong.me/blog/2014/11/20/how-to-use-jacman/ "如何使用 Jacman 主题 ")
[Hexo博客Jacman主题的一些优化](http://www.tuicool.com/articles/FRrQvi3 "Hexo博客Jacman主题的一些优化")


---
