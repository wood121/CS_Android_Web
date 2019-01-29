>一些总结，方便查找使用
## 一、获取keystore信息
#### debug版本的SHA1数值
>* 打开终端，输入cd .Android
>* 输入keytool -list -v -keystore debug.keystore
>* 输入口令 android 
>* 查看keystore相关信息，比如SHA1值、MD5值
#### release版本的SHA1数值
>* 使用IDE工具创建一个keystore
>* 输入keytool -list -v -keystore  上一步创建的keystore
>* 输入自己创建keystore的相关口令
>* 查看keystore相关信息，比如SHA1值、MD5值
## 二、更改IDE默认使用的debug.keystore
#### Eclipse操作
>http://blog.csdn.net/superbigcupid/article/details/48230675
#### Android Studio操作
>http://blog.csdn.net/zouchengxufei/article/details/48747803
