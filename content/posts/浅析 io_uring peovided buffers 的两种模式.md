+++
date = '2025-01-19T21:26:44+08:00'
draft = false
title = '浅析 io_uring provided buffers 的两种模式'
isStarred = false

+++

## 一、echo server（provided buffer）

[这里](https://github.com/frevib/io_uring-echo-server)实现了一个通过 provided buffer 特性的 echo server，其中采用了 liburing 封装了 io_uring 中针对 provided buffer 特性的接口，本文从这个实例出发，介绍 provided buffer 特性。

首先，设置 buffer 的主要的函数调用逻辑为：

![image-20250203174409198](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20250203174409198.png)

通过将用户空间中的 buffer 相关信息写入 sqe，并通过 sqe 请求向内核中的 io_uring 设置 provided buffer，其中 sqe 的参数设置参考 io_uring_prep_provided_buffers，主要是传入 buffer 的大小和内存地址，代码如下：

```c
// 设置 group_id，同时 buffer id 设置为 0
io_uring_prep_provide_buffers(sqe, bufs, MAX_MESSAGE_LEN, BUFFERS_COUNT, group_id, 0);

// 设置 sqe 参数
IOURINGINLINE void io_uring_prep_provide_buffers(struct io_uring_sqe *sqe,
						 void *addr, int len, int nr,
						 int bgid, int bid)
{
	io_uring_prep_rw(IORING_OP_PROVIDE_BUFFERS, sqe, nr, addr, (__u32) len,
				(__u64) bid);
	sqe->buf_group = (__u16) bgid;
}

IOURINGINLINE void io_uring_prep_rw(int op, struct io_uring_sqe *sqe, int fd,
				    const void *addr, unsigned len,
				    __u64 offset)
{
	sqe->opcode = (__u8) op;
	sqe->fd = fd;
	sqe->off = offset;
	sqe->addr = (unsigned long) addr;
	sqe->len = len;
}

```

在这个 sqe 请求完成后，对应的 buffer 已经注册到 io_uring 当中。之后的 socket recv 请求不再需要设置用户缓冲区了，而是通过如下方式，io_uring 会返回对应的 provided buffer，例如，这里为一个 socket 添加的 recv 请求代码为：

```c
// 设置 flags 为 IOSQE_BUFFER_SELECT，传入 group_id，以及单个 buffer 的长度
add_socket_read(&ring, sock_conn_fd, group_id, MAX_MESSAGE_LEN, IOSQE_BUFFER_SELECT);

// 这里的 io_uring_prep_recv 为具体的 recv io_uring 请求
void add_socket_read(struct io_uring *ring, int fd, unsigned gid, size_t message_size, unsigned flags) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    io_uring_prep_recv(sqe, fd, NULL, message_size, 0);
    io_uring_sqe_set_flags(sqe, flags);
    sqe->buf_group = gid;

    conn_info conn_i = {
        .fd = fd,
        .type = READ,
    };
    memcpy(&sqe->user_data, &conn_i, sizeof(conn_i));
}

// 可以看到传入的 buffer 为空，io_uring 会从 buf_group 中找到可用的 buffer
IOURINGINLINE void io_uring_prep_recv(struct io_uring_sqe *sqe, int sockfd,
				      void *buf, size_t len, int flags)
{
	io_uring_prep_rw(IORING_OP_RECV, sqe, sockfd, buf, (__u32) len, 0);
	sqe->msg_flags = (__u32) flags;
}
```

这样的 recv 请求，最终根据 cqe 再拿到 buffer id，读取接收的数据：

```c
...
else if (type == READ) {
  int bytes_read = cqe->res;
  int bid = cqe->flags >> 16;
  if (cqe->res <= 0) {
    // read failed, re-add the buffer
    add_provide_buf(&ring, bid, group_id);
    // connection closed or error
    close(conn_i.fd);
  } else {
    // bytes have been read into bufs, now add write to socket sqe
    add_socket_write(&ring, conn_i.fd, bid, bytes_read, 0);
  }
```

在 recv 失败，或者 buffer 使用完毕以后，通过同样的方式，以 sqe 请求的方式重新在 io_uring 中提供缓冲区，也就是这里的 add_provide_buf 函数。

![image-20250203184344657](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20250203184344657.png)

## 二、UDP server

在 liburing 的 GitHub 仓库中也实现了一个采用 provided buffer 的 UDP server，但是这里的使用方式和上面的 echo server 有些不同。这里首先通过 io_uring_register_buf_ring 为 io_uring 注册一个环形缓冲区，注册成功以后需要调用 io_uring_buf_ring_add() 方法，通知内核 buffer ring group 的 buffer 是否可用，添加以后，其所有权就交给内核了，之后内核就会通过 cqe 将对应的 buffer 再交还用户空间读取。在  io_uring_buf_ring_add() 添加了一些 buffer 以后，需要调用 io_uring_buf_ring_advance() 通知内核 buffer ring 的变化。对于 buffer 在 sqe 和 cqe 的使用，和上面的 echo server 一样，在 sqe 的 flags 设置  IOSQE_BUFFER_SELECT，buf 标记为 NULL，选定 buf_group。一旦 io_uring 准备接收数据，就会从对应 的 buf group 中选定一个空闲的 buf_id，填进 cqe 的 flags 中。

在 liburing 中代码如下：

```c
static int setup_buffer_pool(struct ctx *ctx)
{
	int ret, i;
	void *mapped;
	struct io_uring_buf_reg reg = { .ring_addr = 0,
					.ring_entries = BUFFERS,
					.bgid = 0 };

  // 这里的 struct io_uring_buf 表示一个 buffer 的元数据，包含 addr，len，bid 等等
	ctx->buf_ring_size = (sizeof(struct io_uring_buf) + buffer_size(ctx)) * BUFFERS;
	mapped = mmap(NULL, ctx->buf_ring_size, PROT_READ | PROT_WRITE,
		      MAP_ANONYMOUS | MAP_PRIVATE, 0, 0);
	if (mapped == MAP_FAILED) {
		fprintf(stderr, "buf_ring mmap: %s\n", strerror(errno));
		return -1;
	}
	ctx->buf_ring = (struct io_uring_buf_ring *)mapped;
	io_uring_buf_ring_init(ctx->buf_ring);

	reg = (struct io_uring_buf_reg) {
		.ring_addr = (unsigned long)ctx->buf_ring,
		.ring_entries = BUFFERS,
		.bgid = 0
	};
	ctx->buffer_base = (unsigned char *)ctx->buf_ring +
			   sizeof(struct io_uring_buf) * BUFFERS;

  // 这里是 liburing 封装的注册 ring buffer 的接口
	ret = io_uring_register_buf_ring(&ctx->ring, &reg, 0);
	if (ret) {
		fprintf(stderr, "buf_ring init failed: %s\n"
				"NB This requires a kernel version >= 6.0\n",
				strerror(-ret));
		return ret;
	}

	for (i = 0; i < BUFFERS; i++) {
    // 将 buffer 的所有权移交给内核
		io_uring_buf_ring_add(ctx->buf_ring, get_buffer(ctx, i), buffer_size(ctx), i,
				      io_uring_buf_ring_mask(BUFFERS), i);
	}
  // 通知内核可见 buffer group 的改动
	io_uring_buf_ring_advance(ctx->buf_ring, BUFFERS);

	return 0;
}
```

在这里，整个 ring buffer 的结构如下：

![image-20250203202019529](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20250203202019529.png)

在用户空间使用完一个 buffer 之后，继续通过 add 和 advance 通知内核 ring 的改动，代码中为：

```c
static void recycle_buffer(struct ctx *ctx, int idx)
{
	io_uring_buf_ring_add(ctx->buf_ring, get_buffer(ctx, idx), buffer_size(ctx), idx,
			      io_uring_buf_ring_mask(BUFFERS), 0);
	io_uring_buf_ring_advance(ctx->buf_ring, 1);
}
```

具体的使用流程和 echo server 类似，接下来从 liburing 的视角看这几个函数的作用。

### 1. io_uring_register_buf_ring

这个函数是简单的对 register 的系统调用的封装，调用链为：

io_uring_register_buf_ring->do_register->__sys_io_uring_register

### 2. io_uring_buf_ring_add && io_uring_buf_ring_advance

```c
/*
 * Assign 'buf' with the addr/len/buffer ID supplied
 */
IOURINGINLINE void io_uring_buf_ring_add(struct io_uring_buf_ring *br,
					 void *addr, unsigned int len,
					 unsigned short bid, int mask,
					 int buf_offset)
{
	struct io_uring_buf *buf = &br->bufs[(br->tail + buf_offset) & mask];

	buf->addr = (unsigned long) (uintptr_t) addr;
	buf->len = len;
	buf->bid = bid;
}

/*
 * Make 'count' new buffers visible to the kernel. Called after
 * io_uring_buf_ring_add() has been called 'count' times to fill in new
 * buffers.
 */
IOURINGINLINE void io_uring_buf_ring_advance(struct io_uring_buf_ring *br,
					     int count)
{
	unsigned short new_tail = br->tail + count;

	io_uring_smp_store_release(&br->tail, new_tail);
}
```

在调用这两个函数之后，内核中的 io_uring 就知道当前 ring buffer 的 tail 到了哪里，且整个 ring 已经填满对应的缓冲区。注意环形结构存在在 meta data 中，下面举例说明：

![image-20250203205343714](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20250203205343714.png)

## 三、tokio io-uring 设计方案

在 tokio io-uring 中只实现了 echo server 形式的 provided buffer，而没有第二种形式的 mapped buffer ring，起 opcode 如下：

```rust
opcode! {
    /// Register `nbufs` buffers that each have the length `len` with ids starting from `bid` in the
    /// group `bgid` that can be used for any request. See
    /// [`BUFFER_SELECT`](crate::squeue::Flags::BUFFER_SELECT) for more info.
    pub struct ProvideBuffers {
        addr: { *mut u8 },
        len: { i32 },
        nbufs: { u16 },
        bgid: { u16 },
        bid: { u16 }
        ;;
    }

    pub const CODE = sys::IORING_OP_PROVIDE_BUFFERS;

    pub fn build(self) -> Entry {
        let ProvideBuffers { addr, len, nbufs, bgid, bid } = self;

        let mut sqe = sqe_zeroed();
        sqe.opcode = Self::CODE;
        sqe.fd = nbufs as _;
        sqe.__bindgen_anon_2.addr = addr as _;
        sqe.len = len as _;
        sqe.__bindgen_anon_1.off = bid as _;
        sqe.__bindgen_anon_4.buf_group = bgid;
        Entry(sqe)
    }
}
```

## 四、参考资料

- [echo server with provided buffer](https://github.com/frevib/io_uring-echo-server)
- [liburing](https://github.com/axboe/liburing)
- [Tokio io_uring](https://github.com/tokio-rs/io-uring)