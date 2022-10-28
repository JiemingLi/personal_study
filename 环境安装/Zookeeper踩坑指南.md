# Zookeeper踩坑指南



## 安装JDK8（zk启动需要依赖jdk环境）

[安装教程](https://blog.csdn.net/m0_46475664/article/details/125073580)

**问题:**

腾讯云服务器在执行`java -version`的时候报错`bad ELF interpreter: 没有那个文件或目录`

**解决方法**

系统内缺少glibc库导致，需要安装glibc

`yum install -y glib.i686`

[refrence](https://blog.csdn.net/weixin_44519124/article/details/102837025)



## 启动zk

**问题**

错误: 找不到或无法加载主类org.apache.zookeeper.server.quorum.QuorumPeerMain

**解决方法**

从目前的最新版本3.5.5开始，带有bin名称的包才是我们想要的下载可以直接使用的里面有编译后的二进制的包，而之前的普通的tar.gz的包里面是只是源码的包无法直接使用。

[reference](https://www.cnblogs.com/zhoading/p/11593972.html)





