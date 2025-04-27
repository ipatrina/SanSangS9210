# 三丧S9210 — 安卓root手搓指南

"三丧S9210"是一个安卓智能手机正向提权实践项目。

项目使用搭载 Android 14 操作系统的三星电子 Samsung Galaxy S24 (SM-S9210) 智能手机，实践对REE操作系统层面自主完全控制的最小改动过程。其应用的步骤思想可广泛适用于全部现代安卓生态智能手机。

# 背景

用户使用智能手机普遍无法针对设备软件的底层工作逻辑进行定制。这对实现设备需求扩展、规避不可抗力的监管有着极大的不利。

包括谷歌、三星品牌在内的安卓智能手机，允许用户通过解锁引导加载程序(解BL锁)，定制REE操作系统。

拥有REE操作系统的自主完全控制权利，即可以Linux root身份执行自定义驻守进程，从而实现充分的扩展和管理需求。

"三丧A226B"项目侧重于概念验证，探究安卓OEM安全机制原理及各类设想的正确性和可行性。

"三丧S9210"项目侧重于最优实践，创造实际、可控的主要移动工作环境。

值得注意的是，本项目针对安卓智能手机Linux原生层可操作能力，而非安卓应用提权。如果您的主要目的是为安卓APP获取更多能力，请直接使用Magisk、KernelSU等主流root项目，不要参考本项目的过程和思想。

如果您尚未认识安卓OEM安全机制的通用概念和思想，请先查看"三丧A226B"项目。

# 开始

**01 确定版本文件**

确定您要长期使用的固件版本。一旦您解锁引导加载程序(BL锁)并取得Linux root身份操作权限，一般不再更新设备固件。

使用以下项目，通过三星官方FUS服务器下载工厂固件：

https://github.com/MatrixEditor/samloader3

例如，选择中国香港版本工厂固件(S9210ZHS4AXL4)文件进行工作：SM-S9210_1_20250114074310_0w8z2usw5k_fac.zip

---

**02 写入版本文件**

在解BL锁前，您可以先将带有官方签名的目标软件版本，使用Odin下载工具写入设备。

您可以通过XDA论坛获取Odin下载工具：

https://xdaforums.com/t/patched-odin-3-13-1.3762572

---

**03 解锁引导加载程序**

在"开发者选项"中，拨开"OEM解锁"开关。在Odin下载模式中，解锁引导加载程序。

这不会导致您的保修失效。

---

**04 编辑VBMETA**

您需要将vbmeta.img的第124字节(指针偏移123)修改为0x03。

这将禁用AVB2.0校验，允许ABL接受改动后的安卓系统文件。

您需要将修改后的vbmeta.img打包为vbmeta-patched.tar文件。

---

**05 编辑SUPER**

在本步骤中，我们需要使用基于Debian的操作系统，对super.img中的system、product(可选)、odm分区进行修改。

- 我们需要先将super.img解包。

- 我们需要分别将system、product、odm以EROFS分区格式进行挂载。

- 我们需要分别将system、product、odm分区中的内容，复制到可编辑目录。

- 我们需要将能够驻守自定义root进程的安卓RC服务添加至system分区。

- 在"三丧A226B"项目中，出于概念验证的目的，我们通过修改内核的方法，使SELinux强制放行了所有拒绝的动作。但在主要移动工作环境中，SELinux的拒绝动作失效或将使应用判断为不安全的环境，从而影响应用的使用，严重会威胁联网账号的安全等级。因此，在"三丧S9210"这一实践项目中，我们将维持三星原厂内核文件(boot.img)不变，转为将"u:r:shell:s0"标记加入SELinux许可域(permissive domain)，仅放行我们添加的RC服务，避免普通应用不受SELinux规则的限制。

- 我们可以顺路移除冗余的自带安卓应用(可选)。在以下的示例中，我们移除了预置的谷歌相关应用、微软相关应用、Meta相关服务及部分三星原厂应用。

- 最后，我们需要将修改后的分区，重新创建EROFS镜像，重新打包为super.img，并创建Odin线刷tar文件。

示例修改过程如下：

```
simg2img super.img super-raw.img

imjtool super-raw.img extract

# MMapped: 0x7f2039600000, imgMeta 0x7f2039601000
# liblp dynamic partition (super.img) - Blocksize 0x1000, 2 slots
# LP MD Header @0x3000, version 10.0, with 7 logical partitions @0x0 on block device of 13130 GB, at partition super, first sector: 0x800
# Partitions @0x3080 in 2 groups:
#       Group 0: default
#       Group 1: qti_dynamic_partitions
#             Name: system (read-only, Huawei EROFS Filesystem Image, @0x100000 spanning 1 extents of 5 GB) - extracted
#             Name: odm (read-only, Huawei EROFS Filesystem Image, @0x179500000 spanning 1 extents of 1000 KB) - extracted
#             Name: product (read-only, Huawei EROFS Filesystem Image, @0x179600000 spanning 1 extents of 1 GB) - extracted
#             Name: system_dlkm (read-only, Huawei EROFS Filesystem Image, @0x1d3600000 spanning 1 extents of 14 MB) - extracted
#             Name: system_ext (read-only, Huawei EROFS Filesystem Image, @0x1d4500000 spanning 1 extents of 81 MB) - extracted
#             Name: vendor (read-only, Huawei EROFS Filesystem Image, @0x1d9700000 spanning 1 extents of 1 GB) - extracted
#             Name: vendor_dlkm (read-only, Huawei EROFS Filesystem Image, @0x24dd00000 spanning 1 extents of 28 MB) - extracted

cd extracted

mkdir /mnt/system

mount -t erofs -o loop system.img /mnt/system

cp -a /mnt/system /

cat > /system/system/etc/init/local.rc <<EOF
on property:dev.bootcomplete=1
    start local

service local /system/bin/sh /system/etc/rc.local
    class main
    seclabel u:r:shell:s0
    user root
    group root
    disabled
    oneshot
EOF

chcon --reference /system/system/etc/init/hw/init.rc /system/system/etc/init/local.rc

cat > /system/system/etc/rc.local <<EOF
#! /system/bin/sh

/system/bin/sleep 5

if [ -f /storage/emulated/0/rc.local ]; then
    /system/bin/cp /storage/emulated/0/rc.local /mnt/rc.local
    /system/bin/sh /mnt/rc.local
fi

if [ -f /data/rc.local ]; then
    /system/bin/cp /data/rc.local /mnt/rc.local
    /system/bin/sh /mnt/rc.local
fi
EOF

chcon --reference /system/system/etc/init/hw/init.rc /system/system/etc/rc.local

rm -r /system/system/app/SmartSwitchAgent
rm -r /system/system/app/SmartSwitchStub
rm -r /system/system/app/SamsungPassAutofill_v1
rm -r /system/system/app/FBAppManager_NS
rm -r /system/system/priv-app/GalaxyApps_OPEN
rm -r /system/system/priv-app/SamsungMessages
rm -r /system/system/priv-app/SmartSwitchAssistant
rm -r /system/system/priv-app/SamsungPass
rm -r /system/system/priv-app/OneDrive_Samsung_v3
rm -r /system/system/priv-app/YourPhone_P1_5
rm -r /system/system/priv-app/LinkToWindowsService
rm -r /system/system/priv-app/FBInstaller_NS
rm -r /system/system/priv-app/FBServices

mkfs.erofs -zlz4hc system-patched.img /system

umount /mnt/system

rm -r /mnt/system

rm -r /system

mkdir /mnt/product

mount -t erofs -o loop product.img /mnt/product

cp -a /mnt/product /

rm -r /product/app/DuoStub
rm -r /product/app/Gmail2
rm -r /product/app/YouTube
rm -r /product/app/Maps
rm -r /product/app/AssistantShell
rm -r /product/priv-app/Velvet

mkfs.erofs -zlz4hc product-patched.img /product

umount /mnt/product

rm -r /mnt/product

rm -r /product

mkdir /mnt/odm

mount -t erofs -o loop odm.img /mnt/odm

cp -a /mnt/odm /

# Below commands only needed when chapter Android 15 applies to your case #

mkdir /mnt/system; mkdir /mnt/system_ext; mkdir /mnt/product; mount -t erofs -o loop system.img /mnt/system; mount -t erofs -o loop system_ext.img /mnt/system_ext; mount -t erofs -o loop product.img /mnt/product

cp /mnt/system/system/etc/selinux/plat_sepolicy_and_mapping.sha256 /odm/etc/selinux/precompiled_sepolicy.plat_sepolicy_and_mapping.sha256
cp /mnt/system_ext/etc/selinux/system_ext_sepolicy_and_mapping.sha256 /odm/etc/selinux/precompiled_sepolicy.system_ext_sepolicy_and_mapping.sha256
cp /mnt/product/etc/selinux/product_sepolicy_and_mapping.sha256 /odm/etc/selinux/precompiled_sepolicy.product_sepolicy_and_mapping.sha256

umount /mnt/product; umount /mnt/system_ext; umount /mnt/system; rm -r /mnt/product; rm -r /mnt/system_ext; rm -r /mnt/system

cp policy /odm/etc/selinux/precompiled_sepolicy

################

sepolicy-inject -Z shell -P /odm/etc/selinux/precompiled_sepolicy

mkfs.erofs -zlz4hc odm-patched.img /odm

umount /mnt/odm

rm -r /mnt/odm

rm -r /odm

lpmake --metadata-size 65536 --device-size $(du -b ../super-raw.img | awk '{print $1}') --metadata-slots 2 --group qti_dynamic_partitions:$(du -b system-patched.img odm-patched.img product-patched.img system_dlkm.img system_ext.img vendor.img vendor_dlkm.img | awk '{s+=$1} END {print s}') --partition system:none:$(du -b system-patched.img | awk '{print $1}'):qti_dynamic_partitions --partition odm:none:$(du -b odm-patched.img | awk '{print $1}'):qti_dynamic_partitions --partition product:none:$(du -b product-patched.img | awk '{print $1}'):qti_dynamic_partitions --partition system_dlkm:none:$(du -b system_dlkm.img | awk '{print $1}'):qti_dynamic_partitions --partition system_ext:none:$(du -b system_ext.img | awk '{print $1}'):qti_dynamic_partitions --partition vendor:none:$(du -b vendor.img | awk '{print $1}'):qti_dynamic_partitions --partition vendor_dlkm:none:$(du -b vendor_dlkm.img | awk '{print $1}'):qti_dynamic_partitions --image system=system-patched.img --image odm=odm-patched.img --image product=product-patched.img --image system_dlkm=system_dlkm.img --image system_ext=system_ext.img --image vendor=vendor.img --image vendor_dlkm=vendor_dlkm.img --sparse --output super-patched.img

lz4 -B6 --content-size super-patched.img super.img.lz4

rm super-patched.img

tar -cf super-patched.tar super.img.lz4

rm super.img.lz4
```

对于simg2img、mkfs.erofs和lz4程序，请使用APT软件包管理器获取。

对于imjtool程序，请从官方网站获取：

https://newandroidbook.com/tools/imjtool.html

对于lpmake程序，请访问谷歌Android CI官网，在"aosp_cf_x86_64_phone"标签下，选择一个"userdebug"版本，浏览其"Artifacts"，并找到"cvd-host_package.tar.gz"进行下载(您还需要导入压缩包中的so运行库文件到您的Debian操作系统)：

https://ci.android.com/builds/branches/aosp-main/grid?legacy=1

对于sepolicy-inject程序，请编译以下项目：

https://github.com/xmikos/setools-android

对于编辑SELinux规则的思想，参考文献(仅作为引用)：

https://android.stackexchange.com/questions/218911/how-to-add-selinux-policy-on-a-user-debug-rom-that-has-split-policy-scheme

对于三星Galaxy S24设备，必须压缩为"super.img.lz4"文件，且必须使用特定的lz4压缩指令，才可以使任何SUPER镜像在Odin移动端和PC端均被接受。参考文献(仅作为引用)：

https://github.com/lz4/lz4/issues/802

---

**06 写入修改的文件**

您需要将vbmeta-patched.tar、super-patched.tar使用Odin刷入设备。

您设备的售后(保修)权益、三星安全、NFC卡包等安全功能，将在写入修改后的vbmeta.img，并加载后，永久失效(KNOX熔断)。

如果您尚未决定放弃这些权益，请不要刷写修改后的vbmeta.img，这是您的最后选择机会。

只有设备成功启动并加载了修改后的vbmeta.img(或boot.img、recovery.img等)，才会触发KNOX熔断。只是BL解锁不会导致KNOX熔断。

---

**07 用户数据清除**

通过"三丧A226B"项目，我们知道，"BL锁状态变更后会格式化userdata"这件事情其实是由两部分构成：

ABL在调用平台解锁函数的同时，也会写标记通知Recovery清除用户数据。而TEE仅在环境状态与之前相符时，才会提供先前存储的userdata解密密钥。

因此，才会出现我们熟悉的BL解锁(三星手机是检测到修改后的镜像)会擦除用户数据的认知。

三星手机却很好地为我们展示了这极易被蒙蔽的本质，因为三星的ABL并不会主动通知Recovery清除用户数据。

例如，Galaxy S24在刷入修改的镜像后，并不会直接擦除用户数据，而是会正常启动，显示启动画面，然后画面闪断，黑屏，手机重新启动，并陷入这样的循环。

此时，您完全不必紧张，认为手机已被刷坏。其实，这只是TEE不提供userdata解密密钥，导致init无法进入下一阶段，内核崩溃的常规现象。

您只需要手动进入Recovery，进行工厂格式化，这样，TEE就会为我们创建一个新的密钥了，并且此密钥仅在当前环境状态下(使用我们修改后的镜像启动时)可用。

与Galaxy A22 5G不同的是，A22会在这种情况下，自动跳转至Recovery，提示您"userdata分区无法解密，要不试一下格式化？"，而S24并不会主动给出这样的提示。

---

**08 驻守进程初始化**

我们的RC服务将在开机启动时，依次一次性加载/sdcard/rc.local、/data/rc.local可执行文件。

在您的设备初始状态下，您需要通过Chrome浏览器或其他方式，将rc.local文件下载至/sdcard。

例如，我们可以编写/sdcard/rc.local来加载临时telnet服务：

```
#! /system/bin/sh

/system/bin/cp /storage/emulated/0/busybox /data/local/tmp/busybox
/system/bin/chmod +x /data/local/tmp/busybox
/data/local/tmp/busybox telnetd -F -l /system/bin/sh
```

您可以从此处获取一个busybox可执行文件：

https://github.com/meefik/busybox

当您获得一次性的telnet交互命令行后，您就可以部署自定义的驻守进程了。例如，您可以部署一个长期的telnet服务，/data/rc.local的内容会是：

```
#! /system/bin/sh

/data/system_properties ro.boot.verifiedbootstate green
/data/system_properties ro.boot.flash.locked 1
/data/system_properties ro.boot.warranty_bit 0
/data/system_properties ro.vendor.boot.warranty_bit 0

/data/busybox telnetd -F -l /data/login
```

其中，有关system_properties的事项将在下一章节叙述。

而login程序作为简易的telnet安全登录入口，已由"三丧A226B"项目介绍。

---

**09 安卓属性值**

一些安卓应用会使用getprop的简单方式来判断运行环境是否安全。它们通常会检查以下属性值：

```
getprop ro.boot.verifiedbootstate
getprop ro.boot.flash.locked
getprop ro.boot.warranty_bit
getprop ro.vendor.boot.warranty_bit
```

此时，我们需要恢复BL解锁前的状态值。在"三丧A226B"项目中，我们修改了init2，使setprop命令可以应用于任何属性，包括ro属性，进行概念验证。但在实践中，应用也可通过执行"setprop ro.*"来判定环境不安全。

安卓运行时属性值存放于"/dev/\_\_properties\_\_"目录。我们可以通过以下项目中的第三方可执行文件，对需要的属性值进行修改：

https://github.com/liwugang/android_properties

该项目编译的可执行文件假定了AREA_SIZE为0x20000，与高版本安卓实际不符。故需要跳过以下if条件运行：

```
if (fd_stat.st_size != AREA_SIZE) {
    print_log("file [%s] size is not equal %x\n", file_name, AREA_SIZE);
    close(fd);
    return NULL;
}
```

---

**10 去除开机警告**

搭载高通骁龙系列芯片的三星手机，无法去除开机警告。

搭载联发科、Exynos芯片的三星手机，可以通过编辑up_param分区来移除开机警告。

高通平台ABL展示图片存放在imagefv分区，并可通过以下项目提取图像文件：

https://github.com/LongSoft/UEFITool

但imagefv分区经过UEFI签名保护。

---

**11 已测试的应用**

BL解锁后，在此定制环境下：

- 微信、支付宝、淘宝APP登录和使用正常。

- 支付宝指纹付款正常、面部认证通过。

- 铁路12306指纹登录正常、购票和订餐正常、面部认证通过。

- Google Play商店指纹付款正常。

---

**12 UFS分区规划**

高通现已精简UFS闪存的LUN硬件分区规划，数量由6个LU减至4个LU(不含RPMB)，并全部使用GPT分区表。LBA大小为4096字节。

LU0 - HLOS数据硬件分区 (modem, boot, recovery, super, metadata, userdata等)

LU1 - BOOT1 (xbl, xbl_config等) [16 MB]

LU2 - BOOT2 (xbl_b, xbl_config_b等) [16 MB]

LU3 - 高通平台数据硬件分区 (abl, devcfg, tz, uefi等)

---

**13 可能的用途**

费了这么半天劲手搓出来的Linux root执行权限，我们可以用来做什么呢？一些创意有：

- 可以随意编辑"/data/data/"中的应用数据，修改游戏关卡进度。

- HTML5手机管理控制台。

- 一键隐藏特定的应用和数据，比如有好看的小姐姐要求加好友推销产品时，委婉地拒绝"我没有微信，没法扫码"。

- 制造APP不兼容的假象，比如推销小姐姐一定要你安装微信APP、并注册登录时，可以在后台循环根据包名结束进程。

- 在后台创建长连接tun隧道，配合路由表，即使在使用5G移动网络时，也可以在不使用安卓VPN功能(无VPN客户端、无VPN图标)的情况下，使用卢森堡的IP地址下载盗版电影，从而避免被DMCA。

- 设置照片自动备份，比如追星拍照后，保安大哥让删除，照片表面上删除了、也清空了回收站，可实际当按下快门的那一刻，早已自动备份到了指定目录。

- 紧急情况下，例如朋友聚餐不想请客结帐，一键擦除init_boot分区，快速销毁内核init程序，装作手机坏了、卡在开机界面。回家拿Odin再把init刷回去，手机又可以完美工作。

- 紧急情况下，例如老婆快要翻到婚外恋的证据，连按5次音量按钮，快速数据自毁(只要把metadata分区填零就可以了)，整机格式化。即使老婆请来了最顶尖的技术专家也无济于事，保住了家庭。

---

**14 关于版本号**

以我们工作的三星手机S9210ZHS4AXL4固件版本为例，在官方服务器中所展示的版本号为"S9210ZHS4AXL4/S9210OZS4AXL4/S9210ZCS4AXK6"：

https://fota-cloud-dn.ospserver.net/firmware/TGY/SM-S9210/version.xml

三个分隔的版本号分别为"BL和AP版本/CSC版本/CP(Modem)版本"。

固件版本号按照如下形式分割：S9210 - ZH - S - 4 - A - XL - 4。

- S9210 代表手机型号。

- ZH 代表地区，即中国香港版本固件。（此项若为ZC，则代表中国大陆版本固件。）

- S 代表更新类型，即安全补丁更新。（此项若为U，则代表功能性更新。）

- 4 代表Bootloader版本号。（将进行防降级保险丝熔断。）

- A 代表安卓版本，即Android 14。（此项若为B，则代表Android 15。）

- XL 代表固件构建时的年、月，即2024年12月。X代表2000年过后的第N个字母年，而L代表第N个字母月。

- 4 代表在此年月内的第N次构建。

版本号数字顺序为从1至9，然后从A至Z。

所以，S9210ZHS4AXL4版本表示：S9210机型的中国香港版本，安全补丁更新，Bootloader版本4，安卓版本14，2024年12月第4版。

# 安卓15

在2025年4月三星Galaxy S24设备的One UI 7.0 (Android 15)更新中，章节05中添加至precompiled_sepolicy文件的SELinux许可域不生效。这会导致我们的自定义RC服务无法被SELinux策略允许执行。

当然，无论何时，您都可以通过"三丧A226B"项目中所提及的，破坏内核函数avc_denied()的方式，使SELinux失能，从而允许任何进程不受MAC控制，包括我们的自定义服务进程。

但从生产环境角度考虑，安卓平台无法仅依靠DAC生存。我们依然需要探究precompiled_sepolicy无法生效的原因。

根据安卓官网说明，预编译的SELinux政策(precompiled_sepolicy)文件仅在三组SHA256哈希匹配后，才会被加载，否则会被即时重新编译：

https://source.android.com/docs/security/features/selinux/build

我们发现，三星Galaxy S24设备的Android 15固件中的三组SHA256竟不匹配，这就导致无论我们是否修改precompiled_sepolicy文件，其都不会在系统启动时被加载，而是会使用secilc即时编译SELinux政策文件。

为了解决这一问题，我们可以暂时破坏avc_denied()，在安卓15环境下获得root命令行执行权限后，提取即时编译并加载的SELinux政策文件"/sys/fs/selinux/policy"。然后，在重新打包SUPER分区时，与三个SHA256哈希文件一同覆盖到"/odm/etc/selinux/*"。

之所以需要让precompiled_sepolicy生效作为解决方案，是因为cil条目中不允许存在许可域，否则系统将阻止编译。

# 说人话

- 正向提权 --> 通过非漏洞方式获取root权限

- 自主完全控制 --> 用root权限为所欲为

- 版本文件 --> 固件包

- REE操作系统 --> 安卓系统

- HLOS --> 安卓系统

- 现代安卓手机 --> 系统版本不低于Android 10

- 主要移动工作环境 --> 日常使用的安卓主力机

# 你说的不对 / 我还有问题

提个Issue咯。
