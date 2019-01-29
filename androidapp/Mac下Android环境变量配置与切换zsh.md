## 一 配置环境变量
1.启动Terminal终端工具
2.创建.bash_profile文件
>touch .bash_profile

3.打开并编辑.bash_profile文件
>open .bash_profile

4.在.bash_profile文件中写入以下内容
>export ANDROID_HOME=/Users/wood121/Library/Android/sdk
>export PATH=${PATH}:${ANDROID_HOME}/tools
>export PATH=${PATH}:${ANDROID_HOME}/platform-tools

注意：ANDROID_HOME中的sdk路径填写自己电脑中的路径

5.使.bash_profile文件生效
>source .bash_profile

6.Terminal中输入adb查看是否生效，未生效会出现
>command not found

## 二 切换到zsh
1.查看Mac本中的zsh，Terminal中输入
>cat /etc/shells
可以看到你的mac本中已经存在的各种shell，最下面一种既是/bin/zsh

>zsh --version
或输入这个命令查看zsh版本

2.改变默认的shell
>我们只需要在终端输入zsh，即可完成切换

3.重启Terminal
>Terminal语句开头已是绿色波浪线

4.安装完zsh后使用adb,git会遇到的问题
>"zsh: command not found adb:adb"或者"zsh: command not found: [Git]"

这是因为：原来使用的bash shell和zsh读取的不同的系统环境变量
bash shell读取的是.bash_profile文件中的，而zsh读取的是.zshrc中的

5.解决zsh安装完后找不到adb,git的问题
>open .zshrc
打开zsh的配置文件，在# User configuration部分，
添加读取.bash_profile的语句:source ~/.bash_profile

![](http://upload-images.jianshu.io/upload_images/4330197-e5d17066be15192b.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>source .zshrc
使.zshrc生效

>输入adb进行验证
