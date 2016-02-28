---
layout: post
title: Linux File System Basic - 3
description: Linux file system(文件系统)模块的实现和基本数据结构。关键字：文件系统，内核，samplefs，VFS，存储。
categories:
- [Chinese, Software]
tags:
- [file system, crash, kernel, linux, storage]
---

>本文首发于 <http://oliveryang.net>，转载时请包含原文或者作者网站链接。

## 文件系统 mount 和 Super Block

Samplefs Day2 的代码涉及到了文件系统 mount 和 Super Block (超级块)的实现。
本文将以 [Day2 的代码](https://github.com/yangoliver/lktm/tree/master/fs/samplefs/day2)
为例，讲解相关概念。

### 1. Samplefs Day2

#### 1.1 源代码

与 Day1 相比，Day2 的实现增加了下面几个函数，

- samplefs_fill_super: 初始化 VFS Super Block。

  该回调在文件系统被 mount 时被调用。

  mount 在VFS文件系统的调用路径以系统调用 `sys_mount` 为起点，路径如下(内核版本3.19)，

	  sys_mount->do_mount->do_new_mount->vfs_kern_mount->mount_fs

  在 VFS 层面，由于 samplefs 早已在模块加载时就向其注册了文件系统类型，
  因此 VFS 可以很方便查找到 samplefs 在 samplefs_fs_type 里注册的入口函数 samplefs_mount，
  而 samplefs_mount 使用了 mount_nodev 方法，并把 samplefs_fill_super 回调作为参数传递给 mount_nodev,

	  samplefs_mount->mount_nodev

  mount_nodev 分配了新的 VFS Supoer Block 然后调用 samplefs_fill_super 回调做了如下几件事情，

  1. 初始化了由 mount_nodev 分配好并传入的 VFS Super Block。
     其中把 Super Block 的操作表 samplefs_super_ops 赋值给了 struct super_block 的 s_op 成员。
	 而在 samplefs_super_ops 初始化好了 samplefs_put_super 回调函数用于未来释放 samplefs 自己的 Super Block。
  2. 分配 root inode。
  3. 分配了属于 samplefs 模块的内存 Super Block: samplefs_sb_info，并让它在 VFS 层的 Super Block 指向它。
  4. 根据 root inode，分配 root dentry，作为 mount_nodev，也是 samplefs_mount 的最终返回值。
  5. 使用 load_nls_default() 函数初始化 samplefs 模块的内存 Super Block。
     主要用于 mount 时对不同编码字符集的支持, Linux NLS Kconfig 里有对 Native language support 的说明。
  6. 调用 samplefs_parse_mount_options 来解析 mount 时的选项参数。

- samplefs_parse_mount_options: 解析 mount 文件系统时的选项参数。

  这个函数的实现比较简单，值得说明的有两点，

  1. mount 时的选项参数是在 sys_mount 系统调用时从用户空间拷贝到内核内存中，
     再由 VFS 的代码通过 samplefs 的 mount 入口函数传入进来的。

  2. 解析后的选项参数保存在了 samplefs 模块的 Super Block：samplefs_sb_info 里。
     Samplefs 的 VFS Super Block 结构指向这个结构。

- samplefs_put_super: 释放 samplefs 模块的内存 Super Block。

  smaplefs 在mount的时候一共创建两个内存 Super Block，

  1. 存在于 samplefs 模块这层的内存 Super Block: `struct samplefs_sb_info`

     这个超级块是由 samplefs_fill_super 在 mount 时分配的，因此也正是由 samplefs_put_super 这个函数在 umount 时释放的。

     必须注意的是，samplefs_put_super 是 VFS 定义的标准回调函数，在 `struct super_operations` 里定义的，
     是 VFS Super Block 的标准方法。

  2. 存在于 VFS 层的 Super Block: `struct super_block`

     这个超级块在 mount 文件系统时，由 samplefs_mount 调用 mount_nodev 时由 VFS 的代码分配。
     而释放则是在 umount 命令触发调用 sys_umount 系统调用来释放的。

     在2.6内核，sys_umount 直接在当前上下文一直调用到 deactivate_super 来释放掉 VFS Super Block。

     而在3.19内核，sys_umount 则在调用 mntput_no_expire 时引入了异步执行的逻辑，
     把释放 VFS Super Block的任务交给另外一个线程去做。但如果 MNT_INTERNAL 标志被置位，
     则意味着 umount 是从内核态发起的，对内核态发起的 umount 则仍旧使用当前上下文，
     即同步的方式去释放 VFS Super Block。下面的代码片段就来自 mntput_no_expire，

         if (likely(!(mnt->mnt.mnt_flags & MNT_INTERNAL))) { /* 不属于 MS_KERNMOUNT 的方式 */
             struct task_struct *task = current;
             if (likely(!(task->flags & PF_KTHREAD))) { /* 不是内核线程，要返回用户态 */
                init_task_work(&mnt->mnt_rcu, __cleanup_mnt);
                if (!task_work_add(task, &mnt->mnt_rcu, true)) /* 返回用户态必须用这个函数才保证正确 */
                    return;
             }
             if (llist_add(&mnt->mnt_llist, &delayed_mntput_list)) /* 内核线程，又不属于 MS_KERNMOUNT，为何不用同步方式? */
                 schedule_delayed_work(&delayed_mntput_work, 1);   /* 用 workqueue 异步执行是因为可能中断上下文？*/
             return;
         }
         cleanup_mnt(mnt); /* 因为设置 MS_KERNMOUNT，不返回用户态，可以使用同步方式 */

     这个被称作 [delayed mntput 的patch](https://github.com/torvalds/linux/commit/9ea459e110df32e60a762f311f7939eaa879601d)
     在3.18-rc1被引入。关于为何要引入 delayed mntput 和 task_work_add API 有何特殊的意义，
     [LWN 讲述 delay fput 的文章](https://lwn.net/Articles/494158/)对理解这些问题很有帮助。

#### 1.2 编译和加载

编译 Day2 模块需要先编译 Linux 内核源代码。请参考
[Linux File System Basic - 2](http://oliveryang.net/2016/01/linux-file-system-basic-2/)。

Samplefs 的编译可以在 Linux 内核编译成功后，运行下面的命令单独编译，

	make M=/ws/lktm/fs/samplefs/day2

原版的 Day2 的代码是为 Linux 2.6 写的，在新内核 Linux 3.19 上会因为内核接口的变化引起编译错误。
如果使用本文提供的 Day2 的源码，则可以正确编译，这是因为本文所用代码对新内核做了相应的修改。
请参考[针对新内核接口的 Patch](https://github.com/yangoliver/lktm/commit/dd2b5a7332ff61ee8a4ded3281616b0f77d6eddf#diff-2e79772ae929f397a8bb5817fc4e6c4f)
来查看本文中的 Day2 代码针对原有代码做了哪些修改。

### 2. 关键数据结构和概念

本节对文件系统的一些关键数据结构和概念做简单介绍。

#### 2.1 struct file_system_type: 文件系统类型

用于描述和表示一个具体的文件系统类型。每个文件系统模块都声明和初始化一个文件系统类型数据结构，
然后在模块加载和初始化时通过 VFS register_filesystem API 向 VFS 核心层注册。
模块在被卸载时，可以通过 VFS unregister_filesystem 从 VFS 核心层注销。

VFS 核心层维护一个全局链表，可以查找系统中目前注册的所有文件系统类型，
并且调用该数据结构里提供的 mount 和 kill_sb 方法在 文件系统的 mount/umount 操作时做相应的处理。
[Linux file system basic - 2](http://oliveryang.net/2016/01/linux-file-system-basic-2/)
中已经有过详细介绍，这里就不再展开详述。

#### 2.2 struct super_block: 超级块

Super Block 既是表示一个已经 mount 的文件系统的内存对象，也是关联所有文件系统 Meta Data (元数据) 的核心对象。
讨论 Super Block 这个概念的时候，需要搞清楚是哪个层面上的 Super Block。
否则会引起很多误会和混淆。一个基于磁盘的文件系统，会涉及到三个不同层面上的 Super Block，

- VFS **内存中**的 Super Block

  是 VFS 对所有文件系统**共性**做的数据抽象，所有文件系统都使用相同的定义：`struct super_block`。

- 具体文件系统**内存中**的 Super Block
 
  是具体文件系统基于磁盘介质上的 Super Block 在内存中创建的对象。
  每个文件系统都需要自己定义，属于该文件系统**个性**的部分。Samplefs 的对应数据结构为：`struct samplefs_sb_info`。

- 具体文件系统**磁盘上**存储的 Super Block

  是具体文件系统 Disk Layout (磁盘布局)整体设计的一部分，属于该文件系统**个性**的一部分。
  通常磁盘 Super Block 存储在磁盘设备上的固定偏移的一个或者多个 Block (块)里。
  由于 samplefs 不是一个磁盘文件系统，因此没有磁盘上的 Super Block。

Linux 3.19的 [struct super_block 的定义](https://github.com/torvalds/linux/blob/v3.19/include/linux/fs.h#L1215)
里的部分成员在 samplefs_fill_super 回调里被初始化了, 下面的定义仅列出相关成员，

	struct super_block {

		[...snipped...]

		unsigned char		s_blocksize_bits;
		unsigned long		s_blocksize;
		loff_t			s_maxbytes;	/* Max file size */
		struct file_system_type	*s_type;
		const struct super_operations	*s_op;

		[...snipped...]

		unsigned long		s_magic;
		struct dentry		*s_root;

		[...snipped...]

		void 			*s_fs_info;	/* Filesystem private info */

		[...snipped...]

		/* Granularity of c/m/atime in ns.
		   Cannot be worse than a second */
		u32		   s_time_gran;

		[...snipped...]
	};

这里只介绍其中三个重要的结构成员，

1. s_fs_info 成员

   该成员直接指向 samplefs 模块的内存 Super Block。通过把 s_fs_info 指向 samplefs_sb_info，
   VFS 的 Super Block 结构 super_block 和 samplefs_sb_info 结构关联了起来。

2. s_op 成员

   该成员直接指向 VFS Super Block 的操作表结构：`struct super_operations`，

		struct super_operations samplefs_super_ops = {
			.statfs         = simple_statfs,
			.drop_inode     = generic_delete_inode, /* Not needed, is the default */
			.put_super      = samplefs_put_super,
		};

   Samplefs只初始化了 super_operations 的三个方法，其中前两个是 VFS 代码提供的默认回调。
   而 samplefs 只自定义及使用了第三个方法：put_super，用于释放 samplefs 模块自定的 Super Block。

3. s_root 成员

   指向 root dentry，而 root dentry 又可以指向 root inode。
   Samplefs 通过 VFS 函数，先后分配了 root inode 和 root dentry，并且赋值给 s_root 成员。

#### 2.3 struct inode: 索引节点

inode 数据结构存放了文件系统内的各种对象(常规文件，目录，符号链接，设备文件等)的元数据。
与 Super Block 类似，inode 在文件系统的不同层次都有具体定义，

- VFS **内存中**的 inode
- 具体文件系统**内存中**的 inode
- 具体文件系统**磁盘上**存储的 inode

Samplefs day2 的代码里只涉及了 VFS inode，它在 samplefs_fill_super 中调用了 iget_locked 分配了 root inode。
本文暂不对 inode 做详细说明。

#### 2.4 struct dentry: 目录项

dentry 数据结构描述文件系统对象(常规文件，目录，符号链接，设备文件等)在内核中的文件系统树中的位置。
与 Super Block 类似，理论上 dentry 在文件系统的不同层次也可以有不同定义，

- VFS **内存中**的 dentry
- 具体文件系统**内存中**的 dentry
- 具体文件系统**磁盘上**存储的 dentry

Samplefs day2 的代码里只涉及了 VFS dentry，它在 samplefs_fill_super 中调用了 d_make_root 分配了 root dentry。
这个 root dentry 也是 samplefs_mount 返回给 VFS 的返回值。该 root dentry 也被 VFS Super Block 的 s_root 成员指向。
dentry 结构的 d_inode 成员也会指向它所关联的 root inode，这里即 samplefs 的 root inode。
本文暂不对 dentry 做详细说明。

#### 2.5 struct vfsmount: VFS文件系统装载

vfsmount 代表了文件系统的已装载实例。其中主要由文件系统的 root dentry 和 Super Block 构成。

	struct vfsmount {
		struct dentry *mnt_root;	/* root of the mounted tree */
		struct super_block *mnt_sb;	/* pointer to superblock */
		int mnt_flags;
	};

早期内核里，vfsmount 还用于将局部文件系统的装载实例链接在一起，形成一个全局树状数据结构，
用于访问各文件系统装载实例。因此 vfsmount 有很多其它结构成员。

新内核中，vfsmount 的大部分成员都被转移到 `struct mount` 数据结构中。这样，
链接所有文件系统装载实例的工作改由 `struct mount` 结构完成。由于 vfsmount 也被用于 VFS API 的参数，
因此，把不需要暴露给 VFS API 使用者的成员转移到内部 mount 结构的好处还是显而易见的。

本文暂不对 vfsmount 做更详细的说明。

### 3. 实验和调试

TBD

#### 3.1 文件系统 mount

#### 3.2 遍历 mount 实例

#### 3.3 查看 Super Block

### 4. 小结