---
layout: post
title:  "Full Disk Encryption"
date:   2017-03-09 13:31:01 +0800
categories: Android
tag: migrate
---

* content
{:toc}

# Full Disk Encryption(翻译)
原文:http://source.android.com/security/encryption/index.html

## 什么是全盘加密
全盘加密是使用密钥加密Andr​​oid设备上的所有用户数据的过程。一旦设备被加密，所有用户创建的数据，保存到磁盘之前会自动加密，并在调用数据之前自动解密。

## Android5.0上我们要增加什么
1. 创建快速加密，只对data分区中使用了的块进行加密，以避免第一次启动需要很长的时间。目前只有EXT4和f2fs文件系统支持快速加密。
2. 在首次启动增加forceencrypt标志。
3. 增加无密码加密支持。
4. 增加通过使用TEE签署能力来实现的hardware-backed的密钥存储
>`注意:` 如果是升级到5.0的设备可以通过恢复出厂设置来回到未加密的状态，而全新的5.0设备，会在首次开机时加密，且不可以恢复到未加密的状态。

## Android全盘加密如何工作的
Android的全磁盘加密是基于内核的一项在块设备层工作的特性:dm-crypt。正因为如此，加密适用于eMMC以及类似的块存储设备。加密是不适用于YAFFS，它直接和NAND的raw数据通讯。

加密算法是128的AES中的CBC和ESSIV:SHA256。主密钥是通过调用OpenSSL库采用128位AES加密。您必须使用128位以上的密钥（256位是可选的）。
>`注意:` OEM厂商可以使用128位或更高位去加密主密钥。

在Android 5.0版本，有四种加密状态：
1. default
2. PIN
3. password
4. pattern

在第一次开机时，设备会创建一个随机生成128位的主密钥，然后用默认密码和存储的盐值(salt)哈希它。默认密码是：“default_password”。然而，所得哈希也通过TEE签名，它使用签名的哈希加密主密钥。
你可以在Android开源项目cryptfs.c文件中找到定义的默认密码。

当用户在设备上设置PIN或密码时，只有128位的密钥会被重新加密并存储。 （即PIN /密码图案的变化不会引起data区的重新加密。）请注意，设备本身可能会受到PIN，图案或密码限制。

加密是由init和vold管理的。init调用的vold，并vold通过设置属性来触发init的事件。系统的其他部分也观察属性来执行任务，例如报告状态，要求输入密码，或当出现致命错误的情况下提示恢复出厂设置。要调用在vold的加密功能，系统使用命令行工具（VDC的cryptfs命令）：checkpw，重启，enablecrypto，changepw，cryptocomplete，verifypw，setfield，getfield，mountdefaultencrypted，getpwtype，getpw和clearpw。

在加密，解密或擦拭data区时，data区​​不能被挂在。然而，为了显示的任何用户界面（UI），framework必须运行并且framework需要data区才能运行。要解决这个难题，一个临时的文件系统被挂在到/data。这使得Android可以提示输入密码，显示进度，建议根据需要数据擦除。这种情况增加了一个缺点，为了从临时文件系统切换到真的data区，系统必须停止所有与临时文件系统有关的每个进程，并在真的data区上重新启动。要做到这一点，所有的服务必须在一下三个组之一:
1. core：启动后一直不关闭。
2. main：关闭并在输入密码后重启。
3. late_start：不启动，直到data区已经被解密并挂载。

为了触发这些操作，属性vold.decrypt 被设置为不同的字符串。杀死并重新启动服务的init命令是：
1. class_reset：停止服务，但允许它被class_start重新启动。
2. class_start：重新启动服务。
3. class_stop：停止服务，并增加了SVC_DISABLED标志。不可以被class_start重新启动。

## 流程
加密的设备有四种流程。一个设备只可以被加密一次，之后都遵循正常的启动流程。
1. 加密一个未加密的设备：
>(1) 加密一个带有forceencrypt标志位的新设备：在第一次启动强制加密（Android L后开始）。
>(2) 加密现有设备：用户手动启动加密（Android K和更早版本）。

2. 启动加密设备：
>(1) 启动没有密码的加密设备:启动没有设置密码的加密设备。（Android 5.0及更高版本的设备）
>(2) 启动用密码加密设备

除了这些流程，设备也可能在加密过中失败。将在下面详细说明每个流程。

### 加密一个带有forceencrypt标志位的新设备
这是一个Android 5.0设备正常的第一次启动的情况。
1. 用 forceencrypt标志检测未加密的文件系统
/data没有加密，但是因为forceencrypt被设置了，所以必须加密。卸载/data。
2. 开始加密
属性vold.decrypt =“trigger_encryption”。触发init.rc，这将导致vold不用密码加密/data。 （密码没有被设置，因为这是一个新的设备。）
3. 挂载tmpfs
vold挂载/data到tmpfs（使用从ro.crypto.tmpfs_options得到的tmpfs），并将属性vold.encrypt_progress设置为0。vold 为启动加密系统准备tmpfs /data，并将属性vold.decrypt设置为trigger_restart_min_framework
4. 启动framework，显示进度条
由于设备几乎没有数据需要加密，加密很快，通常不会出现进度条。可以在加密已有的设备章节了解进度条的细节
5. 当/data加密完成，关闭framework
vold设置vold.decrypt为trigger_default_encryption.这将启动defaultcrypto服务。trigger_default_encryption检查加密类型，看看/data是否使用密码加密。由于Android 5.0设备在第一次启动加密，不会有密码设置;因此，我们解密和挂载/data。
6. 挂载data
然后init使用ro.crypto.tmpfs_options参数挂载/data到tmpfs的RAMDisk。ro.crypto.tmpfs_options参数是init.rc.设置的
7. 启动framework
vold设置vold.decrypt为trigger_restart_framework，然后继续正常的开机过程。

### 加密已有的设备
这种情况发生在加密一个 Android K 或者更早的版本升级到L的设备上。
这个过程是用户发起的。当用户选择加密设备时，设备要提示用户电池完全充电和交流电适配器插入来保证有足够的电完成加密过程。
>`警告：如果设备在完成加密前电量耗尽，会导致设置处于一个部分加密状态，这种情况下，只有回恢复出厂设置，用户数据也会丢失`

要启用加密，vold启动一个循环去读取块设备的每个扇区，然后将其写入加密块设备。 vold在读写之前的检查扇区是否被使用过了，这可以让加密在很少或几乎没有数据的新的设备上快得多。

设备的状态：设置ro.crypto.state =“unencrypted”然后执行on nonencrypted init触发器继续启动。

1. 检查密码
UI用 cryptfs enablecrypto inplace调用void，其中passwd是用户的锁屏密码。

2. 削弱framework
vold检查是否可加密。如果它不能加密，返回-1，并打印在日志中的原因。如果它可以加密，它设置属性vold.decrypt为trigger_shutdown_framework。这将导致init.rc停在main和late_start类型的服务。

3. 卸载/data
vold卸载/ mnt / SD和/data。

4. 开始加密/data
vold建立一个加密映射，它创建一个虚拟加密的块设备，虚拟加密的块设备映射到真正的块设备加密，但它被写入时加密扇区，读取时解密扇区。vold然后创建并写入了加密的元数据。

5. 加密的时候，挂载tmpfs
vold挂载/data到tmpfs（使用从ro.crypto.tmpfs_options得到的tmpfs），并将属性vold.encrypt_progress设置为0。vold 为启动加密系统准备tmpfs /data，并将属性vold.decrypt设置为trigger_restart_min_framework
6. 启动framework，显示进度条
trigger_restart_min_framework触发init.rc启动main类的服务。当framework看到vold.encrypt_progress设置为0，它展示进度条界面，且每隔五秒钟查询该属性，并更新进度条。加密进程每次加密1%的分区就更新vold.encrypt_progress。
7. /data加密完成后，重启
当/data被成功加密，vold清除元数据中的标志ENCRYPTION_IN_PROGRESS并重启。
如果由于某种原因重启失败，vold设置vold.encrypt_progress为error_reboot_failed，用户界面应该显示一条消息，要求用户按一个按钮，重新启动。不期望这不断发生。

### 启动默认加密的设备
这是当你启动一个没有密码的加密的设备。由于Android 5.0设备在第一次启动加密，此时没有设置密码，因此，这是默认的加密状态。

1. 检测加密的/data没有密码
因为/data不能挂载且encryptable或 forceencrypt被设置，检测到Android设备加密。
vold设置vold.decrypt为trigger_default_encryption，这将启动defaultcrypto服务。 trigger_default_encryption检查加密类型，看看/data被使用或不使用密码加密。

2. 解密/data
创建了覆盖块设备的DM-crypt设备，因此设备就可以使用了。

3. 挂载/data
vold然后挂载解密后的的真正/ data分区，然后准备新的分区。它设置属性vold.post_fs_data_done为0，设置vold.decrypt到trigger_post_fs_data。这将导致init.rc运行post-fs-data命令。他们将创建必需的目录或链接，然后设置vold.post_fs_data_done为1。
一旦在vold的发现该值为1，它设置属性vold.decrypt为trigger_restart_framework。这将导致init.rc，重新开始在main服务，并且开机以来第一次启动late_start服务。

4. 启动framework
现在framework使用解密的/data的启动的所有服务，并且系统初始化好了。

### 启动非默认加密的设备
这是当你启动一个具有密码的加密设备。该设备的密码可能是PIN，图案或密码。
1. 检测到加密的设备存在密码
由于标志ro.crypto.state =“encrypted”，判断Andr​​oid设备被加密。因为/数据与密码进行加密，vold设置vold.decrypt为trigger_restart_min_framework。
2. 挂载 tmpfs
init将用init.rc传来的参数设置五个属性以保存/data的初始安装选项。vold的使用这些属性来设置加密映射：
```
ro.crypto.fs_type
ro.crypto.fs_real_blkdev
ro.crypto.fs_mnt_point
ro.crypto.fs_options
ro.crypto.fs_flags（以0x开头ASCII 8位十六进制数）
```

3. 启动framework，提示输入密码
framework启动并看到vold.decrypt设为trigger_restart_min_framework。这告诉framework，它是启动在tmpfs的/data上的，且需要获得用户的密码。
但是，首先，它需要确保该磁盘被正确加密。它发命令cryptfs cryptocomplete到VOLD。 如果加密已成功完成，vold的返回0，-1代表内部错误，-2加密未成功完成。vold通过元数据中的CRYPTO_ENCRYPTION_IN_PROGRESS标志来确定这一点。如果它被设置，代表加密过程被中断，并且在该设备上没有可用的/data。如果vold返回一个错误，UI应该​​显示一条消息，让用户重新启动和工厂重置，并给用户一个按钮重启并恢复出厂设置。

4. 用密码解密/data
一旦cryptfs cryptocomplete返回成功，framework将显示UI要求用户输入密码。UI通过发送cryptfs checkpw命令到VOLD来检查密码。如果密码是正确的（这是由在一个临时位置成功挂载解密的/data决定的，然后将其卸载），vold在属性ro.crypto.fs_crypto_blkdev中保存解密的块设备的名称，并返回状态0到UI 。如果密码不正确，则返回-1到用户界面。

5. 停止framework
UI呈现了一个密码启动的界面，然后调用vold的cryptfs restart命令。vold的设置属性vold.decrypt为trigger_reset_main，这将导致init.rc做class_reset main。这将停止所有main类的服务，这样才能卸载tmpfs /data。

6. 挂载/data
接下来，vold的挂载真正解密的/ data分区，并准备新的分区（如果它是用wipe选项加密，它也许从未被准备。首版上是不支持的）。它设置属性vold.post_fs_data_done为0，然后设置vold.decrypt为trigger_post_fs_data。这将导致init.rc运行 post-fs-data命令。他们将创建任何必需的目录或链接，然后设置vold.post_fs_data_done为1。一旦vold发现该属性为1，它设置属性vold.decrypt为trigger_restart_framework。这将导致init.rc，重新开始在main服务，并且开机以来第一次启动late_start服务。

7. 开启完整的framwork
现在framework使用解密的/data的启动的所有服务，并且系统初始化好了。

### 失败
设备通常的以几个步骤去启动:
```
1.检测到加密设备存在密码
2.挂载tmpfs
3.开启framework去提示输入密码
```
但是在framework运行后，可能出现一些错误:
```
1.密码正确，却无法解密data
2.用户输错密码超过30次
```
如果这些错误不被解决，提示用户恢复出场设置。

如果vold的在加密过程中检测到错误，并且没有数据被破坏，franework也启动了，vold设置属性vold.encrypt_progress为error_not_encrypted。用户界面会提示用户重新启动，并提醒他们从来没有开始加密。如果错误发生在franework以停止，但进度条UI还未完成，vold将重新启动系统。如果重新启动失败，则设置为vold.encrypt_progress和error_shutting_down返回-1;但不会有任何捕获错误。这是不期望的情况。

如果vold的在加密过程中出现错误，它设置为vold.encrypt_progress为error_partially_encrypted返回-1。然后，UI应该显示一条消息，说失败的加密并且为用户提供一键重置设备的按钮。

## 存储加密密钥
加密的密钥存储在加密的元数据中。Hardware backing是通过使用TEE签署的能力来实现。先前，我们用通过施加scrypt到用户的口令和所存储的盐生成的密钥加密主密钥。为了使密钥抵抗off-box attacks，我们通过用存储在TEE的密钥签名上步所得密钥。然后通过一个或更多的scrypt应用来变成一个适当的长度密钥。然后，该密钥用于加密和解密的主密钥。
存储此key：
1.	随机生成16个字节的磁盘加密密钥（DEK）和16个字节的盐。
2.	使用scrypt给用户密码和盐，并产生32字节的中间密钥（IK1）。
3.	给IK1填充零字节直到满足hardware-bound private key（HBK）的大小。具体来说，我们这样填充：00 || IK1 || 00..00;一个零点字节，32字节IK1，223零字节。
4.	用HBK签名IK1产生256字节的IK2
5.	使用scrypt到lk2和盐（盐和步骤2相同），以产生32字节IK3。
6.	使用前16个字节的IK3作为KEK，最后16个字节作为IV
7.	用AES_CBC使用KEK作为key加密DEK，初始化向量IV。

## 修改密码
当用户选择更改或删除密码，用户界面​​发送cryptfs changepw命令到vold，vold用新密码盘重新加密主密钥。

## 加密相关属性
vold和init通过设置属性来进行通信。这里是加密相关的属性的列表。
### vold	属性

| 属性      |    描述 |
|:--------|:-------|
| vold.decrypt	=	trigger_encryption  | 不用用户密码加密drive |
| vold.decrypt	=	trigger_default_encryption  | 检查drive，看它是否是不带用户密码的加密。如果是，解密和挂载它，否则设置vold.decrypt为trigger_restart_min_framework |
| vold.decrypt	=	trigger_reset_main  | vold通过设置它，去关闭要求磁盘密码的UI|
| vold.decrypt	=	trigger_post_fs_data  | vold设置它去准备/data相关的目录等 |
| vold.decrypt	=	trigger_restart_framework  | vold通过设置它来启动真正的framework和所有的service |
| vold.decrypt	=	trigger_shutdown_framework  | vold通过设置它来关闭完整的framework然后加密 |
| vold.decrypt	=	trigger_restart_min_framework  | vold通过设置它来启动加密的进度条的UI或提示密码，显示哪个界面取决于ro.crypto.state的值 |
| vold.encrypt_progress  | 当framework启动时，如果此属性被设置，进入显示进度条的UI模式 |
| vold.encrypt_progress	=	0到100	| 进度条显示的百分比 |
| vold.encrypt_progress	=	error_partially_encrypted| 进度条UI应显示加密失败的消息，并给用户出厂重置设备的选项 |
| vold.encrypt_progress	=	error_reboot_failed| 进度条UI应该显示一条消息，加密完成，给用户一个按钮，重新启动设备。但预计该错误不会发生 |
| vold.encrypt_progress	=	error_not_encrypted| 进度条UI应该显示一条消息，说发生了错误，但没有数据加密或丢失，给用户一个按钮，重新启动系统|
| vold.encrypt_progress	=	error_shutting_down| 进度条界面没有运行，所以目前还不清楚谁将会为这个错误响应。它不应该仍会发生|
| vold.post_fs_data_done		=	0| 在设置vold.decrypt为trigger_post_fs_data之前由vold设置它为0|
| vold.post_fs_data_done		=	1| 完成任务post-fs-data后，通过init.rc 设置|

### init	属性

| 属性      |    描述 |
|:--------|:-------|
|ro.crypto.fs_crypto_blkdev|由vold命令checkpw设置,被resatrt命令使用。|
|ro.crypto.state ＝	unencrypted	|由init设置，说明系统运行在一个未加密的/data上。如果改值为encreypted，则相反。|
|ro.crypto.fs_type,ro.crypto.fs_real_blkdev,ro.crypto.fs_mnt_point,ro.crypto.fs_options,ro.crypto.fs_flags 	|这五个属性是由init当它试图从init.rc传递的参数挂载/data。vold使用这些设置加密映射。|
|ro.crypto.tmpfs_options|在init.rc中设置，当init挂载tmpfs /data时使用|

### init	动作
```
on post-fs-data
on nonencrypted
on property:vold.decrypt=trigger_reset_main
on property:vold.decrypt=trigger_post_fs_data
on property:vold.decrypt=trigger_restart_min_framework
on property:vold.decrypt=trigger_restart_framework
on property:vold.decrypt=trigger_shutdown_framework
on property:vold.decrypt=trigger_encryption
on property:vold.decrypt=trigger_default_encryption
```