## BSD层socket

### 相关数据结构
```c
// include/linux/net.h
struct socket {
    socket_state        state;
    short               type;
    unsigned long       flags;
    struct socket_wq*   wq;
    struct file*        file; // 相关联的文件描述符
    struct sock*        sk; // INET层的sock结构
    const struct proto_ops* ops; // socke的操作接口, BSD层只定义了这么一个接口, 具体的实现则由具体类型的子模块完成
};
```

### socket create
- BSD层主要做两件事:
    1. 根据`family`参数选择对应的模块创建socket;
    2. 将socket绑定到当前进程的`fdt`中.

```c
// net/socket.c
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
    return __sys_socket(family, type, protocol);
}

int __sys_socket(int family, int type, int protocol)
{
    // ...
    retval = sock_create(/*current->nsproxy->net_ns,*/ family, type, protocol, &sock/*, 0*/); // 根据`family`选择具体的协议模块创建socket
    // ...
    return sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK)); // 为socket分配一个文件描述符, 并记录到当前进程的`fd`数组中.
}

int __sock_create(struct net* net, int family, int type, int protocol, struct socket** res, int kern)
{
    struct socket* sock;
    const struct net_proto_family* pf;
    // ...
    sock = sock_alloc(); // 分配一个inode用来放socket.
    // ...
    sock->type = type;
    if (rcu_access_pointer(net_families[family]) == NULL) // 如果对应的模块还没加载, 就先加载.
        request_module("net-pf-%d", family);

    pf = rcu_dereference(net_families[family]); // net_families 在sock_register中赋值, 协议模块加载的时候会调用sock_register进行注册.
    // ...
    err = pf->create(net, sock, protocol, kern); // 调用具体的协议模块来创建socket. 这应该算是下一层了, `struct socket`的创建和初始化之前已经完成了, 下一层主要是`struct sock`的创建和初始化.
    // ...
}

static int sock_map_fd(struct socket *sock, int flags)
{
    int fd = get_unused_fd_flags(flags); // 获取一个还未使用的文件描述符
    // ...
    newfile = sock_alloc_file(sock, flags, NULL); // 将INET层的socket和一个file结构绑定
    if (likely(!IS_ERR(newfile))) {
        fd_install(fd, newfile); // 将新的file结构和文件描述符绑定
        return fd;
    }
    // ...
}

struct file *sock_alloc_file(struct socket *sock, int flags, const char *dname)
{
    file = alloc_file_pseudo(SOCK_INODE(sock), sock_mnt, dname,  // 属于fs模块了
                O_RDWR | (flags & O_NONBLOCK),
                &socket_file_ops);
    sock->file = file;
    file->private_data = sock;
    return file;
}
```
