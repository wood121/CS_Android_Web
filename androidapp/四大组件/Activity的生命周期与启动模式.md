### 一、生命周期
![Activity生命周期图.png](https://upload-images.jianshu.io/upload_images/4330197-9e96e6e29e8e91bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>onCreate() ，我们可以做一些初始化工作。
onRestart()，
onStart()，
onResume()，
onPause()，我们可以做一些数据存储，停止动画等工作，但不能太耗时
onStop()，可以做一些稍微重量级的回收工作，同样不能太耗时。
onDestroy()，做一些回收工作和最终的资源释放。

>几种情况
**
### 二、启动模式

### 三、工作过程
