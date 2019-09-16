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

#### socket create (INET层)
- INET层主要完成`sock`结构体的初始化.

```c
// include/net/sock.h
struct sockt_common {
    // ...
    struct proto* skc_prot;
    // ...
};

// include/net/sock.h
struct sock {
    struct sock_common  `__sk_common`;
#define sk_prot         `__sk_common.skc_pro` // 用于操作`sock`的一些接口.
    // ...
    struct socket*      sk_socket;
    struct proto*       sk_prot_creator;
    // ...
};

// include/net/inet_sock
struct inet_sock {
    struct sockt sk;  // 相当于继承了`struct sock`, 到时候直接通过强制类型转换来访问
#if IS_ENABLED(CONFIG_IPV6)
    struct ipv6_pinfo* pinet6;
#endif
    // ...
};
```

```c
// net/ipv4/af_inet.c
static int inet_create(struct net *net, struct socket *sock, int protocol, int kern)
{
    struct sock* sk;
    struct inet_protosw answer;
    // ...
    // 通过一个inet_protosw结构体的链表(inetsw)维护协议的注册信息, 需要创建socket的时候就从inetsw中根据type和protocol进行查找.
    list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {
        if (protocol == answer->protocol) {
            if (protocol != IPPROTO_IP)
                break;
        }
        else {
            // ...
        }
    }
    // ...
    sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer->prot, kern); 给sock分配内存
    // ...
    struct inet_sock* inet;
    inet = inet_sk(sk);
    // ... // inet_sock结构体的初始化
    // ...
    sock_init_data(sock, sk); // sock结构体的初始化, 和上层的socket结构体进行绑定.
    // ...
    err = sk->sk_prot->hash(sk);
    // ...
    err = sk->sk_prot->init(sk); // 调用具体的协议模块对socket进行初始化.
    // ...
}
```
