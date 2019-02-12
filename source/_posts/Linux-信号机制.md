---
title: Linux-信号机制
date: 2019-02-12 10:27:38
tags:
	- Linux
	- IPC
category: Linux
comments: ture
---
`Linux`系统中 `信号机制`是进程间通信的其中一种方法，全称为`软中断信号`，也被称为`软中断`。不过 信号本质上是在软件层面上对硬件中断机制的一种模拟。  
与Linux 系统中其他进程间通信方式(pipe，Shared Memory，Android系统的Binder)相比，信号所能传递的信息比较少，也比较粗糙，只是一个数字。但它也有自己的优点，由于传递的信息量少，所以更便于管理和使用，可以用于系统管理相关的任务，例如通知进程的终结，中止，恢复。  

每一个信号用一个证书常量宏来表示，以 `SIG`来开头,一下是常见的信号量。  

## 常见的信号量类型

|信号量|Value|Desc|demo|
|---|---|---|---|---|
|SIGABRT|6|进程发生错误或者调用了abort()|如在C库函数strlen，如果发生异常会调用abort()
|SIGBUS|7|不存在的物理地址，硬件错误|例如系统错误或硬件出错|
|SIGFPE|8|浮点数运算错误|如除0操作，余0，整数溢出等|
|SIGILL|4|非法指令|损坏的可执行文件或者代码区损坏|
|SIGSEGV|11|段地址错误|空指针，访问不存在的地址空间，访问内核区，对只读空间进行写操作，栈溢出，数组越界，野指针|
|SIGPIPE|13|管道错误，往没有reader的管道中写操作|Linux 中的Socket，如果连接中断了，还继续写的情况下就会触发signal(SIGPIPE,SIG_IGN);|
|SIGHUP|1|终端挂起或控制进程终止。|当用户退出Shell时，由该进程启动的所有进程都会收到这个信号，默认动作为终止进程|
|SIGINT|2|键盘中断|当用户按下<Ctrl+C>组合键时，用户终端向正在运行中的由该终端启动的程序发出此信号。默认动作为终止进程。|
|SIGQUIT|3|键盘退出键被按下|当用户按下<Ctrl+D>或<Ctrl+\>组合键时，用户终端向正在运行中的由该终端启动的程序发出此信号。默认动作为退出程序。|
|SIGKILL|9|无条件终止进程.进程接收到该信号会立即终止，不进行清理和暂存工作。该信号不能被忽略、处理和阻塞，它向系统管理员提供了可以杀死任何进程的方法。|系统管理员无条件杀死任何进行|
|SIGALRM|14|定时器超时，默认动作为终止进程|--|
|SIGTERM|15|程序结束信号，可以由 kill 命令产生。与SIGKILL不同的是，SIGTERM 信号可以被阻塞和终止，以便程序在退出前可以保存工作或清理临时文件等|--|

还可以通过`kill -l` 命令查看系统支持的所有信号: 
 
```shell
$ kill -l
1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL
5) SIGTRAP      6) SIGABRT      7) SIGBUS       8) SIGFPE
9) SIGKILL     10) SIGUSR1     11) SIGSEGV     12) SIGUSR2
13) SIGPIPE     14) SIGALRM     15) SIGTERM     16) SIGSTKFLT
17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU
25) SIGXFSZ     26) SIGVTALRM   27) SIGPROF     28) SIGWINCH
29) SIGIO       30) SIGPWR      31) SIGSYS      34) SIGRTMIN
35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3  38) SIGRTMIN+4
39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12
47) SIGRTMIN+13 48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14
51) SIGRTMAX-13 52) SIGRTMAX-12 53) SIGRTMAX-11 54) SIGRTMAX-10
55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7  58) SIGRTMAX-6
59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```

__注意:不同的Linux发行版支持的信号可能不同。__  

每一种信号都会有一个默认的动作，默认动作就是脚本或程序接收到该信号后所作出的默认操作。常见的默认动作有终止进程，退出程序，忽略信号，重启进程等等。

## 信号的种类 

可以从两个不同的分类角度对信号进行分类：（1）可靠性方面：可靠信号与不可靠信号；（2）与时间的关系上：实时信号与非实时信号。 

#### 不可靠信号与可靠信号

#### 不可靠信号

Linux信号机制基本上是从Unix系统中继承过来的。早期Unix系统中的信号机制比较简单和原始，后来在实践中暴露出一些问题，因此，把那些建立在早期机制上的信号叫做"不可靠信号"，信号值小于SIGRTMIN(Red hat 7.2中，SIGRTMIN=32，SIGRTMAX=63)的信号都是不可靠信号。这就是"不可靠信号"的来源。  
它的主要问题是:  

+ 进程每次处理信号后，就将对信号的响应设置为默认动作。在某些情况下，将导致对信号的错误处理；因此，用户如果不希望这样的操作，那么就要在信号处理函数结尾再一次调用signal()，重新安装该信号。  
+ 信号可能丢失

早期unix下的不可靠信号主要指的是进程可能对信号做出错误的反应以及信号可能丢失。

Linux支持不可靠信号，但是对不可靠信号机制做了改进：在调用完信号处理函数后，不必重新调用该信号的安装函数（信号安装函数是在可靠机制上的实现）。因此，Linux下的不可靠信号问题主要指的是<mark>信号可能丢失</mark>。  

#### 可靠信号  
随着时间的发展，实践证明了有必要对信号的原始机制加以改进和扩充。所以，后来出现的各种Unix版本分别在这方面进行了研究，力图实现"可靠信号"。  

由于原来定义的信号已有许多应用，不好再做改动，最终只好又新增加了一些信号，并在一开始就把它们定义为可靠信号，这些信号支持排队，不会丢失。  

同时，信号的发送和安装也出现了新版本：`信号发送函数sigqueue()`及`信号安装函数sigaction()`。

信号值位于`SIGRTMIN`和`SIGRTMAX`之间的信号都是可靠信号，可靠信号克服了信号可能丢失的问题。Linux在支持新版本的信号安装函数`sigation（）`以及信号发送函数`sigqueue()`的同时，仍然支持早期的`signal（）信号安装函数`，支持`信号发送函数kill()`。

__<mark>注意:</mark>__:  
不要有这样的误解：由`sigqueue()`发送、`sigaction`安装的信号就是可靠的。事实上，可靠信号是指后来添加的新信号（信号值位于`SIGRTMIN及SIGRTMAX`之间）；不可靠信号是信号值小于`SIGRTMIN`的信号。信号的可靠与不可靠只与信号值有关，与信号的发送及安装函数无关。  
目前linux中的signal()是通过sigation()函数实现的，因此，即使通过signal（）安装的信号，在信号处理函数的结尾也不必再调用一次信号安装函数。同时，由signal()安装的实时信号支持排队，同样不会丢失。  

对于目前linux的两个信号安装函数:`signal()及sigaction()`来说，它们都不能把`SIGRTMIN`以前的信号变成可靠信号（都不支持排队，仍有可能丢失，仍然是不可靠信号），而且对`SIGRTMIN`以后的信号都支持排队。这两个函数的最大区别在于，经过`sigaction`安装的信号都能传递信息给信号处理函数（对所有信号这一点都成立），而经过`signal`安装的信号却不能向信号处理函数传递信息。对于信号发送函数来说也是一样的。

#### 实时信号与非实时信号

早期Unix系统只定义了32种信号，每个信号有了确定的用途及含义，并且每种信号都有各自的缺省动作。如按键盘的`CTRL ^C`时，会产生`SIGINT`信号，对该信号的默认反应就是进程终止。之后的信号 都表示 实时信号，等同于前面阐述的`可靠信号`。这保证了发送的多个实时信号都被接收。实时信号是`POSIX标准`的一部分，可用于应用进程。

__非实时信号都不支持排队，都是不可靠信号；实时信号都支持排队，都是可靠信号。__

## 信号机制  

#### 信号的接收
信号的接收工作是由内核来实现的，当内核接收到信号后，会将其放到对应进程的信号队列中，同时向进程发送一个中断，使其陷入内核态。
<mark>这个时候</mark>，虽然进程进入了内核态，但信号还没有被处理，可能在排队。

#### 信号的检测

进程陷入内核态后，有两种场景会对信号进行检测： 

- 进程从内核态返回到用户态前进行信号检测
- 进程在内核态中，从睡眠状态被唤醒的时候进行信号检测

当发现有新信号时，便会进入下一步，信号的处理。  

#### 信号的处理  

`信号处理函数`是运行在用户态的，调用处理函数前，内核会将当前内核栈的内容备份拷贝到用户栈上，并且修改指令寄存器（eip）将其指向信号处理函数。  

接下来进程返回到用户态中，执行相应的`信号处理函数`。  

信号处理函数执行完成后，还需要返回内核态，检查是否还有其它信号未处理。如果所有信号都处理完成，就会将内核栈恢复（从用户栈的备份拷贝回来），同时恢复指令寄存器（eip）将其指向中断前的运行位置，最后回到用户态继续执行进程。  

<mark>注意: 如果自定义信号处理函数，需要提前注册 信号处理函数</mark>  

## 信号处理函数注册

如果进程处理某一个信号，那么久需要在进程中注册(安装)该信号。 信号安装主要用来确定信号值及进程针对该信号值的动作之间的映射关系。即进程要处理哪个信号，该信号传递给进程时，进程执行的操作。

Linux 主要有两个函数实现信号的注册安装: `signal()`.`sigaction()`其中前者 在可靠信号系统调用的基础上实现，是库函数。它只有`两个参数`，不支持信号传递信息，主要是用于`前32种非实时信号的安装`; 而`sigaction()`是较新的函数（由两个系统调用实现：`sys_signal`以及`sys_rt_sigaction`），有三个参数，支持信号传递信息，主要用来与 `sigqueue() 系统调用`配合使用，当然，`sigaction()`同样支持非实时信号的安装。`sigaction()优于signal()`主要体现在支持信号带有参数。  

#### signal() 

```
#include <signal.h> 
void (*signal(int signum, void (*handler))(int)))(int); 
```
`第一个参数`指定信号的值，`第二个参数`指定针对前面信号值的处理，可以忽略该信号（参数设为`SIG_IGN`）; 可以采用`系统默认方式`处理信号(参数设为`SIG_DFL`)；也可以自己实现处理方式(`参数指定一个函数地址`)。    
如果signal()调用成功，返回最后一次为安装信号signum而调用signal()时的handler值；失败则返回SIG_ERR。  

#### sigaction() 

```
#include <signal.h> 
int sigaction(int signum,const struct sigaction *act,struct sigaction *oldact));
```
`sigaction函数`用于改变进程接收到特定信号后的行为。  
该函数的`第一个参数`为信号的值，可以为除`SIGKILL及SIGSTOP`外的任何一个特定有效的信号（**为这两个信号定义自己的处理函数，将导致信号安装错误**）。  
`第二个参数`是指向结构	`sigaction`的一个实例的指针，在`结构sigaction`的实例中，指定了对特定信号的处理，可以为空，进程会以缺省方式对信号处理；  
`第三个参数oldact`指向的对象用来保存原来对相应信号的处理，可指定oldact为NULL。如果把第二、第三个参数都设为NULL，那么该函数可用于检查信号的有效性。  

第二个参数最为重要，其中包含了对指定信号的处理、信号所传递的信息、信号处理函数执行过程中应屏蔽掉哪些函数等等。  

`sigaction`结构定义:  

```
struct sigaction {
         union{
           __sighandler_t _sa_handler;
           void (*_sa_sigaction)(int,struct siginfo *, void *)；
           }_u
                    sigset_t sa_mask；
                   unsigned long sa_flags； 
                 void (*sa_restorer)(void)；
                 }
```

其中，sa_restorer，已过时，POSIX不支持它，不应再被使用。

联合数据结构中的两个元素 **`_sa_handler`** 以及 **`*_sa_sigaction`** 指定信号关联函数，即用户指定的信号处理函数。除了可以是用户自定义的处理函数外，还可以为`SIG_DFL`(采用缺省的处理方式)，也可以为`SIG_IGN`（忽略信号）。   
 
由 **`_sa_handler`** 指定的处理函数只有一个参数,即信号值，所以信号不能传递除信号值之外的任何信息； 由**`_sa_sigaction`**是指定的信号处理函数带有三个参数，是为实时信号而设的（当然同样支持非实时信号），它指定一个3参数信号处理函数。  
第一个参数为信号值，第三个参数没有使用（posix没有规范使用该参数的标准），第二个参数是指向siginfo_t结构的指针，结构中包含信号携带的数据值，参数所指向的结构如下:  

```
siginfo_t {
                 int      si_signo;  /* 信号值，对所有信号有意义*/
                 int      si_errno;  /* errno值，对所有信号有意义*/
                 int      si_code;   /* 信号产生的原因，对所有信号有意义*/
       union{          /* 联合数据结构，不同成员适应不同信号 */  
         //确保分配足够大的存储空间
         int _pad[SI_PAD_SIZE];
         //对SIGKILL有意义的结构
         struct{
             ...
             }...
           ... ...
           ... ...          
         //对SIGILL, SIGFPE, SIGSEGV, SIGBUS有意义的结构
             struct{
             ...
             }...
           ... ...
           }
     }
```

<mark>注：</mark>为了更便于阅读，在说明问题时常把该结构表示为：  

```
siginfo_t {
int      si_signo;  /* 信号值，对所有信号有意义*/
int      si_errno;  /* errno值，对所有信号有意义*/
int      si_code;   /* 信号产生的原因，对所有信号有意义*/
pid_t    si_pid;    /* 发送信号的进程ID,对kill(2),实时信号以及SIGCHLD有意义 */
uid_t    si_uid;    /* 发送信号进程的真实用户ID，对kill(2),实时信号以及SIGCHLD有意义 */
int      si_status; /* 退出状态，对SIGCHLD有意义*/
clock_t  si_utime;  /* 用户消耗的时间，对SIGCHLD有意义 */
clock_t  si_stime;  /* 内核消耗的时间，对SIGCHLD有意义 */
sigval_t si_value;  /* 信号值，对所有实时有意义，是一个联合数据结构，
                          /*可以为一个整数（由si_int标示，也可以为一个指针，由si_ptr标示）*/
     
void *   si_addr;   /* 触发fault的内存地址，对SIGILL,SIGFPE,SIGSEGV,SIGBUS 信号有意义*/
int      si_band;   /* 对SIGPOLL信号有意义 */
int      si_fd;     /* 对SIGPOLL信号有意义 */
}
```

`siginfo_t`结构中的联合数据成员确保该结构适应所有的信号，比如对于实时信号来说，则实际采用下面的结构形式: 

```
typedef struct {
	int si_signo;
	int si_errno;           
	int si_code;            
	union sigval si_value;
} siginfo_t;

```
sigval 也是一个联合数据结构:  

```
union sigval {
    int sival_int;      
    void *sival_ptr;    
    }
```


采用联合数据结构，说明`siginfo_t`结构中的si_value要么持有一个4字节的整数值，要么持有一个指针，这就构成了与信号相关的数据。  

在后面 分析 `sigqueue发送信号` sigqueue的第三个参数就是`sigval`联合数据结构,当调用sigqueue时，该数据结构中的数据就将拷贝到信号处理函数的第二个参数中。这样，在发送信号同时，就可以让信号传递一些附加信息。信号可以传递信息对程序开发是非常有意义的。

信号参数的传递过程 如下: 

调用 sigqueue(pid_t pid, int sig, const union sigval val)   
--->  信号进入队列, 进程进入内核态 
---> 内核接收到 信号之后 ,准备数据，把携带的`sigvall`结构类型的参数 拷贝到 安装信号时 结构体`sigaction`中的 `si_value ` 。  
---> 使进程进入用户态，拷贝数据到用户态，执行信号处理程序   
---> sigaction 中定义的 handler(int,struct siginfo *, void *）

**这就成功的把参数传递到了 信号处理程序。**  
![](/img/sig/sig_install.gif "")  

## 信号的发送  

发送信号的主要函数有:`kill()`、`raise()`、 `sigqueue()`、`alarm()`、`setitimer()` 以及 `abort()`  

#### kill()

```
#include <sys/types.h> 
#include <signal.h> 
int kill(pid_t pid,int signo)
```

参数 pid:  

|pid的值|信号的接收进程|
|---|---|
|pid>0|进程ID为pid的进程接收|
|pid=0|同一个进程组的所有进程|
|pid<0 但 pid!=-1 | 进程组ID为 -pid的所有进程|
|pid=-1|除发送进程自身外，所有进程ID大于1的进程|

参数 signo:  

`sinno`是信号值，当为0时（即空信号），实际不发送任何信号，但照常进行错误检查，因此，可用于检查目标进程是否存在。以及当前进程是否具有向目标发送信号的权限（root权限的进程可以向任何进程发送信号，非root权限的进程只能向属于同一个session或者同一个用户的进程发送信号）。

`Kill()`最常用于pid>0时的信号发送，调用成功返回 0； 否则，返回 -1。 注：对于`pid<0`时的情况，对于哪些进程将接受信号，各种版本说法不一，其实很简单，参阅内核源码`kernal/signal.c`即可，上表中的规则是参考`red hat 7.2`。  

#### raise() 

```
#include <signal.h> 
int raise(int signo) 
```

向进程本身发送信号，参数为即将发送的信号值。调用成功返回 0；否则，返回 -1。  

#### sigqueue()  

```
#include <sys/types.h> 
#include <signal.h> 
int sigqueue(pid_t pid, int sig, const union sigval val) 
```

调用成功返回 0；否则，返回 -1。  
`sigqueue()`是比较新的发送信号系统调用，主要是针对实时信号提出的（当然也支持前32种），支持信号带有参数，与函数`sigaction()`配合使用。  

`sigqueue`的第一个参数是指定接收信号的进程ID，第二个参数确定即将发送的信号，第三个参数是一个联合数据结构`union sigval`，指定了信号传递的参数，即通常所说的`4字节值`。  

```
typedef union sigval {
    int  sival_int;
    void *sival_ptr;
}sigval_t;
```

如前面分析，该数据结构 最后会被传递到 信号处理程序`handler`的第二个参数(`struct siginfo *`)中。

`sigqueue()`比`kill()`传递了更多的附加信息，但`sigqueue()`只能向一个进程发送信号，而不能发送信号给一个进程组。如果`signo=0`，将会执行错误检查，但实际上不发送任何信号，0值信号可用于检查pid的有效性以及当前进程是否有权限向目标进程发送信号。  

在调用`sigqueue`时，`sigval_t`指定的信息会拷贝到`3参数信号处理函数`（3参数信号处理函数指的是信号处理函数由`sigaction`安装，并设定了`sa_sigaction`指针)的`siginfo_t`结构中，这样信号处理函数就可以处理这些信息了。

**注：** `sigqueue（）`发送非实时信号时，第三个参数包含的信息仍然能够传递给信号处理函数； `sigqueue（）`发送非实时信号时，仍然不支持排队，即在信号处理函数执行过程中到来的所有相同信号，都被合并为一个信号。  

#### alarm()  

```
#include <unistd.h> 
unsigned int alarm(unsigned int seconds) 
```

专门为SIGALRM信号而设，在指定的时间seconds秒后，将向进程本身发送SIGALRM信号，又称为闹钟时间。进程调用alarm后，任何以前的alarm()调用都将无效。如果参数seconds为零，那么进程内将不再包含任何闹钟时间。   
返回值，如果调用alarm（）前，进程中已经设置了闹钟时间，则返回上一个闹钟时间的剩余时间，否则返回0。

#### setitimer()  

```
#include <sys/time.h> 
int setitimer(int which, const struct itimerval *value, struct itimerval *ovalue)); 
```
setitimer()比alarm功能强大，支持3种类型的定时器:  

- ITIMER_REAL：	设定绝对时间；经过指定的时间后，内核将发送SIGALRM信号给本进程;
- ITIMER_VIRTUAL 设定程序执行时间；经过指定的时间后，内核将发送SIGVTALRM信号给本进程;
- ITIMER_PROF 设定进程执行以及内核因本进程而消耗的时间和，经过指定的时间后，内核将发送ITIMER_VIRTUAL信号给本进程;

setitimer()第一个参数which指定定时器类型（上面三种之一）；第二个参数是结构itimerval的一个实例，结构itimerval形式见附录1。第三个参数可不做处理。  

setitimer()调用成功返回0，否则返回-1。  

#### abort() 

```
#include <stdlib.h> 
void abort(void);
```

向进程发送SIGABORT信号，默认情况下进程会异常退出，当然可定义自己的信号处理函数。即使SIGABORT被进程设置为阻塞信号，调用abort()后，SIGABORT仍然能被进程接收。  
该函数无返回值。

## 信号机制的系统调用  

信号处理函数运行在用户态，当遇到系统调用、中断或是异常的情况时，程序会进入内核态。信号涉及到了这两种状态之间的转换。

![](/img/sig/sig_interrupt.jpg "")

学习参考:
[IBM-Linux信号上](https://www.ibm.com/developerworks/cn/linux/l-ipc/part2/index1.html)  
[Android平台Native代码的崩溃捕获机制及实现](https://mp.weixin.qq.com/s/g-WzYF3wWAljok1XjPoo7w?)

