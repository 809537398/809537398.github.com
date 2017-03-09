---
layout: post
title:  "SEAndroid!"
date:   2017-03-09 13:31:01 +0800
categories: Android
tag: migrate
---

* content
{:toc}

### SEAndroid安全机制
![enter image description here](http://img.blog.csdn.net/20140710112453799?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
1. 内核空间的SELinux LSM模块负责内核资源的安全访问控制。
>访问向量缓冲（Access Vector Cache）和一个安全服务（Security Server）
2. 用户空间的SEAndroid Policy描述的是资源安全访问策略。
>external/sepolicy/users
>external/sepolicy/roles

3. 用户空间的Security Context描述的是资源安全上下文。
4. 用户空间的Security Server一方面需要到用户空间的Security Context去检索对象的安全上下文，另一方面也需要到内核空间去操作对象的安全上下文。
5. 用户空间的selinux库封装了对SELinux文件系统接口的读写操作。
6. 用户空间的Security Server到用户空间的Security Context去检索对象的安全上下文时，同样也是通过selinux库来进行的。

App进程、App数据文件、系统文件和系统属性。这四种类型对象的安全上下文通过四个文件来描述：mac_permissions.xml、seapp_contexts、file_contexts和property_contexts，它们均位于external/sepolicy目录中。

#### mac_permissions.xml
```
<?xml version="1.0" encoding="utf-8"?>
<policy>

    <!-- Platform dev key in AOSP -->
    <signer signature="@PLATFORM" >
      <seinfo value="platform" />
    </signer>

    <!-- Media dev key in AOSP -->
    <signer signature="@MEDIA" >
      <seinfo value="media" />
    </signer>

    <!-- shared dev key in AOSP -->
    <signer signature="@SHARED" >
      <seinfo value="shared" />
    </signer>

    <!-- release dev key in AOSP -->
    <signer signature="@RELEASE" >
      <seinfo value="release" />
    </signer>

    <!-- All other keys -->
    <default>
      <seinfo value="default" />
    </default>

</policy>
```
`文件mac_permissions.xml给不同签名的App分配不同的seinfo字符串，例如，在AOSP源码环境下编译并且使用平台签名的App获得的seinfo为“platform”，使用第三方签名安装的App获得的seinfo签名为"default"。`
#### seapp_contexts
```
# Input selectors: 
#   isSystemServer (boolean)
#   user (string)
#   seinfo (string)
#   name (string)
#   sebool (string)
# isSystemServer=true can only be used once.
# An unspecified isSystemServer defaults to false.
# An unspecified string selector will match any value.
# A user string selector that ends in * will perform a prefix match.
# user=_app will match any regular app UID.
# user=_isolated will match any isolated service UID.
# All specified input selectors in an entry must match (i.e. logical AND).
# Matching is case-insensitive.
# Precedence rules:
#     (1) isSystemServer=true before isSystemServer=false.
#     (2) Specified user= string before unspecified user= string.
#     (3) Fixed user= string before user= prefix (i.e. ending in *).
#     (4) Longer user= prefix before shorter user= prefix. 
#     (5) Specified seinfo= string before unspecified seinfo= string.
#     (6) Specified name= string before unspecified name= string.
#     (7) Specified sebool= string before unspecified sebool= string.
#
# Outputs:
#   domain (string)
#   type (string)
#   levelFrom (string; one of none, all, app, or user)
#   level (string)
# Only entries that specify domain= will be used for app process labeling.
# Only entries that specify type= will be used for app directory labeling.
# levelFrom=user is only supported for _app or _isolated UIDs.
# levelFrom=app or levelFrom=all is only supported for _app UIDs.
# level may be used to specify a fixed level for any UID. 
#
isSystemServer=true domain=system
user=system domain=system_app type=system_data_file
user=bluetooth domain=bluetooth type=bluetooth_data_file
user=nfc domain=nfc type=nfc_data_file
user=radio domain=radio type=radio_data_file
user=_app domain=untrusted_app type=app_data_file levelFrom=none
user=_app seinfo=platform domain=platform_app type=platform_app_data_file
user=_app seinfo=shared domain=shared_app type=platform_app_data_file
user=_app seinfo=media domain=media_app type=platform_app_data_file
user=_app seinfo=release domain=release_app type=platform_app_data_file
user=_isolated domain=isolated_app
```
`从前面的分析可知，对于使用平台签名的App来说，它的seinfo为“platform”。用户空间的Security Server在为它查找对应的Type时，使用的user输入为"_app"。这样在seapp_contexts文件中，与它匹配的一行即为`
```
user=_app seinfo=platform domain=platform_app type=platform_app_data_file 
```

#### file_contexts
```
###########################################
# Root
/           u:object_r:rootfs:s0

# Data files
/adb_keys       u:object_r:rootfs:s0
/default.prop       u:object_r:rootfs:s0
/fstab\..*      u:object_r:rootfs:s0
/init\..*       u:object_r:rootfs:s0
/res(/.*)?      u:object_r:rootfs:s0
/ueventd\..*        u:object_r:rootfs:s0

# Executables
/charger        u:object_r:rootfs:s0
/init           u:object_r:rootfs:s0
/sbin(/.*)?     u:object_r:rootfs:s0

......

#############################
# System files
#
/system(/.*)?       u:object_r:system_file:s0
/system/bin/ash     u:object_r:shell_exec:s0
/system/bin/mksh    u:object_r:shell_exec:s0
```
#### property_contexts
```
##########################
# property service keys
#
#
net.rmnet0              u:object_r:radio_prop:s0
net.gprs                u:object_r:radio_prop:s0
net.ppp                 u:object_r:radio_prop:s0
net.qmi                 u:object_r:radio_prop:s0
net.lte                 u:object_r:radio_prop:s0
net.cdma                u:object_r:radio_prop:s0
gsm.                    u:object_r:radio_prop:s0
persist.radio           u:object_r:radio_prop:s0
net.dns                 u:object_r:radio_prop:s0
sys.usb.config          u:object_r:radio_prop:s0
......
```

#### 安全策略由所有.te文件构成

#### 启动过程
在init进程启动的过程中，加载所有策略文件到SELinux LSM模块中去
>A. 以/sys/fs/selinux为安装点，安装一个类型为selinuxfs的文件系统，也就是SELinux文件系统，用来与内核空间的SELinux LSM模块通信。
B. 如果不能在/sys/fs/selinux这个安装点安装SELinux文件系统，那么再以/selinux为安装点，安装SELinux文件系统。
C. 成功安装SELinux文件系统之后，接下来就调用另外一个函数selinux_android_reload_policy来将SEAndroid安全策略加载到内核空间的SELinux LSM模块中去。(`sys/fs/selinux/load文件，然后将参数data所描述的安全策略写入到这个文件中去。`)
`在较旧版本的Linux系统中，SELinux文件系统是以/selinux为安装点的，不过后面较新的版本都是以/sys/fs/selinux为安装点的，Android系统使用的是后者`

#### PackageManagerService和installd负责创建App数据目录的安全上下文，Zygote进程负责创建App进程的安全上下文，而init进程负责控制系统属性的安全访问。

#### 文件安全上下文
1.预置在ROM里面的.
2.动态创建的，即在系统在运行的过程中创建的。(`它们的安全上下文如果没有特别指定，就与父目录的安全上下文一致。`)
![enter image description here](http://img.blog.csdn.net/20140715111923780?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
1. 设置打包在ROM里面的文件的安全上下文
 通过file_contexts定义的上下文去设定
2. 设置虚拟文件系统的安全上下文
genfs_contexts
3. 设置应用程序数据文件的安全上下文
mac_permissions.xml

`应用程序在安装的时候分配到的UID由两部分组成，一部分是User Id，另一部分是App Id。常量AID_USER的值定义为100000，用UID除以该值获得的整数就为User Id，而用UID对该值进行取模得到的就为App ID
`
>    App Id小于AID_APP(10000)是保留给系统使用的，它们对应的username保存在数组android_ids中，具体可以参考文件`system/core/include/private/android_filesystem_config.h`。
 App Id大于等于AID_APP(10000)并且小于AID_ISOLATED_START(99000)是分配给`应用程序`使用的，它们对应的username统一设置为“_app”。
 App Id大于等于AID_ISOLATED_START(99000)是分配给运行在独立沙箱（没有任何自己的Permission）的`应用程序`使用的，它们对应的username统一设置为“_isolated”。
       
#### 进程安全上下文关
![enter image description here](http://img.blog.csdn.net/20140723013307393?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
1. 为独立进程静态地设置安全上下文
>概括来说，在external/sepolicy/zygote.te文件中，通过init_daemon_domain指明了Zygote进程的domain为zygote。我们可以从Zygote进程的创建过程来理解这些安全策略。首先， Zygote进程是由init进程fork出来的。在fork出来的时候，Zygote进程的domain来自于父进程init的domain，即此时Zygote进程的domain为init。接下来，刚刚fork出来的Zygote进程会通过系统接口exec将文件/system/bin/app_process加载进来执行。由于上面提到的allow和type_transition规则的存在，使得文件/system/bin/app_process被exec到刚刚fork出来的Zygote进程的时候，它的domain自动地从init转换为zygote。这样我们就可以给init进程和Zygote进程设置不同的domain，以便可以给它们赋予不同的SEAndroid安全权限。
一个新创建的进程的安全上下文在默认情况下来自于其父进程
system/core/rootdir/init.rc
```
on early-init
    ......

    # Set the security context for the init process.
    # This should occur before anything else (e.g. ueventd) is started.
    setcon u:r:init:s0
    ......
```

```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```

1. Zygote进程对应的可执行文件为/system/bin/app_process。
2. 文件/system/bin/app_process的type为zygote_exec
3. external/sepolicy/zygote.te文件中，定义了一个名称为zygote的domain
```
# zygote
type zygote, domain;
type zygote_exec, exec_type, file_type;

permissive zygote;
init_daemon_domain(zygote)
unconfined_domain(zygote)
```
4. init_daemon_domain语句是一个宏，定义在文件external/sepolicy/te_macros中，用来设置zygote这个domain的权限
```
#####################################
# init_daemon_domain(domain)
# Set up a transition from init to the daemon domain
# upon executing its binary.
define(`init_daemon_domain', `
domain_auto_trans(init, $1_exec, $1)
tmpfs_domain($1)
')
```
5. domain_auto_trans 
```
#####################################
# domain_auto_trans(olddomain, type, newdomain)
# Automatically transition from olddomain to newdomain
# upon executing a file labeled with type.
#
define(`domain_auto_trans', `
# Allow the necessary permissions.
domain_trans($1,$2,$3)
# Make the transition occur by default.
type_transition $1 $2:process $3;
')
```
6. type_transition 指定当一个domain为init的进程创建一个子进程执行一个type为zygote_exec的文件时，将该子进程的domain设置为zygote，而不是继承父进程的domain。
7. domain_trans 用来允许进程的domain从init修改为zygote
```
#####################################
# domain_trans(olddomain, type, newdomain)
# Allow a transition from olddomain to newdomain
# upon executing a file labeled with type.
# This only allows the transition; it does not
# cause it to occur automatically - use domain_auto_trans
# if that is what you want.
#
define(`domain_trans', `
# Old domain may exec the file and transition to the new domain.
allow $1 $2:file { getattr open read execute };
allow $1 $3:process transition;
# New domain is entered by executing the file.
allow $3 $2:file { entrypoint read execute };
# New domain can send SIGCHLD to its caller.
allow $3 $1:process sigchld;
# Enable AT_SECURE, i.e. libc secure mode.
dontaudit $1 $3:process noatsecure;
# XXX dontaudit candidate but requires further study.
allow $1 $3:process { siginh rlimitinh };
')
```
2.  为应用程序进程设置安全上下文
ActivityManagerService在请求Zygote进程创建应用程序进程的时候，会传递很多参数，例如应用程序在安装时分配到的uid和gid。增加了SEAndroid安全机制之后，ActivityManagerService传递给Zygote进程的参数包含了一个seinfo。这个seinfo与我们在前面SEAndroid安全机制中的文件安全上下文关联分析一文中介绍的seinfo是一样的，不过它的作用是用来设置应用程序进程的安全上下文，而不是设置应用程序数据文件的安全上下文
Zygote在fork后，进程一开时的contect为u:r:zygote:s0，之后在设置正确context。

#### SEAndroid安全机制对Android属性访问的保护分析 

![enter image description here](http://img.blog.csdn.net/20140727023519555?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
`Android系统的所有属性都是保存在内存中。这块属性内存区是通过将文件/dev/__properties__映射到内存中创建的`
属性内存区域最多可以容纳的属性个数为247个，并且每一个属性在头部的索引表都对应有一个类型为unsigned的索引项。每一个属性占用的字节数为32 + 4 + 92 = 128，每一个属性索引项占用的字节数是4个字节，因此属性内存区头部，即结构体prop_area，总区占用的字节数是4 + 4 + 4 + 4 + 4 * 4 + 4 * 247 = 1020。紧接着属性内存区头部的属性值区域。考虑到对齐因素，属性值区域从属性区域的1024个字节偏移开始。因此，整个属性区域占用的内存大小是1024 + 247 * 128 = 32768个字节，刚好就是32Kb。

```
struct prop_area {
    unsigned volatile count;属性内在区当前包含的属性个数。
    unsigned volatile serial;增加或者修改属性时用到的同步变量。
    unsigned magic;属性内存区魔数，用来识别属性内存区。
    unsigned version; 属性内在区版本号，用来识别属性内存区。
    unsigned reserved[4];保留字段。
    unsigned toc[1];属性内容索引表。
};

struct prop_info {
    char name[PROP_NAME_MAX];属性名称，长度最大为32个字节。
    unsigned volatile serial;修改属性值时用到的同步变量，同时也用来描述属性值的长度。
    (serial的最高字节用来描述对应的属性的值的长度，低三个字用来同步属性值的读写，并且最低的一位是用来描述属性是否正在修改中)
    char value[PROP_VALUE_MAX];属性值，长度最大为92个字节。
};
```
##### 内存区域的创建和初始化
1. 属性内存区域是由init进程在启动的过程中创建和初始化。
2. Init进程在启动的时候，会创建一个属性管理服务。这个属性管理服务会创建一个Server端Socket，用来接收其它进程发送过来的增加或者修改属性的请求。
3. system/core/init/init.c
```
int main(int argc, char **argv)
{
    ......

    property_init();
    ......

    if (!is_charger)
        property_load_boot_defaults();
    ......

    queue_builtin_action(property_service_init_action, "property_service_init");
    ......

    for(;;) {
        ......

        execute_one_command();
        ......

        if (!property_set_fd_init && get_property_set_fd() > 0) {
            ufds[fd_count].fd = get_property_set_fd();
            ufds[fd_count].events = POLLIN;
            ufds[fd_count].revents = 0;
            fd_count++;
            property_set_fd_init = 1;
        }

        ......

        nr = poll(ufds, fd_count, timeout);
        ......

        for (i = 0; i < fd_count; i++) {
            if (ufds[i].revents == POLLIN) {
                if (ufds[i].fd == get_property_set_fd())
                    handle_property_set_fd();
                ......
            }
        }
    }

    return 0;
}  
```
函数property_init的实现很简单，它通过调用另外一个函数init_property_area来创建和初始化一块属性内存区
函数property_load_boot_defaults通过调用另外一个函数load_properties_from_file将定义在文件PROP_PATH_RAMDISK_DEFAULT里面的属性加载前面创建的属性内存区域中。

`
bionic/libc/include/sys/_system_properties.h
    #define PROP_PATH_RAMDISK_DEFAULT  "/default.prop"  
`


函数property_service_init_action的实现很简单，它通过调用另外一个函数`start_property_service`来启动属性管理服务。`(这个函数定义在文件system/core/init/property_service.c中。)`
start_property_service首先是调用函数load_properties_from_file将定义在文件PROP_PATH_SYSTEM_BUILD和PROP_PATH_SYSTEM_DEFAULT中的属性加载到前面创建的属性内存区域中。接着又调用函数load_override_properties判断当前是否是以debug模式启动。如果是的话，那就再将定义在文件PROP_PATH_LOCAL_OVERRIDE中的属性加载到前面创建的属性内存区域中来。再接下来还会调用函数load_persistent_properties将保存在目录PERSISTENT_PROPERTY_DIR中的持久属性加载到前面创建的属性内存区域中来。
```
void start_property_service(void)
{
    int fd;

    load_properties_from_file(PROP_PATH_SYSTEM_BUILD);
    load_properties_from_file(PROP_PATH_SYSTEM_DEFAULT);
    load_override_properties();
    /* Read persistent properties after all default values have been loaded. */
    load_persistent_properties();

    fd = create_socket(PROP_SERVICE_NAME, SOCK_STREAM, 0666, 0, 0);
    if(fd < 0) return;
    fcntl(fd, F_SETFD, FD_CLOEXEC);
    fcntl(fd, F_SETFL, O_NONBLOCK);

    listen(fd, 8);
    property_set_fd = fd;
}
```

#### 其它进程是如何将Init进程创建的属性内存区域映射到自己的进程地址空间
1. 在创建进程之前，调用函数is_selinux_enabled检查系统是否开启了SEAndroid。
2. 在创建进程之后，调用函数properties_inited检查Init进程是否已经创建和初始化好属性内存区域了。如果已经创建和初始化好，那么就继续调用函数get_property_workspace来获得用来描述属性内存区域的文件描述符fd以及属性内存区域的大小sz。我们在前面提到，属性内存区域是通过将文件/dev/__properties__映射到内存中创建的。因此，这里得到的文件描述符fd实际上指向的就是文件/dev/__properties__。再接下来将属性内存区域的文件描述符以及大小拼接成一个字符串，并且将得到的字符串作为环境变量ANDROID_PROPERTY_WORKSPACE的值。
3. 调用函数setsockcreatecon设置新创建的Socket的安全上下文，这是因为接下来要为Zygote进程创建一个名称zygote的Socket。
4. 调用函数execve在新创建的进程中加载参数args[0]所描述的要执行文件，实际上就是/system/bin/app_process文件。从前面SEAndroid安全机制中的进程安全上下文关联分析一文可以知道，加载了/system/bin/app_process文件之后，新进程的安全上下文的domain就会被设置为zygote了。这样就可以与init进程的安全上下文区分开来。
>应用程序进程都是通过Zygote进程fork出来的，这意味着应用程序进程会继续父进程zygote的环境变量，也就是会继续前面第2步创建的环境变量ANDROID_PROPERTY_WORKSPACE。有了这个环境变量之后，就可以得到用来描述在Init进程创建的属性内存区域的文件描述符以及大小了。

读取Android系统的属性是没有权限控制的，并且可以直接从本进程的地址空间读取
设置Android系统的属性，需要通过socket通讯，连接到init去修改。
并且要通过check_control_perms（以“ctl.”开头）和check_perms（包括基于UID/GID的权限检查和基于SEAndroid的权限检查）检测后才能修改属性。
`遍历数组property_perms。检查名称为name的属性是否在它的配置当中。如果在它的配置之中，一方面是要求请求进程具有与配置一样的UID/GID，另一方面是要求请求进程的安全上下文符合SEAndroid安全策略定义的规则。如果不在它的配置当中，那么就拒绝对名称为name的属性进行增加或者修改`

#### SEAndroid安全机制对Binder IPC的保护分析 

![enter image description here](http://img.blog.csdn.net/20140801012725545?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
当Service Manager将自己注册为Context Manager时，Binder驱动会检查它是否具有设置Context Manager的SEAndroid安全权限。
当Client通过Binder驱动请求与Server发送通信时，Binder驱动会检查Client是否具有与Server通信的SEAndroid安全权限。
Client与Server的通信数据带有Binder对象或者文件描述符，那么Binder驱动还会进一步检查源进程是否具有向目标进程传输Binder对象或者文件描述符的SEAndroid权限。
```
struct security_class_mapping secclass_map[] = {
    ......

    { "binder", { "impersonate", "call", "set_context_mgr", "transfer", NULL } },
    
    ......
};

allow unconfineddomain domain:binder { call transfer set_context_mgr };
```
所有类型的应用程序进程的domain都具有unconfineddomain属性

>一个进程可以打开一个有权限访问的文件，得到一个文件描述符，然后再通过Binder IPC将这个文件描述符传递给另外一个进程，使得另外的这个进程也可以访问这个文件。由于文件是属于系统的敏感资源，因此，我们需要禁止一个进程将一个文件描述符传递给一个没有权限访问的进程。

`
在Android系统中，所有类型的应用程序进程，以及系统服务进程，例如Service Manager进程、Zygote进程和System Server进程，它们能不能传递文件描述符给对方，不是取决于它们的domain有没有unconfined_domain属性，而是取决于目标进程是否具有使用传递的文件的描述符以及访问传递的文件的内容的权限。
`