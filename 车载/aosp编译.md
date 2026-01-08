

[toc]

# AOSP编译

> AOSP:*Android Open Source Project*
>
> Android 是适用于各种不同规格设备的操作系统。任何人都可以通过AOSP)查看 Android 的文档和源代码。您可以使用 AOSP 为自己的设备创建自定义 Android OS 变体。
>
> *Android 第一方应用开发者*:有权访问 AOSP 系统 API 并编写特权应用程序和设备制造商应用程序的 Android 应用程序开发人员。*Android 第三方应用开发者*:仅使用 Android 公共 SDK 来创建 Android 应用程序的 Android 应用程序开发人员。

## 1. 系统要求与配置

- 64 位 x86 系统
- Ubuntu 18.04 
- Python 3.6 +

## 2. 获取aosp源码

https://source.android.com/docs/setup/download/downloading?hl=zh-cn

>在服务器上下载/编译 Android，可能需要几个小时。如果你：
>
>- 使用普通终端，断线=任务中断 😫
>- 使用 tmux，断线后可以 **重新连接继续操作** 😎



```bash
tmux new -s aosp          # 新建一个叫 aosp 的会话

tmux attach -t aosp       # 重新连接 aosp 会话

tmux kill-session -t aosp # 杀掉 aosp 会话
```



```shell

# ① 安装Git、Curl、Python，有些系统自带~
sudp apt-get install git
sudo apt-get install curl
sudo apt-get install python

# ② 创建bin，并添加到PATH中
mkdir ~/bin
PATH=~/bin:$PATH

# ③ 下载Repo工具，并设置为可执行
# Tips：国外源可能有问题，可以替换为国内镜像，比如：https://mirrors.tuna.tsinghua.edu.cn/git/git-repo
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo

# ④ 创建工作文件目录 (存源码)
mkdir aosp
cd aosp

# Tips：Repo在运行过程中会尝试访问官方Git源更新自己，如果想用tuna镜像更新，可复制
# 下述内容到~/.bashrc中
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/'

# ⑤ Git设置身份信息(名字、邮箱)
git config --global user.name  "User Name"
git config --global user.email "user@example.com"

# ⑥ 初始化仓库 与 选中分支 (同样支持替换镜像源) 
#可选值查看：https://source.android.com/source/build-numbers#source-code-tags-and-builds
#kenel编译限制：https://source.android.google.cn/docs/setup/reference/bazel-support?hl=en
#gsi分支源码编译限制：https://ci.android.com/builds/branches/aosp-android14-gsi/grid?legacy=1
repo init -c -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-14.0.0_r1

# ⑦ 同步代码 (拉取aosp源码到工作目录，一般要几个小时) https://source.android.com/docs/automotive/start/pixelxl?hl=zh-cn
#加上 -c 参数，仅同步当前android-13.0.0_r82分支，节约下载时间
repo sync -c -j4 --fail-fast --force-sync
repo info #可以查看当前repo信息

#下载过程可能出现某些project找不到，可能是镜像问题，也可能是网的问题，
# 试试可以单独同步该项目，支持-jN参数，-j4如：
repo sync -c platform/frameworks/layoutlib
repo sync -c -j1 --fail-fast --force-sync

# ⑧ Tips：
#下载后进行初始化再同步，效率会高很多
curl -OC - https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar 

tar -xvf aosp-latest.tar  -C aosp/
```

#repo sync issues集合：https://developer.aliyun.com/article/1599065



## 3. 构建Android

|      | 编译系统 | 配置文件   |
| ---- | -------- | ---------- |
| old  | Makefile | Android.mk |
| New  | Soong    | Android.bp |

-  ninja 更底层的编译规则，.ninja，Android.bp 可以被 BlueSprint下翻译成 Ninja语法文件

### aosp源码编译后会生成一系列的产物(out目录下)：

### 3.1编译

**编译产物**

> /out/host → Android开发工具相关的产物，包含各种SDK工具，如adb、dex2oat、aapt等；
>
> /out/target/common → 一些通用的编译产物，包含Java应用代码和Java库；
>
> /out/target/product/[product_name] → 针对特定设备的编译产物，以及平台相关C/C++代码与二进制文件(如system.img、ramdisk.img、userdata.img、boot.img等)；

```shell
# ① 初始化环境
source build/envsetup.sh

# ② 删除out与中间文件，clean会删除本次设置生成的、clobber会删除所有配置生成的
m clobber	#清除所有编译缓存，相当于rm -rf out/
m clean	#清除编译缓存，out/target/product/[product_name]
m installclean	#清除所有二进制文件

# ③ 选择编译目标
#menu 选择
lunch
#直接选择 
lunch aosp_sunfish_car-userdebug

# ④ 开始编译，后面的-jN参数用来设置编译的并行任务数，CPU核心数为6，N值最好选6-12间
# 根据CPU核心数动态修改
m -j20

# 可以把输出结果打印到log文件中：make -j6 2>&1 | tee build_20211206_1403.log

# Tips：有需要还可以键入：make sdk，编译SDK生成修改后的android.jar

#单编
cd package/apps/Setting m 或者 mmm xxx

# 编译单独模块的可选指令如下：
# mm → 编译当前目录下的模块，不编译依赖模块 
# mmm → 编译指定目录下的模块，不编译依赖模块
# mma → 编译当前目录下的模块及其依赖项
# mmmma → 编译指定路径下所有模块，切包含依赖
```

**编译错误解决**

ninja failed with: exit status 137 ，构建过程由于内存问题而终止，具体系统日志/var/log/syslog  OOM->kill ninja

```shell
#执行
sudo vim build/soong/java/config/config.go    
#或     
sudo gedit build /soong/ java/config/ config.go
#把2048修改为4096关闭当前编译Terminal窗⼝
pctx.StaticVariable("JavacHeapSize" , "4096M") 
```

```shell
#配置
sudo vim ~/.bashrc
export MAVEN_OPTS="-Xms8g -Xmx8g"
#编译启动后看是否生效 
grep "JavacHeapSize" out/soong/build.ninja
```

> https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650271897&idx=1&sn=47e3e0c45a1c832f9e1af91d9b9c5d87&chksm=886301f6bf1488e0a23e57857352f8c1d771dae9bcef4ceb96600d76310157ffb74998ab542b&scene=27

![image-20250529172011221](img/aosp%E7%BC%96%E8%AF%91/image-20250529172011221.png)

编译产物为266G  

### 3.2AS查看源码

```shell
#根目录执行，生成idegen.jar
mmm development/tools/idegen/

#源码根目录生成android.ipr和android.iml
sudo development/tools/idegen/idegen.sh 

sudo chmod 777 android.iml
sudo chmod 777 android.ipr

#编辑 android.iml 排除不需要的模块,如下表

#as 打开android.ipr

#修改sdk/jdk指向

```

```xml
<excludeFolder url="file://$MODULE_DIR$/./external/emma"/>
<excludeFolder url="file://$MODULE_DIR$/./external/jdiff"/>
<excludeFolder url="file://$MODULE_DIR$/out/eclipse"/>
<excludeFolder url="file://$MODULE_DIR$/.repo"/>
<excludeFolder url="file://$MODULE_DIR$/external/bluetooth"/>
<excludeFolder url="file://$MODULE_DIR$/external/chromium"/>
<excludeFolder url="file://$MODULE_DIR$/external/icu4c"/>
<excludeFolder url="file://$MODULE_DIR$/external/webkit"/>
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/docs"/>
<excludeFolder url="file://$MODULE_DIR$/out/host"/>
<excludeFolder url="file://$MODULE_DIR$/out/target/common/docs"/>
<excludeFolder url="file://$MODULE_DIR$/out/target/common/obj/JAVA_LIBRARIES/android_stubs_current_intermediates"/>
<excludeFolder url="file://$MODULE_DIR$/out/target/product"/>
<excludeFolder url="file://$MODULE_DIR$/prebuilt"/>
<excludeFolder url="file://$MODULE_DIR$/external/chromium" />
<excludeFolder url="file://$MODULE_DIR$/external/icu4c" />
<excludeFolder url="file://$MODULE_DIR$/external/webkit" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/docs" />
<excludeFolder url="file://$MODULE_DIR$/hardware" />
<excludeFolder url="file://$MODULE_DIR$/kernel" />
<excludeFolder url="file://$MODULE_DIR$/libcore" />
<excludeFolder url="file://$MODULE_DIR$/libnativehelper" />
<excludeFolder url="file://$MODULE_DIR$/ndk" />
<excludeFolder url="file://$MODULE_DIR$/out" />
<excludeFolder url="file://$MODULE_DIR$/out/eclipse" />
<excludeFolder url="file://$MODULE_DIR$/out/host" />
<excludeFolder url="file://$MODULE_DIR$/out/target/common/docs" />
<excludeFolder url="file://$MODULE_DIR$/out/target/common/obj/JAVA_LIBRARIES/android_stubs_current_intermediates" />
<excludeFolder url="file://$MODULE_DIR$/out/target/product" />
<excludeFolder url="file://$MODULE_DIR$/pdk" />
<excludeFolder url="file://$MODULE_DIR$/platform_testing" />
<excludeFolder url="file://$MODULE_DIR$/prebuilt" />
<excludeFolder url="file://$MODULE_DIR$/prebuilts" />
<excludeFolder url="file://$MODULE_DIR$/sdk" />
<excludeFolder url="file://$MODULE_DIR$/shortcut-fe" />
<excludeFolder url="file://$MODULE_DIR$/toolchain" />
<excludeFolder url="file://$MODULE_DIR$/tools" />
<excludeFolder url="file://$MODULE_DIR$/QNX" />
<excludeFolder url="file://$MODULE_DIR$/QNX_vendor" />
<excludeFolder url="file://$MODULE_DIR$/test" />
<excludeFolder url="file://$MODULE_DIR$/system" />
<excludeFolder url="file://$MODULE_DIR$/out_images" />
<excludeFolder url="file://$MODULE_DIR$/external" />
<excludeFolder url="file://$MODULE_DIR$/AMSS" />
<excludeFolder url="file://$MODULE_DIR$/developers" />
<excludeFolder url="file://$MODULE_DIR$/development" />
<excludeFolder url="file://$MODULE_DIR$/build" />
<excludeFolder url="file://$MODULE_DIR$/device" />
<excludeFolder url="file://$MODULE_DIR$/vendor" />
<excludeFolder url="file://$MODULE_DIR$/cts" />
<excludeFolder url="file://$MODULE_DIR$/dalvik" />
<excludeFolder url="file://$MODULE_DIR$/bootable" />
<excludeFolder url="file://$MODULE_DIR$/compatibility" />
<excludeFolder url="file://$MODULE_DIR$/bionic" />
<excludeFolder url="file://$MODULE_DIR$/art" />
<excludeFolder url="file://$MODULE_DIR$/disregard" />
 
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/media" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/apct-tests" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/apex" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/cmds" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/data" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/docs" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/drm" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/location" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/lowpan" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/libs" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/tests" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/tools" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/graphics" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/keystore" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/mime" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/mms" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/native" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/nfc-extras" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/obex" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/opengl" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/proto" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/rs" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/samples" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/sax" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/telecomm" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/telephony" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/test-base" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/test-legacy" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/test-mock" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/test-runner" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/startop" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/identity" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/ex" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/hardware" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/libs" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/opt" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/rs" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/wilhelm" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/multidex" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/minikin" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/layoutlib" />
 
<excludeFolder url="file://$MODULE_DIR$/frameworks/av" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/compile" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/ml" />
<excludeFolder url="file://$MODULE_DIR$/frameworks/native" />
<excludeFolder url="file://$MODULE_DIR$/packages/apps" />
<excludeFolder url="file://$MODULE_DIR$/packages/inputmethods" />
<excludeFolder url="file://$MODULE_DIR$/packages/screensavers" />
<excludeFolder url="file://$MODULE_DIR$/packages/wallpapers" />
<excludeFolder url="file://$MODULE_DIR$/packages/services/BuiltInPrintService" />
<excludeFolder url="file://$MODULE_DIR$/packages/services/Mms" />
<excludeFolder url="file://$MODULE_DIR$/packages/services/Mtp" />
<excludeFolder url="file://$MODULE_DIR$/packages/services/Telecomm" />
<excludeFolder url="file://$MODULE_DIR$/packages/services/Telephony" />
<excludeFolder url="file://$MODULE_DIR$/packages/services/AlternativeNetworkAccess" />
```

### 3.3 aosp下解压执行驱动->vendor

#1.当前repo对应的分支信息
cd .repo/manifests
git logwq

#2.下载
#buildId与tag的对应关系 https://source.android.com/docs/setup/reference/build-numbers?hl=zh-cn
#TQ3A.230805.001.S1	android-13.0.0_r83
#驱动下载地址 with buildId--  https://developers.google.com/android/drivers?hl=zh-cn

#3.放到根目录解压
tar -xvzf qcom-sunfish-tq3a.230805.001.s1-377dd9d9.tgz
tar -xvzf google_devices-sunfish-tq3a.230805.001.s1-2a7bf157.tgz
#解压得到2个sh自释放文件
extract-google_devices-walleye.sh extract-gcom-walleye.sh
#4.安装
./extract-google_devices-sunfish.sh
./extract-qcom-sunfish.sh
#看完license输入 I ACCEPT 安装完成，此时会创建vendor文件夹

## 4.刷入设备

oem解锁：https://wiki.lineageos.org/devices/sunfish/install/#unlocking-the-bootloader

- **重启到Fastboot模式**：确保设备已重启至Fastboot模式，通常通过按住特定的物理按键组合（如[音量减键](https://zhida.zhihu.com/search?content_id=248441599&content_type=Article&match_order=1&q=音量减键&zhida_source=entity)+电源键）实现。
- **烧录镜像**：使用`fastboot flash`命令将编译好的系统镜像烧录至设备的指定分区。 例如，`fastboot flash system system.img`会将`system.img`镜像烧录至设备的[系统分区](https://zhida.zhihu.com/search?content_id=248441599&content_type=Article&match_order=2&q=系统分区&zhida_source=entity)。
- **重启设备**：烧录完成后，使用`fastboot reboot`命令重启设备，使新的系统镜像生效。

### 1. **`boot.img`** (启动映像)

- **必需**：这个文件包含了 Android 的启动加载程序（bootloader）和内核（kernel），它是设备能够启动并加载 Android 操作系统的关键部分。如果你修改了内核或者启动程序，需要刷入该映像。
- 如果你没有修改内核或启动程序，`boot.img` 就不需要更新，但通常在刷机时还是需要刷入。

### 2. **`system.img`** (系统映像)

- **必需**：这是包含 Android 操作系统、系统应用以及库文件的映像文件。如果你已经编译了 AOSP 系统，`system.img` 是你需要刷入的核心文件。它包含了 Android 的所有用户空间程序。
- 必须刷入，尤其是在你修改了 AOSP 系统或添加/更改了应用的情况下。

### 3. **`recovery.img`** (恢复映像)

- **可选**：这个文件包含了 Android 的恢复模式。如果你修改了恢复环境（例如，安装了自定义恢复像 TWRP 或者修改了默认恢复），你需要刷入新的 `recovery.img`。
- 如果你没有做任何恢复模式的修改，通常可以跳过此步骤，设备会使用默认的恢复模式（通常是设备厂商提供的恢复系统）。

### 4. **`vendor.img`** (厂商映像)

- **可选**：这个文件包含设备厂商特定的硬件抽象层（HAL）和驱动程序。它对于设备的硬件支持至关重要（例如：摄像头、传感器、蓝牙等）。
- 如果你修改了或更新了设备特定的硬件驱动（如摄像头、音频驱动等），你需要刷入 `vendor.img`。
- 如果你没有修改任何硬件驱动或厂商特定的部分，通常 `vendor.img` 不需要更新。大多数设备会有一个预先提供的 `vendor.img`，不需要手动修改。

### 5. **`userdata.img`** (用户数据映像)

- **可选**：包含设备的用户数据，如应用数据和设置。通常，在刷机时并不需要刷入 `userdata.img`，除非你想要恢复设备到出厂设置（即清空所有数据）。
- 如果你只想更新系统或内核，并且不需要清除设备上的数据，跳过此步骤。如果你想做一次干净刷机或恢复设备到出厂状态，可以使用 `fastboot erase userdata`。

### 6. **`radio.img` / `modem.img`** (基带和无线电映像)

- **可选**：包含设备的基带（Modem）和无线电（Radio）固件。这些文件对设备的通信功能（例如通话、Wi-Fi、蓝牙等）至关重要。
- 如果你没有更新基带或无线电固件，通常不需要刷入 `radio.img` 或 `modem.img`。但是，如果你的设备需要更新无线电驱动，刷入这些文件是必要的。

### 7. **`bootloader.img`** (启动加载程序映像)

- **可选**：部分设备需要更新启动加载程序（bootloader），尤其是在需要解锁 Bootloader 或设备有固件更新时。如果你没有对 Bootloader 进行修改，通常不需要刷入该映像。

- 这个文件一般只有在设备有特别的要求（例如厂商更新 Bootloader 或解锁）时才需要刷入。


  ```shell
  cd out
  
  #解压成zip
  find . -type f -name "*.img" | zip all_images.zip -@
  
  #传输到mac开始刷入
  
  ```


其他品牌刷机：
https://wiki.lineageos.org/devices/sunfish/install/#unlocking-the-bootloader
