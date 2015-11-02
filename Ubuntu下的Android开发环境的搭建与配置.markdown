## Ubuntu下的Android开发环境的搭建与配置

标签（空格分隔）： android

---

写在前面的话：Android开发环境在Ubuntu下面的配置在32位的时代相对来说还是很方便的，但是当64位的Linux（Ubuntu）大行其道的时候，很多问题就接踵而来了，归根结底是因为很多在Android开发中用的Linux工具都是基于32位的系统编译生成的，所以如果要使这些工具能够在64位的系统上运行，就需要在64位的系统下安装一些32位的兼容包（依赖包），最终通过在网络上的查找以及自己聪明的大脑，总算完成了在Ubuntu下Android开发环境的搭建与配置。

### Eclipse
现在主流的Android开发环境主要*Eclipse*以及*Android* Studio。这里第一部分，我们先来看看如何配置Ecplise的环境（为什么两个我都要配置呢？因为现阶段来说，Android Studio还不能用来进行NDK的开发，如果有NDK的需要，还是要用Eclipse来进行开发；其次Android Studio这个工具在运行起来之后，不知道什么原因总会比Eclipse更加耗费系统资源，所以对于系统资源比较紧张的情况下，我还是要切回Eclipse开发的）。

在配置Ecplise之前，首先要确保你已经在机器上安装了Android SDK，之后才能进行下一步的配置。然后是ADT的安装包，下载ADT的压缩文件，然后在Ecplise下依次*Help->Install New Software->Add->Archive*,然后找到你下载的ADT插件即可完成ADT的安装。上述工作全部之后，我们在创建第一个应用程序并运行时会在下部的窗口提示有类似下列的错误提示信息：`adb: no such file or directory`。总之一句话：对不起，系统里面没有adb这个文件或目录，你瞅瞅是不是压根就没装。但是当我们检索我们的sdk时会发现，我们已经安装了adb工具，之所以会有上面的提示，是因为我们的系统是64位的，而adb所依赖的运行环境是32位的，这是我们就需要安装32位的依赖包啦。首先下面的一条命令（以ubuntu为例）：
```
sudo dpkg --add-architecture i386
```
为系统的软件仓库添加32位的架构，然后升级一下软件仓库：
```
sudo apt-get update
```
然后我们安装ia-32libs的32位支持包，但是当我用`apt-get`安装这个包时会提示已经没有这个包了，相应地该包已经被如下的3个包替代了**lib32z1,lib32ncurses5,lib32bz2-1.0**。那么这就简单了，我们不安ia-32libs，而是安装上面这3个替代包就可以了。
重新编译运行一下前述的示例应用程序，成功运行，Bingo！！！

### Android Studio
Android Studio的配置相对来说就很简单，只要你完成了sdk的安装，在Android Studio里面配置一下就可以使Android Studio运行起来了，因为在Ubuntu下Android Studio的主要问题来自于adb，但是这个问题已经在Eclipse的配置当中解决了。对了
，对于第一次启动Android Studio时卡在更新进度条的问题，只需要在其安装根目录下的bin目录中编辑一下idea.properties文件即可,在文件尾加入如下一行：
```
disable.android.first.run=true
```
保存，退出，重启Android Studio，成功运行。





