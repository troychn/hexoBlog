---
title: macOs使用技巧与问题记录-outlook2016无法搜索
toc: true   // 在文章侧边增加文章目录
date: 6/11/2017 5:26:15 PM 
updated: 6/11/2017 5:26:20 PM 
categories: [macOs]
tags: [macOs,outlook]

---

#使用技巧：

#问题记录：
1. macos最新版本安装outlook无法搜索的问题

| 问题描述 | 问题环境 | 原因 |
| --- | --- | --- |
| macos最新版本安装    outlook无法搜索的问题 | Macos:10.13.3   Outlook:2016-16.11 | outlook2016版本的搜索功能依赖于mac的聚焦搜索(spotlight)功能，这时我用spotlight来测试一下，果然spotlight功能也不好使用，于是想到怎么来重建这个索引 |

* 解决方法：  
首先：关闭outlook2016的应用  
其次：在命令行中输入以下内容

    ```shell
    #该命令用来关闭索引
    sudo mdutil -i off /
    #该命令用来删除索引
    sudo mdutil -E /
    #该命令用来重建索引
    sudo mdutil -i on /
    
    ```
    然后用快捷键呼出spotlight菜单，随便输入一个词，就能看到提示，正在进行索引，并且告诉你重建索引的时间。于是搜索了一下，果然在spotlight里可以搜索了。  
再次：进入outlook2016中，再试一下搜索，发现好像还是不行。于是从官网上找到了如下代码,来重新为outlook邮件创建索引  

    ```shell
    mdimport -g /Applications/Microsoft\ Outlook.app/Contents/Library/Spotlight/Microsoft\ Outlook\ Spotlight\ Importer.mdimporter -d1 ~/Library/Group\ Containers/UBF8T346G9.Office/Outlook/Outlook\ 15\ Profiles/Main\ Profile
    ```    
通过以上命令操作完后，再次打开outlook2016，发现果然在outlook2016中就可以搜索了    

