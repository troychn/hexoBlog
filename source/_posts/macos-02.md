---
title: macOs环境下搭建hexo静态博客
toc: true   // 在文章侧边增加文章目录
date: 6/11/2017 5:26:15 PM 
updated: 6/11/2017 5:26:20 PM 
categories: [macOs]
tags: [macOs,linux,git,hexo]

---

hexo-Github Pages静态博客，需要node js的支持，由于我的环境，之前在有安装过hexo，本身已经用的是之前的较早版本的node js，最新由于工作需要，用到angular js，准备安装一个最新的node js,所以对node js有多版本支持的要求，因此我通过nvm来进行管理node的版本。  

### 安装NVM工具
由于上一节安装了oh my zsh命令增强工具，安装nvm我是通过安装zsh的插件方式安装的，
参考：[macOs终端工具优化iTerm2+oh my zsh](http://www.troylc.cc/macOs/2017/06/11/macos-01.html)
如下所示：  
首页：Clone zsh-nvm 到oh my zsh 自定义插件库中：

```sh
git clone https://github.com/lukechilds/zsh-nvm ~/.oh-my-zsh/custom/plugins/zsh-nvm

```
其次：修改oh my zsh的配置文件.zshrc，增加以下内容

```sh
plugins+=(zsh-nvm) 或者加到之前的后面，用空格分开 plugins=(*** zsh-nvm)
```
最后：如果要即时生效刚执行
```sh
source ~/.zshrc  #或者关闭当前终端重新打开。
```
### 安装node js

```bash
zdgqiuysn@zqy  ~  nvm list-remote
        v0.1.14
        v0.1.15
        v0.1.16
        ......
       v0.12.15
       v0.12.16
       v0.12.17
       v0.12.18
       ......
       v7.7.2
       v7.10.0
       v8.0.0
       v8.1.0
```
由于之前折腾hexo静态博客的时候用的是node js v0.12.17版本，安装此版本以便于和之前的hexo保持一致，最近工作中需要用到新版本的node js,所以再安装一个v8.1.0版本，让两个版本共存  

```sh
zdgqiuysn@zqy  ~  nvm install v0.12.17
Downloading and installing node v0.12.17...
Downloading https://nodejs.org/dist/v0.12.17/node-v0.12.17-darwin-x64.tar.gz...
######################################################################## 100.0%
Computing checksum with shasum -a 256
Checksums matched!
Now using node v0.12.17 (npm v2.15.1)
Creating default alias: default -> v0.12.17

zdgqiuysn@zqy  ~  nvm install v8.1.0
Downloading and installing node v8.1.0...
Downloading https://nodejs.org/dist/v8.1.0/node-v8.1.0-darwin-x64.tar.gz...
######################################################################## 100.0%
Computing checksum with shasum -a 256
Checksums matched!
Now using node v8.1.0 (npm v5.0.3)

zdgqiuysn@zqy  ~  nvm list
       v0.12.17
->       v8.1.0
default -> v0.12.17
node -> stable (-> v8.1.0) (default)
stable -> 8.1 (-> v8.1.0) (default)
iojs -> N/A (default)
lts/* -> lts/boron (-> N/A)
lts/argon -> v4.8.3 (-> N/A)
lts/boron -> v6.11.0 (-> N/A)
```
node js已经安装完了，由于先安装的v0.12.17版本，所以被设置为默认的node版本，这里先用这个为默认版本，后续用到新版本时，再切换到新的版本中。

### 安装hexo环境

```sh
zdgqiuysn@zqy  ~  npm install hexo-cli -g

> dtrace-provider@0.8.2 install /Users/zdgqiuysn/.nvm/versions/node/v0.12.17/lib/node_modules/hexo-cli/node_modules/hexo-log/node_modules/bunyan/node_modules/dtrace-provider
> node scripts/install.js
......
/Users/zdgqiuysn/.nvm/versions/node/v0.12.17/bin/hexo -> /Users/zdgqiuysn/.nvm/versions/node/v0.12.17/lib/node_modules/hexo-cli/bin/hexo
hexo-cli@1.0.3 /Users/zdgqiuysn/.nvm/versions/node/v0.12.17/lib/node_modules/hexo-cli
├── abbrev@1.1.0
├── object-assign@4.1.1
├── command-exists@1.2.2
├── minimist@1.2.0
├── bluebird@3.5.0
├── tildify@1.2.0 (os-homedir@1.0.2)
├── chalk@1.1.3 (supports-color@2.0.0, escape-string-regexp@1.0.5, ansi-styles@2.2.1, strip-ansi@3.0.1, has-ansi@2.0.0)
├── hexo-util@0.6.0 (striptags@2.2.1, html-entities@1.2.1, highlight.js@9.12.0, camel-case@3.0.0, cross-spawn@4.0.2)
├── hexo-log@0.1.3 (bunyan@1.8.10)
└── hexo-fs@0.2.0 (escape-string-regexp@1.0.5, graceful-fs@4.1.11, chokidar@1.7.0)

 zdgqiuysn@zqy  ~  npm install hexo --save
npm WARN deprecated swig@1.4.2: This package is no longer maintained
npm WARN engine esprima@3.1.3: wanted: {"node":">=4"} (current: {"node":"0.12.17","npm":"2.15.1"})

> dtrace-provider@0.8.2 install /Users/zdgqiuysn/node_modules/hexo/node_modules/hexo-log/node_modules/bunyan/node_modules/dtrace-provider
> node scripts/install.js
hexo@3.3.7 node_modules/hexo
├── abbrev@1.1.0
├── pretty-hrtime@1.0.3
├── hexo-front-matter@0.2.3
├── archy@1.0.0
├── titlecase@1.1.2
├── text-table@0.2.0
├── bluebird@3.5.0
├── tildify@1.2.0 (os-homedir@1.0.2)
├── moment-timezone@0.5.13
├── moment@2.13.0
├── lodash@4.17.4
├── deep-assign@2.0.0 (is-obj@1.0.1)
├── hexo-i18n@0.2.1 (sprintf-js@1.1.1)
├── minimatch@3.0.4 (brace-expansion@1.1.7)
├── strip-indent@1.0.1 (get-stdin@4.0.1)
├── chalk@1.1.3 (escape-string-regexp@1.0.5, ansi-styles@2.2.1, supports-color@2.0.0, has-ansi@2.0.0, strip-ansi@3.0.1)
├── hexo-util@0.6.0 (striptags@2.2.1, html-entities@1.2.1, highlight.js@9.12.0, cross-spawn@4.0.2, camel-case@3.0.0)
├── js-yaml@3.8.4 (esprima@3.1.3, argparse@1.0.9)
├── hexo-log@0.1.3 (bunyan@1.8.10)
├── swig-extras@0.0.1 (markdown@0.5.0)
├── swig@1.4.2 (optimist@0.6.1, uglify-js@2.4.24)
├── hexo-fs@0.1.6 (escape-string-regexp@1.0.5, graceful-fs@4.1.11, chokidar@1.7.0)
├── hexo-cli@1.0.3 (object-assign@4.1.1, command-exists@1.2.2, minimist@1.2.0, hexo-fs@0.2.0)
├── nunjucks@2.5.2 (asap@2.0.5, yargs@3.32.0, chokidar@1.7.0)
├── warehouse@2.2.0 (graceful-fs@4.1.11, JSONStream@1.3.1, is-plain-object@2.0.3, cuid@1.3.8)
└── cheerio@0.20.0 (entities@1.1.1, dom-serializer@0.1.0, css-select@1.2.0, htmlparser2@3.8.3, jsdom@7.2.2)
```
### hexo命令生成静态页面
由于之前已经搭建过了hexo的静态博客，这次只要安装完hexo，通过hexo命令生成页面和上传到github上就行，
参考：[Hexo教程(系列)-hexo静态博客](http://www.troylc.cc/categories/hexo/)

创建一篇新的文章，执行hexo g -d 操作

```sh
zdgqiuysn@zqy pwd
/myspace/java/githubPages/Blog/hexo

zdgqiuysn@zqy  hexo g -d
 
[master c4513b5] Site updated: 2017-06-11 19:31:39
 107 files changed, 598 insertions(+), 325 deletions(-)
 rewrite archives/2017/06/index.html (72%)
 rewrite archives/2017/index.html (79%)
 rewrite archives/index.html (82%)
 create mode 100644 categories/macOs/index.html
 create mode 100644 images/macos/macos-01/1.png
 create mode 100644 images/macos/macos-01/10.jpg
 create mode 100644 images/macos/macos-01/11.jpg
 create mode 100644 images/macos/macos-01/12.jpg
 create mode 100644 images/macos/macos-01/2.jpg
 create mode 100644 images/macos/macos-01/3.jpg
 create mode 100644 images/macos/macos-01/4.jpg
 create mode 100644 images/macos/macos-01/5.jpg
 create mode 100644 images/macos/macos-01/6.jpg
 create mode 100644 images/macos/macos-01/7.jpg
 create mode 100644 images/macos/macos-01/8.jpg
 create mode 100644 images/macos/macos-01/9.jpg
 create mode 100644 macOs/2017/06/11/macos-01.html
 create mode 100644 macOs/2017/06/11/macos-02.html
 rewrite page/4/index.html (76%)
 create mode 100644 page/5/index.html
 create mode 100644 tags/git/index.html
 rewrite tags/hexo/index.html (74%)
 rewrite tags/linux/index.html (76%)
 create mode 100644 tags/macOs/index.html
 create mode 100644 tags/zsh/index.html
Enter passphrase for key '/Users/zdgqiuysn/.ssh/id_rsa':
To github.com:troychn/troychn.github.io.git
   79d9285..c4513b5  HEAD -> master
Branch master set up to track remote branch master from git@github.com:troychn/troychn.github.io.git.
On branch master
nothing to commit, working tree clean
Enter passphrase for key '/Users/zdgqiuysn/.ssh/id_rsa':
To git.coding.net:troylc/troylc.git
   79d9285..c4513b5  HEAD -> master
Branch master set up to track remote branch master from git@git.coding.net:troylc/troylc.git.
INFO  Deploy done: git
```
打开浏览器输入http://www.troylc.cc/
![1](/images/macos/macos-02/1.png)

**如何卸载Hexo：**

```sh
3.0.0版本执行
npm uninstall hexo-cli -g

之前版本执行
npm uninstall hexo -g
```



