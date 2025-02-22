---

title: Occlum 异步网络框架
date: 2024-10-11
author: charming-c
description: 分析并详细解释整个 Occlum 异步网络框架.
isStarred: true
---

## 1. io_uring_callback

这是 libos 中对 occlum io_uring 的进一步封装。对于原始的 occlum io_uring，他最大的缺点是，使用者必须手动的从 CQ 中获得 CQE，然后映射到对应的 SQE 的请求进行处理，这使得整个使用流程略显繁琐。io_uring_callback 提供了用户更加友好的 io_uring api，其特点主要表现在下面三个方面：

- callback_based，在一个 IO 请求被完成后，对应的用户注册的回调函数会被调用，而不需要手动分配 IO 的完成事件。
- Async/await，在提交完 io 请求之后，用户会获得一个代表这个正在进行的 io 请求的 handle，可以 await 这个 handle，类似 future。
- polling_based IO，无论是 IO 的提交还是完成都便于使用轮询模式。

下面先介绍 io_uring 的总体设计，然后从上面三个方面详细介绍具体实现原理。

![image-20241025141435122](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241025141435122.png)

这里介绍一下各个模块的作用：

- Builder：直接套壳的 io_uring 中的 builder，也是 occlum 中创建 IoUring 的唯一方式，可以在创建前配置对应的参数，以便在内核实例化创建的 io_uring 具有不同的特性。
- IoUring：核心结构体，主要包括的字段有：
    - ring：io_uring 实例，调用 crate 进行具体的操作，
    - fd_map：当前 io_uring 中所有处理过的 IO 事件的 fd 以及 io 事件数，除非 unregister 否则一直存在
    - sq_lock：为向 SQ 提交 SQE 加锁，防止同一时间操作同一个 SQ，线程安全
    - token_table：标识 SQE 和 CQE 的一一映射，是回调和 handle 实现的核心
- 在 IoUring 中的方法主要分为两类：
    - op_method：主要是针对特定的 opcode 的对 socket 的封装，比如 accept 等，创建出 SQE，随后通过调用 push_entry，传递进 SQ
    - 其他方法，这里特别指出 poll_completions，此方法包含了 IoUring 对 CQ 操作的封装

解下来分析其中几个重点方法：`push_entry`，`poll_completions`，`cancel`。

io_uring_callback 每次创建 SQE 时会生成一个对应的 IoHandle，这个 Iohandle 的结构大致如下所示：

```c
IoHandle
	-> IoToken
    	->Inner
    		->  state
    			completion_callback
    			token_key
```

在注册 SQE 时，传入 callback 参数，然后在提交 SQE 时，通过生成一个 token_key，注册在 IoUring 的 token_table 中，不仅关联了 SQE 和 IoHandle，通过将 userdata 设置成 tokenkey 还可以将 SQE 和 CQE 对应起来，大致流程如下所示：

![image-20241025162339077](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241025162339077.png)

回调机制的实现流程：

1. 用户通过 fd、callback 调用对应的 op method 创建 sqe
2. 在 IoUring 的 token_table 中生成一个 token_key，利用 token_key 和 callback 生成 token，将 token_key 和 token 关联注册到 table 中
3. 在 sqe 的 userdata 中写入 token_key，push 到 SQ 中
4. 在 CQ 中对应的 cqe 从 ring 中出来后根据 key 的值找到 token
5. 将 token 从 table 中注销
6. 调用 token 中的 callback 实现回调

这只是单个 sqe 的同步流程，对于已经在运行的异步 IoUring 的宏观形势如下：

![image-20241025163108539](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241025163108539.png)

token_table 注册大量已经运行，或者正在被取消的 IO 事件，通过`cancel`方法取消一个已经提交的事件，同时 handle 的存在还可以实时维护在内核运行中的 IO 事件的状态，具体表现为：

- 一开始创建的 sqe，在 push_entry 后提交到内核后为 submitted 状态
- 如果想要取消 IO，则会向内核发起取消请求，此时 Token 标记为 cacelling

即：在提交到内核 IO 事件处理完成前，Token 只有两个状态，即 submitted 和 cancelling

- 在`poll_completions` 后，cqe 出现，利用 key 找到对应的 Token
    - Token 为 submitted，一个正常的 IO 事件完成，调用回调，标记为 processed
    - Token 为 cacelling
        - 如果返回值为 CANCEL_RETVAL，则表示已取消，标记为 cacelled
        - 如果返回值不是，则表示取消失败，标记为 processed，仍旧回调

整个大致的基于回调和 handle 的机制大概就是这样。

## 2. Libos UringSet 单例

在 Occlum libos 中整个网络的 socket 层使用单例的 UringSet 进行 IO 操作，结构为：

![image-20241025174208630](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241025174208630.png)

主要对外暴露两个接口：

- get_uring：分为初始阶段和非初始阶段
    - 初始阶段：依据配置创建足够数量的 IoUring 实例
    - 非初始阶段：不创建，直接找工作负载最少的 IoUring 实例（两个方面，注册的 socketfd 以及 op num）
- disattach_uring
    - 从 IoUring 中取消注册一个 socketfd

## 3. Event 模型

occlum 中事件机制共有三种：

- Waiter、Waker 和 Waiter Queue，和线程休眠唤醒相关
- Event、Observer 和 Notifier 用来处理和广播事件
- WaiterQueueObserver 实现了当事件发生唤醒对应线程的机制

这里主要介绍如何处理和广播事件，其机制大致如下图所示：

![image-20241026210855401](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241026210855401.png)

对于一个 Notifier，其内部动态维护者一个 subscriber 队列，可以通过 register/unregister 从 notifier 中添加和删除 subscriber，Notifiter 每次向队列中的每一个 subscriber 广播事件。而每一个 subscriber 中由一个 observer 和 filter 组成，filter 过滤当前 subscriber 不感兴趣的事件，在 Notifiter 广播事件时，不做处理，observer 则在得到事件时执行对应的 on_event 行为。

**Pollee-Poller 模型：**

![image-20241026215720502](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241026215720502.png)

Pollee 作为 Notifier，Poller 作为 subscriber，在 Poller 通过 connect 和 Pollee 关联后，每次有 add_event 事件添加时，都会向所有的 subscriber 广播，Poller 会执行对应的线程唤醒操作，其他的 Observer 则会执行对应的 on_event。

## 4. Socket file

在 Linux 中将 socket 看做一个文件，在 Occlum 中也是如此。对于所有使用 io_uring 实现的异步 socket，其对应的系统调用经过 Occlum 包装后返回的都是 SocketFile 类型，不同的 socket 类型在 SocketFile 中的 AnySocket 枚举中实现，其对外的接口由 SocketFile 暴露，总体架构如下：

![image-20241107230439318](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241107230439318.png)

当对 Occlum libos 发起一个 Socket 系统调用时，首先会判断 socket 的类型。代码示例如下(这里节选自 bind 系统调用：

```rust
 let file_ref = current!().file(fd as FileDesc)?;
    if let Ok(socket) = file_ref.as_host_socket() {
        let raw_addr = addr.to_raw();
        socket.bind(&raw_addr)?;
    } else if let Ok(unix_socket) = file_ref.as_unix_socket() {
        let unix_addr = addr.to_unix()?;
        unix_socket.bind(unix_addr)?;
    } else if let Ok(uring_socket) = file_ref.as_uring_socket() {
        uring_socket.bind(&addr)?;
    } else {
        return_errno!(ENOTSOCK, "not a socket");
    }
```

这里的第一行`current!().file(fd as FileDesc)?`如果成功的话，会返回一个 FileRef 类型，在 socket_file 中，有如下实现：

```rust
pub trait UringSocketType {
    fn as_uring_socket(&self) -> Result<&SocketFile>;
}

impl UringSocketType for FileRef {
    fn as_uring_socket(&self) -> Result<&SocketFile> {
        self.as_any()
            .downcast_ref::<SocketFile>()
            .ok_or_else(|| errno!(ENOTSOCK, "not a uring socket"))
    }
}
```

因此对于一个 uring socket，可以通过普通的 fd 和 as_uring_socket 方法拿到 SocketFile，而 SocketFile 的实现很简单：

```rust
pub struct SocketFile {
    socket: AnySocket,
}

enum AnySocket {
    Ipv4Stream(Ipv4Stream),
    Ipv6Stream(Ipv6Stream),
    Ipv4Datagram(Ipv4Datagram),
    Ipv6Datagram(Ipv6Datagram),
}
```

SocketFile 中拥有 syscall 对应的接口，对于 Socket 中比较通用的操作，比如 write、read，采用 apply_fn_on_any_socket! 宏实现，而其它不同 Socket 实现逻辑不同的接口则通过 AnySocket 调用具体枚举类实现，比如 bind、listen、accept 等。

## 5. Common

common 中集合了不同状态下 Stream Socket 中的一些通用的属性和方法，比如是否阻塞、是否已经关闭、绑定的 io_uring 实例等等，其定义如下：

```rust
// A 泛型标识地址类型，比如 IPv4、IPv6，R 泛型表示运行时，这里主要用于 io_uring
pub struct Common<A: Addr + 'static, R: Runtime> {
    host_fd: FileDesc,					// host_fd：主机上的文件描述符
    type_: SocketType,					// socket 类型
    nonblocking: AtomicBool,			// 是否是阻塞类型
    is_closed: AtomicBool,				// socket 是否关闭
    pollee: Pollee,						// 用于 Pollee-Poller 机制，实现异步 IO
    inner: Mutex<Inner<A>>,				// 保存了一个 Stream Socket 连接（bind）的 C/S 地址
    timeout: Mutex<Timeout>,			// 等待时延
    errno: Mutex<Option<Errno>>,		// 错误码
    io_uring: Arc<IoUring>,				// socket 绑定的 IoUring 实例
    phantom_data: PhantomData<(A, R)>,	// 标识 Common 结构体拥有和泛型一样的生命周期
}
```

common 中主要就是维护一些 socket 共有的属性和方法，同时也封装了除去 IO 相关的系统调用，例如 do_bind、do_close 等。

## 6. Datagram Socket

在 Occlum 中 Datagram 类型的 socket 实现如下：

```rust
// 泛型类型和 Common 类似
pub struct DatagramSocket<A: Addr + 'static, R: Runtime> {
    common: Arc<Common<A, R>>,	// 持有一个 common 结构
    state: RwLock<State>,				// 当前 DG socket 的状态，包含绑定状态和是否连接
    sender: Arc<Sender<A, R>>,	// socket 的发送端，封装发送行为，对应 output 事件
    receiver: Arc<Receiver<A, R>>,	// socket 的接受端，封装接收行为，对应 input 事件
}

struct State {
    bind_state: BindState,	// 这里的绑定状态有三种：没有绑定、显式和隐式
    is_connected: bool,			// 是否建立连接
}
```

### 6.1 State 的转化

**没有绑定的状态：**

1. new 方法：在初次创建 socket 时，BoundState 为 unbound，is_connected 为 false
2. new_connection方法：BoundState 为 unbound，is_connected 为 true

> new_connection 方法主要是为了创建一个具有连接的 socket 对，此时这个连接就应该是固定的，不需要 bind 一个地址，有任何其他的 socket 向这个 socket 对的任何一个发送数据。

当创建一个 Datagram socket 以后，我们可以通过 bind，将 socket 和一个 Addr 进行绑定，之后其他的 socket 就可以向这个地址发送数据，绑定 addr 的 socket 负责接收数据。在这里这种绑定分为显式和隐式两种。一个 Datagram socket 只能绑定一次，在没有显式绑定的情况下，一些方法的调用会触发隐式绑定，在这之后，再进行显式绑定返回错误。

**显式绑定：**

通过 bind 函数，显式设置当前 socket 绑定的地址，并修改 state：

```rust
pub fn bind(&self, addr: &A) -> Result<()> {
  let mut state = self.state.write().unwrap();
  if state.is_bound() {
    return_errno!(EINVAL, "The socket is already bound to an address");
  }

  do_bind(self.host_fd(), addr)?;

  self.common.set_addr(addr);
  state.mark_explicit_bind();
  // Start async recv after explicit binding or implicit binding
  self.receiver.initiate_async_recv();

  Ok(())
}
```

所谓绑定，就是设置 common inner 中的 addr 的值。

**隐式绑定：**

当调用 connect、sendto、sendmsg 会触发隐式绑定。直接将 BoundState 设置成隐式绑定，具体的端口分配应该是有 host os 完成，在这之后 socket 就不能再使用 bind 绑定显式的地址。

### 6.2 Sender && Receiver

**数据接收：**

![image-20241116133759836](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241116133759836.png)

整个数据的接收流程如下：

1. 在显式绑定或者隐式绑定之后，Datagram socket 会对内核发送一个 recv 请求，也就是说在没有进行 recvmsg 调用之前，IoUring 已经开始接收网卡数据，并且保留在缓冲区中。
2. 之后当应用真的调用 recvmsg 之后，会通过 try_recvmsg 方法尝试从 recv_buf 中拷贝数据
3. 如果没有拷贝到数据，则判断是否出错或特殊处理，如果都不是则继续调用 do_recv

对于整个请求的构造，其结构如图：

![image-20241116142457541](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241116142457541.png)

在 Receiver 的 Inner 中包含了数据缓冲区和控制信息缓冲区，构造一条 request msg 到内核中的方式如下：

1. 初始化 recv_buf，msg_control
2. 构造一个 RecvReq，包含 msghdr、iovec、addr，其中 iovec 指向 recv_buf，缓存接收的数据，addr 保存原地址信息，msghdr 用于提交到 host os，传递 recv 的数据
3. 在 iouring recv 事件完成后，msghdr 中 msg_iovec（指向 recv_buf）保存了数据，msg_name（指向 addr）保存了数据的源地址、msg_control（指向 inner 的 msg_control）保存控制信息，后续可以直接通过 Inner 获取到对应的数据，完成 recv。

**数据发送：**

整个数据结构的组织如下：

![image-20241116185953561](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241116185953561.png)

对于应用来说，当 sendmsg 将数据写到 DataMsg 中的 send_buf 之后，会有 io_uring 将 msg 给到 host os 的内核进行实际的 send 系统调用。所有的发送数据被组织到一个 MsgQueue 中，队列先进先出，io_uring 中每完成一次 send 操作会 pop 出当前的队头，然后继续执行队列中后续数据的发送的操作。

## 7. Stream Socket

StreamSocket 的实现是面向连接的，因此，这里实现了一个状态机，按照连接中的 socket 的不同状态将 socket 分为：

```rust
pub struct StreamSocket<A: Addr + 'static, R: Runtime> {
    state: RwLock<State<A, R>>,
    common: Arc<Common<A, R>>,
}

enum State<A: Addr + 'static, R: Runtime> {
    // Start state 初始状态
    Init(Arc<InitStream<A, R>>),
    // Intermediate state 瞬时状态
    Connect(Arc<ConnectingStream<A, R>>),
    // Final state 1 终态 1
    Connected(Arc<ConnectedStream<A, R>>),
    // Final state 2 终态 2
    Listen(Arc<ListenerStream<A, R>>),
}
```

下面先介绍各个状态的实现和方法：

**Init Stream 状态：**

```rust
pub struct InitStream<A: Addr + 'static, R: Runtime> {
    common: Arc<Common<A, R>>,
    inner: Mutex<Inner>,
}
struct Inner {
    has_bound: bool,
}
```

在调用了构造方法之后，一个 Stream Socket 的初始状态为 InitStream，InitStream 只有是否 bind 的标识。

**Connecting Stream 状态：**

```rust
pub struct ConnectingStream<A: Addr + 'static, R: Runtime> {
    common: Arc<Common<A, R>>,
    peer_addr: A,	// 连接对侧地址
    req: Mutex<ConnectReq<A>>,	// 连接请求
    connected: AtomicBool, // Mainly use for nonblocking socket to update status asynchronously
}

struct ConnectReq<A: Addr> {
    io_handle: Option<IoHandle>,	// 请求的 io_handle
    c_addr: UntrustedBox<libc::sockaddr_storage>,	// socket 地址
    c_addr_len: usize,	// 地址长度
    errno: Option<Errno>,
    phantom_data: PhantomData<A>,
}
```

这是一个瞬时状态，会在建立连接的请求后发生改变，所以整个状态过程中就是发送建立连接的请求。在传入 peer_addr 创建 Connecting Stream 之后，可以调用 connect 方法发送连接请求，其具体实现在 initiate_async_connect 中：

1. 根据传入的 Addr 翻译为 C 的 addr，然后向 io_uring 注册一个 connect IO 请求。
2. 请求的 callback 为：
    1. 正常，则将 connected 参数设置为 true 标识连接建立成功
    2. 有错误码，根据具体的错误码，向 common 中的 pollee 添加事件，比如重置连接

**Connected Stream 状态：**

```rust
pub struct ConnectedStream<A: Addr + 'static, R: Runtime> {
    common: Arc<Common<A, R>>,
    sender: Sender,
    receiver: Receiver,
}
```

在建立连接之后，进入 ConnectStream 状态，通过 sender 和 receiver 进行数据的发送和接收，这个在之后详细解释。

**Listener Stream 状态：**

```rust
pub struct ListenerStream<A: Addr + 'static, R: Runtime> {
    common: Arc<Common<A, R>>,
    inner: Mutex<Inner<A>>,
}
struct Inner<A: Addr> {
    backlog: Backlog<A>,
    fatal: Option<Errno>,
}
struct Backlog<A: Addr> {
    // The entries in the backlog.
    entries: Box<[Entry]>,
    // Arguments of the io_uring requests submitted for the entries in the backlog.
    reqs: UntrustedBox<[AcceptReq]>,
    // The indexes of completed entries.
    completed: VecDeque<usize>,
    // The number of free entries.
    num_free: usize,
    phantom_data: PhantomData<A>,
}
```

进入 Listener Stream 状态的 socket 等待即将到来的 connect，主要维护了一组 backlog，即还未 connect 成功的 socket。

### 7.1 状态转化

状态转化如下图所示：

![image-20241117234937815](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241117234937815.png)

重点解析 connect 方法，首先将 connect 方法分成两部分来看，第一部分，转变瞬时状态 connecting：

```rust
let (init_stream, connecting_stream) = {
  let mut state = self.state.write().unwrap();
  match &*state {
    State::Init(init_stream) => {
      let connecting_stream = {
        let common = init_stream.common().clone();
        ConnectingStream::new(peer_addr, common)?
      };
      let init_stream = init_stream.clone();
      *state = State::Connect(connecting_stream.clone());
      (init_stream, connecting_stream)
    }
    State::Connect(connecting_stream) => {
      if let Some(connected_stream) =
      Self::try_switch_to_connected_state(connecting_stream)
      {
        *state = State::Connected(connected_stream);
        return_errno!(EISCONN, "the socket is already connected");
      } else {
        // Not connected, keep the connecting state and try connect
        let init_stream =
        InitStream::new_with_common(connecting_stream.common().clone())?;
        (init_stream, connecting_stream.clone())
      }
    }
    State::Connected(_) => {
      return_errno!(EISCONN, "the socket is already connected");
    }
    State::Listen(_) => {
      return_errno!(EINVAL, "the socket is listening");
    }
  }
};
```

由于 Stream Socket 的 connect 方法是一个耗时任务，需要连接到远程的 socket 地址，所以创建了 connecting 表示一个连接的请求发出后 socket 所处的状态。对于一个 Stream Socket 对于 connect 的调用，将情况分为三种：

1. Init 状态下：创建出新的瞬时状态 connecting
2. Connecting 状态：已经有 connect 请求发出，这里先检查是否已经建立连接，若有则直接返回，没有则创建出一个 Init 状态，为防止之后连接失败状态回退
3. 其他：返回不同的错误

在创建完瞬时状态后，进入第二部分，发起 connect 的异步请求：

```rust
let res = connecting_stream.connect();
  // If success, then the state is switched to connected; otherwise, for blocking socket
  // the state is restored to the init state, and for non-blocking socket, the state
  // keeps in connecting state.
  match &res {
    Ok(()) => {
      let connected_stream = {
        let common = init_stream.common().clone();
        common.set_peer_addr(peer_addr);
        ConnectedStream::new(common)
      };

      let mut state = self.state.write().unwrap();
      *state = State::Connected(connected_stream);
    }
    Err(_) => {
      if !connecting_stream.common().nonblocking() {
        let mut state = self.state.write().unwrap();
        *state = State::Init(init_stream);
      }
    }
  }
```

如果请求成功，则进入 connected 状态可以进行收发数据。对于阻塞式的 socket，不成功会进入 Init 状态，非阻塞则进入 connecting。

### 7.2 ListenerStream

从构造函数开始：

```rust
pub fn new(backlog: u32, common: Arc<Common<A, R>>) -> Result<Arc<Self>> {
  // Here we use different variables for the backlog. For the libos, as we will issue async accept request
  // ahead of time, and cacel when the socket closes, we set the libos backlog to a reasonable value which
  // is no greater than the max value we set to save resources and make it more efficient. For the host,
  // we just use the backlog value for maximum connection.
  let libos_backlog = std::cmp::min(backlog, PENDING_ASYNC_ACCEPT_NUM_MAX as u32);
  let host_backlog = backlog;

  let inner = Inner::new(libos_backlog)?;
  Self::do_listen(common.host_fd(), host_backlog)?;

  common.pollee().reset_events();
  let new_self = Arc::new(Self {
    common,
    inner: Mutex::new(inner),
  });

  // Start async accept requests right as early as possible to improve performance
  {
    let inner = new_self.inner.lock();
    new_self.initiate_async_accepts(inner);
  }

  Ok(new_self)
}
```

这里的 backlog 有两个，一个是 libos_backlog，用于系统调用，并且在 Listener Stream 的 inner 中作为监听队列的长度，模拟内核，存储还未被 accept 的连接，一个是 host_backlog，表示在 listen 时传入的 backlog 参数。在创建本状态时，就会调用 listen 接收连接，并且调用 initiate_async_accepts 将所有处在空闲状态的 Entry 都进行 accept 请求，以便之后的 accept 调用，和之前的 Datagram 的 recv 类似，都是预先操作。

![image-20241118004032893](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241118004032893.png)

在 new 之后，所有的 Backlog entry 都会发送 accept 请求到 IoUring，然后转化为 pending 状态，在连接进入之后，其回调如下：

```rust
// 部分
let host_fd = retval as FileDesc;
inner.backlog.entries[entry_idx] = Entry::Completed { host_fd };
inner.backlog.completed.push_back(entry_idx);

stream.common.pollee().add_events(IoEvents::IN);

stream.initiate_async_accepts(inner);
```

pending 转化为 completed，并且继续让所有的 free entry 向 IoUring 提交 accept 请求。对于外部调用的 accept 方法，并不发起请求，而是直接向已完成的 compelted queue 中拿到 fd 和 addr，如果没有则阻塞等待，知道有连接到达，在 callback 中添加唤醒事件。

### 7.3 ConnectedStream

在这个状态下进行数据的收发，主要实现和 DataGram 类似，不同的是在组装 request 时，不是简单的将缓冲区的地址写到 msg，而是需要在缓冲区记录位置偏移量，且在必要的时候更新缓冲区。