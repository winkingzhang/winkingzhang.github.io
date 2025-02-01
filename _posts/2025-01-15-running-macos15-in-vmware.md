---
title:  "在 VMware 里运行 macOS Sequoia"
header:
  teaser: "/assets/images/teaser-2.jpg"
categories:
  - VMware
tags:
  - macOS Sequoia
  - VMware
excerpt: >
    本文介绍在 Windows 11 下使用 VMware 简单、稳定且高效的安装 macOS Sequoia 方法。
---

## 前言

自从 Broadcom 收购 VMware 之后，一再修改 VMware Workstation(Fusion) 的授权协议，
终于在 [24 年 11 月份宣布所有人免费](https://blogs.vmware.com/cloud-foundation/2024/11/11/vmware-fusion-and-workstation-are-now-free-for-all-users/) 使用了，
虽然长远上不知道此举的是非功过，但大家确实能切身体会并得到了一些利益，姑且信了，心存感激。

话题回到本文，在 VMware 里运行 macOS Sequoia，这需要一些前提，下文会详细展开。

- 可以启动的 MacOS ISO 镜像
- 已经解锁的 WMware Workstation

> **注意**
> 
> 虽然最新版 VMware 已经支持了 MacOS 15 的版本，但 MacOS 15 会自动检测是否在虚拟机运行
> 而且本机毕竟不是 arm64 CPU，稳妥起见还是建议安装 MacOS 14.x，然后升级到 MacOS 15。
> 

## 制作 MacOS ISO 启动盘

> **Tips**
>
> 这一步可能会劝退很多人，因为需要运行在 Mac 环境下。
> 好在，万能的网络上有人已经准备好了，在确认安全的情况下，拿来主义也是不错的。
> 
> 为了避免广告嫌疑，大家自行网络找寻，或者翻一下这里：[macOS 下载汇总 (系统、应用和教程)](https://sysin.org/blog/macOS/)

好在博主自己有一个 macOS 环境，所以还是自己动手，探究一下其中的细节。

首先需要下载 `gibmacos` 这个工具，简单的就是从 github 拉取下来，然后本地运行就行。

```shell
$ git clone https://github.com/corpnewt/gibMacOS
```

然后切换到 repo 目录，执行 `gibMacOS.command`，应该能看到类似下面的输出，这里能看到当前能从 Apple 站点下载到的各种系统镜像。

```shell
$ ./gibMacOS.command

#######################################################
#                      gibMacOS                       #
#######################################################

Available Products:

1. macOS Sequoia 15.0.1 (24A348)
   - 072-01382 - Added 2024-10-03 21:26:40 - 14.48 GB
2. macOS Ventura 13.7 (22H123)
   - 062-78643 - Added 2024-09-16 17:44:05 - 12.22 GB
3. macOS Sonoma 14.7 (23H124)
   - 062-78824 - Added 2024-09-16 17:42:25 - 13.68 GB
...
25. macOS Mojave 10.14.6 (18G103)
   - 061-26589 - Added 2019-10-14 20:51:08 - 6.52 GB
26. macOS Mojave 10.14.5 (18F2059)
   - 061-26578 - Added 2019-10-14 20:38:26 - 6.52 GB

M. Change Max-OS Version (Currently 12)
C. Change Catalog (Currently publicrelease)
I. Only Print URLs (Currently Off)
S. Set Current Catalog to SoftwareUpdate Catalog
L. Clear SoftwareUpdate Catalog
R. Toggle Recovery-Only (Currently Off)
U. Show Catalog URL
Q. Quit

Please select an option: 3

Downloading InstallAssistant.pkg for 062-78824 - 14.7 macOS Sonoma (23H124)...

1.35 GB/14.48 GB | =                    9.34% | 101.7 MB/s | 00:02:10 left

Succeeded:
  InstallAssistant.pkg
  MajorOSInfo.pkg
  com_apple_MobileAsset_MacSoftwareUpdate.plist
  InstallInfo.plist
  UpdateBrain.zip

Failed:
  None

Files saved to:
  /Users/wenqing/works/github/corpnewt/gibMacOS/macOS Downloads/publicrelease/062-78824 - 14.7 macOS Sonoma (23H124)

```

等待下载完成，就可以到上面程序的下载目录里（上面示例第 97 行类似路径），
找到 `InstallAssistant.pkg`，并执行它，后续制作启动盘就是利用这个包安装的工具软件。

首先使用 `hdiutil` 来创建一个不小于 16GB 的磁盘镜像（这是一个 DMG 文件）。
这里为了方便，指定输出路径在 `/tmp` 下，同时避免其他目录的权限问题：
```shell
$ hdiutil create -o /tmp/MacOS -size 16000m -volname MacOS -layout SPUD -fs HFS+J
created: /tmp/MacOS.dmg
```

接下来，挂载刚刚创建出来的镜像（注意，这里实际看到的磁盘名，即左侧的 `/dev/disk6`，会不太一样，好在后续只会关注 `mountpoint`）：
```shell
$ hdiutil attach /tmp/MacOS.dmg -noverify -mountpoint /Volumes/MacOSISO
/dev/disk6          	Apple_partition_scheme         	
/dev/disk6s1        	Apple_partition_map            	
/dev/disk6s2        	Apple_HFS                      	/Volumes/MacOSISO
```

然后就可以利用 在 `Install masOS app` 里的 `createinstallmedia` 应用来创建 macOS 启动盘了，
需要注意的是，这里的要指定输出 volume 是上面挂载的逻辑路径（即 `/Volumes/MacOSISO`）：
```shell
$ sudo /Applications/Install\ macOS\ Sonoma.app/Contents/Resources/createinstallmedia \
    --volume /Volumes/MacOSISO --nointeraction
Erasing disk: 0%... 10%... 20%... 30%... 100%
Copying essential files...
Copying the macOS RecoveryOS...
Making disk bootable...
Copying to disk: 0%... 10%... 20%... 30%... 40%... 50%... 60%... 100%
Install media now available at "/Volumes/Install macOS Sonoma"
```

接下来还需要按部就班的移除磁盘挂载，将 DMG 文件装换成 ISO 格式的镜像 （当然少不了扫尾和清除垃圾工作）：
```shell
$ hdiutil detach -force /Volumes/Install\ macOS\ Sonoma
"disk6" ejected.

$ ls -al /tmp/MacOS.dmg
-rw-r--r--@ 1 wenqing  wheel  16777216000 15 Oct 21:54 /tmp/MacOS.dmg

$ hdiutil convert /tmp/MacOS.dmg -format UDTO -o /tmp/MacOS-Sonoma-14.7.cdr
Reading Driver Descriptor Map (DDM : 0)…
Reading Apple (Apple_partition_map : 1)…
Reading  (Apple_Free : 2)…
Reading disk image (Apple_HFS : 3)…
........................................
Elapsed Time: 19.139s
Speed: 835.9MB/s
Savings: 0.0%
created: /tmp/MacOS-Sonoma-14.7.cdr

$ mv /tmp/MacOS-Sonoma-14.7.cdr /tmp/MacOS-Sonoma-14.7.iso

$ rm /tmp/MacOS.dmg
```

至此，就可以备份保存这个 iso 文件，方便后续使用。

## 安装 VMware Workstation 并解锁 `Apple macOS`

如前文所说，Broadcom 收购了 VMware，随后进行了资源整合，这导致网络上很多的下载链接都失效了，
因为绝大部分的 `vmware.com` 地址都会跳转到 `broadcom.com` 首页，
然后提示需要注册，bla bla 一堆事，再之后就迷失在一堆链接里，找不到 WMware 的下载页面，这再一次劝退了很多朋友。

经过一些尝试，大家可以从这个链接下载到当前最新的 VMware Workstation Pro v17.6.2：

[https://support.broadcom.com/group/ecx/productfiles?subFamily=VMware%20Workstation%20Pro&displayGroup=VMware%20Workstation%20Pro%2017.0%20for%20Windows&release=17.6.2&os=&servicePk=526672&language=EN](https://support.broadcom.com/group/ecx/productfiles?subFamily=VMware%20Workstation%20Pro&displayGroup=VMware%20Workstation%20Pro%2017.0%20for%20Windows&release=17.6.2&os=&servicePk=526672&language=EN)

当然，这里还是要注册的，好在是免费的，就是填的资料有点多，忍耐一下就好。
如果注册之后没有跳转到下载页面，那么把上面地址贴浏览器，再试一下就能看到下载页面了。

> **Tips**
> 
> 如果你需要其他版本的 VMware Workstation Pro，那么从这个列表地址进入，可以选择 Window/Linux，或者较旧一点的版本：
> [https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Workstation%20Pro](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Workstation%20Pro)

下载之后安装就容易了，看着屏幕一路 next 就行，遇到输入 License 的时候选择个人跳过。

#### 解锁 `Apple macOS`

有一个开源项目实现了对 WMware v15/16/17 所有版本的 patch，可以解锁 `Apple macOS` 选项，需要做的就是克隆这个 repo：
```powershell
PS C:\Work\github\paolo-projects> git clone https://github.com/paolo-projects/unlocker
```

然后到 repo 目录执行 `win-install.cmd`：
```powershell
PS C:\Work\github\paolo-projects\unlocker> .\win-install.cmd
Unlocker 3.0.2 for VMware Workstation
=====================================
(c) Dave Parsons 2011-18

Set encoding parameters...
Active code page: 850

VMware is installed at: C:\Program Files (x86)\VMware\VMware Workstation\
VMware product version: 17.6.2.24409262

Stopping VMware services...

...

Starting VMware services...

Finished!
```

解锁成功之后，可以启动 VMware Workstation，新建一个空的虚拟机，在向导页面里可以看到 `Apple macOS` 选项。

![VMware - Apple macOS](/assets/images/run_macos15_in_vmware/VMware_Apple_macOS.png)

#### 导入 `VMware Tools`

`unlocker` 工具自己会尝试从 softwareupdate.vmware.com 网站上面下载 fusion 的安装包，并自动提取两个 iso 文件，
分别叫 `darwin.iso` 和 `darwinPre15.iso`， 成功的话会自动将他们拷贝到 VMware 的安装目录下。

但随着 VMware Fusion 13.6 系列发布，安装之后就不再带有这两个镜像文件，因此需要自己手动去下载旧版本的。
比如下载 13.5.2，地址是：[https://softwareupdate.vmware.com/cds/vmw-desktop/fusion/13.5.2/23775688/universal/core/com.vmware.fusion.zip.tar](https://softwareupdate.vmware.com/cds/vmw-desktop/fusion/13.5.2/23775688/universal/core/com.vmware.fusion.zip.tar)

下载下来的是一个 `tar` 文件，使用 `WinRAR`/`WinZip`/`7Zip` 都可以解压，里面还是一个 `zip` 压缩文件，继续解压。
然后在目录 `payload/VMware Fusion.app/Contents/Library/isoimages/x86_x64` 下可以看到上面提到的两个 iso 文件。
选中这两个 iso，复制到 VMware 的安装目录（`C:\Program Files (x86)\VMware\VMware Workstation\`）下。


#### 创建虚拟机并调整参数

VMware Workstation 很成熟，创建虚拟机只需要参照向导一步一步进行，需要注意的地方有以下几点：
* 是选择客户机操作版本对话框，客户机操作系统选择 `Apple macOS`，`version` 选择 `macOS 10.15`；
* 虚拟磁盘大小至少 60GB，并且选择单文件模式（单文件性能更好一点）；
* 调整内存大小至少 8GB（建议 16GB），CPU 选择 4 核（注意不要选多 CPU，如果 CPU 支持 Intel VT-x/EPT 或 AMD-V/RVI，则可以勾选这里）
* 网络建议设置成桥接（Bridged）

> 为了方便后续描述，我把虚拟机命名成 `macOS 10.15`，对应生成的 vmx 文件名就是 `macOS 10.15.vmx`

创建完成之后不要立即启动虚拟机，我们还要手动调整一些参数，使用趁手的编辑器打开 vmx 文件（此处我示例文件叫 `macOS 10.15.vmx`）

首先要添加的的是 `smc.version = "0"`，然后搜索 `board-id.reflectHost` 并将其值修改成 `FALSE`。

接下来的部分需要另一个工具来生成 Apple 设备序列号，它就是 `GenSMBIOS`，
github repo 在 [https://github.com/corpnewt/GenSMBIOS](https://github.com/corpnewt/GenSMBIOS)，
克隆下来之后，要在 Window 系统下运行：

```powershell
PS C:\Work\github\corpnewt\GenSMBIOS>.\GenSMBIOS.bat
#######################################################
#                     GenSMBIOS                       #
#######################################################

MacSerial v2.1.8
Current plist: None
Plist type:    Unknown

1. Install/Update MacSerial
2. Select config.plist
3. Generate SMBIOS
4. Generate UUID
5. Generate ROM
6. List Current SMBIOS
7. Generate ROM With SMBIOS (Currently Enabled)
8. Additional Args (Currently: None)

Q. Quit

Please select an option:  3

#######################################################
#                  Generate SMBIOS                    #
#######################################################

M. Main Menu
Q. Quit

Please type the SMBIOS to gen and the number
of times to generate [max 20] (i.e. iMac18,3 5): iMac20,1

#######################################################
#                iMac20,1 SMBIOS Info                 #
#######################################################
Type:         iMac20,1
Serial:       C0..HK....5T
Board Serial: C0.....04......CB
SmUUID:       8F048939-F5D0-480C-9938-7233BF9BD635
Apple ROM:    90........81

Press [enter] to return...
```

这里需要记录的信息是 `Serial` 、 `Board Serial` 和 `Apple ROM`，然后对应修改 `macOS.vmx` 这一段 （如果已存在则替换）：

```ini
board-id = "Mac-A61BADE1FDAD7B05"
hw.model.reflectHost = "FALSE"
hw.model = "iMac20,1"
serialNumber.reflectHost = "FALSE"
serialNumber = "C0..HK....5T"
smbios.reflectHost = "FALSE"
efi.nvram.var.ROM.reflectHost = "FALSE"
efi.nvram.var.MLB.reflectHost = "FALSE"
efi.nvram.var.ROM = "90........81"
efi.nvram.var.MLB = "C0.....04......CB"
```

接下来还需要修改网络配置，这里需要确保网络 MAC 地址符合苹果的要求（标准参考：[ https://hwaddress.com/company/apple-inc/]( https://hwaddress.com/company/apple-inc/)），
搜索 `ethernet0`，然后修改成下面这样：

```ini
ethernet0.addressType = "static"
ethernet0.address = "00:21:E9:c0:92:76"
ethernet0.checkMacAddress = "FALSE"
```

最后看起像这样的（我这里设置16GB内存，4核CPU）：
![VMware macOS](/assets/images/run_macos15_in_vmware/VMware_macOS.png)

下一步就可以挂载 macOS ISO 安装镜像，开始安装了。

## 安装 macOS 并升级到 macOS 15

在虚拟机配置页面，选中 CD/DVD(SATA)，这里勾选启动时连接，然后选择使用 ISO 镜像文件，
在浏览文件对话框里找到前文制作好的镜像文件（`MacOS-Sonoma-14.7.iso`），点击确定，回到虚拟机主页。
此时点击“开启此虚拟机”，稍等一小会儿就可以看到 Apple 标志：

![VMware - install mac](/assets/images/run_macos15_in_vmware/VMware_install_mac_1.png)

这里不用任何操作，稍等片刻就能看到安装语言界面：

![VMware - install mac](/assets/images/run_macos15_in_vmware/VMware_install_mac_2.png)

这里我选的中文，点击下一步按钮（通常是一个向右的箭头）就会进入启动盘菜单界面，这里能看到好几个选项，我们只关注其中两个：

* 安装 macOS Sonoma
* 磁盘工具

![VMware - install mac](/assets/images/run_macos15_in_vmware/VMware_install_mac_3.png)

因为虚拟机创建的是全新的裸盘，因此首先需要选中“磁盘工具”，点击继续，看到磁盘界面:

![VMware - install mac](/assets/images/run_macos15_in_vmware/VMware_install_mac_4.png)

左侧菜单里面通常有一个名字是 NECVMWar VMware SATA 开头的磁盘，这个是安装镜像挂载位置；
下面还有一个随机名字的磁盘，选中它，然后在右边的第四个按钮，叫“抹掉”，点击抹掉，在弹出的界面输入新名字（比如 `mac`），
确定并等待结束，回到磁盘工具界面，点击左上角的关闭，回到上一个截图的启动盘菜单界面，
此时就可以选择第二个，安装 macOS Sonoma，然后看到许可协议，两次确认之后就看到了下面的安装界面。

![VMware - install mac](/assets/images/run_macos15_in_vmware/VMware_install_mac_5.png)

此后就是漫长的等待（大约20分钟左右），期间虚拟机会自动重启几次，然后看到“选择国家或地区”界面（这里不重要，随意选择）：

![VMware - install mac](/assets/images/run_macos15_in_vmware/VMware_install_mac_6.png)

接下来的几个界面都可以一路点击继续，在迁移助理界面，选择左边的 “以后” 按钮跳过。
在 “通过 Apple ID 登录” 界面可以输入苹果 ID 和密码并登录，

![VMware - install mac](/assets/images/run_macos15_in_vmware/VMware_install_mac_7.png)

跳过之后又是许可协议，两次点击确认，就会转到 “创建电脑账户” 界面：

![VMware - install mac](/assets/images/run_macos15_in_vmware/VMware_install_mac_8.png)

接下来几个无关紧要的界面，然后看到“启用定位服务”，此处强烈建议**不要**勾选，在弹出的对话框选择 “**不使用**”，
因为定位服务会导致后续界面的时区和时间都不能自己指定。

![VMware - install mac](/assets/images/run_macos15_in_vmware/VMware_install_mac_9.png)

之后一路继续，根据自己喜好，设置时区和系统时间，选择系统外观，很快就会进入 macOS 14 的桌面，
不过此时可能看到的是一个白板，底部浮着程序坞，右上角显示系统安装盘。

![VMware - install mac](/assets/images/run_macos15_in_vmware/VMware_install_mac_A.png)

打开设置，点击左边“通用”，进入右边的“更新”，等待一会儿就会看到系统可以升级到 `macOS 15.3`，点击立刻升级，大约30分钟后升级完成。

![VMware - install mac](/assets/images/run_macos15_in_vmware/VMware_install_mac_B.png)

到这里安装基本完成，接下来还需要做一些配置。

#### 为 macOS 安装 VMware Tools

前文提到已经将 `darwin.iso` 拷贝到 VMware Workstation 的安装目录，这里需要将这个 `darwin.iso` 挂载到虚拟机的 CD/DVD 上。
记得同时点击连接，确保在桌面右上角看到 `VMware Tools` 的新磁盘，然后双击这个新磁盘，可以看到安装程序图标，双击头标开始安装：

![VMware - config mac](/assets/images/run_macos15_in_vmware/VMware_config_mac_1.png)

> **注意**
> 
> 新版本 macOS 会校验程序的开发者，默认仅运行 App Store 程序，对于 VMware Tools 这样来自三方直接发布的程序，安装的时候，
> 会有提醒，需要在设置里修改信任选项，将“允许以下来源的应用程序”配置项的值修改成 “App Store 与已知开发者”。
> 
> 第一次确认之后会要求重启系统（同时安装程序提示安装失败），按要求重启，重启之后重复上述安装过程即可成功！

基本上安装 VMware Tools 没有特殊配置的地方，一路 next 到底，然后看到 “重新启动” 按钮，点击重启电脑就行了。

![VMware - config mac](/assets/images/run_macos15_in_vmware/VMware_config_mac_2.png)


#### 一些 macOS 基本配置

作为程序员群体，macOS 必不可少的是 `homebrew`，要想安装 `homebrew`，需要先安装命令行开发工具（`Command Line Tools (CLT) for Xcode`）：

```shell
$ xcode-select --install
```

在弹出的对话框点击 “安装”，然后接受许可协议，一小段时间的等待之后就安装成功了。

![VMware - config mac](/assets/images/run_macos15_in_vmware/VMware_config_mac_3.png)

另外， `homebrew` 是依赖 github 的一个工具链，但是由于国内网络环境的特殊性，因此需要配置一下镜像站。
这里我选择清华大学 tuna 镜像站， 速度快而且稳定，详细文档可以参考这里：[Homebrew 软件仓库](https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/)。

接着配置一些环境变量，指定通过 tuna 镜像来拉取数据：

```shell
export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git"
export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git"
export HOMEBREW_INSTALL_FROM_API=1
# export HOMEBREW_API_DOMAIN
# export HOMEBREW_BOTTLE_DOMAIN
# export HOMEBREW_PIP_INDEX_URL
```

> 注：自 brew 4.0 起，`HOMEBREW_INSTALL_FROM_API` 会成为默认行为，无需设置

然后，在终端运行以下三行命令以安装 `Homebrew`

```shell
git clone --depth=1 https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/install.git brew-install
/bin/bash brew-install/install.sh
rm -rf brew-install
```

安装成功后需将 brew 程序的相关路径加入到环境变量中（这一步信息可以在 console 里看到）：

```shell
echo 'export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git"' >> ~/.zprofile
echo 'export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git"' >> ~/.zprofile
```

然后就可以使用 Homebrew 来安装软件啦，比如使用更新的 git，使用 iTerm2 替代系统 Terminal，等等：

```shell
$ brew install git iterm2
```

## 总结

使用 VMware Workstation 安装 macOS 并不算复杂，主要的关键点总结如下：

* 使用已有 mac 系统自己创建 macOS 安装引导镜像（或者从网络下载）
* 使用 `unlocker` 工具解锁 VMware， 支持安装 Apple macOS
* 从 VMware Fusion 获取 `VMware Tools` 镜像（`darwin.iso`）
* 创建 VMware 虚拟机之后需要调整 vmx 参数，重点是生成 Apple 设备的 `Serial`，`Board Serial` 和 `Apple ROM`
* 系统安装完成之后升级到 macOS 15，然后安装 `VMware Tools`

