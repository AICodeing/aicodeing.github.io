---
title: Linux-共享内存(Shared Memory)
date: 2019-02-24 12:00:49
tags:
	- Linux
	- IPC
category: Linux
comments: ture
---

之前[Linux-信号机制](/Linux-信号机制/)这篇文章介绍了 `Linux`系统中的一个进程间通信方式- `信号机制`。这次我们来介绍以下 另一种进程间通信方式-`共享内存`。  

共享内存的原理是: __两个进程可以直接访问同一块内存区域。__ 由于两个进程交互的数据在同一块内存区域，避免了 copy 操作，所以速度是比较快的。  

我们知道之所以划分进程这一概念，就是为了各个进程运行在各自独立的内存空间之中，避免了进程的非法访问和数据破坏。所以，共享内存肯定不是简单的，两个进程直接分配同一块内存区域。 它所占用的空间既不属于进程A 也不属于进程B，而是属于系统内核。 需要使用到 内存映射，将块所属内核的内存区域映射到两个进程之中。  如下图:  

![](/img/share_mem/share_mem.jpg)

实现共享内存比较简单，只需要厦门几步:  

1. 创建内存共享区
2. 映射内存共享区到进程空间
3. 访问共享内存
4. 对共享内存进行读写操作，也即进行进程间通信
5. 撤销共享内存的进程映射
6. 删除共享内存区域，回收内存

#### 1. 创建内存共享区

进程A 通过操作系统提供的API 从内存中申请一块共享区域，在Linux 环境中，可以通过 `shmget`函数来创建或获取共享内存区域  

```
#include <sys/ipc.h>
#include <sys/shm.h>

/*
 * key：SHM 标识
 * size：SHM 大小
 * shmflg：创建或得到的属性，例如 IPC_CREAT
 * return：成功返回 shmid，失败返回 -1，并设置 erron
 */
int shmget(key_t key, size_t size, int shmflg);
```

__参数 `key`__: 创建的内存区域会和给定的 `key`进行绑定，另外一个进程B 可以通过传入相同的 `key`来获取进程A创建的共享内存区域。在 以下两种key 的取值情况下，会创建一个新的内存共享区，否则就是返回已有的内存共享区  

```
1. key 值为 `IPC_PRIVATE`  
2. key不为 `IPC_PRIVATE` ，但是 另一个参数`shmflg `指定了 `IPC_CREATE`标记
```
__参数 `size`__: 指定申请的共享内存的大小，以字节为单位， <mark>注意:</mark>Linux系统下，分配的内存大小都是页的整数倍

__参数shmflg__: 有以下几种取值   

```
IPC_CREATE：申请新建区域
IPC_EXCL：和 IPC_CREATE共同使用，如果指定的区域已经存在，则返回错误。
mode_flags：同 open 函数的 mode 参数，用来指定 文件的权限
SHM_HUGETLB：使用"huge pages"机制来申请
SHM_NORESERVE: 此区域不保留 swap 空间
```
对于 进程A要创建一个共享内存区域，参数 shmflg 设置为 `IPC_CREATE `即可。

__返回值__: 是内存共享区域的id值，用于唯一标识该区域。进程需要映射该区域时 需要使用 此 id 值。

#### 2.映射内存共享区到进程空间

将进程A申请创建的共享内存映射到进程A的进程空间中，需要使用 `shmat`函数  

```
#include <sys/types.h>
#include <sys/shm.h>
#include <sys/ipc.h>

/*
 * shmid：SHM ID
 * shmaddr：SHM 内存地址
 * shmflg：SHM 权限
 * return：成功返回 SHM 的地址，失败返回 (void *) -1，并设置 erron
 */
void *shmat(int shmid, const void *shmaddr, int shmflg);
```

__参数 shmid__: 就是 共享内存区域的id值，由`shmget`函数返回的。  

__参数 shmaddr__: 将内存共享区域映射到指定的地址，可以为 `0`,此时系统将自动分配地址  

__参数shmflg__: 和 `shmget`方法中的 参数`shmflg` 一样。

__返回值__: 如果成功执行后，返回该内存区域的其实地址

#### 3.访问共享内存

经过前面 2 步，进程A 成功创建了 共享内存，并将共享内存区域映射到了 进程A的进程空间之中，那么现在 就该进程B 访问 共享内存区域了。进程B 就是利用 进程A 创建 内存空间时绑定的`key`,通过函数`shmget`来 获取 内存共享区域，然后再使用 `shmat`函数将共享内存区域也映射到进程B的空间中。

#### 4.进行进程间通信

这一步就是进程A或进程B 往 共享内存区域写入自己的信息，实现数据交换或者说实现通信。<mark>注意</mark>: 如果涉及到进程A和进程B 写入数据同步问题，还需要 进程A和进程B协议处理，因为 共享内存没有同步机制。往共享内存区域 copy 数据可以使用 函数`memcpy`。

```
#include <stdio.h>

/*
 * dest: 指向用于存储复制内容的目标地址
 * src: 指向要复制的数据源
 * count: 要被复制的字节数
 * return：返回一个指向目标存储区 dest 的指针
 */
void *memcpy(void *dest, const void *src, size_t count)
```

#### 5.撤销内存映射

Linux中 使用 函数`shmdt`来解除当前进程与共享内存区域的映射关系。

```
#include <sys/types.h>
#include <sys/shm.h>
#include <sys/ipc.h>

/*
 * shmaddr：已经映射的 SHM 地址
 * return：成功返回 0，失败返回 -1，并设置 erron
 */
int shmdt(const void *shmaddr);
```

#### 6.删除共享内存区

经过第5步骤取消了 进程A和进程B 对共享内存的映射关系，如果不需要再次使用的话就可以释放掉 这块共享内存区域了，在Linux 系统中，可以通过 函数`shmctl`来实现。  

```
#include <sys/shm.h>

/*
 * shmid：SHM ID
 * cmd: 控制命令
 * buf: 共享内存区域需要更新的数据，或需要写出的数据
 * return：成功返回 0，否则 失败
 */
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```

__参数 cmd__: 释放掉共享内存区域时需要执行的命令，可选值如下:  

```
IPC_STAT：状态查询，结果存入 参数buf
IPC_SET: 在权限允许的情况下，将共享内存状态更新为 buf 中的数据
IPC_RMID: 删除共享内存段
```

### 示例

`write_shm.c` 

进程 A 创建 共享内存区域 并写入数据

```
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/types.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

int main() {
	// 1. 创建 SHM
	int shm_id = shmget(13, 2048, IPC_CREAT | 0666);
	if (shm_id != -1) {
		// 2. 映射 SHM
		void* shm = shmat(shm_id, NULL, 0);
		if (shm != (void*)-1) {
			// 3. 写 SHM
			char str[] = "share memory";
			memcpy(shm, str, strlen(str) + 1);
			// 4. 关闭 SHM
			shmdt(shm);
		} else {
			perror("shmat:");
		}
	} else {
		perror("shmget:");
	}
	return 0;
}
```

进程B 从共享内存中读取数据  

`read_shm.c`

```
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/types.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

int main() {
	// 1. 获取 SHM
	int shm_id = shmget(13, 2048, IPC_CREAT | 0666);

	if (shm_id != -1) {
		// 2. 映射 SHM
		void* shm = shmat(shm_id, NULL, 0);
		if (shm != (void*)-1) {
			// 3. 读取 SHM
			char str[50] = { 0 };
			memcpy(str, shm, strlen("share memory"));
			printf("shm = %s\n", (char *)shm);
			// 4. 关闭 SHM
			shmdt(shm);
		} else {
			perror("shmat:");
		}
	} else {
		perror("shmget:");
	}
	if (0 == shmctl(shm_id, IPC_RMID))
		printf("delete shm success.\n");
	return 0;
}
```

使用 `gcc ` 命令将 文件 `write_shm.c` 和 `read_shm.c` 编译成可执行文件  

```shell
gcc write_shm.c -o write_shm
gcc read_shm.c -o read_shm
```

先运行 `write_shm` 在运行 `read_shm`, `read_shm`执行后会输出 结果 
`share memory`。 这就意味着 我们成功的通过 共享内存实现了进程A 和进程B的通信。
