---
layout:     post
title:      一个极简容器运行时实践
subtitle:   从一个入门示例窥探Linux 容器底层工作机制
date:       2024-09-29
author:     pandaychen
header-img:
catalog: true
tags:
    - Golang
---

##  0x00    前言

本文从一个非常经典的入门例子了解容器运行时的相关原理和机制，主要参考[Linux 容器底层工作机制：从 500 行 C 代码到生产级容器运行时（2023）](https://arthurchiao.art/blog/linux-container-and-runtime-zh/)一文，配套[代码](https://github.com/ArthurChiao/arthurchiao.github.io/tree/master/assets/code/linux-container-and-runtime)，部分内容有扩充

##  0x01    Linux 容器底层机制：NS/cgroups/Capabilities/Seccomp

![Kernel-machanisms-Linux-containers]()

1.  namespace：内核提供的资源视图**隔离机制**，决定了进程的可见性；例如，pid namespace 限制了进程能看到哪些其他的进程；network namespace 限制了进程能看到哪些网络设备等等
2.  cgroup：一种限制资源使用量的机制，例如一个进程最多能使用多少内存、磁盘 I/O、CPU 等
3.  capabilities：拆分 root privilege，精细化用户/进程权限
4.  seccomp：内核的一种安全计算 （secure computation）机制，例如限制进程只能使用某些特定的系统调用，相对于bpf-lsm等机制安全能力偏弱

以上几种机制中，cgroups 是通过文件系统完成的，其他几种都是通过系统调用完成的，先罗列一些基础概念

####    Namespaces

-   Mount：挂载空间，例如指定某个目录作为进程看到的系统根目录、独立挂载 /proc /sys 等
-   PID：进程空间，例如最大进程数量是独立的
-   Network (netns)：网络空间，例如能看到的网络设备是独立的，部分网络参数是独立的
-   IPC (inter-process communication)
-   UTS (UNIX Time-Sharing)：让进程可以有独立的 host name 和 domain name；
-   cgroup：独立的 cgroup 文件系统和资源管控


##  0x02    一个简单容器的c实现（500 lines）
代码参考
-   [v1](https://github.com/ArthurChiao/arthurchiao.github.io/tree/master/assets/code/linux-container-and-runtime)
-   [v2](https://blog.lizzie.io/linux-containers-in-500-loc.html)

####    入口

```c
int main (int argc, char **argv) {
    // Step 1: parse CLI parameters, validate Linux kernel/cpu/..., generate hostname for container

    // Step 2: setup a socket pair for container sending messages to the parent process
    socketpair(AF_LOCAL, SOCK_SEQPACKET, 0, sockets);
    fcntl(sockets[0], F_SETFD, FD_CLOEXEC);
    config.fd = sockets[1];

    // Step 3: allocate stack space for `execve()`
    stack = malloc(STACK_SIZE));

    // Step 4: setup cgroup for the container for resource isolation
    setup_cgroups(&config);

    // Step 5: launch container 将 mount, pid, ipc, network device, hostname/domainname 放到独立的 namespace
    int flags = CLONE_NEWNS | CLONE_NEWCGROUP | CLONE_NEWPID | CLONE_NEWIPC | CLONE_NEWNET | CLONE_NEWUTS;
    child_pid = clone(child, stack + STACK_SIZE, flags | SIGCHLD, &config);

    // Step 6: error handling and cleanup
}
```
上述主要流程（核心科技在 4 和 5）：

1.  初始化
    -   解析命令行参数
    -   检查内核版本、CPU 架构等等
    -   给容器随机生成一个 hostname
2.  创建一个 socket pair，用于容器进程和主进程之间的通信
3.  分配栈空间，供后面 execve() 执行容器进程时使用
4.  设置 cgroups 资源隔离
5.  通过 clone 创建子进程，在里面通过 namespace/capabilities/seccomp 等技术实现资源管控和安全

####    1、初始化&&命令行参数
程序提供了如下命令行参数

-   -u <uid>：以什么用户权限运行
-   -m <ctn rootfs path>：镜像解压之后的目录
-   -c <command>：容器启动后运行的命令

####    2、创建 socket pair，供容器进程和父进程通信

```c
    if (socketpair(AF_LOCAL, SOCK_SEQPACKET, 0, sockets)) {
        fprintf(stderr, "socketpair failed: %m\n");
        goto error;
    }
    if (fcntl(sockets[0], F_SETFD, FD_CLOEXEC)) {
        fprintf(stderr, "fcntl failed: %m\n");
        goto error;
    }
    config.fd = sockets[1];
```

子进程（容器进程）需要发消息给父进程，因此初始化一个 socketpair。容器进程会告诉父进程是否需要设置 uid/gid mappings， 如果需要，就会执行 setgroups/setresgid/setresuid（权限相关）

####    3、分配栈空间，供随后 execve() 执行容器启动进程使用

```c
//通过 execve() 来执行容器内进程的准备工作
#define STACK_SIZE (1024 * 1024)
    char *stack = 0;
    if (!(stack = malloc(STACK_SIZE))) {
        fprintf(stderr, "=> malloc failed, out of memory?\n");
        goto error;
    }
```

####    4、创建 cgroup，为容器设置资源限额
cgroups 用来限制进程和进程组的资源使用量，避免有问题的容器进程影响整个系统，代码使用v1版本

```c
struct cgrp_control {
    char control[256];
    struct cgrp_setting {
        char name[256];
        char value[256];
    } **settings;
};
struct cgrp_setting add_to_tasks = { // echo 0 > /sys/fs/cgroup/<controller>/<hostname>/tasks
    .name = "tasks",
    .value = "0"
};

struct cgrp_control *cgrps[] = {
    & (struct cgrp_control) {
        .control = "memory",
        .settings = (struct cgrp_setting *[]) {
            & (struct cgrp_setting) {
                .name = "memory.limit_in_bytes",
                .value = "1073741824"
            },
            /* & (struct cgrp_setting) { */
            /* 	.name = "memory.kmem.limit_in_bytes", */
            /* 	.value = "1073741824" */
            /* }, */
            &add_to_tasks,
            NULL
        }
    },
    & (struct cgrp_control) {
        .control = "cpu",
        .settings = (struct cgrp_setting *[]) {
            & (struct cgrp_setting) {
                .name = "cpu.shares",
                .value = "256" // CPU shares
            },
            &add_to_tasks,
            NULL
        }
    },
    & (struct cgrp_control) {
        .control = "pids",
        .settings = (struct cgrp_setting *[]) {
            & (struct cgrp_setting) {
                .name = "pids.max",
                .value = "64"
            },
            &add_to_tasks,
            NULL
        }
    },
    /* & (struct cgrp_control) { */
    /* 	.control = "blkio", */
    /* 	.settings = (struct cgrp_setting *[]) { */
    /* 		& (struct cgrp_setting) { */
    /* 			.name = "blkio.weight", */
    /* 			.value = "10" */
    /* 		}, */
    /* 		&add_to_tasks, */
    /* 		NULL */
    /* 	} */
    /* }, */
    NULL
};

int setup_cgroups(struct ctn_config *config) {
    for (struct cgrp_control **cgrp = cgrps; *cgrp; cgrp++) {
        char dir[PATH_MAX] = {0};
        snprintf(dir, sizeof(dir), "/sys/fs/cgroup/%s/%s", (*cgrp)->control, config->hostname);
        mkdir(dir, S_IRUSR | S_IWUSR | S_IXUSR);
        for (struct cgrp_setting **setting = (*cgrp)->settings; *setting; setting++) {
            char path[PATH_MAX] = {0};
            int fd = 0;
            snprintf(path, sizeof(path), "%s/%s", dir, (*setting)->name);
            fd = open(path, O_WRONLY);
            write(fd, (*setting)->value, strlen((*setting)->value));
            close(fd);
        }
    }

    setrlimit(RLIMIT_NOFILE, & (struct rlimit) {.rlim_max = 64, .rlim_cur = 64,});
    return 0;
}
```

设置 cgroups 的逻辑是创建 cgroup 目录， 以及往 cgroups 配置文件写入配置。上述代码配置了以下几个资源限额：

-   /sys/fs/cgroup/memory/$hostname/memory.limit_in_bytes=1GB：容器进程及其子进程使用的总内存不超过 1GB
-   /sys/fs/cgroup/memory/$hostname/memory.kmem.limit_in_bytes=1GB：容器进程及其子进程使用的总内存不超过 1GB
-   /sys/fs/cgroup/cpu/$hostname/cpu.shares=256：CPU 总 slice 是 1024，因此限制进程最多只能占用 1/4 CPU 时间
-   /sys/fs/cgroup/pids/$hostname/pid.max=64：允许容器进程及其子进程最多拥有 64 个 PID
-   /sys/fs/cgroup/blkio/$hostname/weight=50：确保容器进程的 IO 优先级比系统其他进程低
-   降低文件描述符的 hard limit。fd 与 pid 类似，都是 per-user 的。这里设置上限之后， 后面还会 drop CAP_SYS_RESOURCE，因此容器内的用户是改不了的

另外，程序还通过 `add_to_tasks` 向 {memory,cpu,blkio,pids}/$hostname/tasks 文件写入 0，即实现下面的效果：

```bash
$ echo 0 > /sys/fs/cgroup/<controller>/<hostname>/tasks
```

tasks 里面存放的是受这个 cgroup 管控的进程 ID（PID）列表， 0 是一个特殊值，表示“执行当前写入操作的进程”。

####    5、 clone() 启动子进程，运行容器
fork() 是常见的创建子进程的方式，但它背后调用的是 clone()，是创建容器的关键技术。 我们创建的子进程要能 mount 自己的根目录、设置自己的 hostname 以及做其他一些事情， 要实现这个效果，需要向 clone() 传递相应的 flags 参数：

```c
    int flags = CLONE_NEWNS | CLONE_NEWCGROUP | CLONE_NEWPID | CLONE_NEWIPC | CLONE_NEWNET | CLONE_NEWUTS;
    child_pid = clone(child, stack + STACK_SIZE, flags | SIGCHLD, &config);
```

上述配置说明了

-   flags 规定了这个新创建的子进程要有多个独立的 namespace
-   x86 平台栈是向下增长的，因此将栈指针指向 stack+STACK_SIZE 作为起始位置
-   最后还还加上了 SIGCHLD 标志位，这样就能 wait 子进程了

下面是子进程内做的事情：

```c
int child(void *arg) {
    struct ctn_config *config = arg;
    sethostname(config->hostname, strlen(config->hostname);

    setup_mounts(config);
    setup_userns(config);
    setup_capabilities();
    setup_seccomp();
    close(config->fd);

    execve(config->argv[0], config->argv, NULL);
}
```

现在分别来讨论下：

**1、sethostname()：感知 UTS namespace**

sethostname/gethostname 系统调用用于设置或获取主机名（hostname），hostname 是 UTS namespace 隔离的， 刚才创建子进程时指定了要有独立的 UTS namespace， 因此在这个子进程内设置 hostname 时，**影响的只是这个 UTS namespace 内（即这个容器内）所有进程 看到的 hostname**

**2、setup_mounts()：安全考虑，unmount 不需要的目录**

flags 指定了子进程有独立的 mount namespace。**按照最小权限原则应该 unmount 掉子进程不应该访问的目录**

```c
int setup_mounts(struct ctn_config *config) {
    // remounting everything with MS_PRIVATE...
    mount(NULL, "/", NULL, MS_REC | MS_PRIVATE, NULL));

    // making a temp directory and a bind mount there...
    char mount_dir[] = "/tmp/tmp.XXXXXX";
    mkdtemp(mount_dir);
    mount(config->mount_dir, mount_dir, NULL, MS_BIND | MS_PRIVATE, NULL);

    char inner_mount_dir[] = "/tmp/tmp.XXXXXX/oldroot.XXXXXX";
    memcpy(inner_mount_dir, mount_dir, sizeof(mount_dir) - 1);
    mkdtemp(inner_mount_dir);

    // pivoting root...
    syscall(SYS_pivot_root, mount_dir, inner_mount_dir);

    char *old_root_dir = basename(inner_mount_dir);
    char old_root[sizeof(inner_mount_dir) + 1] = { "/" };
    strcpy(&old_root[1], old_root_dir);

    // unmounting old_root
    chdir("/");
    umount2(old_root, MNT_DETACH);
    rmdir(old_root);

    return 0;
}
```

`setup_mounts`的主要实现步骤如下：

1.  使用 MS_PRIVATE 重新挂载所有内容
2.  创建一个临时目录，并在其中创建一个子目录
3.  将用户命令行指定的目录（config->mount_dir）bind mount 到临时目录上
4.  使用 pivot_root 将 bind mount 作为根目录，并将旧的根目录挂载到内部临时目录上。 pivot_root 系统调用允许交换 / 处的挂载点与另一个挂载点
5.  卸载旧的根目录，删除内部临时目录


**3、setup_userns()**

```c
int setup_userns(struct ctn_config *config) {
    // trying a user namespace...
    int has_userns = !unshare(CLONE_NEWUSER);
    write(config->fd, &has_userns, sizeof(has_userns));

    if (has_userns) {
        fprintf(stderr, "done.\n");
    } else {
        fprintf(stderr, "unsupported? continuing.\n");
    }

    // switching to uid %d / gid %d...", config->uid, config->uid
    setgroups(1, & (gid_t) { config->uid });
    setresgid(config->uid, config->uid, config->uid);
    setresuid(config->uid, config->uid, config->uid);
}
```

**4、setup_capabilities()：禁用部分 capabilities**

同样，基于最小权限原则drop 所有不需要的 capabilities

```c
int setup_capabilities() {
    // dropping capabilities...
    int drop_caps[] = {
        CAP_AUDIT_CONTROL,
        CAP_AUDIT_READ,
        CAP_AUDIT_WRITE,
        CAP_BLOCK_SUSPEND,
        CAP_DAC_READ_SEARCH,
        CAP_FSETID,
        CAP_IPC_LOCK,
        CAP_MAC_ADMIN,
        CAP_MAC_OVERRIDE,
        CAP_MKNOD,
        CAP_SETFCAP,
        CAP_SYSLOG,
        CAP_SYS_ADMIN,
        CAP_SYS_BOOT,
        CAP_SYS_MODULE,
        CAP_SYS_NICE,
        CAP_SYS_RAWIO,
        CAP_SYS_RESOURCE,
        CAP_SYS_TIME,
        CAP_WAKE_ALARM
    };
    size_t num_caps = sizeof(drop_caps) / sizeof(*drop_caps);

    // bounding...
    for (size_t i = 0; i < num_caps; i++) {
        prctl(PR_CAPBSET_DROP, drop_caps[i], 0, 0, 0);
    }

    // inheritable...
    cap_t caps = NULL;
    if (!(caps = cap_get_proc())
            || cap_set_flag(caps, CAP_INHERITABLE, num_caps, drop_caps, CAP_CLEAR)
            || cap_set_proc(caps)) {
        if (caps) cap_free(caps);
        return 1;
    }

    cap_free(caps);
    return 0;
}
```

**5、setup_seccomp()：禁用部分系统调用**

使用seccomp机制，将有安全风险的系统调用都放到了黑名单

```c
int setup_seccomp() {
    scmp_filter_ctx ctx = NULL;

    // filtering syscalls...
    if (!(ctx = seccomp_init(SCMP_ACT_ALLOW))
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(chmod), 1, SCMP_A1(SCMP_CMP_MASKED_EQ, S_ISUID, S_ISUID))
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(chmod), 1, SCMP_A1(SCMP_CMP_MASKED_EQ, S_ISGID, S_ISGID))
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(fchmod), 1, SCMP_A1(SCMP_CMP_MASKED_EQ, S_ISUID, S_ISUID))
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(fchmod), 1, SCMP_A1(SCMP_CMP_MASKED_EQ, S_ISGID, S_ISGID))
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(fchmodat), 1, SCMP_A2(SCMP_CMP_MASKED_EQ, S_ISUID, S_ISUID))
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(fchmodat), 1, SCMP_A2(SCMP_CMP_MASKED_EQ, S_ISGID, S_ISGID))
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(unshare), 1, SCMP_A0(SCMP_CMP_MASKED_EQ, CLONE_NEWUSER, CLONE_NEWUSER))
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(clone), 1, SCMP_A0(SCMP_CMP_MASKED_EQ, CLONE_NEWUSER, CLONE_NEWUSER))
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(ioctl), 1, SCMP_A1(SCMP_CMP_MASKED_EQ, TIOCSTI, TIOCSTI))
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(keyctl), 0)
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(add_key), 0)
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(request_key), 0)
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(ptrace), 0)
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(mbind), 0)
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(migrate_pages), 0)
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(move_pages), 0)
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(set_mempolicy), 0)
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(userfaultfd), 0)
            || seccomp_rule_add(ctx, SCMP_FAIL, SCMP_SYS(perf_event_open), 0)
            || seccomp_attr_set(ctx, SCMP_FLTATR_CTL_NNP, 0)
            || seccomp_load(ctx)) {
                if (ctx) seccomp_release(ctx);
                return 1;
            }

    // all pass, return success
    seccomp_release(ctx);
    return 0;
}
```

**6、execve()：执行指定的容器启动命令**

前面资源视图隔离（namespace）、资源限额（cgroups）、文件目录（mount）、权限（capabilities）、安全（seccomp）等工作都做好之后，就可以启动用户指定的容器进程了：

```c
    execve(config->argv[0], config->argv, NULL);
```

至此，如果一切正常，容器进程就起来了。接下来还要容器进程与主进程的通信， 类似于真实环境中 containerd 处理来自具体容器的消息


####    6、处理容器事件：user namespace 相关

```c
#define USERNS_OFFSET 10000
#define USERNS_COUNT 2000

int handle_child_uid_map (pid_t child_pid, int fd) {
    int uid_map = 0;
    int has_userns = -1;
    if (read(fd, &has_userns, sizeof(has_userns)) != sizeof(has_userns)) {
        fprintf(stderr, "couldn't read from child!\n");
        return -1;
    }

    if (has_userns) {
        char path[PATH_MAX] = {0};
        for (char **file = (char *[]) { "uid_map", "gid_map", 0 }; *file; file++) {
            if (snprintf(path, sizeof(path), "/proc/%d/%s", child_pid, *file) > sizeof(path)) {
                fprintf(stderr, "snprintf too big? %m\n");
                return -1;
            }
            fprintf(stderr, "writing %s...", path);
            if ((uid_map = open(path, O_WRONLY)) == -1) {
                fprintf(stderr, "open failed: %m\n");
                return -1;
            }
            if (dprintf(uid_map, "0 %d %d\n", USERNS_OFFSET, USERNS_COUNT) == -1) {
                fprintf(stderr, "dprintf failed: %m\n");
                close(uid_map);
                return -1;
            }
            close(uid_map);
        }
    }

    if (write(fd, & (int) { 0 }, sizeof(int)) != sizeof(int)) {
        fprintf(stderr, "couldn't write: %m\n");
        return -1;
    }
    return 0;
}
```

##  0x03    验证过程
本文创建容器时，是先手动将一个 docker 镜像解压到本地目录，然再指定这个目录创建容器。主要验证过程：

```bash
$ dk pull busybox:1.36  #下载官方 busybox 容器镜像
$ dk save busybox:1.36 -o busybox-1.36.tar  #保存为本地 tar 文件

$ sudo tar xvf busybox-1.36.tar #layer.tar 是镜像本身，其他都是元数据
244ed32d6820f8861f94beda2456fa7032a832a4e7ed7e72fa66b802518f9adc/
244ed32d6820f8861f94beda2456fa7032a832a4e7ed7e72fa66b802518f9adc/VERSION
244ed32d6820f8861f94beda2456fa7032a832a4e7ed7e72fa66b802518f9adc/json
244ed32d6820f8861f94beda2456fa7032a832a4e7ed7e72fa66b802518f9adc/layer.tar
f5fb98afcf9f5c6e8e069557f605b15b52643166c82ac5695f49fc6b0be04ee8.json
manifest.json
repositories


$ mkdir ~/busybox-1.36-rootfs #
$ tar xvf busybox-1.36/244ed32d6820f8861f94beda2456fa7032a832a4e7ed7e72fa66b802518f9adc/layer.tar -C ~/busybox-1.36-rootfs
```

如目录`busybox-1.36-rootfs`（形如Linux node 根目录的文件夹），下面会使用这个目录作为容器根目录来启动容器

```bash
$ ls ~/busybox-1.36-rootfs/
bin  dev  etc  home  lib  lib64  root  tmp  usr  var
```

编译容器demo代码

```bash
$ sudo apt install libseccomp-dev libcap-dev -y
$ gcc -Wl,--no-as-needed -lcap -lseccomp ctn-d.c
```

####    测试
运行 `/bin/sh`交互式，指定解压之后的 busybox 容器镜像目录作为容器根目录（`-m ~/busybox-rootfs-1.36/`）， 以 root 权限（`-u 0`）运行 shell（`-c /bin/sh`）：

```bash
$ sudo ./a.out -u 0 -m ~/busybox-rootfs-1.36/ -c /bin/sh
=> validating Linux version...6.1.11-060111-generic on x86_64.
=> generating hostname for container ... 1cd132c-ten-of-pentacles done
=> setting cgroups...memory...cpu...pids...done.
=> setting rlimit...done.
=> remounting everything with MS_PRIVATE...remounted.
=> making a temp directory and a bind mount there...
=> temp directory /tmp/tmp.j1SPbh
done.
=> pivoting root...done.
=> unmounting /oldroot.p216lf...done.
=> trying a user namespace...writing /proc/3544398/uid_map...writing /proc/3544398/gid_map...done.
=> switching to uid 0 / gid 0...done.
=> dropping capabilities...bounding...inheritable...done.
=> filtering syscalls...done.
/ #     #成功进入容器内的shell
```

执行几个 busybox 镜像内有的命令来测试：

```bash
/ # whoami                       # <-- 已经进入到容器内
root
/ # hostname
17bb49-death
/ # ls
bin    dev    etc    home   lib    lib64  root   tmp    usr    var
```

最后通过 `exit` 正常退出 shell，由于这是容器主进程，因此 shell 退出之后容器也就退出了

```bash
/ # exit                         # <-- 退出容器
=> cleaning cgroups...done
```


再测试下直接在容器内执行一段任务（非交互式），运行完就会自动退出：

```bash
$ sudo ./a.out -u 0 -m ~/busybox-rootfs-1.36/ -c /bin/echo "hello from inside container"
=> validating Linux version...6.1.11-060111-generic on x86_64.
=> generating hostname for container ... 1cd132c-ten-of-pentacles done
=> setting cgroups...memory...cpu...pids...done.
=> setting rlimit...done.
=> remounting everything with MS_PRIVATE...remounted.
=> making a temp directory and a bind mount there...
=> temp directory /tmp/tmp.j1SPbh
done.
=> pivoting root...done.
=> unmounting /oldroot.p216lf...done.
=> trying a user namespace...writing /proc/3544398/uid_map...writing /proc/3544398/gid_map...done.
=> switching to uid 0 / gid 0...done.
=> dropping capabilities...bounding...inheritable...done.
=> filtering syscalls...done.
hello from inside container  #   <-- 容器内执行 echo，执行完就退出了
=> cleaning cgroups...done.
```

##  0x04    对比 containerd+runc
本文的代码对应容器技术栈中的哪些组件呢？如下图展示了k8s 中容器相关的组件，其中dockerd 是可选的（较新版本的 k8s 中已经去掉了 dockerd），但其实 containerd 是跟 kubelet/docker 一样，都是负责管理工作（container manager）， 真正干活的是更底层的 runc组件。本文的demo其实就是一个简化版的 containerd+runc

![k8s-arch-simple]()

runc 是基于一个 config.json 创建容器的，当k8s 创建容器时，通常是由 containerd 通过 config.json 传给 runc 。一个真实的 k8s pod 对应的 config.json文件配置内容可能如下：

```json
(node1) $ cat /run/containerd/io.containerd.runtime.v1.linux/moby/eedd6341c/config.json | jq
{
  "process": {
    "user": {
      "uid": 0,
      "gid": 0
    },
    "args": [ ],
    ...
    "memory": {
      "limit": 2147483648,
      "swap": 2147483648,
    },
    "cpu": {
      "shares": 1024,
      "quota": 100000,
      "period": 100000,
      "cpus": "0-31"
    },
    "blockIO": {
      "weight": 0
    },
    "cgroupsPath": "/kubepods/besteffort/pod187acdb9/eedd6341c",
    "namespaces": [
      {
        "type": "mount"
      },
      {
        "type": "network"
      },
      ...
    ],
    "maskedPaths": [
      "/proc/asound",
      "/proc/acpi",
      "/proc/kcore",
      "/proc/keys",
      "/proc/latency_stats",
      "/proc/timer_list",
      "/proc/timer_stats",
      "/proc/sched_debug",
      "/proc/scsi",
      "/sys/firmware"
    ],
    "readonlyPaths": [
      "/proc/bus",
      "/proc/fs",
      "/proc/irq",
      "/proc/sys",
      "/proc/sysrq-trigger"
    ]
  }
}
```

####   容器网络
对容器而言，常规网络方案来说分几步：

1.  在宿主机上创建一个 Linux bridge
2.  创建一个 veth pair，一端连接到 Linux bridge，一端放到容器的 network namespace 内
3.  配置 IP/MAC/NAT 规则等，让容器网络能连通

####   特殊目录
通常，容器内的目录（就是busybox 镜像解压之后的目录）

```bash
# ls /
bin    dev    etc    home   lib    lib64  root   tmp    usr    var
```

对比宿主机根目录，少了 `boot/mnt/opt/proc/run/sbin/srv/sys` 这几个目录

```bash
# In the container
# ls /
bin        dev  etc  home  lib  lib64                         root                       tmp  usr  var

# On the node
$ ls /
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

其中，最重要的两个：`/proc`（procfs） 和 `/sys`（sysfs），这两个大部分是内核暴露给用户空间的接口（如/proc是宿主机的相关信息）。需要容器运行时单独处理，在创建容器时挂载进去

####    容器 /proc 存在的问题以及解决方案
若 docker/containerd/runc 之类的容器运行时，会发现它是把宿主机 `/proc` 挂载到了容器， 这会导致什么问题呢？ 

1.  容器内能看到宿主机以及其他容器的进程信息（虽然原则上是只读的），存在安全风险
2.  容器内读取某些文件，如 /proc/meminfo，本来是希望得到容器自己的内存信息，但实际上拿到的是宿主机的，因此数据不准

那如何解决呢？先看原作者给出的kubernetes的pod例子，查看容器 /proc/cpuinfo 和 /proc/meminfo

```bash
#容器需要 125m = 125/1000 = 1/8 个 CPU，751 MB 内存； 最多能使用 1 个 CPU，1GB 内存
$ k get pod -n <ns> <pod> -o yaml
...
    resources:
      requests:
        cpu: 125m
        memory: 751619276800m
      limits:
        cpu: "1"
        memory: 1Gi
```

在容器里面看 /proc 挂载信息以及cpu、内存等信息。可以看到，Memory 是对的，CPU 不对，实际上是宿主机的 CPU 数量

```bash
(container) # cat /proc/mounts | grep proc
proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0
lxcfs /proc/meminfo fuse.lxcfs ro,relatime,user_id=0,group_id=0,allow_other 0 0
lxcfs /proc/diskstats fuse.lxcfs ro,relatime,user_id=0,group_id=0,allow_other 0 0
proc /proc/bus proc ro,relatime 0 0
proc /proc/fs proc ro,relatime 0 0
proc /proc/irq proc ro,relatime 0 0
proc /proc/sys proc ro,relatime 0 0
proc /proc/sysrq-trigger proc ro,relatime 0 0
tmpfs /proc/acpi tmpfs ro,relatime 0 0
tmpfs /proc/kcore tmpfs rw,nosuid,size=65536k,mode=755 0 0
tmpfs /proc/keys tmpfs rw,nosuid,size=65536k,mode=755 0 0
tmpfs /proc/timer_list tmpfs rw,nosuid,size=65536k,mode=755 0 0
tmpfs /proc/sched_debug tmpfs rw,nosuid,size=65536k,mode=755 0 0
tmpfs /proc/scsi tmpfs ro,relatime 0 0

(container) # cat /proc/cpuinfo
processor       : 0
vendor_id       : GenuineIntel
...
processor       : 31            # 32 CPU
vendor_id       : GenuineIntel

(container) # cat /proc/meminfo
MemTotal:        1048576 kB     # 1GB
MemFree:          893416 kB
MemAvailable:     893416 kB
...
```

原因如下：

-   信息正确的 /proc/meminfo，是通过 lxcfs 挂载来实现的
-   /proc/cpuinfo 不正确是因为没有配置 lxcfs 或者不支持

在宿主机上查看容器的 config.json 文件，可以看到有 meminfo 的挂载，但是没有 cpuinfo 的挂载

```bash
$ cat /run/containerd/io.containerd.runtime.v1.linux/moby/d32b1bf.config.json | jq .
#文件的挂载信息

    {
      "destination": "/proc/meminfo",
      "type": "bind",
      "source": "/var/lib/lxcfs/proc/meminfo",
      "options": [
        "rbind",
        "ro",
        "rprivate"
      ]
    },
    {
      "destination": "/proc/diskstats",
      "type": "bind",
      "source": "/var/lib/lxcfs/proc/diskstats",
      "options": [
        "rbind",
        "ro",
        "rprivate"
      ]
    },
```

再去k8s 里再看看 pod spec配置，如下：

```bash
$ k get pod xxx -n <ns> <pod> -o yaml
    volumeMounts:
    - mountPath: /proc/meminfo
      name: meminfo
      readOnly: true
    - mountPath: /proc/diskstats
      name: diskstats
      readOnly: true

  volumes:
  - hostPath:
      path: /var/lib/lxcfs/proc/meminfo
      type: File
    name: meminfo   #仅指定了meminfo的挂载，没有 cpuinfo 的挂载
  - hostPath:
      path: /var/lib/lxcfs/proc/diskstats
      type: File
    name: diskstats
```

####    /proc采集不准的解决方案

方案一：从上层解决，应用主动适配 cgroup

需要应用程序主动适配 cgroup，需要的信息都去容器的 /sys/fs/cgroup 目录读取，不要去容器的 /proc/ 目录读。很多监控采集程序都是这么做的，比如 k8s 用的 cadvisor，采集 cpuload，示例代码如：

-   [cadvisor-1](https://github.com/google/cadvisor/blob/v0.48.1/manager/container.go#L652-L654)
-   [cadvisor-2](https://github.com/google/cadvisor/blob/v0.48.1/utils/cpuload/netlink/example/example.go)

方案二：从底层解决，使用lxcfs等特殊文件系统挂载来模拟部分 /proc 文件


####    lxcfs

本小节简单介绍下lxcfs+fuse 基本工作原理。lxcfs 是一个简单的用户空间文件系统，设计目的就是 workaround 当前的一些 Linux 内核限制

-   一些文件：能 bind-mount 到 /proc，做到 CGroup-aware
-   一个 cgroupfs-like tree：container aware

前述用 lxcfs 接管了 /proc/meminfo 和 /proc/diskstats，但是没有接管 /proc/cpuinfo，这也是为什么 Memory 是对的，CPU 数量不对

```bash
(container) # cat /proc/mounts | grep proc
...
lxcfs /proc/meminfo fuse.lxcfs ro,relatime,user_id=0,group_id=0,allow_other 0 0   # 只读
lxcfs /proc/diskstats fuse.lxcfs ro,relatime,user_id=0,group_id=0,allow_other 0 0 # 只读
```

lxcfs 工作原理如下图，**node 上起一个 daemon 进程， 劫持容器内的 /proc 读操作， 通过 fuse 文件系统转到对容器 cgroupfs 的读写，然后将信息返回给请求方**

![lxcfs-workflow]()

而lxcfs组件只有一个 daemon 进程，运行在每个 node 上（通常以DaemonSet方式部署）

```bash
(node) $ systemctl status lxcfs
● lxcfs.service - FUSE filesystem for LXC
   Loaded: loaded (/usr/lib/systemd/system/lxcfs.service; enabled; vendor preset: disabled)
   Active: active (running)
```

lxcfs提供的目录挂载范围如下，默认 fuse 挂载路径 /var/lib/lxcfs/

```bash
$ tree /var/lib/lxcfs/ -L 2
/var/lib/lxcfs/
├── cgroup
│   ├── blkio
│   ├── cpu,cpuacct
│   ├── cpuset
│   ├── devices
│   ├── freezer
│   ├── hugetlb
│   ├── memory
│   ├── name=systemd
│   ├── net_cls,net_prio
│   ├── perf_event
│   ├── pids
│   └── rdma
└── proc
    ├── cpuinfo
    ├── diskstats
    ├── meminfo
    ├── stat
    ├── swaps
    └── uptime
```

##  0x05 参考
-   [a linux-container-and-runtime example](https://github.com/ArthurChiao/arthurchiao.github.io/blob/master/assets/code/linux-container-and-runtime/ctn-d.c)
-   [100 行 C 代码创建一个 KVM 虚拟机（2019）](https://arthurchiao.art/blog/kvm-host-in-a-few-lines-of-code-zh/)