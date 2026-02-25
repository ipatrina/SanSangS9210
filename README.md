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

mkdir /mnt/system; mount -t erofs -o loop system.img /mnt/system; cp -a /mnt/system /

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

rm -rf /system/system/app/SmartSwitchAgent
rm -rf /system/system/app/SmartSwitchStub
rm -rf /system/system/app/SamsungPassAutofill_v1
rm -rf /system/system/app/FBAppManager_NS
rm -rf /system/system/priv-app/GalaxyApps_OPEN
rm -rf /system/system/priv-app/SamsungMessages
rm -rf /system/system/priv-app/SmartSwitchAssistant
rm -rf /system/system/priv-app/SamsungPass
rm -rf /system/system/priv-app/OneDrive_Samsung_v3
rm -rf /system/system/priv-app/YourPhone_P1_5
rm -rf /system/system/priv-app/LinkToWindowsService
rm -rf /system/system/priv-app/FBInstaller_NS
rm -rf /system/system/priv-app/FBServices

mkfs.erofs -zlz4hc system-patched.img /system

umount /mnt/system; rm -rf /mnt/system; rm -rf /system

mkdir /mnt/product; mount -t erofs -o loop product.img /mnt/product; cp -a /mnt/product /

rm -rf /product/app/DuoStub
rm -rf /product/app/Gmail2
rm -rf /product/app/YouTube
rm -rf /product/app/Maps
rm -rf /product/app/AssistantShell
rm -rf /product/priv-app/Velvet

mkfs.erofs -zlz4hc product-patched.img /product

umount /mnt/product; rm -rf /mnt/product; rm -rf /product

mkdir /mnt/odm; mount -t erofs -o loop odm.img /mnt/odm; cp -a /mnt/odm /

# Below commands only needed when chapter Android 15 applies to your case #

mkdir /mnt/system; mkdir /mnt/system_ext; mkdir /mnt/product; mkdir /mnt/vendor; mount -t erofs -o loop system.img /mnt/system; mount -t erofs -o loop system_ext.img /mnt/system_ext; mount -t erofs -o loop product.img /mnt/product; mount -t erofs -o loop vendor.img /mnt/vendor

cp -f /mnt/system/system/etc/selinux/plat_sepolicy_and_mapping.sha256 /odm/etc/selinux/precompiled_sepolicy.plat_sepolicy_and_mapping.sha256
cp -f /mnt/system_ext/etc/selinux/system_ext_sepolicy_and_mapping.sha256 /odm/etc/selinux/precompiled_sepolicy.system_ext_sepolicy_and_mapping.sha256
cp -f /mnt/product/etc/selinux/product_sepolicy_and_mapping.sha256 /odm/etc/selinux/precompiled_sepolicy.product_sepolicy_and_mapping.sha256

secilc /mnt/system/system/etc/selinux/plat_sepolicy.cil -m -M true -G -N -c 30 /mnt/system/system/etc/selinux/mapping/34.0.cil -o sepolicy.XXXXXX -f /dev/null /mnt/system/system/etc/selinux/mapping/34.0.compat.cil /mnt/system_ext/etc/selinux/system_ext_sepolicy.cil /mnt/system_ext/etc/selinux/mapping/34.0.cil /mnt/system_ext/etc/selinux/mapping/34.0.compat.cil /mnt/product/etc/selinux/product_sepolicy.cil /mnt/product/etc/selinux/mapping/34.0.cil /mnt/vendor/etc/selinux/plat_pub_versioned.cil /mnt/vendor/etc/selinux/vendor_sepolicy.cil

umount /mnt/vendor; umount /mnt/product; umount /mnt/system_ext; umount /mnt/system; rm -rf /mnt/vendor; rm -rf /mnt/product; rm -rf /mnt/system_ext; rm -rf /mnt/system

cp -f sepolicy.XXXXXX /odm/etc/selinux/precompiled_sepolicy

################

sepolicy-inject -Z shell -P /odm/etc/selinux/precompiled_sepolicy

mkfs.erofs -zlz4hc odm-patched.img /odm

umount /mnt/odm; rm -rf /mnt/odm; rm -rf /odm

lpmake --metadata-size 65536 --device-size $(du -b ../super-raw.img | awk '{print $1}') --metadata-slots 2 --group qti_dynamic_partitions:$(du -b ../super-raw.img | awk '{print $1 - 4194304}') --partition system:none:$(du -b system-patched.img | awk '{print $1}'):qti_dynamic_partitions --partition odm:none:$(du -b odm-patched.img | awk '{print $1}'):qti_dynamic_partitions --partition product:none:$(du -b product-patched.img | awk '{print $1}'):qti_dynamic_partitions --partition system_dlkm:none:$(du -b system_dlkm.img | awk '{print $1}'):qti_dynamic_partitions --partition system_ext:none:$(du -b system_ext.img | awk '{print $1}'):qti_dynamic_partitions --partition vendor:none:$(du -b vendor.img | awk '{print $1}'):qti_dynamic_partitions --partition vendor_dlkm:none:$(du -b vendor_dlkm.img | awk '{print $1}'):qti_dynamic_partitions --image system=system-patched.img --image odm=odm-patched.img --image product=product-patched.img --image system_dlkm=system_dlkm.img --image system_ext=system_ext.img --image vendor=vendor.img --image vendor_dlkm=vendor_dlkm.img --sparse --output super-patched.img

lz4 -B6 --content-size super-patched.img super.img.lz4

rm -f super-patched.img

tar -cf super-patched.tar super.img.lz4

rm -f super.img.lz4
```

对于simg2img、mkfs.erofs、secilc和lz4程序，请使用APT软件包管理器获取。请同时安装gawk软件包，才可以使lpmake中的awk命令输出正确结果。

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

- 支付宝指纹付款正常、红包领取正常、面部认证通过。

- 铁路12306指纹登录正常、购票和订餐正常、面部认证通过。

- Google Play商店指纹付款正常。

- 《鱿鱼游戏：杀出重围》游戏正常。

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

- 紧急情况下，例如朋友聚餐不想请客结帐，连按5次侧面按钮，快速擦除init_boot分区，销毁内核init程序，装作手机坏了、卡在开机界面。回家拿Odin再把init刷回去，手机又可以完美工作。

- 紧急情况下，例如老婆快要翻到婚外恋的证据，连按5次侧面按钮，快速进行整机数据自毁。即使老婆请来了世界最顶尖的技术专家也无济于事，保住了家庭。

---

**14 关于版本号**

以我们工作的三星手机S9210ZHS4AXL4固件版本为例，在官方服务器中所展示的版本号为"S9210ZHS4AXL4/S9210OZS4AXL4/S9210ZCS4AXK6"：

https://fota-cloud-dn.ospserver.net/firmware/TGY/SM-S9210/version.xml

三个分隔的版本号分别为"BL和AP版本/CSC版本/CP(Modem)版本"。

固件版本号按照如下形式分割：S9210 - ZH - S - 4 - A - XL - 4。

- S9210 代表手机型号。

- ZH 代表地区，即中国香港版本固件。（此项若为ZC，则代表中国大陆版本固件。）

- S 代表更新类型，即安全补丁更新。（此项若为U，则代表功能性更新。）

- 4 代表Bootloader (ARB BIT)版本号。（将进行防降级保险丝熔断。）

- A 代表安卓版本，即Android 14。（此项若为B，则代表Android 15。）

- XL 代表固件构建时的年、月，即2024年12月。X代表2000年过后的第N个字母年，而L代表第N个字母月。

- 4 代表在此年月内的第N次构建。

版本号数字顺序为从1至9，然后从A至Z。

所以，S9210ZHS4AXL4版本表示：S9210机型的中国香港版本，安全补丁更新，Bootloader版本4，安卓版本14，2024年12月第4版。

# 说人话

- 正向提权 --> 通过非漏洞方式获取root权限

- 自主完全控制 --> 用root权限为所欲为

- 版本文件 --> 固件包

- REE操作系统 --> 安卓系统

- HLOS --> 安卓系统

- 现代安卓手机 --> 系统版本不低于Android 10

- 主要移动工作环境 --> 日常使用的安卓主力机

# 安卓15

**SELinux政策文件**

在2025年4月三星Galaxy S24设备的One UI 7.0 (Android 15)更新中，章节05中添加至precompiled_sepolicy文件的SELinux许可域不生效。这会导致我们的自定义RC服务无法被SELinux策略允许执行。

当然，无论何时，您都可以通过"三丧A226B"项目中所提及的，破坏内核函数avc_denied()的方式，使SELinux失能，从而允许任何进程(不包括init)不受MAC控制，包括我们的自定义服务进程。

但从生产环境角度考虑，安卓平台无法仅依靠DAC生存。我们依然需要探究precompiled_sepolicy无法生效的原因。

根据安卓官网说明，预编译的SELinux政策(precompiled_sepolicy)文件仅在三组SHA256哈希匹配后，才会被加载，否则会被即时重新编译：

https://source.android.com/docs/security/features/selinux/build

我们发现，三星Galaxy S24设备的Android 15固件中的三组SHA256哈希竟不匹配，这就导致无论我们是否修改precompiled_sepolicy文件，其都不会在系统启动时被加载，而是会使用secilc命令即时编译SELinux政策文件。

为了解决这一问题，我们可以暂时破坏avc_denied()，在安卓15环境下获得root命令行执行权限后，提取即时编译并加载的SELinux政策文件"/sys/fs/selinux/policy"。

但更加便捷的做法是，参考安卓源代码中secilc的命令用法，直接在Debian操作系统中使用secilc命令编译SELinux政策文件。"vend_plat_vers"版本号可通过"/vendor/etc/selinux/plat_sepolicy_vers.txt"文件获取。

https://android.googlesource.com/platform/system/core/+/master/init/selinux.cpp

最后，在重新打包SUPER分区时，与三个SHA256哈希文件一同覆盖到"/odm/etc/selinux/*"。

之所以需要让precompiled_sepolicy生效作为解决方案，是因为cil条目中不允许存在许可域，否则系统将阻止编译。

# 安卓16

**OEM解锁**

因不明原因，自One UI 8.0起，三星不再允许OEM解锁。

请保持您的设备使用One UI 6 / 7。

**升级**

如您有意使用One UI 8，在未解锁的设备上，请先降级至One UI 7。

在已解锁并正常使用的设备上，请严格按照以下步骤操作，您的数据不会丢失：

1. 必须确定您的OUI7固件包的防降级熔断版本号(ARB BIT)与您的OUI8固件包的防降级熔断版本号(ARB BIT)完全一致。若不一致，您不能刷写OUI8，请立即终止操作。

例如：您可以使用"S9210ZHS**4**BYDF"与"S9210ZHS**4**CYJ7"版本文件工作，因为它们的ARB BIT都是"4"。

2. 提取OUI7 BL包中的abl.elf.lz4。此步骤后，您不再需要使用OUI7固件包。

3. 解压OUI8 BL包。

4. 在OUI8 BL解压目录中，使用lz4解压vbmeta.img.lz4。
  
5. 在OUI8 BL解压目录中，按照章节04的描述修改上一步骤解压的vbmeta.img。

6. 在OUI8 BL解压目录中，删除vbmeta.img.lz4。

7. 在OUI8 BL解压目录中，删除abl.elf.lz4。

8. 将第二步提取的OUI7 BL包中的abl.elf.lz4，放置于OUI8 BL解压目录。

此时，您的OUI8 BL解压目录中，包含了两个被替换的文件：abl.elf.lz4 (来自OUI7) 和 vbmeta.img (不需要重新压缩为lz4)。

9. 使用tar命令将OUI8 BL解压目录重新打包为tar文件。

```
tar -cf BL-patched.tar --exclude=BL-patched.tar *
```

10. 使用Odin选择以下文件组合，刷入设备。(对于AP、CP、CSC，请选择您的实际版本文件。CSC文件必须以"HOME_CSC_"开头，否则会丢失用户数据。)

[BL] BL-patched.tar

[AP] AP_S9210ZHS4CYJ7_S9210ZHS4CYJ7_MQB102654672_REV00_user_low_ship_MULTI_CERT_meta_OS16.tar.md5

[CP] CP_S9210ZCS4BYDF_CP30183922_MQB96147879_REV00_user_low_ship_MULTI_CERT.tar.md5

[CSC] HOME_CSC_OZS_S9210OZS4BYDF_MQB96147879_REV00_user_low_ship_MULTI_CERT.tar.md5

刷写完成后，您可以正常启动进入One UI 8系统。

11. 根据章节05的描述，修改OUI8 AP包中SUPER分区。然后重新进入Odin下载模式，刷写修改后SUPER分区。

[AP] super-patched.tar

# 权限用例

**21 录音通知**

One UI 7.0针对中国、美国等地机型增加了原生通话录音功能，但会向通话方强制播放录音通知。

- One UI 7 (Android 15)

您可以在"/data/user_de/0/com.samsung.android.incallui/shared_prefs/com.samsung.android.incallui_preferences.xml"文件中增加以下两个选项，方可激活"测试模式"中的"传统录音方式"，以禁用录音通知功能：

```
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <boolean name="incallui_test_mode" value="true" />
    <boolean name="record_call_original" value="true" />
</map>
```

- One UI 8 (Android 16)

新版"电话"应用程序通过全局设置项决定是否使用带有录音通知的交互界面。

"电话"应用程序在每次初始化时，均会重新读取CSC设置，并将"settings global call_recording_ui_type"重新设置为"1"(中国机型CSC配置)。

而每次拨号后，仅当程序获取"call_recording_ui_type"的值为"0"时，才会选择进入"传统录音方式"界面。

故，若要禁用录音通知功能，您必须循环检测该值是否被设置为"1"，并立即更改为"0"。(每当重新启动"电话"APP时会被一次性重置为"1"，而如果您能立即将其置"0"，则在拨号后APP就会实时使用"0"值来展示界面。)

相关自动化脚本的实现已合并至下一章节。

除此之外，在"call_recording_ui_type"的值为"0"时，可以实现通话自动录制。

```
settings put system record_calls_automatically_on_off 1
```

"call_recording_ui_type"为"1"时，该值("record_calls_automatically_on_off")无影响。"record_calls_automatically_on_off"默认值为"0"，该值不会被自动覆盖。

---

**22 应急自毁**

为应对不可抗力的监管，以下"/data/emerg.sh"脚本将监听"按下侧面按钮5次"事件，并立即擦除init1及重置，使手机停留在启动界面，无法正常使用。

![Splash preview](https://thumbs2.imgbox.com/04/c4/xTHfH8wL_t.png)

仅擦除init1是无损操作，不会造成数据丢失。您只需使用Odin将init_boot分区恢复，即可如初使用手机。

若您需要更高的安全级别，您可以使能"--wipe_data"，彻底销毁手机数据。

```
#! /system/bin/sh

while true; do
  if [[ $(getprop debug.tracing.screen_state) != "1" ]]; then
    # Put Chapter 33 code here if needed

    if [[ $(settings get secure emergency_state_machine_state) == "1" ]]; then
      settings put secure emergency_state_machine_state 0
      echo "Emerg event triggered! Responding."
      touch /mnt/emerg.flag
      #dd if=/dev/zero of=/dev/block/by-name/init_boot bs=1048576
      #dd if=/dev/zero of=/dev/block/by-name/metadata bs=1048576
      mkdir /cache/recovery
      echo "--wipe_cache" > /cache/recovery/command
      #echo "--wipe_data" > /cache/recovery/command
      sync
      settings put secure emergency_state_machine_state 0
      reboot recovery
      break
    fi

    if [[ $(settings get global call_recording_ui_type) == "1" ]]; then
      settings put global call_recording_ui_type 0
    fi
  fi
  sleep 1
done
```

![Emergency preview](https://thumbs2.imgbox.com/46/8e/L5HGyLtO_t.jpg)

您可以在"/data/user_de/0/com.samsung.android.emergency/shared_prefs/com.samsung.android.emergency_pref.xml"文件中自定义显示的紧急号码：

```
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="emergency_selected_number">设备立即自毁</string>
    <string name="emergency_custom_number">设备立即自毁</string>
</map>
```

---

**23 查找我的女朋友**

当您需要查找女朋友的实时位置，可以使用curl将"dumpsys location | grep location=Location"的坐标缓存信息在其手机上通过自定义HTTP API循环发送。

坐标缓存仅在安卓APP使用位置信息时才会更新。因此，我们需要使用安卓APP在后台不断刷新位置信息。请参阅以下开源项目：

https://github.com/barbeau/gpstest

该APP可在通知栏进行长期工作，实时刷新位置数据。您可以通过shell命令来隐藏和取消隐藏该通知(不会影响APP工作)：

```
cmd notification snooze --for 99999999 "0|com.android.gpstest|12345678|null|10308"

cmd notification unsnooze "0|com.android.gpstest|12345678|null|10308"
```

实时刷新位置数据将增加手机耗电。因此，您可以通过脚本控制该服务的启停：

```
am start-foreground-service -n com.android.gpstest/.ForegroundOnlyLocationService

am stopservice -n com.android.gpstest/.ForegroundOnlyLocationService
```

您的配偶可通过系统"最近使用"界面(多任务管理键)，自行决定是否提供实时位置更新数据。

---

**24 短信息拉取**

当您使用"Google 信息"APP作为默认的短信收发应用时，您可以通过以下命令查询消息数据库，将最近的50条短信息输出为CSV格式：

```
/data/sqlite3 -csv /data/data/com.google.android.apps.messaging/databases/bugle_db "SELECT TABLE2.name, TABLE1.text, TABLE1.timestamp FROM parts AS TABLE1 JOIN conversations AS TABLE2 ON TABLE1.conversation_id = TABLE2._id ORDER BY TABLE1.timestamp DESC LIMIT 50"
```

您可以从此处获取一个sqlite3可执行文件：

https://xdaforums.com/t/new-sqlite3-cli-binary-v3-50-1-for-all-devices.4273049

---

**25 玩玩小游戏**

- 单机游戏《死亡独轮车》(Happy Wheels)可通过编辑"/data/data/com.fancyforce.happywheels/shared_prefs/Cocos2dxPrefsFile.xml"进行移除广告和关卡解锁。

```
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <boolean name="remove_ads" value="true" />
    <string name="c6_l0">600</string>
</map>
```

添加"remove_ads"选项可以永久移除游戏中厌人的广告。

添加"cX_lY"选项，值为"Z"，可进行关卡解锁。其中X表示章节号(0-6)，Y表示关卡号(0-14)，Z表示通关用时(推荐使用600秒，不会影响正常的游戏体验)。重复添加此选项即可解锁多个关卡。多次通关时间使用冒号分隔，例如"300.25:600:900.81"。

- 联机游戏《狂野飙车8》和《狂野飙车：传奇集结》(狂野飙车9)，在云端保存用户数据。

使用社交帐户登录《狂野飙车》存在后续需要注销社交账户的可能。

直接备份本地用户数据目录"/data/data/com.gameloft.android.ANMP.GloftA8HM"(或A9)，并在新设备上恢复"databases"、"files"、"shared_prefs"文件夹，即可完成云端账户迁移。

- 奥飞特七(Outfit7)"会说话的朋友"系列互动应用，可通过编辑XML和SQLite数据库文件，增加猫币或道具数量、解锁动画并移除广告。

《会说话的汤姆猫》(2.0)
```
sed -i 's/\&quot;:5/\&quot;:2000/g' /data/data/com.outfit7.talkingtom/shared_prefs/com.outfit7.enterprise.persistence.xml
sed -i 's/\&quot;:3/\&quot;:2000/g' /data/data/com.outfit7.talkingtom/shared_prefs/com.outfit7.enterprise.persistence.xml

sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.talkingtom/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.talkingtom/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.talkingtom/shared_prefs/FelisBillingCore.xml
```

《会说话的汤姆猫》(1.0)
```
sed -i 's|</map>|    <boolean name="unlimited" value="true" \/>\
</map>|g' /data/data/com.outfit7.talkingtom/shared_prefs/prefs.xml
```

《会说话的狗狗本》
```
/data/sqlite3 /data/data/com.outfit7.talkingben/databases/tube.db "UPDATE state SET value = '{\"number\":2000}';"

sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.talkingben/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.talkingben/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.talkingben/shared_prefs/FelisBillingCore.xml
```

《会说话的新闻》
```
sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.talkingnewsfree/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.talkingnewsfree/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.talkingnewsfree/shared_prefs/FelisBillingCore.xml
```

《会说话的汤姆猫2》
```
/data/sqlite3 /data/data/com.outfit7.talkingtom2free/databases/vca.db "UPDATE state SET value = '{\"balance\":200000}';"

sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.talkingtom2free/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.talkingtom2free/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.talkingtom2free/shared_prefs/FelisBillingCore.xml
```

《会说话的鹦鹉》
```
sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.talkingpierrefree/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.talkingpierrefree/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.talkingpierrefree/shared_prefs/FelisBillingCore.xml
```

《会说话的安吉拉》
```
/data/sqlite3 /data/data/com.outfit7.talkingangelafree/databases/vca.db "UPDATE state SET value = '{\"balance\":200000}';"

sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.talkingangelafree/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.talkingangelafree/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.talkingangelafree/shared_prefs/FelisBillingCore.xml
```

《会说话的金杰猫》
```
/data/sqlite3 /data/data/com.outfit7.talkinggingerfree/databases/toothpaste.db "UPDATE state SET value = '{\"number\":2000}';"

sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.talkinggingerfree/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.talkinggingerfree/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.talkinggingerfree/shared_prefs/FelisBillingCore.xml
```

《会说话的金杰猫2》
```
/data/sqlite3 /data/data/com.outfit7.gingersbirthdayfree/databases/food.db "UPDATE state SET value = '{\"number\":2000}';"

sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.gingersbirthdayfree/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.gingersbirthdayfree/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.gingersbirthdayfree/shared_prefs/FelisBillingCore.xml
```

《我的汤姆猫》
```
sed -i 's/\&quot;:100000,/\&quot;:200000,/g' /data/data/com.outfit7.mytalkingtomfree/shared_prefs/com.outfit7.enterprise.persistence.xml
sed -i 's/\&quot;:10000,/\&quot;:200000,/g' /data/data/com.outfit7.mytalkingtomfree/shared_prefs/com.outfit7.enterprise.persistence.xml

sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.mytalkingtomfree/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.mytalkingtomfree/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.mytalkingtomfree/shared_prefs/FelisBillingCore.xml
```

《我的汤姆猫2》
```
sed -i 's/\&quot;:100000,/\&quot;:200000,/g' /data/data/com.outfit7.mytalkingtom2/shared_prefs/com.outfit7.starlite.mytalkingtom2.xml

sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.mytalkingtom2/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.mytalkingtom2/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.mytalkingtom2/shared_prefs/FelisBillingCore.xml
```

《我的安吉拉》
```
sed -i 's/\&quot;100000/\&quot;200000/g' /data/data/com.outfit7.mytalkingangelafree/shared_prefs/com.outfit7.mytalkingangela.game.xml
sed -i 's/\&quot;10000/\&quot;200000/g' /data/data/com.outfit7.mytalkingangelafree/shared_prefs/com.outfit7.mytalkingangela.game.xml

sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.mytalkingangelafree/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.mytalkingangelafree/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.mytalkingangelafree/shared_prefs/FelisBillingCore.xml
```

《我的安吉拉2》
```
sed -i 's/\&quot;:100000,/\&quot;:200000,/g' /data/data/com.outfit7.mytalkingangela2/shared_prefs/com.outfit7.starlite.mytalkingangela2.user.xml

sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.mytalkingangela2/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.mytalkingangela2/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.mytalkingangela2/shared_prefs/FelisBillingCore.xml
```

《汤姆猫总动员》
```
sed -i 's/\&quot;:300,/\&quot;:200000,/g' /data/data/com.outfit7.mytalkingtomfriends/shared_prefs/com.outfit7.starlite.mytalkingtomfriends.user.xml

sed -i 's/<boolean name="PaidUser.isPaidUser" value="false" \/>/<boolean name="PaidUser.isPaidUser" value="true" \/>/' /data/data/com.outfit7.mytalkingtomfriends/shared_prefs/FelisBillingCore.xml
sed -i 's/<boolean name="PaidUser.ignoreConfigUpdate" value="false" \/>/<boolean name="PaidUser.ignoreConfigUpdate" value="true" \/>/' /data/data/com.outfit7.mytalkingtomfriends/shared_prefs/FelisBillingCore.xml

chattr +i /data/data/com.outfit7.mytalkingtomfriends/shared_prefs/FelisBillingCore.xml
```

《我的汉克狗》
```
# 这只太丑了，不搞。
```

请将猫币、钻石数量替换为实际原始数量。

sqlite3可执行文件可通过章节24获取。

应用会每日重置FelisBillingCore.xml，因此需要通过chattr命令锁定文件。

---

**26 网络抓包**

您可以从此处获取一个tcpdump可执行文件进行网络数据抓包：

https://github.com/S-trace/tcpdump_static_aarch64

---

**27 WLAN MAC地址**

- 设备MAC地址

WLAN配置存储于以下文件：

```
/data/misc/apexdata/com.android.wifi/WifiConfigStore.xml

/data/misc/apexdata/com.android.wifi/WifiConfigStoreSoftAp.xml
```

修改"WifiConfigStore.xml"文件中的以下选项可在安卓系统层面自定义设备MAC地址：

```
sed -i 's#^<string name="wifi_sta_factory_mac_address">.*</string>#<string name="wifi_sta_factory_mac_address">aa:bb:cc:dd:ee:ff</string>#' /data/misc/apexdata/com.android.wifi/WifiConfigStore.xml
```

修改后，必须使用以下命令生效：

```
pkill system_server
```

参考文献(仅作为引用)：

https://xdaforums.com/t/guide-how-to-configure-the-wifi-in-android-via-script.4548489

笔者不很清楚该做法的生效原理。但其实这有关"WifiConfigStore.xml"参数的加载机制。

当系统启动时，system_server从"WifiConfigStore.xml"加载参数。当用户普通关机时，system_server会将已加载至内存的参数写回"WifiConfigStore.xml"。

也就是说，当用户手动编辑参数后，必须避免system_server重写"WifiConfigStore.xml"的过程。

使用"echo b > /proc/sysrq-trigger"方式或物理按键强制重置设备，可避免system_server的重写过程，但同样可能导致磁盘缓存未同步，造成XML文件损坏。因此，强制重置设备前，需要执行"sync"。

使用"reboot"命令也可避免system_server的重写过程，且这种生效方式相对和平。

一旦XML文件损坏，system_server会重新从WLAN驱动读取设备MAC地址，否则采用XML中存储的WLAN MAC地址。

- 随机MAC地址

若要在每次忘记网络后重新生成随机MAC地址，请使用以下命令配置WLAN非持久随机分配MAC地址功能：

```
settings put global non_persistent_mac_randomization_force_enabled 1
```

---

**28 触摸屏去使能**

为制造手机故障的假象，您可以安全、暂时地使触控屏幕完全失灵。

```
echo -n "spi1.0" > /sys/bus/spi/drivers/stm_ts_spi/unbind
```

该命令可将SPI总线上的设备"spi1.0"从"stm_ts_spi"驱动中解绑，从而停止内核驱动对触控设备的控制，属于无损级别操作。

重新启动手机可使设备功能完全恢复。

若您希望了解如何查找正确的触控设备与驱动名称，请遵循以下步骤：

1. 使用"getevent -l"命令定位触摸屏幕设备编号。

```
add device 7: /dev/input/event7
  name:     "sec_touchscreen"
```

2. 使用"readlink /sys/class/input/input7/device"命令获取设备名称。

```
../../../spi1.0
```

3. 使用"readlink /sys/class/input/input7/device/driver"命令获取驱动名称。

```
../../../../../../../../bus/spi/drivers/stm_ts_spi
```

---

**29 消耗移动数据使用量**

在手机正常连接并使用Wi-Fi网络的情况下，您依然可以通过命令行消耗4G/5G移动数据流量。这适用于移动网络套餐流量过多、无法用尽的情况。示例命令如下：

```
curl --interface rmnet_data7 -o /dev/null https://wa-us-ping.vultr.com/vultr.com.1000MB.bin
```

您需要在开发者选项中使能"始终开启移动数据网络"选项，否则系统不会创建相应的rmnet_data网络接口。

---

**30 去使能敏感信息窗口保护**

一些安卓应用通过设置"FLAG_SECURE"窗口标志，来禁止屏幕截图和录制。

https://developer.android.com/security/fraud-prevention/activities

通过使能开发调试开关，可以避免该窗口标志生效。

```
/data/system_properties ro.debuggable 1
settings put secure disable_secure_windows 1
/data/system_properties ro.debuggable 0
settings delete secure disable_secure_windows
```

仅当"ro.debuggable"为"1"时，"disable_secure_windows"才会生效。而消费者操作系统的"ro.debuggable"值一定为"0"。因此，执行上述命令依赖章节09的过程。

执行上述命令后，系统将在内存中标记去使能敏感信息窗口保护，本次开机期间有效。

已测试Chrome浏览器无痕模式、支付宝付款码屏幕截图和录制有效。

(这里是研究人员提供的一些非正式研究记录：https://pastebin.com/n3UsieBE)

---

**31 屏幕亮度**

当安卓应用控制并锁定屏幕亮度时，您可以直接操作内核驱动来强制调整。您还可以获取驱动最大屏幕亮度。

```
cat /sys/class/backlight/panel0-backlight/brightness

cat /sys/class/backlight/panel0-backlight/max_brightness

echo 100 > /sys/class/backlight/panel0-backlight/brightness
```

---

**32 数字健康**

手机会为用户统计设备使用量数据。除了正常使用GUI操作禁用统计和重置数据外，也可直接通过命令来浏览或删除这些存储文件。

```
ls -l /data/system_ce/0/usagestats
ls -l /data/system_ce/0/usagestats/daily
ls -l /data/system_ce/0/usagestats/weekly
ls -l /data/system_ce/0/usagestats/monthly
ls -l /data/system_ce/0/usagestats/yearly
```

```
rm /data/system_ce/0/usagestats/daily/*
rm /data/system_ce/0/usagestats/weekly/*
rm /data/system_ce/0/usagestats/monthly/*
rm /data/system_ce/0/usagestats/yearly/*
rm /data/system_ce/0/usagestats/mappings
rm /data/system_ce/0/usagestats/migrated
rm /data/system_ce/0/usagestats/version
```

请务必注意，不要删除任何文件夹。

这些存储文件同样由system_server管理，因此，生效原理请参考章节27。

通过移除"数字健康"APP(com.samsung.android.forest)，可以使"设置"APP中无法查看使用量数据。

另外，可以通过以下命令重置网络流量使用量。

```
rm /data/misc/apexdata/com.android.tethering/netstats/*
```

---

**33 USB反制**

如果您希望设备在无锁屏状态下，接入USB数据线缆(不包括充电器)或“USB调试”选项被启用时，立即视为设备已被入侵，并触发章节22的自毁流程，您可以在以上脚本中增加这些代码：

```
    if [[ $(getprop sys.locksecured) == "false" ]]; then
      if [[ "$(cat /sys/class/android_usb/android0/state)" == "CONNECTED" || "$(getprop sys.usb.config)" == *adb* ]]; then
      	setprop persist.sys.usb.config sec_charging
        am start -a android.intent.action.VIEW -d "file:///data/invaded.html" -t "text/html"
        cmd uimode night yes
        settings delete secure ui_night_mode
        echo -n "spi1.0" > /sys/bus/spi/drivers/stm_ts_spi/unbind
        echo -n "c42d000.qcom,spmi:qcom,pmk8550@0:pon_hlos@1300:pwrkey" > /sys/bus/platform/drivers/pm8941-pwrkey/unbind
        sleep 8
        settings put secure emergency_state_machine_state 1
      fi
    fi
```

脚本将通过卸载内核驱动的方式，禁用触摸屏幕操作及电源按键，并展示用户预设的HTML警告页面。这不会影响“电源+音量下”的按键强制重置方式，原因是，这是SoC物理中断。

当WebView加载本地HTML文件时，必须为文件赋予相应权限：

```
chmod 777 /data/invaded.html
chcon u:object_r:shell_data_file:s0 /data/invaded.html
```

# 你说的不对 / 我还有问题

提个Issue咯。
