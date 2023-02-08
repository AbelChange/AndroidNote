### 1.环境

python3环境pip install frida-tools

pip install frida-dexdump

下载[Frida](https://github.com/frida/frida/releases)  server (frida-server-xxx-android-xxx 和 gadget-android-xxx）

### 2.手机端配置

#启动frida server 

adb push xxx-frida-server  /data/local/tmp
adb shell
cd /data/local/tmp/ 
chmod 755 frida-server-15.2.2-android-x86
./frida-server-15.2.2-android-x86

//设置tcp转接
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043

### 3.本地配置

gadget-android-xxx放到本地目录
C:\Users\haoshuaihui\AppData\Local\Microsoft\Windows\INetCache\frida\gadget-android-arm64.so(重命名)


查看连接状态 : frida-ps -U 

反编前台 app:  frida-dexdump -FU -d -o .     

反编目标apk：frida-dexdump -U -f  com.xxx  -d 