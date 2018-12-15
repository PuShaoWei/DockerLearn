# Docker的内核知识

`docker` 本质上是宿主机上的进程。`docker`通过`namespace`实现了资源隔离，通过`cgroups`实现了资源限制，通过写时复制机制`copy-on-write`实现了高效的文件操作


### namespace 资源隔离

Linux 内核中提供了6种namespace隔离的系统调用，这里一半是针对Linux内核3.8及以后，因为user namespace在3.8后才支持

namespace|系统调用参数|隔离内容
---|---|---
UTS|CLONE_NEWUTS|主机名与域名
IPC|CLONE_NEWIPC|信号量、消息队列和内存共享
PID|CLONE_NEWPID|进程编号
Network|CLONE_NEWNET|网络设备、网络栈、端口
Mount|CLONE_NEWS|挂载点（文件系统）
User|CLONE_NEWUSER|用户和用户组

#### 进行namespace API 操作的4种方式

`namespace`的API包括`clone()`,`stens()`,`unshare()`，还有`/proc`下的文件

1. 通过`clone()`创建新的进程的同时创建`namespace`

```c
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

- `child_func`是函数指针，我们知道进程的4要素，这个就是指向程序的指针，就是所谓的`剧本`，传入子进程运行的程序主函数
- `child_stack` 是为子进程分配系统堆栈空间（在linux下系统堆栈空间是2页面，就是8K的内存，其中在这块内存中，低地址上放入了值，这个值就是进程控制块task_struct的值）
- flags 就是标志用来描述你需要从父进程继承那些资源， 
- arg 传入可用于用户的参数 

标志| 含义
---|---
CLONE_PARENT|  创建的子进程的父进程是调用者的父进程，新进程与创建它的进程成了“兄弟”而不是“父子”
CLONE_FS    |   子进程与父进程共享相同的文件系统，包括root、当前目录、umask
CLONE_FILES |  子进程与父进程共享相同的文件描述符（file descriptor）表
CLONE_NEWNS | 在新的namespace启动子进程，namespace描述了进程的文件hierarchy
CLONE_SIGHAND | 子进程与父进程共享相同的信号处理（signal handler）表
CLONE_PTRACE|  若父进程被trace，子进程也被trace
CLONE_VFORK |   父进程被挂起，直至子进程释放虚拟内存资源
CLONE_VM |   子进程与父进程运行于相同的内存空间
CLONE_PID| 子进程在创建时PID与父进程一致
CLONE_THREAD  | Linux 2.4中增加以支持POSIX线程标准，子进程与父进程共享相同的线程群

下面的例子是创建一个线程（子进程共享了父进程虚存空间，没有自己独立的虚存空间不能称其为进程）。
父进程被挂起当子线程释放虚存资源后再继续执行。
与系统调用clone功能相似的系统调用有fork,但fork事实上只是clone的功能的一部分，clone与fork的主要区别在于传递了几个参数，而当中最重要的参数就是conle_flags,下表是系统定义的几个clone_flags标志：

标志 |Value| 含义
---|---|---
CLONE_VM |0x00000100 | 置起此标志在进程间共享地址空间
CLONE_FS |0x00000200 | 置起此标志在进程间共享文件系统信息
CLONE_FILES| 0x00000400| 置起此标志在进程间共享打开的文件
CLONE_SIGHAND |0x00000800| 置起此标志在进程间共享信号处理程序

如果置起以上标志所做的处理分别是置起CLONE_VM标志： 

```c
mmget(current->mm); 
/* 
* Set up the LDT descriptor for the clone task. 
*/ 
copy_segments(nr, tsk, NULL); 
SET_PAGE_DIR(tsk, current->mm->pgd); 
置起CLONE_ FS标志： 
atomic_inc(¤t->fs->count); 
置起CLONE_ FILES标志： 
atomic_inc(&oldf->count); 
置起CLONE_ SIGHAND标志： 
atomic_inc(¤t->sig->count); 
```

2. 查看 /proc/[pid]/ns 文件

```shell
[pushaowei@izm5ej1n0xcwfs3iuh67o5z ~]$ ls -l /proc/$$/ns # $$ 在shell中表示当前运行的进程ID号
总用量 0
lrwxrwxrwx 1 pushaowei pushaowei 0 12月 15 16:19 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 pushaowei pushaowei 0 12月 15 16:19 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 pushaowei pushaowei 0 12月 15 16:19 net -> net:[4026531956]
lrwxrwxrwx 1 pushaowei pushaowei 0 12月 15 16:19 pid -> pid:[4026531836]
lrwxrwxrwx 1 pushaowei pushaowei 0 12月 15 16:19 user -> user:[4026531837]
lrwxrwxrwx 1 pushaowei pushaowei 0 12月 15 16:19 uts -> uts:[4026531838]
```

如果两个进程只想的`namespace`编号相同，就说明它们在同一个`namespace`下，否在便在不同的`namespace`里面。

3. 通过`setns()`加入一个已经存在的`namespace`
  在进程都结束的情况下， 也可以通过挂载的形式把`namespace`保留下来，保留`namespace`的目的是否了以后有进程加入做准备。
  在`docker`中使用`docker exec`命令在已经运行着的容器中执行一个新的命令，就需要用到该方法.
  为了不影响进程的调用者，也为了使新加的`pid namespace`生效，会在`setns`函数执行后使用`clone`创建子进程继续执行命令

```c
int stens(int fd,int nstype);
```
- `fd`表示要加入`namespace`的文件描述符，它是一个指向`proc/[pid]/ns`目录的文件描述符
- `nstype`让调用者可以检查fd 指向的namespace类型是否符合实际要求，为0表示不检查

为了把新加入的`namespace`利用起来，需要引入`execve()`系列函数，该函数可以执行用户命令，最宠用的就是调用`/bin/bash`并接受参数。运行一个`shell`

```c
fd = open(argv[1],0_RDONLY); // 获取 namespace 文件描述符
setns(fd,0); // 加入新的namespace
execvp(argv[0],&argv[2]); // 执行程序
```

假设编译后的程序名叫 `sentns-ts`

```bash
./sentns-ts ~/uts /bin/bash # ～/uts 是绑定的 /proc/[pid]/ns/uts
```

至此就可以在新加入的namespace中执行shell

4. 通过`unshare()`在原先进程上执行namespace隔离

最后要说明的系统调用时`unshare()`，他与`clone()`很像，不同的是，`unshare()`运行在原先的进程上，不需要启动一个新进程

```c
int unshare(int flags)
```

调用 `unshare()`主要作用就是，不启动新进程就可以起到隔离效果，相当于跳出原先的`namespace`进行操作，这样就可以在原进程上进行一些需要的隔离操作


####  巩固一下`fork()`系统调用

当程序调用`fork()`函数时，系统会创建新的进程，为其分配资源，例如存储数据和代码的空间，然后把原来进程的所有值都复制到新进程中，只有少量数值与原来的进程值不同，相当于复制了本身

`fork()`的神奇之处在于仅仅被调用一次，却能返回两次（父进程与子进程各返回一次）通过返回值的不同就可以区分父进程和子进程

```c
int fork()
```

- 在父进程中，`fork()`返回新创建的子进程ID
- 在子进程中，`fork()`返回0
- 如果出现错误，`fork()`返回一个负值

```bash
cat << EOF > fork_example.c
#include <unistd.h>
#include <stdio.h>
int main()
{
  pid_t fpid; //fpid 表示`fork`函数返回值
  fpid=fork();
  if(fpid < 0 ){
    printf("error in fork! \n");
  }else if(fpid == 0){ 
    printf("I am child. process id is %d \n",getpid());
  }else{
    printf("i am parent, process id is %d \n",getpid());
  }
  return 0;
}
EOF
```

验证`fork`调用结果

```bash
$ gcc -Wall fork_example.c && ./a.out # a.out 是linux/unix环境下gcc编译源代码(c/c++)并连接产生的默认执行文件名

i am parent, process id is 10521
I am child. process id is 10522
```

> 使用完`fork`后，父进程有义务监控子进程的运行状态，并在子进程推出后自己才能正常退出，否则子进程就会成为孤儿进程

代码执行过程中，在`fpid=fork()`之前，只有一个进程在执行这段代码，在这条语句之后就变成父进程和子进程同时执行了，这两个进程几乎完全相同，将要执行的下一句语句都是`if(fpid<0)`，同时`fpid=fork()`的返回值会一句所属进程返回不同的值


Unix标准的复制进程的系统调用时fork（即分叉），但是Linux，BSD等操作系统并不止实现这一个，确切的说linux实现了三个，fork,vfork,clone（确切说vfork创造出来的是轻量级进程，也叫线程，是共享资源的进程）

####  fork，vfork，clone

Unix标准的复制进程的系统调用时fork（即分叉），但是Linux，BSD等操作系统并不止实现这一个，确切的说linux实现了三个，fork,vfork,clone（确切说vfork创造出来的是轻量级进程，也叫线程，是共享资源的进程）

系统调用|  描述
---|---
fork  |fork创造的子进程是父进程的完整副本，复制了父亲进程的资源，包括内存的内容task_struct内容
vfork |vfork创建的子进程与父进程共享数据段,而且由vfork()创建的子进程将先于父进程运行
clone| Linux上创建线程一般使用的是pthread库 实际上linux也给我们提供了创建线程的系统调用，就是clone
fork

#### UTS namespace

 `UTS(UNIX Time-sharing System) namespace` 提供了主机名和域名的隔离，这样每个`Docker`容器就可以拥有独立的主机名和域名了，在网络上可以被视作一个独立的节点，而非宿主机上的一个进程。
  `Docker`中，每个镜像基本都以自身提供的服务名称来命名镜像的`hostname`且不会对宿主机产生任何影响，其原理就是`UTS namespace` 

```bash
cat << EOF > uts_clone.c
#define _GUN_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <stdlib.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE];
char* const child_args[] = {
    "/bin/bash",
    NULL
};

int child_main(void* args){
  printf("在子进程中！\n");
  execv(child_args[0],child_args);
  return 1;
}

int main()
{
  printf("程序开始运行！\n");
  int child_pid = clone(child_main, child_stack + STACK_SIZE, SIGCHLD, NULL);
  waitpid(child_pid,NULL,0);
  printf("已经退出！\n");
  return 0;
}
EOF
```  

验证效果

```bash
[pushaowei@izm5ej1n0xcwfs3iuh67o5z ~]$ gcc -Wall uts_clone.c && ./a.out
程序开始运行！
在子进程中！
[pushaowei@izm5ej1n0xcwfs3iuh67o5z ~]$ exit
exit
已经退出！
```
将上面代码改一改，加入`UTS`隔离，运行代码需要root权限，以防止普通用户任意修改系统主机名导致`set-user-ID`相关运行错误


```c
cat << EOF > uts_example.c
#define _GUN_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
#include <stdlib.h>

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE];
char* const child_args[] = {
    "/bin/bash",
    NULL
};

int child_main(void* args){
  printf("在子进程中！\n");
  sethostname("NewNamespace",12); // here
  execv(child_args[0],child_args);
  return 1;
}

int main()
{
  printf("程序开始运行！\n");
  int child_pid = clone(child_main, child_stack + STACK_SIZE, CLONE_NEWUTS|SIGCHLD, NULL);
  waitpid(child_pid,NULL,0);
  printf("已经退出！\n");
  return 0;
}
EOF
```

验证执行结果

```bash
[root@izm5ej1n0xcwfs3iuh67o5z pushaowei]# gcc -Wall uts_example.c && ./a.out
程序开始运行！
在子进程中！
[root@NewNamespace pushaowei]# exit
exit
已经退出！
[root@izm5ej1n0xcwfs3iuh67o5z pushaowei]#
```

#### IPC namespace

