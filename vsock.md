# VSock (AF_VSOCK) in the Linux Kernel

> Source tree: `net/vmw_vsock/`, `include/net/af_vsock.h`, `uapi/linux/vm_sockets.h`

---

## 1. What is VSock?

VSock is a socket address family (`AF_VSOCK = 40`) designed for communication between **virtual machines and their hypervisor** — or between VMs on the same host — without needing a full network stack. It provides a standard POSIX socket API (`socket`, `bind`, `listen`, `accept`, `connect`, `send`, `recv`) over a hypervisor-specific transport.

---

## 2. The Address Model — CID + Port

Instead of IP addresses, vsock uses:

- **CID** (Context ID) — identifies a VM or the host. Well-known values:
  - `VMADDR_CID_LOCAL (1)` — loopback within the same endpoint
  - `VMADDR_CID_HOST (2)` — the hypervisor
  - CID ≥ 3 — guest VMs (assigned by the hypervisor)
- **Port** — analogous to a TCP/UDP port number

`struct sockaddr_vm` is the address type (`uapi/linux/vm_sockets.h`):

```c
struct sockaddr_vm {
    sa_family_t     svm_family;   // AF_VSOCK
    unsigned short  svm_reserved1;
    unsigned int    svm_port;
    unsigned int    svm_cid;
    __u8            svm_flags;
    unsigned char   svm_zero[...];
};
```

---

## 3. Socket Types Supported

| Type | Ops table (af_vsock.c) | Notes |
|---|---|---|
| `SOCK_DGRAM` | `vsock_dgram_ops` (line 1615) | Connectionless |
| `SOCK_STREAM` | `vsock_stream_ops` (line 2633) | Reliable, ordered byte stream |
| `SOCK_SEQPACKET` | `vsock_seqpacket_ops` (line 2655) | Reliable, ordered, message-preserving |

DGRAM sockets never enter the bound hash table buckets — they stay on the unbound list (`vsock_bind_table[VSOCK_HASH_SIZE]`) to avoid polluting stream socket lookups.

---

## 4. Core Data Structures

### `struct vsock_sock` (`include/net/af_vsock.h:29`)

Embeds `struct sock` as its **first member** (required by the kernel socket framework):

```c
struct vsock_sock {
    struct sock sk;                       // generic socket — MUST be first
    const struct vsock_transport *transport;  // pluggable backend
    struct sockaddr_vm local_addr;
    struct sockaddr_vm remote_addr;

    // Global hash table links
    struct list_head bound_table;         // link in vsock_bind_table
    struct list_head connected_table;     // link in vsock_connected_table

    // Connection queues (on the listener socket)
    struct sock     *listener;            // for pending sockets: their server
    struct list_head pending_links;       // half-open connections (multi-step handshakes)
    struct list_head accept_queue;        // fully established, waiting for accept()
    bool             rejected;            // accept() rejected this socket

    // Deferred work
    struct delayed_work connect_work;     // fires on connect() timeout
    struct delayed_work pending_work;     // cleans up stale half-open sockets
    struct delayed_work close_work;

    u64  buffer_size;
    u64  buffer_min_size;
    u64  buffer_max_size;

    void *trans;                          // transport-private data
};
```

Cast macros:
```c
#define vsock_sk(__sk)   ((struct vsock_sock *)__sk)
#define sk_vsock(__vsk)  (&(__vsk)->sk)
```

### Global Hash Tables (`af_vsock.c:244`)

```c
struct list_head vsock_bind_table[VSOCK_HASH_SIZE + 1];  // keyed by port
struct list_head vsock_connected_table[VSOCK_HASH_SIZE]; // keyed by CID^port
DEFINE_SPINLOCK(vsock_table_lock);
```

- `vsock_bind_table[VSOCK_HASH(addr)]` — bound sockets, looked up when a packet arrives
- `vsock_bind_table[VSOCK_HASH_SIZE]` — special bucket for unbound/DGRAM sockets
- `vsock_connected_table[VSOCK_CONN_HASH(src,dst)]` — connected sockets, looked up for in-flow data

---

## 5. Pluggable Transport Layer

The most important architectural choice: **vsock has no transport of its own**. All actual I/O is delegated through `struct vsock_transport` (`include/net/af_vsock.h:108`), a vtable of function pointers.

### Four Transport Slots (`af_vsock.c:205–211`)

```c
static const struct vsock_transport *transport_h2g;    // host → guest
static const struct vsock_transport *transport_g2h;    // guest → host
static const struct vsock_transport *transport_dgram;
static const struct vsock_transport *transport_local;  // loopback
```

### Transport Feature Flags

```c
#define VSOCK_TRANSPORT_F_H2G    0x00000001
#define VSOCK_TRANSPORT_F_G2H    0x00000002
#define VSOCK_TRANSPORT_F_DGRAM  0x00000004
#define VSOCK_TRANSPORT_F_LOCAL  0x00000008
```

### Concrete Backends

| File | Transport | Used when |
|---|---|---|
| `virtio_transport.c` | virtio-vsock | QEMU/KVM |
| `vmci_transport.c` | VMware VMCI | VMware hypervisors |
| `hyperv_transport.c` | Hyper-V | Microsoft Hyper-V |
| `vsock_loopback.c` | local loopback | same-VM communication |

### Transport Selection (`vsock_assign_transport`, af_vsock.c:557)

Called at `connect()` / listen-side `accept()` time, based on remote CID:

```c
switch (sk->sk_type) {
case SOCK_DGRAM:
    new_transport = transport_dgram;
case SOCK_STREAM / SOCK_SEQPACKET:
    if (vsock_use_local_transport(remote_cid))  → transport_local
    else if (remote_cid <= VMADDR_CID_HOST)     → transport_g2h
    else if (transport_h2g->has_remote_cid(...)) → transport_h2g
    else fallback via g2h with TO_HOST flag
}
```

`try_module_get()` is called on the selected transport module to prevent unloading while sockets are open.

---

## 6. Socket State Machine

VSock reuses TCP state constants (they are already exported to userspace tools like `ss(8)`):

| State | Meaning |
|---|---|
| `TCP_CLOSE` | Unconnected (initial) |
| `TCP_SYN_SENT` | `connect()` in progress |
| `TCP_ESTABLISHED` | Connected |
| `TCP_CLOSING` | `shutdown()` in progress |
| `TCP_LISTEN` | Listening for connections |

---

## 7. Server-Side Connection Workflow

### Phase 1 — Userspace Setup

```
socket(AF_VSOCK, SOCK_STREAM, 0)
    └─ __vsock_create()          [af_vsock.c:897]
           sk_alloc() → allocates vsock_sock via SLUB
           sock_init_data()
           state = TCP_CLOSE
           pending_links, accept_queue initialized as empty lists
           connect_work, pending_work armed but not scheduled

bind(fd, {AF_VSOCK, port, cid})
    └─ __vsock_bind()            [af_vsock.c:858]
           inserts sock into vsock_bind_table (keyed by port)

listen(fd, backlog)
    └─ vsock_listen()            [af_vsock.c:1940]
           sk->sk_max_ack_backlog = backlog
           sk->sk_state = TCP_LISTEN
           (no transport involvement)
```

### Phase 2 — Incoming Packet (Transport IRQ/Softirq → Core)

```
virtio_transport_recv_pkt()      [virtio_transport_common.c:1608]
  ├─ parse src/dst CID+port from virtio packet header
  ├─ vsock_find_bound_socket()   ← lookup listener in vsock_bind_table
  ├─ lock_sock(listener_sk)
  └─ switch (sk->sk_state):
       case TCP_LISTEN:
           virtio_transport_recv_listen()
```

Inside `virtio_transport_recv_listen()` (`virtio_transport_common.c:1531`):

```
1. Validate: packet op must be VIRTIO_VSOCK_OP_REQUEST
2. sk_acceptq_is_full(sk)?       → send RST, return -ENOMEM
3. sk->sk_shutdown == MASK?      → send RST, return -ESHUTDOWN
4. child = vsock_create_connected(listener_sk)
       inherits: connect_timeout, buffer sizes, credentials from parent
5. lock_sock_nested(child, SINGLE_DEPTH_NESTING)
6. child->sk_state = TCP_ESTABLISHED    ← virtio does one-shot handshake
7. Set child local_addr (hdr->dst) and remote_addr (hdr->src)
8. vsock_assign_transport(vchild, vsk)  ← picks and inits transport backend
9. sk_acceptq_added(listener_sk)        ← sk_ack_backlog++
10. vsock_insert_connected(vchild)      ← add to vsock_connected_table
11. vsock_enqueue_accept(listener, child)
        sock_hold(child); sock_hold(listener)
        list_add_tail(&vchild->accept_queue, &vlistener->accept_queue)
12. virtio_transport_send_response()    ← send OP_RESPONSE to peer
13. release_sock(child)
14. listener->sk_data_ready(listener)  ← wake up accept() sleeper
        └─ sock_def_readable()
               wake_up_interruptible_sync_poll(&wq->wait, EPOLLIN|...)
```

**Key**: virtio-vsock does a **one-shot handshake**. The child goes directly to `TCP_ESTABLISHED`. The `pending_links` list is not used (it was designed for VMCI's multi-step handshake).

### Phase 3 — `accept()` returns

Handled by `vsock_accept()` — see detailed breakdown in section 8.

### Full Flow Diagram

```
Userspace                    vsock core                      Transport (IRQ/softirq)
──────────────────────────────────────────────────────────────────────────────────────
socket()  ──────────────►  __vsock_create()
                            state=TCP_CLOSE
bind()    ──────────────►  insert into vsock_bind_table
listen()  ──────────────►  state=TCP_LISTEN
accept()  ──────────────►  vsock_accept()
                            schedule_timeout() ◄─ sleeping ──────────────────────────┐
                                                                                      │
                                               ◄── virtio_transport_recv_pkt() ───────┤
                                                     vsock_find_bound_socket()        │
                                                     virtio_transport_recv_listen()   │
                                                       vsock_create_connected()       │
                                                       child state=TCP_ESTABLISHED    │
                                                       vsock_assign_transport()       │
                                                       sk_acceptq_added()             │
                                                       vsock_insert_connected()       │
                                                       vsock_enqueue_accept()         │
                                                       send OP_RESPONSE to peer       │
                                                       sk_data_ready() ───────────────┘
                            vsock_dequeue_accept()
                            sock_graft() → wire child to newsock fd
accept() returns ◄────────  release_sock()
```

---

## 8. Deep Dive: `vsock_accept()` (`af_vsock.c:1854`)

```c
static int vsock_accept(struct socket *sock,      // listening socket
                        struct socket *newsock,   // fresh empty socket from VFS
                        struct proto_accept_arg *arg)
```

`newsock` is allocated by the VFS before calling here. `arg->flags` carries `O_NONBLOCK`.

### Step 1 — Guard checks (lines 1867–1877)

```c
lock_sock(listener);

if (!sock_type_connectible(sock->type))  → -EOPNOTSUPP  (DGRAM can't accept)
if (listener->sk_state != TCP_LISTEN)   → -EINVAL
```

`lock_sock()` acquires `sk->sk_lock.slock` (spinlock) and sets `owned=1`, serializing all socket operations. Softirq can still run but will queue to the backlog instead of calling into this socket directly.

### Step 2 — Timeout calculation (line 1882)

```c
timeout = sock_rcvtimeo(listener, arg->flags & O_NONBLOCK);
// → O_NONBLOCK set:  timeout = 0   (skip sleep loop, return -EAGAIN immediately)
// → blocking:        timeout = sk->sk_rcvtimeo  (MAX_SCHEDULE_TIMEOUT if no SO_RCVTIMEO)
```

### Step 3 — The sleep loop (lines 1884–1896)

```c
while ((connected = vsock_dequeue_accept(listener)) == NULL
       && listener->sk_err == 0
       && timeout != 0)
{
    prepare_to_wait(sk_sleep(listener), &wait, TASK_INTERRUPTIBLE);
    release_sock(listener);
    timeout = schedule_timeout(timeout);
    finish_wait(sk_sleep(listener), &wait);
    lock_sock(listener);

    if (signal_pending(current)) {
        err = sock_intr_errno(timeout);
        goto out;
    }
}
```

This is the **standard kernel sleep-on-condition** pattern. Each piece:

#### `vsock_dequeue_accept()` (line 703)

```c
vconnected = list_entry(vlistener->accept_queue.next,
                        struct vsock_sock, accept_queue);
list_del_init(&vconnected->accept_queue);
sock_put(listener);   // drop the ref held by vsock_enqueue_accept()
return sk_vsock(vconnected);
```

Returns `NULL` if the list is empty. Requires caller to hold `lock_sock(listener)`.

#### `sk_sleep(listener)` (sock.h:2124)

```c
return &rcu_dereference_raw(sk->sk_wq)->wait;
```

Returns the `wait_queue_head_t` embedded in the socket's `socket_wq`. The same queue that `sk_data_ready()` wakes.

#### `prepare_to_wait()` (wait.c:248)

```c
wq_entry->flags &= ~WQ_FLAG_EXCLUSIVE;   // non-exclusive: ALL waiters wake on one connection
spin_lock_irqsave(&wq_head->lock, flags);
if (list_empty(&wq_entry->entry))
    __add_wait_queue(wq_head, wq_entry);  // add to head of wait list
set_current_state(TASK_INTERRUPTIBLE);   // smp_store_mb() — full memory barrier
spin_unlock_irqrestore(&wq_head->lock, flags);
```

Two things happen atomically (under the wait queue spinlock):
1. Task is enqueued onto the socket's wait queue.
2. `TASK_INTERRUPTIBLE` is set — the scheduler will not run this task until woken or signalled.

`set_current_state()` uses `smp_store_mb()` — a **full memory barrier**. This ensures the subsequent `vsock_dequeue_accept()` check (inside the waker's `sk_data_ready()`) cannot be reordered before the state change, preventing a lost-wakeup race.

#### `release_sock(listener)` — why release before sleeping?

The sock lock **must** be released before `schedule_timeout()`. If it were held, the transport's RX path (which needs `lock_sock` to call `vsock_enqueue_accept`) would deadlock.

The race window between `release_sock` and `schedule_timeout` is safe because:
- `prepare_to_wait` already set `TASK_INTERRUPTIBLE` and added the task to the wait queue.
- If `sk_data_ready()` fires in the window, it calls `wake_up_interruptible_sync_poll()`, which sets the task back to `TASK_RUNNING`. `schedule_timeout()` then returns immediately without sleeping.

#### `schedule_timeout(timeout)`

Calls `schedule()` — yields the CPU. The task sleeps until:
- Timeout expires → returns 0
- `wake_up()` is called on `sk_sleep(listener)` → returns remaining time

#### `finish_wait()` (wait.c:375)

```c
__set_current_state(TASK_RUNNING);
if (!list_empty_careful(&wq_entry->entry)) {
    spin_lock_irqsave(&wq_head->lock, flags);
    list_del_init(&wq_entry->entry);    // remove from wait queue if not already removed
    spin_unlock_irqrestore(&wq_head->lock, flags);
}
```

`list_empty_careful()` checks both `.next` and `.prev` without a lock — safe because if the entry is empty, both pointers are self-referential and only this CPU touches them.

`autoremove_wake_function` (called by the waker) removes the entry from the queue during the wakeup itself, so `finish_wait` usually finds it already gone.

#### `signal_pending(current)` + `sock_intr_errno()`

```c
// sock.h:2749
static inline int sock_intr_errno(long timeo) {
    return timeo == MAX_SCHEDULE_TIMEOUT ? -ERESTARTSYS : -EINTR;
}
```

- `-ERESTARTSYS` — syscall restarts automatically if the signal handler has `SA_RESTART`.
- `-EINTR` — syscall fails permanently (used when `SO_RCVTIMEO` was set, since restart would change the timeout semantics).

### Step 4 — Error disambiguation (lines 1898–1902)

```c
if (listener->sk_err) {
    err = -listener->sk_err;   // transport error (e.g. VM went away)
} else if (!connected) {
    err = -EAGAIN;             // timeout expired with no connection
}
```

`sk_err` wins over `!connected` — a transport failure on the listener overrides the timeout path.

### Step 5 — Taking ownership of the connected socket (lines 1904–1932)

```c
sk_acceptq_removed(listener);
// WRITE_ONCE(sk->sk_ack_backlog, sk->sk_ack_backlog - 1)
// WRITE_ONCE used because sk_acceptq_is_full() reads it locklessly with READ_ONCE
```

```c
lock_sock_nested(connected, SINGLE_DEPTH_NESTING);
// Both listener and connected are now held simultaneously.
// SINGLE_DEPTH_NESTING tells lockdep this is intentional nested locking
// of the same lock class (AF_VSOCK). Lock order is always: listener first.
```

```c
if (err) {
    vconnected->rejected = true;
    // Don't call sock_graft. vsock_pending_work() will clean up.
} else {
    newsock->state = SS_CONNECTED;
    sock_graft(connected, newsock);
```

#### `sock_graft()` (sock.h:2145) — the critical wiring step

```c
write_lock_bh(&sk->sk_callback_lock);
rcu_assign_pointer(sk->sk_wq, &parent->wq);  // child uses newsock's wait queue
parent->sk = sk;                              // newsock->sk = connected
sk_set_socket(sk, parent);                    // connected->sk_socket = newsock
security_sock_graft(sk, parent);              // LSM hook (SELinux, etc.)
write_unlock_bh(&sk->sk_callback_lock);
```

Before `sock_graft`, `connected` is a "naked" `struct sock` — no associated `struct socket`, no fd. After:

| Pointer | Before | After |
|---|---|---|
| `newsock->sk` | `NULL` | `connected` |
| `connected->sk_socket` | `NULL` | `newsock` |
| `connected->sk_wq` | transport's internal wq | `newsock->wq` |

`sk_callback_lock` (rwlock) is used because `sk_wq` is read from softirq context (e.g. `sock_def_readable` uses `rcu_dereference`). The `write_lock_bh` excludes both other writers and BH.

```c
    set_bit(SOCK_CUSTOM_SOCKOPT, &connected->sk_socket->flags);
```
Tells `setsockopt` to call vsock's own handler for unknown option levels instead of returning `ENOPROTOOPT`. Required for vsock-specific buffer size options.

```c
    if (vsock_msgzerocopy_allow(vconnected->transport))
        set_bit(SOCK_SUPPORT_ZC, &connected->sk_socket->flags);
```
If the transport supports `MSG_ZEROCOPY` sendmsg, advertise that on the new socket.

```c
    release_sock(connected);
    sock_put(connected);   // drop the ref from vsock_enqueue_accept()
```

`vsock_enqueue_accept()` called `sock_hold(connected)`. Now that the VFS holds a reference through `newsock`, we drop the accept-queue reference.

### Wakeup Path: Transport → `accept()`

```
Transport (IRQ/softirq context):
  vsock_enqueue_accept(listener, child)
      list_add_tail(...)              ← child on accept_queue
  listener->sk_data_ready(listener)
      └─ sock_def_readable()          [net/core/sock.c:3647]
             rcu_read_lock()
             wq = rcu_dereference(sk->sk_wq)
             if (skwq_has_sleeper(wq))
                 wake_up_interruptible_sync_poll(&wq->wait, EPOLLIN|EPOLLPRI|...)
                     └─ autoremove_wake_function()
                            default_wake_function()
                                try_to_wake_up(wq_entry->private)  ← the blocked task
                            list_del_init_careful(&wq_entry->entry) ← removes from waitq

Sleeping task resumes in schedule_timeout():
  finish_wait() → TASK_RUNNING (entry already gone via autoremove_wake_function)
  lock_sock(listener)
  loop re-evaluates: vsock_dequeue_accept() → returns child
  loop exits
```

### Reference Count Lifecycle

```
Event                         refcnt  Note
──────────────────────────────────────────────────────────────────────────
__vsock_create()              1       initial alloc
vsock_insert_connected()      2       connected_table holds a ref
vsock_enqueue_accept()        3       accept_queue holds a ref

In accept():
  vsock_dequeue_accept()      3→2*    *drops listener's ref, not child's
  sock_graft()                —       VFS/fd now holds a ref (via inode)
  sock_put(connected)         2→1     accept_queue ref dropped

After accept() returns:
  fd (inode)                  1
  vsock_connected_table       1
  Total                       2
```

### The 3 Critical Design Points

1. **`release_sock` before `schedule_timeout`** — mandatory. Without it, the transport's RX path could never lock the listener to enqueue the child. The lost-wakeup race is closed by `prepare_to_wait` installing the waiter *before* `release_sock`.

2. **`accept_queue` manipulated under `lock_sock`** — both `vsock_enqueue_accept()` (transport side) and `vsock_dequeue_accept()` (accept side) run with the listener's sock lock held. No additional list lock is needed.

3. **`sock_graft` under `sk_callback_lock`** — because `sk_wq` is accessed from softirq via RCU, the pointer swap requires `rcu_assign_pointer` on write and `write_lock_bh` to exclude concurrent writers (`sock_orphan`, etc.).

---

## 9. Namespace Modes

Each network namespace has a **mode** (`af_vsock.c:87–135`):

- **global** (default for `init_net`): CIDs are system-wide; sockets can reach any global-namespace VM.
- **local**: CID pool is private per-namespace; sockets can only communicate within their own namespace.

Controlled via sysctl:
- `/proc/sys/net/vsock/ns_mode` — read-only, set at namespace creation
- `/proc/sys/net/vsock/child_ns_mode` — write-once, controls mode for future child namespaces

---

## 10. Module Initialization (`vsock_init`, af_vsock.c:2989)

```
vsock_init_tables()             ← zero-initialize both hash tables
misc_register(&vsock_device)    ← register /dev/vsock char device
proto_register(&vsock_proto, 1) ← register protocol with SLUB (own slab cache)
sock_register(&vsock_family_ops)← register AF_VSOCK with the socket layer
register_pernet_subsys()        ← per-netns sysctl setup
vsock_bpf_build_proto()         ← BPF sockmap support
```

---

## 11. Key Files Reference

| Path | Purpose |
|---|---|
| `net/vmw_vsock/af_vsock.c` | Address family core: bind, listen, accept, connect, send, recv |
| `include/net/af_vsock.h` | `vsock_sock` and `vsock_transport` struct definitions |
| `net/vmw_vsock/virtio_transport_common.c` | Virtio transport implementation (most readable backend) |
| `net/vmw_vsock/virtio_transport.c` | Virtio PCI device glue |
| `net/vmw_vsock/vsock_loopback.c` | Local loopback transport |
| `net/vmw_vsock/diag.c` | Netlink diag — how `ss(8)` reads vsock state |
| `net/vmw_vsock/vsock_bpf.c` | BPF sockmap integration |
| `net/vmw_vsock/af_vsock_tap.c` | Packet tap for tracing/debugging |
| `uapi/linux/vm_sockets.h` | Userspace API: CID constants, `sockaddr_vm`, ioctls |
