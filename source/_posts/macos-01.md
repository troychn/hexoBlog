---
title: macOs终端工具优化iTerm2+oh my zsh
toc: true   // 在文章侧边增加文章目录
date: 6/11/2017 5:26:15 PM 
updated: 6/11/2017 5:26:20 PM 
categories: [macOs]
tags: [macOs,linux,git,zsh]

---

工欲善其事,必先利其器。能从千百年传承下来，必定是经过各行各业的实践与验证，到今年仍然适用，今天就分享一些软件行业对程序员来说的利器(zsh+oh my zsh)这款屌炸天linux下的增加命令行的工具。

之前一直都是用着各种环境下的默认的终端工具，在逛github时，发现了zsh和oh my zsh增强终端工具，觉得很强大，让程序员的逼格瞬间提升一个T，但发现这工具其实老外都是几年前就开始逼格了，一直持续到现在，而且star都是好几万，要不怎么说开源质量方面老个都这么强大呢。虚心学习什么时候都不晚，既然发现了，咋也来逼格一下，并分享自己的安装与配置的过程，记录下逼格过程的快感；扯远了，回正题。

本次安装环境为macOS Sierra 10.12.5下安装与配置iTerm2+zsh+oh my zs，当然网上大神们也有其它环境下的针对(zsh+oh my zs)这工具的安装，后续也给出相关的链接地址。

## iTerm2终端工具

虽然mac系统自带有终端工具，但今天说的Iterm2在mac上显得更加强大与突出，它能使命令行工具变得更加美观与便捷。
### iTerm2安装
在iTerm2[官方网站](http://iterm2.com)下载最新版本的iTerm2安装包，解压之后将iTerm2.app程序文件移动或复制到应用程序(Applications)目录下，即可以完成安装。安装完成后，启动iTerm2，这里先分享我配置好的iTerm2的界面：  
![iTerm2管理界面](/images/macos/macos-01/1.png)

### iTerm2基本配置与操作
#### 修改iTerm2配置方案-配置
首页要选择一个iTerm2的配置主题，我们这里选择的是solarized，这个主题也很强大，有很多工具的配色方案可以选择，从github上下载主题到本地的一个目录

```sh
git clone git@github.com:altercation/solarized.git
```
![配置方案下载](/images/macos/macos-01//2.jpg)
进入iterm2-colors-solarized目录，双击Solarized Dark.itermcolors和Solarized Light.itermcolors这两个文件。
然后进入iTerm2设置preferences->Profiles->Colors->Color Presets->选择刚刚安装的solarized Dark;
![设置iTerm2配置方案](/images/macos/macos-01/3.jpg)
#### 让新窗口的命令行跟随上一个打开的窗口的目录-配置
iTerm2设置preferences->Profiles/Default/General,Working DIrectory选择 Reuse previous session’s directory  
![跟随目录](/images/macos/macos-01/4.jpg)
#### 保存ssh远程登录指令及相关用户名密码-配置
在iTerm2中可以直接存储登录命令，如：`ssh root@localhost`然后就会出现提示，让我们输入密码，但这样每次输入密码也很烦恼，有时远程机器太多，真记不住，然而iTerm2中可以使用expect脚本实现。   

首先打开iTerm2设置preferences->Profiles->在左侧最下方有一个`+`和`-`，或者旁边的Other Actions来新建，删除，复制一个现有的profiles。  

然后接着上面的配置，在配置profiles的界面右侧中，填写相关的信息与快捷键，tags等。
最后在在配置profiles的界面右侧的Command下面的Send Text at start 输入我们编写的expect脚本来运行，如下所示  
编写expect脚本  

```sh
***@localhost ~ vim /usr/local/bin/item2login.sh
#!/usr/bin/expect

set timeout 30
spawn ssh -o ServerAliveInterval=60 -p [lindex $argv 0] [lindex $argv 1]@[lindex $argv 2]
expect {
        "(yes/no)?"
        {send "yes\n";exp_continue}
        "password:"
        {send "[lindex $argv 3]\n"}
}
interact
```
这里`[lindex $argv 0]， [lindex $argv 1]， [lindex $argv 2]， [lindex $argv 3]` 分别代表着4个参数。  
将item2login.sh保存到 /usr/local/bin 就可以了，然后在iTerm2中设置：脚本 端口号 用户名 服务器地址 `密码` 一定要一一对应,如果密码中含有特殊字符，就需要把密码这个参数用``包起来
`item2login.sh 22 root 10.211.55.4 'edfr@#3'  这里最后密码参数中含有@#，所以需要把密码用''单引号包括起来的。`

![](/images/macos/macos-01/5.jpg)

#### iTerm2选择即为复制-操作
在用命令行中进行复制，之前的命令行要么是不能使用常规复制快键来操作,要么是选中后再按复制command+C,然而在iTerm2这个功能就非常出色了,选中就自动复制成功.  
然后我们只需要在粘贴的地方按下 command + v 即可粘贴成功。
#### 命令行界面全文查找功能-操作
全文查找功能和在文本编辑器中一样,只需按下command + f 输入要查找的内容,即可在当前命令行界面查找并高亮显示,点击搜索框右侧箭头可以循环逐个定位，如下所示：
![](/images/macos/macos-01/6.jpg)
#### 分隔屏幕显示-操作
水平分隔 `command +shift +d`，垂直分隔 `command+d`
![](/images/macos/macos-01/7.jpg)
#### 命令行补全-操作
`command + ;`  自动补全命令，`command + shift +h` 把历史输入命令全部显示出来  
![](/images/macos/macos-01/8.jpg)
![](/images/macos/macos-01/9.jpg)
iTerm2就先说这么多了，以后用到了新的功能，再更新分享吧。


## 终端命令增强工具zsh和oh my zsh  

这两个命令行增强工具不单单是适用macOs，其它linux上也是支持的。安装说明可以参考[github[https://github.com/robbyrussell/oh-my-zsh/wiki/Installing-ZSH]](https://github.com/robbyrussell/oh-my-zsh/wiki/Installing-ZSH)上的说明  

### macOs下安装zsh工具  

```sh
***@localhost ~ brew install zsh zsh-completions
......
***@localhost ~ zsh --version
zsh 5.2 (x86_64-apple-darwin16.0)
```

### macOs下安装oh my zsh  

oh my zsh是在zsh基础上增加了许多插件化的增强功能，安装说明也可以在[github[https://github.com/robbyrussell/oh-my-zsh]](https://github.com/robbyrussell/oh-my-zsh)找到说明，以下以github参考进行安装：
**curl:**  

```sh 
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

**wget**

```bash
sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```

#### oh my zsh 配置主题及添加自带的插件和自定义的插件
oh my zsh安装完后，一般配置都会~/.zshrc配置文件中进行配置，
**配置主题agnoster：**  

修改~/.zshrc文件，把主题修改为agnoster,`ZSH_THEME="agnoster"`，修改完后重新打开终端就会生效，或者用`source ~/.zshrc`,让配置即时生效。由于agnoster需要Powerline字体的支持，不然终端的命令中中的三角会显示乱码。  

**[安装Powerline字体](https://github.com/powerline/fonts)：**

```bash
# clone
git clone https://github.com/powerline/fonts.git
# install
cd fonts
./install.sh
# clean-up a bit
cd ..
rm -rf fonts
```
如果终端是iTerm2,需要在Term2设置preferences->Profiles/Default/Text->chang font中选择上面安装的powerline字体  
![](/images/macos/macos-01/10.jpg)

**配置oh my zsh插件**
oh my zsh中插件分为两种:
一种是oh my zsh默认自带的插件，自带的插件是在~/.oh-my-zsh/plugins/目录下，这里面有很多插件，一般应该是足够逼格了。配置插件只需要修改~/.zshrc文件中的`plugins=(git cp z vim-interaction npm)`  

另一种是oh my zsh自定义插件，处定义的插件是放在~/ .oh-my-zsh/custom/plugins/目录下，这个目录下的插件，就需要我们自己到网络上下载，然后再放到这个目录下，然后也是修改~/.zshrc文件中的`plugins=(git zsh-syntax-highlighting cp z vim-interaction npm zsh-nvm)`,
![](/images/macos/macos-01/11.jpg)
比如我这里安装了一个自定义的插件zsh-syntax-highlighting，用来增强zsh在命令行中输入正确的命令为绿色，如果输入错误的命令则显示为红色。如图所示：
![](/images/macos/macos-01/12.jpg)
**安装oh my zsh自定义插件的方法，参考[github源码](https://github.com/zsh-users/zsh-syntax-highlighting/blob/master/INSTALL.md)**：
首先将这个插件的源码克隆到oh-my-zsh的自定义plugins目录中：

```sh
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```
其实在oh my zsh配置文件中配置激活插件，修改~/.zshrc：

```sh
plugins=( [plugins...] zsh-syntax-highlighting)
```

最后执行`source ~/.zshrc`让插件即时生效：
 
```
source ~/.zshrc
```
其它自定义的插件，安装方法类似，大家可以自行到网络上搜索oh my zsh的插件，我也是在慢慢的探索新逼格的新插件中。至此分享结束，后续也会持续关注有关macOs上的工具优化。希望能够帮到大家，相互学习。


