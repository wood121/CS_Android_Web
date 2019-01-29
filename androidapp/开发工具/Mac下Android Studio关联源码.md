### 一、关联一般源码
&emsp;&emsp;首先打开Android Studio的SDK Manager查看是否下载了对应的Sources for Android xx。如果没有，先选中下载。
&emsp;&emsp;打开Finder，窗口栏点击&ensp;**前往**，在下拉窗口中点击&ensp;**前往文件夹**。找到AndroidStudioXX/options/jdk.table.xml。
![打开设置文件夹](https://upload-images.jianshu.io/upload_images/4330197-c209a09175b15fa3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&emsp;&emsp;打开jdk.table.xml文件,找到源码相应版本的 < sourcePath>标签,把源码路径写进去就可以了。
![写进源码路径](https://upload-images.jianshu.io/upload_images/4330197-4a31b5597d744c3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 二、关联@hide标注的源码
&emsp;&emsp;有些源码，比如ActivityThread一类，因为被标注了@hide，在IDE中是无法查看到的。但是我们可以通过替换sdk中platforms中对应android-xx中的android.jar包来实现。
&emsp;&emsp;[jar包下载链接](https://github.com/anggrayudi/android-hidden-api)
