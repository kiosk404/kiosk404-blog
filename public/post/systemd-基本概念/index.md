# systemd 基本概念

systemd 是 PID 为1的一个程序，负责初始化系统。所有的程序不是systemd直接启动就是由systemd的子系统启动。systemd是内核直接启动，所以信号9(KILL)对systemd也无效。

systemd 使用Linux控制组跟踪进程，维护安装和自动挂载点，并实现基于事务性依赖关系的详尽服务控制逻辑。其他部分包括日志记录守护程序，用于控制基本系统配置的实用程序，例如主机名，日期，区域设置，维护已登录用户和正在运行的容器和虚拟机的列表，系统帐户，运行时目录和设置，以及用于管理简单网络的守护程序配置，网络时间同步，日志转发和名称解析。

<!--more-->
<img src='https://img1.kiosk007.top/static/images/systemd/systemd.png' style="height:400px" />

# 简介
## 对比 systemV

从CentOS 7.x 以后，Red Hat 系列的 distribution 放弃沿用多年的System V 开机启动服务的流程. systemd 也是当年我从6.x 到 7.x 过度时感觉到最大的变化了，直到今天，MySQL的启动脚本还是可以通过 `service mysqld start ` 启动的，但是升级到 CentOS 7.x 之后，就变成了 `systemd start mysqld` 了。当然还有经典的`init 0`的关机指令。

相比于 initd，systemd有以下几个大的进步。
- <font color=blue >并行启动</font>：旧的 init 启动脚本是一项一项任务依序启动的模式，因此不相依的服务也是得要一个一个的等待。systemd可以让所有的服务同时启动，因此你会发现到，系统启动的速度变快了！
- <font color=blue >一经要求就回应的on-demand启动方式</font>：systemd全部就是仅有一只systemd服务搭配systemctl指令来处理，无须其他额外的指令来支援。不像systemV还要init, chkconfig, service...等等指令。此外， systemd由于常驻记忆体，因此任何要求(on-demand)都可以立即处理后续的daemon启动的任务。
- <font color=blue >服务相依性的自我检查</font>：由于systemd可以自订服务相依性的检查，因此如果B服务是架构在A服务上面启动的，那当你在没有启动A服务的情况下仅手动启动B服务时， systemd会自动帮你启动A服务。
- <font color=blue >依daemon功能分类</font>：systemd下管理的服务非常多，为了厘清所有服务的功能，因此，首先systemd先定义所有的服务为一个服务单位(unit)，并将该unit归类到不同的服务类型(type)去。旧的init仅分为stand alone与super daemon，systemd将服务单位(unit)区分为service, socket, target, path, snapshot, timer等多种不同的类型(type)，方便管理员的分类与记忆。
- <font color=blue >将多个daemons集合成为一个群组</font>：如同systemV的init里头有个runlevel的特色，systemd亦将许多的功能集合成为一个所谓的target项目，这个项目主要在设计操作环境的建置，所以是集合了许多的daemons，亦即是执行某个target就是执行好多个daemon的意思！
- <font color=blue >向下相容旧有的init服务脚本</font>：基本上， systemd是可以相容于init的启动脚本的，因此，旧的init启动脚本也能够透过systemd来管理，这也是为什么到现在还可以使用`serivce mysqld start` 这样命令的原因。

> **综上可知，systemd 已经足够强大，可以管理一个进程的生命周期，如果是自己写的一套代码完全可以交给systemd 来维护呀，以前公司使用 [god](https://ruby-china.org/topics/21354)(进程监控守护工具) 来管理服务，god 提供了服务启动、服务宕机自动拉起、环境变量和chroot、服务资源限制、定时任务等，但是systemd 的出现已经足够替代 god 这样的第三方服务，systemd 已经足够实现服务托管。**

<img src='https://img1.kiosk007.top/static/images/systemd/systemd_components2.png' style='height:400px' />

## systemd 的配置文件

基本上， systemd 将过去所谓的daemon 执行脚本通通称为一个服务单位(unit)，而每种服务单位依据功能来区分时，就分类为不同的类型(type)。在 Systemd 的生态圈中，Unit 文件统一了过去各种不同系统资源配置格式，例如服务的启/停、定时任务、设备自动挂载、网络配置、虚拟内存配置等。而 Systemd 通过不同的文件后缀来区分这些配置文件。

- <font color=blue >automount</font>：用于控制自动挂载文件系统，相当于 SysV-init 的 autofs 服务
- <font color=blue >device</font>：对于 /dev 目录下的设备，主要用于定义设备之间的依赖关系
- <font color=blue >mount</font>：定义系统结构层次中的一个挂载点，可以替代过去的 /etc/fstab 配置文件
- <font color=blue >path</font>：用于监控指定目录或文件的变化，并触发其它 Unit 运行
- <font color=blue >scope</font>：这种 Unit 文件不是用户创建的，而是 Systemd 运行时产生的，描述一些系统服务的分组信息
- <font color=blue >service</font>：封装守护进程的启动、停止、重启和重载操作，是最常见的一种 Unit 文件
- <font color=blue >slice</font>：用于表示一个 CGroup 的树，通常用户不会自己创建这样的 Unit 文件
- <font color=blue >snapshot</font>：用于表示一个由 systemctl snapshot 命令创建的 Systemd Units 运行状态快照
- <font color=blue >socket</font>：监控来自于系统或网络的数据消息，用于实现基于数据自动触发服务启动
- <font color=blue >swap</font>：定义一个用户做虚拟内存的交换分区
- <font color=blue >target</font>：用于对 Unit 文件进行逻辑分组，引导其它 Unit 的执行。它替代了 SysV-init 运行级别的作用，并提供更灵活的基于特定设备事件的启动方式。其实是一群unit 的集合，例如上面表格中谈到的multi-user.target 其实就是一堆服务的集合～也就是说， 选择执行multi-user.target 就是执行一堆其他.service 或/及.socket 之类的服务就是了！
- <font color=blue >timer</font>：用于配置在特定时间触发的任务，替代了 Crontab 的功能

**文件目录**
Unit 文件按照 Systemd 约定，应该被放置指定的三个系统目录之一中。这三个目录是有优先级的，如下所示，越靠上的优先级越高。因此，在三个目录中有同名文件的时候，只有优先级最高的目录里的那个文件会被使用。

- <font color=blue >/etc/systemd/system/</font>：管理员依据主机系统的需求所建立的执行脚本，其实这个目录有点像以前/etc/rc.d/rc5.d/Sxx之类的功能。
- <font color=blue >/run/systemd/system/</font>：系统执行过程中所产生的服务脚本，这些脚本的优先序要比/usr/lib/systemd/system/高！
- <font color=blue >/usr/lib/systemd/system/</font>：每个服务最主要的启动脚本设定，有点类似以前的/etc/init.d底下的档案；

Systemd 默认从目录 /etc/systemd/system/ 读取配置文件。但是，里面存放的大部分文件都是符号链接，指向目录 /usr/lib/systemd/system/，真正的配置文件存放在那个目录。

# Systemd Service Unit

## Unit 文件结构
一般可以使用 <font style='color=blue'>systemctl cat networking</font> 查看 Unit 文件。

``` bash
[Unit]
Description=Hello World
After=docker.service
Requires=docker.service
[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill busybox1
ExecStartPre=-/usr/bin/docker rm busybox1
ExecStartPre=/usr/bin/docker pull busybox
ExecStart=/usr/bin/docker run --name busybox1 busybox /bin/ sh -c "while true; do echo Hello World; sleep 1; done"
ExecStop="/usr/bin/docker stop busybox1"
ExecStopPost="/usr/bin/docker rm busybox1"
[Install]
WantedBy=multi-user.target
```
**<font style='color:red'>`[Unit]`</font>** 区块通常是配置文件的第一个区块，用来定义 Unit 的元数据，以及配置与其他 Unit 的关系。它的主要字段如下。

>- `Description`：简短描述
>- `Documentation`：文档地址
>- `Requires`：当前 Unit 依赖的其他 Unit，如果它们没有运行，当前 Unit 会启动失败
>- `Wants`：与当前 Unit 配合的其他 Unit，如果它们没有运行，当前 Unit 不会启动失败
>- `BindsTo`：与Requires类似，它指定的 Unit 如果退出，会导致当前 Unit 停止运行
>- `Before`：如果该字段指定的 Unit 也要启动，那么必须在当前 Unit 之后启动
>- `After`：如果该字段指定的 Unit 也要启动，那么必须在当前 Unit 之前启动
>- `Conflicts`：这里指定的 Unit 不能与当前 Unit 同时运行
>- `Condition`：当前 Unit 运行必须满足的条件，否则不会运行
>- `Assert`：当前 Unit 运行必须满足的条件，否则会报启动失败


**<font style='color:red'>`[Service]`</font>** 区块用来 Service 的配置，只有 Service 类型的 Unit 才有这个区块。它的主要字段如下。
>- `Type`：定义启动时的进程行为。它有以下几种值。
>- `Type=simple`：默认值，执行ExecStart指定的命令，启动主进程
>- `Type=forking`：以 fork 方式从父进程创建子进程，创建后父进程会立即退出
>- `Type=oneshot`：一次性进程，Systemd 会等当前服务退出，再继续往下执行
>- `Type=dbus`：当前服务通过D-Bus启动
>- `Type=notify`：当前服务启动完毕，会通知Systemd，再继续往下执行
>- `Type=idle`：若有其他任务执行完毕，当前服务才会运行
>- `ExecStart`：启动当前服务的命令
>- `ExecStartPre`：启动当前服务之前执行的命令
>- `ExecStartPost`：启动当前服务之后执行的命令
>- `ExecReload`：重启当前服务时执行的命令
>- `ExecStop`：停止当前服务时执行的命令
>- `ExecStopPost`：停止当其服务之后执行的命令
>- `RestartSec`：自动重启当前服务间隔的秒数
>- `Restart`：定义何种情况 Systemd 会自动重启当前服务，可能的值包括always（总是重启）、on-success、on-failure、on-abnormal、on-abort、on-watchdog
>- `TimeoutSec`：定义 Systemd 停止当前服务之前等待的秒数
>- `Environment`：指定环境变量
>- `EnvironmentFile`：指定加载一个包含服务所需的环境变量的列表的文件，文件中的每一行都是一个环境变量的定义
>- `Nice`：服务的进程优先级，值越小优先级越高，默认为 0。其中 -20 为最高优先级，19 为最低优先级
>- `WorkingDirectory`：指定服务的工作目录
>- `RootDirectory`：指定服务进程的根目录（/ 目录）。如果配置了这个参数，服务将无法访问指定目录以外的任何文件
>- `User`：指定运行服务的用户
>- `Group`：指定运行服务的用户组
>- `MountFlags`：服务的 Mount Namespace 配置，会影响进程上下文中挂载点的信息，即服务是否会继承主机上已有挂载点，以及如果服务运行执行了挂载或卸载设备的操作，是否会真实地在主机上产生效果。可选值为 shared、slaved 或 private
   - shared：服务与主机共用一个 Mount Namespace，继承主机挂载点，且服务挂载或卸载设备会真实地反映到主机上
   - slave：服务使用独立的 Mount Namespace，它会继承主机挂载点，但服务对挂载点的操作只有在自己的 Namespace 内生效，不会反映到主机上
   - private：服务使用独立的 Mount Namespace，它在启动时没有任何任何挂载点，服务对挂载点的操作也不会反映到主机上
>- `LimitCPU` / LimitSTACK / LimitNOFILE / LimitNPROC 等：限制特定服务的系统资源量，例如 CPU、程序堆栈、文件句柄数量、子进程数量等

**<font style='color:red'>`[Install]`</font>** 通常是配置文件的最后一个区块，用来定义如何启动，以及是否开机启动。它的主要字段如下。
>- `WantedBy`：它的值是一个或多个 Target，当前 Unit 激活时（enable）符号链接会放入/etc/systemd/system目录下面以 Target 名 + .wants后缀构成的子目录中
>- `RequiredBy`：它的值是一个或多个 Target，当前 Unit 激活时，符号链接会放入/etc/systemd/system目录下面以 Target 名 + .required后缀构成的子目录中
>- `Alias`：当前 Unit 可用于启动的别名
>- `Also`：当前 Unit 激活（enable）时，会被同时激活的其他 Unit

**Unit 配置文件的完整字段清单，请参考 [官方文档](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)。**

## Unit 管理
<font style='color:blue'>systemctl list-units </font>命令可以查看当前系统的所有 Unit 。
``` bash
# 列出正在运行的 Unit
$ systemctl list-units

# 列出所有Unit，包括没有找到配置文件的或者启动失败的
$ systemctl list-units --all

# 列出所有没有运行的 Unit
$ systemctl list-units --all --state=inactive

# 列出所有加载失败的 Unit
$ systemctl list-units --failed

# 列出所有正在运行的、类型为 service 的 Unit
$ systemctl list-units --type=service

# 查看开机自启动的 service
$ systemctl list-unit-files --type service |grep enabled
```

# Systemd Target

启动计算机的时候，需要启动大量的 Unit。如果每一次启动，都要一一写明本次启动需要哪些 Unit，显然非常不方便。Systemd 的解决方案就是 Target。

简单说，Target 就是一个 Unit 组，包含许多相关的 Unit 。启动某个 Target 的时候，Systemd 就会启动里面所有的 Unit。

传统的init启动模式里面，有 RunLevel 的概念，跟 Target 的作用很类似。不同的是，RunLevel 是互斥的，不可能多个 RunLevel 同时启动，但是多个 Target 可以同时启动。

``` bash
# 查看当前系统的所有 Target
$ systemctl list-unit-files --type=target

# 查看一个 Target 包含的所有 Unit
$ systemctl list-dependencies multi-user.target

# 查看启动时的默认 Target
$ systemctl get-default

# 设置启动时的默认 Target
$ sudo systemctl set-default multi-user.target

# 切换 Target 时，默认不关闭前一个 Target 启动的进程，
# systemctl isolate 命令改变这种行为，
# 关闭前一个 Target 里面所有不属于后一个 Target 的进程
$ sudo systemctl isolate multi-user.target
```

Target 与 传统 RunLevel 的对应关系如下。
``` bash

Traditional runlevel      New target name     Symbolically linked to...

Runlevel 0           |    runlevel0.target -> poweroff.target
Runlevel 1           |    runlevel1.target -> rescue.target
Runlevel 2           |    runlevel2.target -> multi-user.target
Runlevel 3           |    runlevel3.target -> multi-user.target
Runlevel 4           |    runlevel4.target -> multi-user.target
Runlevel 5           |    runlevel5.target -> graphical.target
Runlevel 6           |    runlevel6.target -> reboot.target

```

它与<font style='color:blue'>init</font>进程的主要差别如下。

（1）默认的 RunLevel（在`/etc/inittab`文件设置）现在被默认的 Target 取代，位置是`/etc/systemd/system/default.target`，通常符号链接到graphical.target（图形界面）或者multi-user.target（多用户命令行）。

（2）启动脚本的位置，以前是`/etc/init.d`目录，符号链接到不同的 RunLevel 目录 （比如`/etc/rc3.d`、`/etc/rc5.d`等），现在则存放在`/lib/systemd/system`和`/etc/systemd/system`目录。

（3）配置文件的位置，以前init进程的配置文件是`/etc/inittab`，各种服务的配置文件存放在`/etc/sysconfig`目录。现在的配置文件主要存放在`/lib/systemd`目录，在`/etc/systemd`目录里面的修改可以覆盖原始设置。


# Systemd 管理
Systemd 并不是一个命令，而是一组命令，涉及到系统管理的方方面面。

## systemctl
<font style='color:blue'> systemctl </font> 是 Systemd 的主命令，用于管理系统。

``` bash
# 重启系统
$ sudo systemctl reboot

# 关闭系统，切断电源
$ sudo systemctl poweroff

# CPU停止工作
$ sudo systemctl halt

# 暂停系统
$ sudo systemctl suspend

# 让系统进入冬眠状态
$ sudo systemctl hibernate

# 让系统进入交互式休眠状态
$ sudo systemctl hybrid-sleep

# 启动进入救援状态（单用户状态）
$ sudo systemctl rescue
```
## systemd-analyze
<font style='color:blue'> systemd-analyze </font> 命令用于查看启动耗时。
``` bash
# 查看启动耗时
$ systemd-analyze                                                                                       

# 查看每个服务的启动耗时
$ systemd-analyze blame

# 显示瀑布状的启动过程流
$ systemd-analyze critical-chain

# 显示指定服务的启动流
$ systemd-analyze critical-chain atd.service
```
## hostnamectl
<font style='color:blue'> hostnamectl </font> 命令用于查看当前主机的信息。

``` bash
# 显示当前主机的信息
$ hostnamectl

# 设置主机名。
$ sudo hostnamectl set-hostname rhel7
```

## localectl
<font style='color:blue'> localectl </font> 命令用于查看本地化设置。
``` bash
# 查看本地化设置
$ localectl

# 设置本地化参数。
$ sudo localectl set-locale LANG=en_GB.utf8
$ sudo localectl set-keymap en_GB
```
## timedatectl

<font style='color:blue'> timedatectl </font>命令用于查看当前时区设置。

``` bash
# 查看当前时区设置
$ timedatectl

# 显示所有可用的时区
$ timedatectl list-timezones                                                                                   

# 设置当前时区
$ sudo timedatectl set-timezone America/New_York
$ sudo timedatectl set-time YYYY-MM-DD
$ sudo timedatectl set-time HH:MM:SS
```

## loginctl
<font style='color:blue'> loginctl </font> 命令用于查看当前登录的用户。

``` bash
# 列出当前session
$ loginctl list-sessions

# 列出当前登录用户
$ loginctl list-users

# 列出显示指定用户的信息
$ loginctl show-user ruanyf
```

# 日志管理

Systemd 统一管理所有 Unit 的启动日志。带来的好处就是，可以只用<font  style='color:blue'>`journalctl`</font>一个命令，查看所有日志（内核日志和应用日志）。日志的配置文件是`/etc/systemd/journald.conf`。

``` bash
# 查看所有日志（默认情况下 ，只保存本次启动的日志）
$ sudo journalctl

# 查看内核日志（不显示应用日志）
$ sudo journalctl -k

# 查看系统本次启动的日志
$ sudo journalctl -b
$ sudo journalctl -b -0

# 查看上一次启动的日志（需更改设置）
$ sudo journalctl -b -1

# 查看指定时间的日志
$ sudo journalctl --since="2012-10-30 18:17:16"
$ sudo journalctl --since "20 min ago"
$ sudo journalctl --since yesterday
$ sudo journalctl --since "2015-01-10" --until "2015-01-11 03:00"
$ sudo journalctl --since 09:00 --until "1 hour ago"

# 显示尾部的最新10行日志
$ sudo journalctl -n

# 显示尾部指定行数的日志
$ sudo journalctl -n 20

# 实时滚动显示最新日志
$ sudo journalctl -f

# 查看指定服务的日志
$ sudo journalctl /usr/lib/systemd/systemd

# 查看指定进程的日志
$ sudo journalctl _PID=1

# 查看某个路径的脚本的日志
$ sudo journalctl /usr/bin/bash

# 查看指定用户的日志
$ sudo journalctl _UID=33 --since today

# 查看某个 Unit 的日志
$ sudo journalctl -u nginx.service
$ sudo journalctl -u nginx.service --since today

# 实时滚动显示某个 Unit 的最新日志
$ sudo journalctl -u nginx.service -f

# 合并显示多个 Unit 的日志
$ journalctl -u nginx.service -u php-fpm.service --since today

# 查看指定优先级（及其以上级别）的日志，共有8级
# 0: emerg
# 1: alert
# 2: crit
# 3: err
# 4: warning
# 5: notice
# 6: info
# 7: debug
$ sudo journalctl -p err -b

# 日志默认分页输出，--no-pager 改为正常的标准输出
$ sudo journalctl --no-pager

# 以 JSON 格式（单行）输出
$ sudo journalctl -b -u nginx.service -o json

# 以 JSON 格式（多行）输出，可读性更好
$ sudo journalctl -b -u nginx.serviceqq
 -o json-pretty

# 显示日志占据的硬盘空间
$ sudo journalctl --disk-usage

# 指定日志文件占据的最大空间
$ sudo journalctl --vacuum-size=1G

# 指定日志文件保存多久
$ sudo journalctl --vacuum-time=1years
```




参考：

[systemd 入门教程](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html)
