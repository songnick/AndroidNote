在APP程序开发时，有的时候也会需要抓取手机中的网络数据包，分析数据包中的信息以便我们更好的理解和实现网络程序，最近在做项目时也遇到的这种情况，在我们组长的帮助下实现了手机网络数据包的抓取，主要是利用NetCat和WireShaker实现，这里记录一下实现流程。
1.我们必须要做的就是将手机root，具体的root方法和过程我就不说了，可以根据自己手机的具体情况想办法了。

2.下载工具：(1)tcpdump工具，主要功能是抓取网卡中的网络数据；(2)NetCat工具，实现网络端口的监听和重定向；(3)WireShak工具，主要利用它分析抓取的网络数据包。这里的NetCat工具需要根据不同的平台进行编译，以便适合自己使用。这里我把tcpdump和NetCat工具打包了，可以从这里下载，下载后可以直接使用，WireShak就自行下载吧。

3.配置好刚才下载好的工具：
(1)将tcpdump和nc(编译好的NetCat)放到手机中，并使其成为可执行文件。
push tcpdump工具并change mode
adb push tcpdump /data/local/tmp/
adb shell chmod 6755 /data/local/tmp/tcpdump

push nc工具并change mode 
adb push nc /data/local/tmp/
adb shell chmod 6755 /data/data/local/tmp/nc

(2)将下载好的WireShak安装好，并将安装目录配置到path路径中，以便在命令行可以直接执行。

4.使用配置好的工具进行抓包吧！

打开命令行重新启动adb
adb shell "su -c 'sleep 1'"
adb start-server

利用tcpdump抓包并定向给nc
adb shell su -c "/data/local/tmp/tcpdump -i wlan0 -n -s 0 -w - | /data/local/tmp/nc -l -p 11233"

另启命令行启动wirshak分析刚才抓取的数据包
 adb forward tcp:11233 tcp:11233 && nc 127.0.0.1 11233 | wireshark -k -S -i -

启动wireshak后就可以看到抓取的手机网络数据包了，就可以根据需要分析对应的数据包信息。
以上所有的这些配置和运行都是在window 7下面完成的。今天就记录这么多~——~