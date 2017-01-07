---
title: Hexo教程(二)-hexo-jacman主题优化 #可以改成中文的，如“新文章”
toc: true
date: 2016-05-31 18:09:23 #发表日期，一般不改动
updated: 2016-06-21 13:09:23 #更新时间
categories: hexo #文章文类
tags: [hexo优化,jacman,hexo] #文章标签，多于一项时用这种格式，只有一项时使用tags: blog

---
### 前言
上一篇博文把我们的博客已经部署到了github pages服务上，别人可以通过网址来登陆我们的博客了，但是我们这时博客并不好看，怎么优化hexo博客，在Hexo下已经有很多人开发了各种主题给我们使用，我们只需要把他克隆过来，然后通过修改配置文件即可达到要的效果。那么我们应该怎么修改呢？
### 正文
#### [Hexo官网](https://hexo.io/themes/ "hexo主题专栏")主题专栏
![Hexo官网主题](/images/hexo-2/themes.png)
可以看到有很多主题给我们选，我们只要选择喜欢的主题点击进去，然后进入到它的github地址，我们只要把这个地址复制下来(例如我是选择：jacman这个主题)
![Hexo官网主题](/images/hexo-2/jacmangithub.png)



#### 克隆主题jacman
再打开Hexo文件夹下的themes目录（F:\Blog\hexo\themes），右键Git Bash，在命令行输入:
```bash
$ git clone https://github.com/wuchong/jacman.git themes/jacman
```
等待下载完成

#### 启用主题jacman
修改你的博客根目录(F:\Blog\hexo)下的_config.yml配置文件中的theme属性，将其设置为jacman。
```bash
......
#theme: landscape
theme: jacman
......
```
#### 部署主题jacman
返回Hexo目录，右键Git Bash，输入
```bash
hexo g
hexo s
```
打开本地浏览器，输入`http://localhost:4000/` 即可看见我们的主题已经更换了
![Hexo官网主题](/images/hexo-2/usejacman.png)

如果效果满意，将它部署到Github上;打开Hexo文件夹，右键Git Bash，输入
```bash
hexo clean   (必须要，不然有时因为缓存问题，服务器更新不了主题)
hexo g -d
```
打开自己github pages的主页(`https://xxxxx.github.io`)，即可看到修改后的效果

#### 优化hexo主配置
修改你的博客根目录(F:\Blog\hexo)下的_config.yml配置
```bash
# Hexo Configuration
## Docs: https://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site 网站
title: troylc技术博客                   #网站标题
subtitle: 爱生活爱编程，一起来进步吧！  #网站副标题
description:  hello,every body!         #网站描述
author: troylc                          #您的名字 
language: zh-CN                         #网站使用的语言
timezone:                               #网站时区。Hexo 默认使用您电脑的时区

# URL 网址
## 如果您的网站存放在子目录中，例如 http://yoursite.com/blog，则请将您的 url 设为 http://yoursite.com/blog 并把 root 设为 /blog/
url: https://www.troylc.cc/
root: /
#permalink: :year/:month/:day/:title/
permalink: :category/:year/:month/:day/:title.html  #生成文章的链接格式
permalink_defaults:

# Directory 目录配置
source_dir: source              #源文件夹，这个文件夹用来存放内容。
public_dir: public              #公共文件夹，这个文件夹用于存放生成的站点文件。
tag_dir: tags                   #标签文件夹
archive_dir: archives           #归档文件夹
category_dir: categories        #分类文件夹
code_dir: downloads/code        #nclude code 文件夹
i18n_dir: :lang                 #国际化（i18n）文件夹
skip_render:                    #跳过指定文件的渲染，您可使用 glob 表达式来匹配路径。

# Writing 文章
new_post_name: :title.md        # File name of new posts 新建文章默认文件名
default_layout: post            # 默认布局
titlecase: false                # Transform title into titlecase
external_link: true             # Open external links in new tab  在新标签中打开一个外部链接，默认为true
filename_case: 0                #转换文件名，1代表小写；2代表大写；默认为0，意思就是创建文章的时候，是否自动帮你转换文件名，默认就行，意义不大。
render_drafts: false            #是否渲染_drafts目录下的文章，默认为false
post_asset_folder: false        #启动 Asset 文件夹
relative_link: false            #把链接改为与根目录的相对位址，默认false
future: true                    #显示未来的文章，默认false
highlight:    #代码块的设置
  enable: true
  line_number: true
  auto_detect: false
  tab_replace:
# 代码高亮主题
# available: default | night
available: night
highlight_theme: night  

# Category & Tag  分类和标签的设置
default_category: uncategorized   #默认分类         
category_map:                     #分类别名 category_map 是为了让url中尽量少出现中文，做的映射。如下配置
#    编程: programming
#    生活: life
#    其他: other
tag_map:                          #标签别名

# Date / Time format
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: YYYY-MM-DD
time_format: HH:mm:ss

# Pagination
## Set per_page to 0 to disable pagination
per_page: 10
pagination_dir: page

# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
#theme: landscape
theme: jacman  #主题
stylus: 
  compress: true
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: 
      github: git@github.com:xxxx/xxxxx.github.io.git #部署到github pages
      coding: git@git.coding.net:xxxx/xxxx.git #部署到coding pages
  branch: master #上传到git服务的主分支上

# 自动生成sitemap
#sitemap:
#path: sitemap.xml
#baidusitemap:
#path: baidusitemap.xml
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
#### 优化jacman主题配置
修改你的博客主题根目录(F:\Blog\hexo\themes\下的_config.yml配置
```bash
##### Menu  #一级菜单
menu:
  home: /
  categories: /categories
  tags: /tags
  archives: /archives
  life: /life
  about: /about
## you can create `tags` and `categories` folders in `../source`.
## And create a `index.md` file in each of them.
## set `front-matter`as
## layout: tags (or categories)
## title: tags (or categories)
## ---

#### Widgets #内容区域的左侧内容
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
  
#### RSS 
rss: /atom.xml ## RSS address.

#### Image  #网站相关logo图片 及主题颜色
imglogo:
  enable: true             ## display image logo true/false.
  src: img/logo.png        ## `.svg` and `.png` are recommended,please put image into the theme folder `/jacman/source/img`.
favicon: img/favicon.ico   ## size:32px*32px,`.ico` is recommended,please put image into the theme folder `/jacman/source/img`.     
apple_icon: img/jacman.jpg ## size:114px*114px,please put image into the theme folder `/jacman/source/img`.
author_img: img/author.jpg ## size:220px*220px.display author avatar picture.if don't want to display,please don't set this.
banner_img: #img/banner.jpg ## size:1920px*200px+. Banner Picture
### Theme Color #主题颜色
theme_color:
    theme: '#2ca6cb'    ##the defaut theme color is blue

# 代码高亮主题
# available: default | night
available: night
highlight_theme: night

#### index post is expanding or not  该主题首页文章列表默认是全部展开，这里关闭，只展示标题和描述
index:
  expand: false   ## default is unexpanding,so you can only see the short description of each post.
  excerpt_link: Read More  

close_aside: false  #close sidebar in post page if true
mathjax: false      #enable mathjax if true

### Creative Commons License Support, see http://creativecommons.org/ 
### you can choose: by , by-nc , by-nc-nd , by-nc-sa , by-nd , by-sa , zero
creative_commons: none

#### Author information  #作者信息
author:
  intro_line1:  "Hello ,I'm troylc Page in github."    ## your introduction on the bottom of the page
  intro_line2:  "This is my blog,believe it or not."  ## the 2nd line
  weibo_verifier: 0f81d6c8    ## e.g. 0f81d6c8 Your weibo-show widget verifier ,if you use weibo-show it is needed.
  tsina: xxxxxx    ## e.g. xxxxxx  Your weibo ID,It will be used in share button.
  weibo: xxxxxx     ## e.g. xxxxx or xxxxx for http://weibo.com/xxxxxx
  douban:     ## e.g. wuchong1014 or your id for https://www.douban.com/people/wuchong1014
  zhihu:      ## e.g. jark  for http://www.zhihu.com/people/jark
  email:      ## e.g. imjark@gmail.com
  twitter:    ## e.g. jarkwu for https://twitter.com/jarkwu
  github: xxxxx     ## e.g. xxxxxx for https://github.com/xxxxx
  facebook:   ## e.g. imjark for https://facebook.com/imjark
  linkedin:   ## e.g. wuchong1014 for https://www.linkedin.com/in/wuchong1014
  google_plus:    ## e.g. "111190881341800841449" for https://plus.google.com/u/0/111190881341800841449, the "" is needed!
  stackoverflow:  ## e.g. 3222790 for http://stackoverflow.com/users/3222790/jark
## if you set them, the corresponding  share button will show on the footer

#### Toc
toc:
  article: true   ## show contents in article.
  aside: true     ## show contents in aside.
## you can set both of the value to true of neither of them.
## if you don't want display contents in a specified post,you can modify `front-matter` and add `toc: false`.

#### Links
links:
  攻城狮: https://www.troylc.cc/，一个面向程序员交流分享的博客
  troylc's Blog: https://www.troylc.cc/

#### Comment
duoshuo_shortname: xxxx   ## e.g. xxxx   your duoshuo short name.
disqus_shortname:     ## e.g. wuchong   your disqus short name.

# You can visit https://leancloud.cn get AppID and AppKey.
#leancloud_visitors:
#  enable: true
#  app_id: CqxYlI0bMsyMrs3ODUxr2mpV-gzGzoHsz # your app_id
#  app_key eX5MnY61mQfgQx2aiXnajze5 # your app_key

#### Share button 采用jiathis分享
jiathis: 
  enable: true ## if you use jiathis as your share tool,the built-in share tool won't be display.
  id: xxxxx   ## e.g. 1889330 your jiathis ID. 
  tsina: xxxxx2176287895 Your weibo id,It will be used in share button.

#### Analytics
google_analytics:
  enable: false
  id:        ## e.g. UA-46321946-2 your google analytics ID.
  site:      ## e.g. wuchong.me your google analytics site or set the value as auto.
## You MUST upgrade to Universal Analytics first!
## https://developers.google.com/analytics/devguides/collection/upgrade/?hl=zh_CN
baidu_tongji:
  enable: true
  sitecode: b991924b21cd080f0a0e761dd1a69288 ## e.g. b991924b21cd080f0a0e761dd1a69288 your baidu tongji site code
cnzz_tongji:
  enable: false
  siteid:    ## e.g. 1253575964 your cnzz tongji site id

#### Miscellaneous
ShowCustomFont: true  ## you can change custom font in `variable.styl` and `font.styl` which in the theme folder `/jacman/source/css`.
fancybox: true        ## if you use gallery post or want use fancybox please set the value to true.
totop: true           ## if you want to scroll to top in every post set the value to true返回顶部按钮，totop。在博客右下脚显示一件返回顶部按钮


#### Custom Search
google_cse: 
  enable: false
  cx:   ## e.g. 018294693190868310296:abnhpuysycw your Custom Search ID.
## https://www.google.com/cse/ 
## To enable the custom search You must create a "search" folder in '/source' and a "index.md" file
## set the 'front-matter' as
## layout: search 
## title: search
## ---
baidu_search:     ## http://zn.baidu.com/  极速体验版本http://zn.baidu.com/cse/wiki/index?category_id=25
  enable: true
  id: troylc.cc  ## e.g. "783281470518440642"  for your baidu search id
  site: http://zhannei.baidu.com/cse/site  ## your can change to your site instead of the default site
  
tinysou_search:     ## http://tinysou.com/
  enable: false
  id:  ## e.g. "4ac092ad8d749fdc6293" for your tiny search id

#swiftype搜索
swift_search:
  enable: false

```
注意：凡是有XXXXX处的，都要换成自己对应的账号。

### 参考
[如何使用Jacman主题](http://jacman.wuchong.me/2014/11/20/how-to-use-jacman/ "如何使用Jacman主题")
[jacman-github](https://github.com/wuchong/jacman "jacman-github")



后续针对具体的jacman的首页和文章页进行优化。持续更新中



---
