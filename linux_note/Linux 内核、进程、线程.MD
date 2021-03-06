# Linux 内核、进程、线程
## linux 内核 kernel
内核 kernel ：</br>
linux内核本质上是一个文件，文件内记录有检测、驱动、控制硬件的程序。内核会在系统读完BIOS和加载好引导程序后被直接加载到内存中。</br>
举个简化的例子，如果玩过单片机的话会比较好理解，比较常用的80C51式通用型单片机，外搭一些电路可实现跑马灯（一串小灯珠以不同的规律亮起来）的功能，制作的时候通常会把一段C语言程序拷到单片机，通过这段C语言程序来控制单片机里的集成电路，进而达到控制单片机输出电压的效果，再搭配外围简单的电路即可实现跑马灯的不同花式显示。linux上的内核就好比单片机里的这段C语言程序，同样起到控制物理硬件的作用。实际上，linux内核也是用C语言编写的。

## 进程 process
进程 process ：</br>
Linux系统中，触发任何一个事件时，系统都会将它定义为一个进程，并给予这个进程一个ID，称为PID。同时，依据触发这个进程的用户的相关属性关系，给予这个PID一组有效的权限设置。每个进程都有三组人的权限rwx，文件所有者权限、同用户组用户权限、其他用户权限，每个文件都其对应的所有者ID（UID）和用户组ID（GID），PID就是根据UID/GID(/etc/passwd)得出的，同一程序每次触发获得的PID不是固定的。进程其实就是一个具有触发者（用户）特征的正在运行的程序program。
</br>
查看进程相关信息</br>
```
$ ps -l
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
0 S  1000  3291  3281  0  80   0 -  2016 do_wai pts/0    00:00:00 bash
0 R  1000  3336  3291  0  80   0 -  2423 -      pts/0    00:00:00 ps
```
bash是ps的父进程，ps其实就是执行ps -l触发的，是bash的子进程。

### 父进程和子进程
分叉 fork-and-exec ：</br>
具体可分为两步</br>
1.fork 分叉</br>
从父进程的当前状态复制出一个分支（暂存进程），这个分支的PPID（父进程ID）为父进程的PID且会有一个独立的PID，其余内容和父进程是一模一样的。</br>
2.exec 执行</br>
之后，暂存进程会以exec的方式加载实际要执行的程序，然后改变程序名，成为一个真正的子进程。</br>
</br>

实际上，当计算机开机的时候，内核(kernel)只建立了一个init进程（systemd）。Linux内核并不提供直接建立新进程的系统调用。剩下的所有进程都是init进程通过fork机制建立的。每个子进程都在内存中分配得属于自己的空间 (address space)，之后父进程空间中的内容会被复制到子进程的空间中，此后两个进程同时运行。</br>
关于Linux 系统中最新的初始化系统（init）systemd ： </br>
https://www.ibm.com/developerworks/cn/linux/1407_liuming_init3/index.html</br>
</br>
一些常用的简单命令如ls -al、rm、mkdir等，它们执行完就结束了，所以PID很快就停止了。也有一些常驻进程，他们一般被称为系统服务daemon，例如crond是开机自启的，它的PID是1223。
```
$ service crond status
Redirecting to /bin/systemctl status crond.service
● crond.service - Command Scheduler
   Loaded: loaded (/usr/lib/systemd/system/crond.service; enabled; vendor preset: enabled)
   Active: active (running) since 六 2018-09-15 19:27:57 CST; 23min ago
 Main PID: 1223 (crond)
   CGroup: /system.slice/crond.service
           └─1223 /usr/sbin/crond -n
```

### 进程管理
#### 进程查看
ps 静态输出进程状态；</br>
top 动态输出进程状态；</br>
pstree 输出进程树。</br>
 
##### 查询操作环境bash中和当前终端有关的进程
```
$ ps -l
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
0 S  1000  3291  3281  0  80   0 -  2016 do_wai pts/0    00:00:00 bash
0 T  1000  5039  3291  3  80   0 -  2850 do_sig pts/0    00:00:00 python3
0 R  1000  5046  3291  0  80   0 -  2423 -      pts/0    00:00:00 ps

# 同一账号X windows下打开第二个terminal
$ ps -l
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
0 S  1000  5200  3281  2  80   0 -  2016 do_wai pts/1    00:00:00 bash
0 R  1000  5244  5200  0  80   0 -  2423 -      pts/1    00:00:00 ps
```

由ps -l可见，对于用户个人，最上层的父进程是bash，而没有扩展到init处。</br>
F：process flags 代表进程权限 </br>
S：status 进程状态 </br>
```
S  sleep 正在睡眠状态idle，可以被唤醒signal；
R  running 正在运行；
T  stop 后台暂停或除错traced状态；
D  不可被唤醒的睡眠状态，通常是进程处于正在等待I/O（阻塞）的状态；
Z  zombie 传说中的“僵尸进程”，进程被终止但无法释放其占用的内存空间，CMD后面标<defunct>。
```
C：CPU占用率</br>
PRI/NI：被CPU执行的优先级，数字越小，优先级越高</br>
ADDR：kernel function 指出进程在内存的哪个部分，如果进程处于running则显示“-”</br>
SZ：占用多少内存</br>
WCHAN：进程运行状态</br>
TTY：登录者终端机位置
```
tty 即tty1-6，当在tty下执行ps -l 就会显示出当前tty的号码，例如X window的TTY就是tty1；
pts 虚拟终端，当在图形化界面下打开一个终端，这个终端叫pts/0，再打开一个终端，这个新的终端就叫pts /1，如果是远程登录的会显示为pts/n。
```
TIME：CPU使用时间

##### 查询系统所有的进程
```
$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.3  27052  3760 ?        Ss   02:12   0:16 /usr/lib/system
root         2  0.0  0.0      0     0 ?        S    02:12   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        S    02:12   0:04 [ksoftirqd/0]
root         5  0.0  0.0      0     0 ?        S<   02:12   0:00 [kworker/0:0H]
# 输出太多，省略部分
sunnyli+  3291  0.0  0.1   8064  1960 pts/0    Ss   14:52   0:00 bash
sunnyli+  5039  0.0  0.4  11400  4552 pts/0    T    21:49   0:00 python3
sunnyli+  5200  0.0  0.2   8064  2500 pts/1    Ss+  22:25   0:00 bash
root      5254  0.1  0.3  11076  3660 ?        Ss   22:31   0:00 login -- sunnylinu
sunnyli+  5268  0.0  0.2   7964  2744 tty2     Ss+  22:31   0:00 -bash
postfix   5349  0.0  0.2  16540  3004 ?        S    22:34   0:00 pickup -l -t unix 
sunnyli+  5384  0.0  0.1   9768  1628 pts/0    R+   22:41   0:00 ps aux
```
%MEM：占用物理内存空间百分比；</br>
VSZ：占用虚拟内存空间KB；</br>
RSS：占用固定内存空间KB；</br>
STAT：同ps -l中的S；</br>
START：进程被触发的时间。</br>
</br>
使用管道命令和正则表达式查询
```
$ ps aux | egrep '(python3|cron)'
root      1223  0.0  0.0   7984   680 ?        Ss   14:47   0:01 /usr/sbin/crond -n
sunnyli+  5039  0.0  0.4  11400  4552 pts/0    T    21:49   0:00 python3
sunnyli+  5429  0.0  0.0   6708   892 pts/0    R+   22:58   0:00 grep -E --color=auto (python3|cron)
```
##### 列出进程树
适用于找进程间的关系
```
$ pstree
systemd─┬─ModemManager───2*[{ModemManager}]
        ├─NetworkManager───2*[{NetworkManager}]
        ├─2*[abrt-watch-log]
        ├─abrtd
        ├─accounts-daemon───2*[{accounts-daemon}]
        ├─alsactl
        ├─at-spi-bus-laun─┬─dbus-daemon───{dbus-daemon}
        │                 └─3*[{at-spi-bus-laun}]
        ├─at-spi2-registr───2*[{at-spi2-registr}]
        ├─atd
        ├─auditd─┬─audispd─┬─sedispatch
        │        │         └─{audispd}
        │        └─{auditd}
# 省略极多输出
```
##### 动态查询进程状态
top可动态查询进程状况，可设置刷新间隔（默认5s）和次数，可根据需求对输出进行排序，可指定查询特定的PID进程，退出top按q。
一般用top内查找最耗费CPU或其他资源的进程，因为它可以排序（按P大写！！）。
```
$ top

# 当前时间，开机持续时间，当前用户数，1/5/15min平均工作负载 
top - 23:34:15 up  8:47,  4 users,  load average: 0.42, 0.18, 0.10
# 进程数量，进程状态
Tasks: 202 total,   3 running, 198 sleeping,   1 stopped,   0 zombie
# CPU整体负载，%wa指I/Owait要注意，多核CPU按1切换显示不同CPU负载
%Cpu(s): 35.3 us, 20.3 sy,  0.0 ni, 44.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
# 物理内存和虚拟内存，虚拟内存用量要尽量地少
KiB Mem :  1027732 total,    78428 free,   683992 used,   265312 buff/cache
KiB Swap:  1048572 total,   962044 free,    86528 used.   136816 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                 
 1399 root      20   0  118352  22596   5676 R 35.0  2.2   4:11.44 X                       
 2380 sunnyli+  20   0 1072824 169672  28616 R 11.9 16.5   3:53.77 gnome-shell             
 3281 sunnyli+  20   0  144308  25456  11480 S  6.9  2.5   0:37.70 gnome-terminal-         
  833 root      20   0   31224   1840   1512 S  0.3  0.2   1:41.20 vmtoolsd                
 1635 mysql     20   0  589928  29832    528 S  0.3  2.9   0:45.20 mysqld                  
 2736 sunnyli+  20   0   64748   5588   3072 S  0.3  0.5   1:37.27 vmtoolsd                
 3189 sunnyli+  20   0   55956   2712   1268 S  0.3  0.3   0:03.99 ibus-daemon             
 5557 sunnyli+  20   0   10236   1932   1340 R  0.3  0.2   0:00.67 top                     
    1 root      20   0   27052   3896   2664 S  0.0  0.4   0:17.70 systemd   
```
top 中的 load average 是系统内所有CPU的 load average，实际生活中我们需要将得到的这个值除以我们的核数来看</br>
平均工作负载： 代表在1,、5、15分钟内，系统平均要负责运行几个进程，数字越小，系统越闲。</br>
load average 有以下几个区间：</br>
* 若是 load = 0 cpu 没有任何任务；
* 若是 load < 0.7 一切都还是很流畅的，cpu 的任务并不多，资源还很充足；
* 若是 0.7< load < 1 的时候我们就需要稍微关注一下了，虽然还可以应付但是这个值已经离临界不远了；
* 若是 load = 1 的时候我们就需要警惕了，因为这个时候已经没有更多的资源的了，已经在全力以赴了；
* 若是 load > 5 的时候系统已经快不行了，需要紧急处理。</br>
通常我们都会先看 15 分钟的值来看这个大体的趋势，然后再看 5 分钟的值对比来看是否有下降的趋势。</br>
```
load average: 0.04, 0.14, 0.25
```
CPU 利用率是对一个时间段内 CPU 使用状况的统计，通过这个指标可以看出在某一个时间段内 CPU 被占用的情况</br>
注意：和 load average 的不同，load average 是在一段时间内 CPU 正在处理以及等待 CPU 处理的进程数情况统计信息
```
%Cpu(s): 35.3 us, 20.3 sy,  0.0 ni, 44.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st

Cpu(s): 1.0%us	用户空间进程占用CPU百分比
1.0% sy	内核空间运行占用CPU百分比
0.0%ni	用户进程空间内改变过优先级的进程占用CPU百分比
97.9%id	空闲CPU百分比
0.0%wa	等待输入输出的CPU时间百分比
0.1%hi	硬中断(Hardware IRQ)占用CPU的百分比
0.0%si	软中断(Software IRQ)占用CPU的百分比
0.0%st	(Steal time) 是 hypervisor 等虚拟服务中，虚拟 CPU 等待实际 CPU 的时间的百分比
```
系统中可用的物理内存最大值并不是 free 这个单一的值，而是 free + buffers + swap 中的 cached 的和
```
KiB Mem :  1027732 total,    78428 free,   683992 used,   265312 buff/cache
KiB Swap:  1048572 total,   962044 free,    86528 used.   136816 avail Mem 

total	物理内存总量
used	使用的物理内存总量
free	空闲内存总量
buffers	用作内核缓存的内存量


total	交换区总量
used	使用的交换区总量
free	空闲交换区总量
cached	缓冲的交换区总量,内存中的内容被换出到交换区，而后又被换入到内存，但使用过的交换区尚未被覆盖
```

在 top 执行过程当中可以使用的按键指令：
```
? ：显示在 top 当中可以输入的按键指令；
P ：以 CPU 的使用资源排序显示；
M ：以 Memory 的使用资源排序显示；
N ：以 PID 来排序；
T ：由该 Process 使用的 CPU 时间累积 (TIME+) 排序；
k ：给予某个 PID 一个讯号  (signal)；
r ：给予某个 PID 重新制订一个 nice 值；
q ：离开 top 软件的按键。
```
```
$ top -d 2 -p $$  # d参数修改刷新时间, p指定PID，此处$$表示bash的进程号
  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                 
 3291 sunnyli+  20   0    8064   1964   1032 S  0.0  0.2   0:01.13 bash  
```
```
$ ps -l|egrep 'python3'
0 T  1000  5039  3291  0  80   0 -  2850 do_sig pts/0    00:00:00 python3
$ top -d 2 -p 5039
  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                 
 5039 sunnyli+  20   0   11400   4552   2560 T  0.0  0.4   0:00.32 python3    
```
#### 进程优先级
和进程优先级有关的两个参数是PRI和NI：</br>
PRI由内核动态决定，PRI越小，优先级越高；</br>
NI可由用户决定，root NI范围为 -20到19，一般用户NI范围0到19且仅能把NI值调高；</br>
PRI_new = PRI_old + NI，但是不是绝对的，最终PRI值还是由系统决定的，NI值会从父进程传递给子进程。</br>

##### 调整NI且触发进程
```
nice [-n NI数值] command
```
注意：有参数-n且注意后面NI值的符号，如果是负数必须root权限</br>
```
# 修改vi的NI且让vi在后台执行，修改前PRI为80
# nice -n -5 vi &
[1]6204  # [1]表示vi的job number，6204为PID
# ps -l
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
4 T     0  6103  3291  0  80   0 -  5379 do_sig pts/0    00:00:00 sudo
4 S     0  6147  3291  0  80   0 -  3759 do_wai pts/0    00:00:00 su
4 S     0  6155  6147  0  80   0 -  1984 do_wai pts/0    00:00:00 bash
4 T     0  6204  6155  0  75  -5 -  1928 do_sig pts/0    00:00:00 vi
0 R     0  6211  6155  0  80   0 -  2423 -      pts/0    00:00:00 ps
# 通过PID杀掉vi进程，-9为signal中SIGKILL的代号，强制中断进程
# kill -9 6204
[1]+  已杀死               nice -n -5 vi
```

##### 修改已有进程的NI
```
renice [NI数值] PID
```
注意：子进程NI值会跟随父进程</br>
```
# 修改bash的NI
# renice 2 6155
6155 (进程 ID) 旧优先级为 0，新优先级为 2
# ps -l
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
4 T     0  6103  3291  0  80   0 -  5379 do_sig pts/0    00:00:00 sudo
4 S     0  6147  3291  0  80   0 -  3759 do_wai pts/0    00:00:00 su
4 S     0  6155  6147  0  82   2 -  2004 do_wai pts/0    00:00:00 bash
0 R     0  6250  6155  0  82   2 -  2423 -      pts/0    00:00:00 ps
```
#### 杀死进程
##### 杀死单个进程
```
kill -signal PID或%jobnumber
```
Signal 内容：
```
1   SIGHUP	
启动被终止的程序，可让该 PID 重新读取自己的设定档，类似重新启动；
2   SIGINT	
相当于用键盘输入 [ctrl]-c 来中断一个程序的进行；
9   SIGKILL	
代表强制中断一个程序的进行，如果该程序进行到一半， 那么尚未完成的部分可能会导致半产品产生，如vim会有 .filename.swp 保留下来；
15  SIGTERM	
以正常的结束程序来终止该程序。由于是正常的终止， 所以后续的动作会将他完成。不过，如果该程序已经发生问题，就是无法使用正常的方法终止时， 输入这个 signal 也是没有用的；
19  SIGSTOP	
相当于用键盘输入 [ctrl]-z 来暂停一个程序的进行。
```
注意：1用于重启，9用于在不正常状态下关闭中断进程，15让进程跑完之后正常关闭。
```
# ps aux|egrep 'vi'
sunnyli+  6036  0.0  0.1   7712  1372 pts/0    T    07:57   0:00 vi
# 杀掉vi进程
# kill -9 6036
```
##### 杀死由某个命令触发的多个进程（服务）
```
killall -signal command
```
注意：被处理的对象是命令，不是PID，被kill的是命令所触发的多个进程
```
# killall -9 python3
[1]+  已杀死               python3
```
#### 前台后台工作管理
后台执行 command & </br>
暂停 [ctrl]+[z]</br>
查看后台状态 jobs -l</br>
将后台进程调到前台   fg %jobnumber</br>
让暂停进程在后台run  bg %jobnumber</br>
删除后台进程 kill -signal %jobnumber</br>
```
# vim vitest_man.conf 
（vi一般模式下按 [ctrl]+[z]）
[1]+  已停止               vim vitest_man.conf
# jobs -l
[1]+  6786 停止                  vim vitest_man.conf
# fg %1
```

### linux多用户、多任务环境
linux的进程设计能很好地利用CPU的性能，每一个任务(进程)被创建时，系统会为他分配存储空间等必要资源，然后在内核管理区为该进程创建管理节点，以便后来控制和调度该任务的执行。进程真正进入执行阶段，系统会给进程分配必要的CPU资源，这个行为被称为“调度”。除CPU而外，系统会为每个进程分配独有的存储空间，还有其他外设的可使用状态等。即是子进程和父进程一样，有它自己独立的硬件资源。</br>
</br>
Linux默认启动6个终端机登陆环境进程，[Alt]+[F1] X-windows [Alt]+[F2-7] tty1-tty6，可以根据实际需要调整数量。linux上，如果某个进程卡死了，可以从当前的tty转到其他tty，然后ps -aux 找出出错的进程并kill了它，再返回到原来的tty就可以了。尽管现在CPU性能已经很好，可以支持不同进程间的快速切换，但由于系统的服务会不断升级，所以总有一天还是会耗尽PC的CPU和RAM资源。</br>
</br>
在一个使用分叉的服务器中，每一个客户端机连接都利用分叉创造一个子进程。父进程继续监听新的连接，同时子进程处理客户端。当客户端的请求结束时，子进程就退出了。因此分叉的进程是并行运行的，客户端之间不必互相等待。Window是不支持分叉的，但是现代的linux/Unix就支持得很好，但是因为每个进程都要求有自己的独立内存，所以以进程的方式实现多连接会很耗费资源，所以有了线程这一概念。Win比较着重多线程，linux比较着重分叉（多进程）。</br>
</br>
## 线程  thread 或Lightweight Process
线程 thread或 Lightweight Process ：</br>
线程 thread 或 Lightweight Process，简称LWP，轻量级进程的意思。在Unix SystemV及SunOS中也被称为轻量进程（lightweight processes），但轻量进程更多指内核线程(kernel thread)，而把用户线程(user thread)称为线程。计算机科学术语，指运行中的程序的调度单位。</br>
</br>
进程是资源分配的基本单位，线程是调度的基本单位。进程和线程都可以被调度，只是线程是更小的可以调度的单位而且线程不单独占有系统的资源，而是多个线程共享一个进程中的资源。一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并行多个线程，每条线程并行执行不同的任务。</br>
</br>
当每个 CPU 核心运行一个进程的时，由于每个进程的资源都独立， CPU 核心之间切换的时候无需考虑上下文。当每个 CPU 核心运行一个线程的时，由于每个线程需要共享资源，所以这些资源必须从 CPU 的一个核心被复制到另外一个核心，才能继续运算，这占用了额外的开销。因此，在 CPU 为多核的情况下，多线程在性能上不如多进程，所以当前面向多核的服务器端编程中多为多进程而非多线程。</br>
